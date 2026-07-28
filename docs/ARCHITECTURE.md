# Как устроен neko-achievements

`neko-achievements` — отдельный Elixir/Erlang-сервис, который считает ачивки
(достижения) пользователей Shikimori по их аниме-списку. Сам Shikimori (Ruby on
Rails) не хранит логику подсчёта ачивок — он лишь уведомляет `neko` об
изменениях в списке пользователя и получает в ответ diff (что добавилось,
пропало или обновилось).

Ачивки в этом сервисе называются `neko_id` + `level` (например
`animelist` level 3, `mecha` level 1) и описаны декларативно в YAML-файлах
(`priv/rules/*.yml`), а не в коде.

## Взаимодействие с внешним миром

```
Shikimori (Rails)  --HTTP POST /user_rate-->  neko (этот сервис)
Shikimori (Rails)  <--HTTP GET /api/v2/user_rates, /api/achievements, /api/animes/neko--  neko
```

neko выступает в двух ролях:

1. **HTTP-сервер** (Cowboy + Plug, `Neko.Router`) — принимает уведомления от
   Shikimori об изменении записи в списке аниме пользователя.
2. **HTTP-клиент к Shikimori API** (`Neko.Shikimori.HTTPClient`, на базе
   HTTPoison/hackney) — подтягивает данные, которых уведомление не содержит:
   полный список оценок пользователя, список аниме и справочники.

### Входящий запрос: `POST /user_rate`

Роут описан в `lib/neko/router.ex:35`. Пайплайн плагов:

1. `Plug.Logger` — логирование.
2. `Neko.Plug.Authenticate` (`lib/neko/plug/authenticate.ex`) — сверяет
   заголовок `Authorization` со статическим токеном (`@token` в
   `Neko.Router`, сейчас placeholder `"foo"` — в проде задаётся отдельно).
   Несовпадение → `401`.
3. `Plug.Parsers` — парсит JSON-тело в `conn.body_params`.
4. Тело оборачивается в структуру `Neko.Request` (`lib/neko/request.ex`):

   ```elixir
   %Neko.Request{id, user_id, target_id, score, action, status, episodes}
   ```

   Поле `action` определяет тип события от Shikimori:
   - `"put"` — запись создана/обновлена (добавление тайтла в список или
     смена статуса/оценки/эпизодов);
   - `"delete"` — запись удалена;
   - `"reset"` — принудительная полная перезагрузка данных пользователя
     (без изменения конкретной записи);
   - `"noop"` — просто триггер пересчёта без изменений.

5. Сервер отвечает `201` с JSON-diff’ом ачивок:
   `%{added: [...], removed: [...], updated: [...]}`.

### Исходящие запросы к Shikimori API

`Neko.Shikimori.HTTPClient` (`lib/neko/shikimori/http_client.ex`) ходит на
`https://shikimori.rip/api/` (см. `config/config.exs`) через пул соединений
hackney (`shikimori_pool`, до 150 коннектов, TTL 30 мин):

| Метод                | Эндпоинт             | Когда вызывается                                   |
|-----------------------|-----------------------|-----------------------------------------------------|
| `get_user_rates!/1`   | `GET v2/user_rates`   | Первая загрузка пользователя или `action: "reset"` — тянет ВСЕ записи со статусами `completed,rewatching,watching,on_hold` |
| `get_achievements!/1` | `GET achievements`    | То же самое — подтягивает уже выданные ранее ачивки |
| `get_animes!/0`       | `GET animes/neko`     | Загрузка/перезагрузка глобального справочника аниме (жанры, длительность, франшиза и т.д.), не привязана к пользователю |

Таймауты настроены каскадом (см. комментарий в `config/config.exs:40-46`):
connect timeout 20s → recv timeout 90s → total 110s на HTTP-запрос, поверх
которого ещё есть таймаут очереди запроса на пользователя (120s) и таймаут
воркера пула правил (10s).

## Модель процессов (Erlang/OTP)

Каждый пользователь обслуживается собственным изолированным набором
процессов, которые создаются лениво и умирают по неактивности:

