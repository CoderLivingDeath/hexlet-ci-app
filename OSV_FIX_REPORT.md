# Отчёт: починка проверки уязвимостей в CI

- Репозиторий: `CoderLivingDeath/hexlet-ci-app`
- Ветка: `main`
- Коммит с исправлением: `463982b` — `chore(deps): обновить Strapi и закрыть уязвимости в локе`
- Предыдущий коммит: `70b8984`
- Дата работ: 2026-10-31

## 1. Симптом

Job `Check` падал на шаге:

```sh
osv-scanner scan source --lockfile pnpm-lock.yaml
```

`osv-scanner` находил 13 уязвимостей в 9 пакетах (3 High, 10 Medium), из них
12 помечались как исправимые, и завершался с кодом 1.

## 2. Причина

Основная часть уязвимых пакетов — транзитивные зависимости Strapi. В `@strapi/*`
они закреплены **точными версиями** (не диапазонами), поэтому простое обновление
лока их не поднимало:

| Пакет | Версия до | Кто закреплял |
| --- | --- | --- |
| `@modelcontextprotocol/sdk` | 1.30.0 | `@strapi/core`, `@strapi/types` 5.52.2 |
| `glob` | 13.0.6 | `@strapi/core`, `rimraf@6` |
| `minimatch` | 10.2.5 / 10.2.6 | `@strapi/cloud-cli`, `typedoc` |
| `minimatch` | 3.1.5 | `fork-ts-checker-webpack-plugin@8`, `glob@7`, `nodemon@3` |
| `nodemailer` | 9.0.1 | `@strapi/provider-email-sendmail@5.52.2` |
| `qs` | 6.15.3 | `@strapi/admin`, `co-body`, `formidable` |
| `undici` | 6.28.0 | `@strapi/core` |
| `deepmerge` | 4.3.1 | `fork-ts-checker-webpack-plugin@8` |

Дополнительно после первого обновления проявилась `GHSA-rmxm-3fg6-px4f`
в `@hono/node-server@1.19.17` (зависимость `@modelcontextprotocol/node`).

## 3. Что сделано

### 3.1. Обновление Strapi

`package.json`:

```diff
-    "@strapi/plugin-users-permissions": "5.52.2",
-    "@strapi/strapi": "5.52.2",
+    "@strapi/plugin-users-permissions": "5.60.0",
+    "@strapi/strapi": "5.60.0",
```

В 5.60.0 Strapi перешёл с `@modelcontextprotocol/sdk` на
`@modelcontextprotocol/node` + `@modelcontextprotocol/server`, что убрало
уязвимый `sdk` из дерева.

### 3.2. Overrides для закреплённых версий

Добавлены в `pnpm-workspace.yaml`:

```yaml
overrides:
  '@modelcontextprotocol/node': 2.1.2
  '@modelcontextprotocol/node>@hono/node-server': 2.1.3
  '@strapi/cloud-cli>minimatch': 11.0.1
  glob@13: 14.0.2
  nodemailer: 10.3.1
  qs@6.16.0: 6.16.1
  typedoc>minimatch: 11.0.1
  undici@6: 6.29.1
```

Важно: в pnpm 12 поле `pnpm.overrides` в `package.json` **не читается**
(pnpm выдаёт предупреждение и игнорирует его). Актуальное место настроек —
`pnpm-workspace.yaml`, где уже находился `allowBuilds`.

### 3.3. Пересборка лока

```sh
pnpm install
```

Пересобран `pnpm-lock.yaml`, проверено, что `pnpm install --frozen-lockfile`
проходит.

### 3.4. Исключения для непоправимых находок

В `osv-scanner.toml` добавлены две записи со сроком на месяц (до 2026-11-30):

| ID | Пакет | Почему исключение |
| --- | --- | --- |
| `GHSA-gcqx-w48m-w2h4` | `deepmerge 4.3.1` | Исправленной версии пакета ещё нет |
| `GHSA-g3hv-75wg-m249` | `minimatch 3.1.5` | Версия закреплена в `fork-ts-checker-webpack-plugin` и `glob@7`; совместимого исправления нет — `11.x` ломает API |

## 4. Результат по пакетам

| Пакет | До | После |
| --- | --- | --- |
| `@modelcontextprotocol/sdk` | 1.30.0 | удалён из дерева |
| `@modelcontextprotocol/node` | 2.1.1 | 2.1.2 |
| `@hono/node-server` | 1.19.17 | 2.1.3 |
| `glob` | 13.0.6 | 14.0.2 |
| `minimatch` | 10.2.5 / 10.2.6 | 11.0.1 |
| `minimatch` | 3.1.5 | 3.1.5 (исключение) |
| `nodemailer` | 9.0.1 | 10.3.1 |
| `qs` | 6.15.3 | 6.16.1 |
| `undici` | 6.28.0 | 6.29.1 (+ 8.12.3) |
| `deepmerge` | 4.3.1 | 4.3.1 (исключение) |

## 5. Проверки

| Проверка | Результат |
| --- | --- |
| `pnpm install --frozen-lockfile` | OK, лок актуален |
| `osv-scanner scan source --lockfile pnpm-lock.yaml` | `No issues found` |
| `pnpm test` | 1 файл, 1 тест — passed |
| `pnpm run lint` | без ошибок |

Примечание: локально на Windows `format:check` может ругаться на 14 файлов
из-за CRLF (`core.autocrlf=true` при `end_of_line=lf` в `.editorconfig`).
Это не связано с исправлением; на Linux/CI с LF проверка проходит.

## 6. Изменённые файлы

```text
package.json          — версии Strapi
pnpm-workspace.yaml   — overrides
pnpm-lock.yaml        — пересобранный лок
osv-scanner.toml      — два временных исключения
```

## 7. Как повторить проверку

```sh
pnpm install --frozen-lockfile
osv-scanner scan source --lockfile pnpm-lock.yaml
pnpm test
pnpm run lint
```

## 8. Риски и что дальше

- Обновление Strapi 5.52.2 → 5.60.0 — минорное в пределах 5.x, тесты и линтер
  проходят.
- Overrides на мажорные версии (`glob 13 → 14`, `undici`, `nodemailer`) выбраны
  вынужденно, потому что Strapi закрепляет точные версии. Их стоит пересмотреть
  при следующем обновлении Strapi: возможно, upstream поднимет зависимости сам
  и overrides можно будет убрать.
- Исключения `osv-scanner.toml` действуют до 2026-11-30. После этой даты
  проверка снова начнёт падать, если `deepmerge` или `minimatch 3.1.5` не
  получат совместимого исправления.
