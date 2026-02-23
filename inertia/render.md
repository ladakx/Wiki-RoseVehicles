# 👽 Render (визуализация) — `render.yml`

`render.yml` описывает, **как физическое тело Inertia выглядит для игроков**: какие визуальные сущности создать (Display / ArmorStand / и т.д.), как их трансформировать и когда показывать.

## Где используется

- `bodies.yml` связывает физическую “префаб” с моделью рендера:
  - для блоков/цепей: `render.model: <id из render.yml>`
  - для ragdoll: у каждой части свой `render-model: <id из render.yml>`
- `items.yml` хранит ItemStack’и, на которые ссылается `item-model`.
- После правок обычно достаточно `/inertia reload`.

## Базовая структура

Каждый верхнеуровневый ключ — это **render-модель** (её `id`), внутри — флаги синхронизации и набор “частей” (`entities`):

```yml
<model-id>:
  sync:
    position: true
    rotation: true
  entities:
    <entity-key>:
      type: ITEM_DISPLAY
      item-model: items.some_item
      local-offset: "0 0 0"
      local-rotation: "0 0 0"
```

### `sync`

Модельные флаги синхронизации применяются ко всем `entities` внутри модели:

- `sync.position` (по умолчанию `true`) — двигать часть вместе с физическим телом.
- `sync.rotation` (по умолчанию `true`) — вращать часть вместе с физическим телом.

Практика:
- Для “декора”, который должен “ехать” с телом — `position: true`.
- Для эффектов, которые должны оставаться с фиксированной ориентацией — `rotation: false`.

## `entities`: части модели

`entities` — это map `<entity-key> -> definition`. `entity-key` — произвольный идентификатор части (например `main`, `rim`, `hitbox`).

### Обязательные поля

- `type` — вид визуальной сущности:
  - `BLOCK_DISPLAY`
  - `ITEM_DISPLAY`
  - `ARMOR_STAND` (часто как fallback для старых клиентов)
  - `BOAT`, `SHULKER`, `INTERACTION` (доп. “network-side” виды)

### Координаты и вращения

- `local-offset: "x y z"` — смещение части относительно центра физического тела.
  - Если `sync.position: true`, смещение **поворотится** вместе с телом.
- `local-rotation: "x y z"` — локальное вращение части.
  - Если `sync.rotation: true`, итоговое вращение считается как `bodyRotation * localRotation`.
  - Важно: значения трактуются как углы для `rotationXYZ` (обычно **радианы**, не градусы).

Формат векторов в конфиге: строка из 3 чисел, разделённых пробелами или запятыми (например `"1 0.5 -2"` или `"1,0.5,-2"`).

## Display-сущности (`BLOCK_DISPLAY`, `ITEM_DISPLAY`)

### `BLOCK_DISPLAY`

Минимальный набор:

```yml
some_block:
  entities:
    block:
      type: BLOCK_DISPLAY
      block: STONE
      scale: "1 1 1"
```

Поля:

- `block: <Material>` — тип блока (например `STONE`, `OAK_LOG`).
- `scale: "x y z"` — масштаб (по умолчанию `"1 1 1"`).
- `translation: "x y z"` — сдвиг внутри `Transformation` (по умолчанию `"0 0 0"`).
- `rotate-translation: true|false` (по умолчанию `true`) — как применять `translation`:
  - `true`: трансляция поворачивается вместе с телом (удобно для составных форм).
  - `false`: трансляция применяется “после” вращения.

### `ITEM_DISPLAY`

Минимальный набор:

```yml
some_item:
  entities:
    item:
      type: ITEM_DISPLAY
      item-model: items.iron_chain
      display-mode: NONE
```

Поля:

- `item-model: <ключ>` — ссылка на `items.yml`.
  - Допускаются префиксы `items.` / `item.` или просто `<id>`.
  - Если ключ не найден, есть fallback (Material или BARRIER).
- `display-mode` — transform для ItemDisplay:
  - `NONE`, `THIRDPERSON_LEFTHAND`, `THIRDPERSON_RIGHTHAND`,
    `FIRSTPERSON_LEFTHAND`, `FIRSTPERSON_RIGHTHAND`, `HEAD`,
    `GUI`, `GROUND`, `FIXED`.

#### Параметры `@skin=...` в `item-model`

`item-model` может содержать параметры после `@`, например:

```yml
item-model: items.ragdoll_head@skin=alex
```

Это используется для динамической подстановки скина (актуально для предметов-голов). Если `skin` — ник онлайн-игрока, будет взят его скин; иначе можно передать текстуру/URL (как в `items.yml`).