```
Neko.UserHandler.DynamicSupervisor
  └── Neko.UserHandler (GenServer, per user_id, via Registry)
        — сериализует все запросы одного пользователя (backpressure)
        — 4 часа recv_timeout => процесс "засыпает" и удаляется

Neko.UserRate.Store.DynamicSupervisor
  └── Neko.UserRate.Store (Agent, per user_id) — кэш записей списка аниме

Neko.Achievement.Store.DynamicSupervisor
  └── Neko.Achievement.Store (Agent, per user_id) — кэш уже выданных ачивок
```

Оба стора регистрируются через `Registry` (`Neko.UserRate.Store.Registry`,
`Neko.Achievement.Store.Registry`) по `user_id`, создаются "лениво" через
`create_missing_handler` при первом запросе и останавливаются вместе, когда
`UserHandler` получает таймаут (`lib/neko/user_handler/user_handler.ex:62`).

Глобальные (не per-user) данные — справочник аниме и правила ачивок —
хранятся в singleton `Agent`-хранилищах (`Neko.Anime.Store`,
`Neko.Rule.CountRule.Store`, `Neko.Rule.DurationRule.Store`), загружаемых при
старте приложения (`lib/neko/application.ex`).

Расчёт правил распараллелен через пул воркеров `poolboy`
(`rule_worker_pool`, 30 воркеров `Neko.Rule.Worker`) — каждый воркер держит
у себя копию правил и справочника аниме, чтобы не гонять их по памяти между
процессами при каждом запросе.

## Пошаговый жизненный цикл запроса

`Neko.Request.process/1` (`lib/neko/request.ex:25`) — центральная точка:

```
1. load_user_data/1
   - action == "reset"  → параллельно Neko.UserRate.reload + Neko.Achievement.reload
                           (полная перезагрузка с Shikimori API)
   - иначе              → Neko.UserRate.load + Neko.Achievement.load
                           (загрузка с API только если для user_id ещё нет
                            стора в реестре; иначеno-op, берём закэшированное)

2. process_action/1  — применяет изменение к локальному кэшу UserRate:
   - action=="put", status in [completed, rewatching, watching, on_hold]
       → Neko.UserRate.put (upsert записи в Agent)
   - action=="put" с любым другим статусом (напр. "planned", "dropped")
       → Neko.UserRate.delete (запись больше не считается для ачивок)
   - action=="delete" → Neko.UserRate.delete
   - action in ["noop","reset"] → ничего (кэш уже актуален)

3. calc_new_achievements/1 = Neko.Achievement.Calculator.call(user_id)
   — пересчитывает вообще ВСЕ ачивки пользователя с нуля по текущему кэшу

4. calc_diff/2 = Neko.Achievement.Diff.call(старые, новые)
   — сравнивает с тем, что было в Achievement.Store, отдаёт
     %{added, removed, updated}

5. save_new_achievements/2 — перезаписывает Achievement.Store новым
   полным набором ачивок

6. diff возвращается в HTTP-ответ (201)
```

Важно: сервис не хранит персистентно ничего, кроме собственного
in-memory состояния процессов. При падении/рестарте всё грузится заново
с Shikimori API. `action: "reset"` — способ Shikimori форсировать полную
пересинхронизацию (например, если локальный кэш разошёлся с реальностью).

## Как именно вычисляется, что ачивка выдана

### 1. Правила описаны в YAML, а не в коде

`priv/rules/*.yml` — 38 файлов, каждый описывает одну "линейку" ачивок
(`neko_id`) с несколькими уровнями (`level`). Пример
(`priv/rules/mecha.yml`):

```yaml
- &defaults
  neko_id: mecha
  level: 1
  algo: count
  threshold: 35
  filters:
    genre_v2_ids: [18]
  metadata: {...}   # тексты, картинка, цвет рамки — не участвует в расчёте

- <<: *defaults
  level: 2
  threshold: 70
```

Есть два алгоритма (`algo`):

- **`count`** (`Neko.Rule.CountRule`) — считает количество тайтлов из
  подходящего под фильтр набора, которые пользователь **досмотрел**
  (см. ниже, какие статусы считаются).
- **`duration`** (`Neko.Rule.DurationRule`) — считает суммарную
  длительность просмотра (в минутах) тайтлов из набора.

`threshold` может быть числом (абсолютный порог) либо строкой вида
`"100%"` — тогда порог считается как процент от максимально возможного
значения для этого правила:
  - для `count` — процент от количества аниме, попавших под фильтр
    (`Neko.Rule.CountRule.threshold/1`);
  - для `duration` — процент от суммарной длительности всех аниме,
    попавших под фильтр (`Neko.Rule.DurationRule.threshold/1`).

### 2. Какие аниме попадают в правило — `filters`

`Neko.Rule.Filters.filter_animes/2` (`lib/neko/rule/filters.ex`) применяет к
глобальному справочнику аниме (`Neko.Anime.Store`, из `GET animes/neko`)
цепочку фильтров, заданных в YAML: `genre_ids`, `genre_v2_ids`,
`anime_ids`, `not_anime_ids`, `year_lte`, `episodes_gte`, `duration_lte`,
`franchise`, а также `or` для объединения нескольких наборов условий.
Результат — множество `anime_ids`, зафиксированное в правиле при загрузке
(пересчитывается при `Neko.Anime.reload/0` или изменении самих правил).

### 3. Какие записи пользователя учитываются

`Neko.Rule.achievements/4` (`lib/neko/rule/rule.ex:39`) для каждого запроса
строит из кэша `Neko.UserRate.all(user_id)`:

- `by_anime_id` — `%{anime_id => %{user_rate, anime}}` для всех тайтлов
  пользователя;
- `user_anime_ids` — подмножество, где статус **не** `"watching"` и **не**
  `"on_hold"` (то есть `completed` или `rewatching`) — это то, что
  используется алгоритмом `count`.

Соответственно:
- `count` учитывает только **полностью просмотренные** (`completed`/
  `rewatching`) тайтлы;
- `duration` учитывает все статусы `completed, rewatching, watching,
  on_hold` — для `watching`/`on_hold` берётся частичная длительность
  `anime.duration * user_rate.episodes` (сколько уже просмотрено), для
  остальных — `anime.total_duration` (полная длительность,
  `episodes * duration`, см. `Neko.Anime.Calculations.calc_total_durations/1`).

### 4. Сравнение с порогом и прогресс

Для каждого правила вычисляется `value` (см. выше), и:

```elixir
rule_applies? = value >= rule.threshold
```

(`lib/neko/rule/rule.ex:71`). Если условие выполнено — формируется
`%Neko.Achievement{user_id, neko_id, level, progress}`.

`progress` (`Neko.Rule.Progress`, 0–100) — вспомогательное значение для
UI (насколько пользователь продвинулся к **следующему** уровню этой же
ачивки):
- если следующего уровня нет — `100`;
- если `value == threshold` текущего уровня — `0`;
- если `value >= next_threshold` — `100` (уже пора считать следующий
  уровень, дальше сработает отдельная ачивка);
- иначе линейная интерполяция между `threshold` и `next_threshold`.

### 5. Итог — полный пересчёт, а не инкремент

`Neko.Achievement.Calculator.call/1` (`lib/neko/achievement/calculator.ex`)
на **каждый** запрос прогоняет пользователя через **все** правила из
**обоих** алгоритмов (`Neko.Rule.CountRule`, `Neko.Rule.DurationRule`),
используя пул воркеров `poolboy`, и складывает результат в `MapSet` —
это и есть полный актуальный набор ачивок пользователя "с нуля",
без попытки посчитать дельту на уровне самого алгоритма. Дельта же
(`added/removed/updated`) считается отдельно, сравнением двух полных
множеств (`Neko.Achievement.Diff`), просто чтобы Shikimori не пришлось
самому вычислять разницу и понять, что показать пользователю
(всплывающее уведомление о новой ачивке и т.п.).

## Резюме: сквозной пример

1. Пользователь на Shikimori отмечает аниме как `completed`.
2. Shikimori шлёт `POST /user_rate` с `action=put, status=completed,
   target_id=..., episodes=...`.
3. neko находит/создаёт `UserHandler` этого пользователя, подгружает (если
   не закэшировано) его список оценок и текущие ачивки с Shikimori API.
4. Обновляет запись в локальном кэше `UserRate.Store`.
5. Прогоняет пользователя через все YAML-правила (`count`/`duration`) во
   всех 30 воркерах пула, получает полный набор "заслуженных" ачивок.
6. Сравнивает с тем, что было раньше → diff.
7. Сохраняет новый набор как текущий, возвращает diff Shikimori (`201`),
   который решает, что показать пользователю.