### Доп. параметры Display (необязательно)

Для `BLOCK_DISPLAY` / `ITEM_DISPLAY` поддерживаются (по версиям Bukkit/MC; если метода нет — будет проигнорировано):

- `view-range: <float>`
- `shadow-radius: <float>`
- `shadow-strength: <float>`
- `interpolation-duration: <int>`
- `teleport-duration: <int>`
- `billboard: FIXED|VERTICAL|HORIZONTAL|CENTER`
- `brightness.block: <int>`, `brightness.sky: <int>`

## ArmorStand (`ARMOR_STAND`)

Используется как простой “legacy” визуал (например в `variants` для старых клиентов).

Поля:

- `item-model: ...` — предмет/модель, обычно идёт в шлем.
- `small: true|false` (по умолчанию `false`)
- `invisible: true|false` (по умолчанию `true`)
- `marker: true|false` (по умолчанию `true`)
- `base-plate: true|false` (по умолчанию `false`)
- `arms: true|false` (по умолчанию `false`)

## Видимость: active/sleeping + LOD + теги

### Простая настройка

По умолчанию часть показывается всегда:

- `show-when.active` (по умолчанию `true`)
- `show-when.sleeping` (по умолчанию `true`)

Можно скрывать часть принудительно:

- `hide-when.active` (по умолчанию `false`)
- `hide-when.sleeping` (по умолчанию `false`)

### LOD-маски

Можно ограничивать видимость по уровню детализации (LOD). Поддерживаемые значения: `NEAR`, `MID`, `FAR` (допускаются также `LOD_NEAR`, `LOD_MID`, `LOD_FAR`).

- `show-when.lod: [NEAR, MID]` (или строкой)
- `hide-when.lod: [FAR]` (или строкой)

### `hide-tags` и `show-tags`

Упрощённая запись “тегами”:

- `hide-tags: [ACTIVE, SLEEPING, LOD_FAR, ...]`
- `show-tags: [ACTIVE, LOD_NEAR, ...]` — whitelist; если указать `ACTIVE/SLEEPING`, это **перекроет** `show-when.active/sleeping`.

## Per-client variants (`variants`)

Одна render-модель может иметь разные определения для разных **версий клиента** (например, для 1.16 показывать ArmorStand, а для 1.19.4+ — Display).

Формат ключа диапазона:
- `"1.16.5-1.19.3"`
- `"1.19.4-latest"`
- `"1.20.1"` (одна версия)

Пример:

```yml
crate:
  variants:
    "1.16.5-1.19.3":
      sync: { position: true, rotation: true }
      entities:
        legacy:
          type: ARMOR_STAND
          item-model: items.crate
          invisible: true
          marker: true
    "1.19.4-latest":
      sync: { position: true, rotation: true }
      entities:
        modern:
          type: BLOCK_DISPLAY
          block: BARREL
          scale: "1 1 1"
```

## `settings:` (общие и специфичные)

`settings:` — произвольный YAML-раздел, который валидируется при загрузке (с предупреждениями) и используется как набор флагов/параметров на уровне сущности.

Общие ключи (примерный список):

- `silent: true|false`
- `gravity: true|false`
- `no-gravity: true|false`
- `invulnerable: true|false`
- `glowing: true|false`
- `collidable: true|false`
- `persistent: true|false`
- `custom-name: "<text>"`
- `custom-name-visible: true|false`

Специфичные секции (и “короткие” legacy-ключи тоже поддерживаются):

- `boat.type: OAK|SPRUCE|...` и `boat.chest: true|false`
- `shulker.color: PURPLE|...` / `shulker.color-id: -1..15`, `shulker.peek: 0..100`, `shulker.ai: true|false`
- `interaction.width: >0`, `interaction.height: >0`, `interaction.responsive: true|false`

Пример `INTERACTION` hitbox:

```yml
interaction_hitbox:
  sync: { position: true, rotation: true }
  entities:
    hitbox:
      type: INTERACTION
      invisible: true
      settings:
        no-gravity: true
        silent: true
        interaction:
          width: 1.0
          height: 1.0
          responsive: true
```

## Быстрый чек-лист при отладке

- В `bodies.yml` указан корректный `render.model` / `render-model` (для ragdoll частей).
- В `render.yml` у модели есть `entities`.
- Для `ITEM_DISPLAY`/`ARMOR_STAND` корректный `item-model` и он существует в `items.yml` (или это валидный Material).
- Для `BLOCK_DISPLAY` указан `block`.
- После правок выполнен `/inertia reload`.


