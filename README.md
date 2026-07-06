# Pipeline — инструкция по `instruction.txt`

Сервис **Pipeline** принимает ZIP-архив и полностью готовит iOS-приложение в App Store
Connect: создаёт bundle и приложение, заполняет метаданные, скриншоты, возрастной
рейтинг, доступность, цену, покупки/подписки и (по желанию) отправляет на модерацию.

Всё описывается **одним файлом внутри ZIP** — `instruction.json` (рекомендуется) или
`instruction.txt`.

---

## Содержание

1. [Как работает: главный принцип](#главный-принцип)
2. [Структура ZIP-архива](#структура-zip-архива)
3. [Как запускается процесс (сценарии аккаунта)](#сценарии-аккаунта)
4. [Формат файла](#формат-файла)
5. [Ключи по разделам](#ключи-по-разделам)
   - [Аккаунт и доступ](#1-аккаунт-и-доступ)
   - [Bundle / App](#2-bundle--app)
   - [Общие ссылки](#3-общие-ссылки)
   - [App Information (метаданные)](#4-app-information)
   - [Категории](#5-категории)
   - [Content Rights](#6-content-rights)
   - [Distribution Page (тексты + скриншоты)](#7-distribution-page)
   - [App Review Information](#8-app-review-information)
   - [Copyright](#9-copyright)
   - [Age Rating](#10-age-rating)
   - [Data Collection (App Privacy)](#11-data-collection)
   - [Pricing & Availability](#12-pricing--availability)
   - [Build (выбор билда + encryption)](#13-build)
   - [Release (отправка на модерацию)](#14-release)
   - [Монетизация: IAP и подписки](#15-монетизация)
6. [Порядок выполнения операций](#порядок-выполнения)
7. [Справочники констант](#справочники-констант)
   - [Коды локалей](#коды-локалей)
   - [Коды территорий](#коды-территорий)
   - [Категории](#категории-appcategories)
   - [Длительности подписок](#длительности-подписок)
   - [Платформы](#платформы)
8. [Полный пример `instruction.txt`](#полный-пример)

---

## Главный принцип

> **Наличие ключа = операция выполняется. Отсутствие ключа = операция пропускается.**

Отдельных флагов `update_*` нет. Если вы не хотите трогать какой-то раздел App Store
Connect — просто **не указывайте** соответствующий ключ.

Булевы значения: `yes / no` (также принимаются `true / false`, `1 / 0`, `on / off`).

---

## Структура ZIP-архива

```
archive.zip
├── instruction.json             ← инструкция (json ИЛИ txt, в любом месте архива)
├── screenshots_en_us/           ← папки со скриншотами приложения (имя = как в ключе)
│   ├── 6.9_1.png
│   └── 6.9_2.png
├── screenshots_en_gb/
│   └── ...
├── iap_shot_1/                  ← папки со скринами ревью IAP/подписок
│   └── review.png
└── AuthKey_ABCDE12345.p8        ← опционально: .p8 ключ (если задан api_key_path)
```

- `instruction.json` / `instruction.txt` ищется на любой глубине архива (json в приоритете).
- Папки скриншотов сопоставляются **по имени** (регистр не важен, ищется рекурсивно).
- Для скринов ревью IAP/подписок берётся **первый файл** из папки.

---

## Сценарии аккаунта

Порядок разрешения аккаунта: **БД → ключи из инструкции → web-логин**.

### Сценарий A — аккаунт уже есть в БД

Достаточно указать имя аккаунта (плюс креды на случай, если понадобится веб-сессия
для создания приложения / privacy / content rights):

```
account_name: dev@example.com
account_email: dev@example.com
account_password: MyP@ssw0rd
account_phone_number: 12184033620
```

Сервис найдёт аккаунт по `account_name` и возьмёт из БД: **proxy, issuer_id, key_id,
путь к .p8**. Прокси указывать не нужно — он уже в БД.

### Сценарий B — аккаунта нет в БД (создастся автоматически)

Указываем креды **и прокси**. Сервис сам залогинится (2FA), создаст API-ключ, скачает
`.p8`, получит issuer_id/key_id и **сохранит аккаунт в БД** для будущих запусков.

```
account_name: dev@example.com
account_email: dev@example.com
account_password: MyP@ssw0rd
account_phone_number: 12184033620
proxy_url: http://user:pass@1.2.3.4:8080
```

### Сценарий C — свой `.p8` ключ в архиве

Кладём `.p8` в ZIP и указываем issuer_id + путь. Web-логин и 2FA не потребуются
(если не понадобятся чисто web-операции вроде создания приложения):

```
account_name: dev@example.com
issuer_id: 69a6de70-xxxx-xxxx-xxxx-xxxxxxxxxxxx
api_key_path: AuthKey_ABCDE12345.p8
proxy_url: http://user:pass@1.2.3.4:8080
```

> **Про 2FA и сессию.** При web-логине код 2FA запрашивается через модалку. После
> первого успешного логина cookie-сессия **сохраняется в БД** и переиспользуется —
> повторные запуски на том же аккаунте **не спрашивают 2FA**, пока сессия жива. Когда
> Apple её протухнет — код запросится снова.

---

## Формат файла

Инструкцию можно положить в архив в одном из двух форматов (имена в приоритете
именно в таком порядке):

- **`instruction.json`** — JSON с нативными типами (рекомендуется), либо
- **`instruction.txt`** — текстовый `ключ: значение`.

Набор ключей и правила (наличие ключа = поле задано) одинаковые. Если в архиве
есть оба файла — используется `instruction.json`.

### Вариант JSON (`instruction.json`)

Ключи те же, но значения — нативные JSON-типы: строки строками, `true/false`
булевыми, списки массивами, `app_info`/`distribution_page` — массивы объектов,
`consumable_iap_products`/… — объекты, матрицы локализаций — массивы массивов.

```json
{
  "account_name": "dev@example.com",
  "create_bundle": true,
  "create_app": true,
  "bundle_id": "com.company.app",
  "app_name": "My Cool App",
  "platform": "iOS",
  "uses_third_party_content": false,
  "primary_category": "GAMES",
  "app_availability_excepts": [],
  "app_pre_order_release_date": "01.12.2026",
  "app_info": [
    { "locale": "en-US", "app_name": "My Cool App", "subtitle": "Best app" },
    { "locale": "en-GB", "app_name": "My Cool App UK" }
  ],
  "distribution_page": [
    { "locale": "en-US", "description_text": "Line 1\nLine 2", "keywords": "game,fun",
      "screenshots_folder_name": "screenshots_en_us", "replace_screenshots": true }
  ],
  "sub_products": {
    "group_name": "Premium",
    "sub_product_ref_names": ["Monthly", "Yearly"],
    "sub_product_prices": ["9.99", "99.99"],
    "sub_product_durations": ["ONE_MONTH", "ONE_YEAR"],
    "sub_three_days_trial": [0],
    "sub_product_localization_locales": ["en-US"],
    "sub_product_localization_names": [["Monthly Premium", "Yearly Premium"]]
  }
}
```

Заметки по JSON:
- Многострочный текст (`description_text`, `app_review_info_notes`) — обычная строка
  с `\n`.
- `app_availability_excepts`: `[]` = все страны, отсутствие ключа / `null` = не трогать.
- Даты — либо `"DD.MM.YYYY"`, либо ISO `"YYYY-MM-DD"`.
- Цены можно числами (`9.99`) или строками (`"9.99"`).

### Вариант TXT (`instruction.txt`)

- Одна пара `ключ: значение` на строку.
- **Списки объектов** (`app_info`, `distribution_page`) — в квадратных скобках с
  фигурными блоками:
  ```
  app_info: [
  {
  locale: en-US
  app_name: My App
  },
  {
  locale: en-GB
  app_name: My App UK
  }
  ]
  ```
- **Многострочные значения** (например `description_text`) — продолжаются до
  следующего распознанного ключа; пустые строки внутри сохраняются.
- **Простые списки** — `[ a, b, c ]`.
- **Объекты покупок** — `{ ключ: [список] ... }`.
- **Даты** — `DD.MM.YYYY` (например `01.12.2026`).

---

## Ключи по разделам

### 1. Аккаунт и доступ

| Ключ | Тип | Описание |
|---|---|---|
| `account_name` | str | Имя аккаунта (обычно email). Ключ поиска в БД. |
| `account_email` | str | Apple ID для web-логина. |
| `account_password` | str | Пароль Apple ID. |
| `account_phone_number` | str | Номер для 2FA (SMS). |
| `proxy_url` | str | Прокси (обязателен для нового аккаунта / если нет в БД). |
| `issuer_id` | str | Issuer ID (сценарий C). |
| `api_key_path` | str | Имя `.p8`-файла в архиве, формат `AuthKey_<KEY_ID>.p8`. |

```
account_name: dev@example.com
account_email: dev@example.com
account_password: MyP@ssw0rd
account_phone_number: 12184033620
proxy_url: http://user:pass@1.2.3.4:8080
```

### 2. Bundle / App

| Ключ | Тип | Описание |
|---|---|---|
| `create_bundle` | bool | `yes` → создать bundle id. |
| `create_app` | bool | `yes` → создать приложение (иначе резолвится существующее). |
| `bundle_id` | str | Bundle identifier, например `com.company.app`. |
| `app_name` | str | Имя приложения. |
| `sku` | str | SKU (обычно = bundle_id). |
| `platform` | str | Платформа: `iOS` / `macOS` / `tvOS`. |
| `app_card_id` | str | Короткий ид для имени API-ключа: `{app_name}{app_card_id}ApiKey`. |

> **Company Name** при создании приложения (обязателен для аккаунтов-организаций)
> задавать не нужно — если Apple его требует, имя компании берётся автоматически
> из API (`provider.name` web-сессии).
| `create_new_version` | bool | `yes` → создать новую версию App Store (1.0). |
| `show_completion_log` | bool | `yes` → в конце вывести копируемый блок с созданными данными. |

```
create_bundle: yes
create_app: yes
bundle_id: com.company.app
app_name: My Cool App
sku: com.company.app
platform: iOS
app_card_id: wa409
create_new_version: yes
show_completion_log: yes
```

### 3. Общие ссылки

Одни на всё приложение (применяются ко всем локалям, где нужно).

| Ключ | Тип | Описание |
|---|---|---|
| `privacy_policy_url` | str | URL политики конфиденциальности. |
| `support_url` | str | URL поддержки. |
| `marketing_url` | str | Маркетинговый URL. |

```
privacy_policy_url: https://example.com/privacy
support_url: https://example.com/support
marketing_url: https://example.com
```

### 4. App Information

Список блоков по локалям. Внутри блока — ключи `locale`, `app_name`, `subtitle`.

```
app_info: [
{
locale: en-US
app_name: My Cool App
subtitle: The best app ever
},
{
locale: en-GB
app_name: My Cool App UK
}
]
```

> `locale` можно указывать кодом (`en-US`) или ассоциативным именем (`EnglishUS`,
> `English (U.S.)`) — сервис приведёт к коду через ассоциации локалей.

> **Занятое имя.** Если имя приложения для локали уже занято, сервис
> автоматически добавляет к нему невидимый символ (zero-width space) и повторяет
> попытку — и так, пока длина не достигнет 30 символов (лимит App Store). Если
> места не осталось — локаль помечается ошибкой, остальные продолжают.

#### Имена из `.docx` (альтернатива `app_info`)

| Ключ | Тип | Описание |
|---|---|---|
| `docx_format_app_names_file` | str | Имя `.docx`-файла в архиве со стандартизированными именами по локалям. |

Если ключ задан, **`app_info` игнорируется** — имена берутся из docx. Формат
каждой строки файла:
```
<номер>. <Локаль>: <Имя приложения>
```
например:
```
1. German: Chicken Road 2:Sockelkunst
6. English (U.K.): Ghuw:PlinthArt
```
Локаль/имя разделяются **первым `": "`** (двоеточие + пробел), поэтому имя может
содержать двоеточие. Имя локали — ассоциативное (`German`, `English (U.K.)`, …),
резолвится в код ASC через ассоциации локалей.

```
docx_format_app_names_file: app_names.docx
```

### 5. Категории

| Ключ | Тип | Описание |
|---|---|---|
| `primary_category` | str | Основная категория (ID из appCategories). |
| `secondary_category` | str | Вторичная категория (ID). |

```
primary_category: GAMES
secondary_category: ENTERTAINMENT
```

> Значения — **ID категорий** (см. [справочник](#категории-appcategories)), не названия.

### 6. Content Rights

Ответ на вопрос «Does your app contain, show, or access third-party content?».

| Ключ | Тип | Описание |
|---|---|---|
| `uses_third_party_content` | bool | `no` → «No» (по умолчанию); `yes` → «Yes». |

```
uses_third_party_content: no
```

### 7. Distribution Page

Список блоков по локалям — тексты страницы версии и скриншоты.

Ключи внутри блока:

| Ключ | Описание |
|---|---|
| `locale` | Код/имя локали. |
| `description_text` | Описание (многострочное). Алиас: `description`. |
| `keywords` | Ключевые слова через запятую. |
| `whats_new` | «Что нового» (для обновлений). Игнорируется, если задан `whats_new_unique`. |
| `promotional_text` | Промо-текст. |
| `screenshots_folder_name` | Имя папки со скриншотами в архиве. |
| `replace_screenshots` | `yes` → заменить существующие скрины, `no` → добавить. |

> **Единый What's New на все локали** — верхнеуровневый ключ `whats_new_unique`
> (многострочный). Если он задан, `whats_new` из блоков игнорируется, и на все
> локали ставится один и тот же текст:
> ```
> whats_new_unique: Исправлены ошибки
> и улучшена производительность
> ```

```
distribution_page: [
{
locale: en-US
description_text: This is a description.
It can be multiline.

Even with empty lines.
keywords: game,fun,arcade,casual
promotional_text: Play now!
screenshots_folder_name: screenshots_en_us
replace_screenshots: yes
},
{
locale: en-GB
description_text: UK description.
keywords: game,fun,uk
screenshots_folder_name: screenshots_en_gb
}
]
```

### 8. App Review Information

Данные для модерации (демо-аккаунт, контакты, заметки). Шаг запускается, если задано
**любое** из полей.

| Ключ | Тип | ASC-атрибут |
|---|---|---|
| `app_review_info_signin_required` | bool | `no` → снять «Sign-in required»; `yes` → требуется демо-аккаунт. |
| `app_review_info_signin_username` | str | Логин демо-аккаунта (при `yes`). |
| `app_review_info_signin_password` | str | Пароль демо-аккаунта (при `yes`). |
| `app_review_info_contact_name` | str | Имя контакта (contactFirstName). |
| `app_review_info_contact_last_name` | str | Фамилия (contactLastName). |
| `app_review_info_contact_phone_number` | str | Телефон (contactPhone). |
| `app_review_info_contact_email` | str | Email (contactEmail). |
| `app_review_info_notes` | str | Заметки для ревьюера (notes). **Многострочный** — продолжается до следующего ключа. |
| `set_app_review_contact_as_account_holder` | bool | `yes` → имя/фамилию контакта взять из держателя аккаунта (ACCOUNT_HOLDER). |

> При `set_app_review_contact_as_account_holder: yes` ключи `app_review_info_contact_name`
> и `app_review_info_contact_last_name` **игнорируются** — вместо них подставляются
> имя и фамилия держателя аккаунта (получаются через `GET /v1/users`, нужен
> ключ с ролью Admin).

```
app_review_info_signin_required: no
app_review_info_contact_name: John
app_review_info_contact_last_name: Doe
app_review_info_contact_phone_number: +12184033620
app_review_info_contact_email: john@example.com
app_review_info_notes: No login required. Just open and play.
```

### 9. Copyright

| Ключ | Тип | Описание |
|---|---|---|
| `copyright_text` | str | Текст копирайта. Наличие ключа = обновляем. |
| `set_copyright_as_account_holder` | bool | `yes` → copyright = имя+фамилия держателя (индивидуальный аккаунт). |
| `set_copyright_as_company` | bool | `yes` → copyright = имя компании из API (корпоративный аккаунт). |

> Любой из флагов **игнорирует** `copyright_text`. Приоритет: `set_copyright_as_company`
> → `set_copyright_as_account_holder` → `copyright_text`.
> - `set_copyright_as_company` берёт имя компании из API (`provider.name` web-сессии) —
>   заполнять вручную ничего не нужно;
> - `set_copyright_as_account_holder` берёт «Имя Фамилия» держателя (через `GET /v1/users`).

```
copyright_text: 2026 My Company Inc.
```
или:
```
set_copyright_as_account_holder: yes
```

### 10. Age Rating

| Ключ | Тип | Описание |
|---|---|---|
| `age_ratings_setup` | bool | `yes` → выставить минимальный возрастной рейтинг (все «нет»). |

```
age_ratings_setup: yes
```

### 11. Data Collection

| Ключ | Тип | Описание |
|---|---|---|
| `setup_data_collection` | bool | `yes` → App Privacy: «данные не собираются» + публикация. |

```
setup_data_collection: yes
```

### 12. Pricing & Availability

| Ключ | Тип | Описание |
|---|---|---|
| `pricing_default` | bool | `yes` → приложение бесплатное (цена 0 во всех странах). |
| `app_availability_excepts` | list | Территории-исключения (см. ниже). |
| `app_pre_order_enabled` | bool | `yes` → включить пре-ордер. |
| `app_pre_order_release_date` | date | Дата релиза пре-ордера, `DD.MM.YYYY`. |
| `disable_apple_silicon_macos_available` | bool | `yes` → выключить доступность на Apple Silicon Mac. |
| `disable_apple_vision_pro_available` | bool | `yes` → выключить доступность на Apple Vision Pro. |

> Чекбоксы «iPhone and iPad Apps on Apple Silicon Macs / Apple Vision Pro» в ASC
> включены по умолчанию. `disable_*: yes` их **выключает** (через приватный
> web-API). `no` или отсутствие ключа — не трогаем (останется включено).

Логика `app_availability_excepts`:
- `[]` — доступно во **всех** странах;
- `[ USA, CAN ]` — все страны, **кроме** указанных;
- ключ **не указан** — доступность не трогаем.

```
pricing_default: yes
app_availability_excepts: []
app_pre_order_enabled: yes
app_pre_order_release_date: 01.12.2026
disable_apple_silicon_macos_available: yes
disable_apple_vision_pro_available: yes
```

### 13. Build

| Ключ | Тип | Описание |
|---|---|---|
| `set_last_build` | bool | `yes` → выбрать последний загруженный билд. |

При `yes`: берётся последний загруженный билд → ожидается окончание processing →
**encryption = none** (`usesNonExemptEncryption=false`, официальный API) → билд
привязывается к версии.

```
set_last_build: yes
```

### 14. Release

| Ключ | Тип | Описание |
|---|---|---|
| `release_type` | str | Тип релиза версии: `manual` / `automatic` / `scheduled`. |
| `release_date` | date | Дата релиза для `scheduled` (`DD.MM.YYYY` или ISO). |
| `release` | bool | `yes` → отправить приложение на модерацию (в самом конце). |

`release_type` (официальный атрибут `releaseType` версии):
- `manual` → `MANUAL` — релиз вручную после одобрения;
- `automatic` → `AFTER_APPROVAL` — автоматически сразу после одобрения;
- `scheduled` → `SCHEDULED` — на дату из `release_date` (обязательна).

```
release_type: scheduled
release_date: 01.12.2026
release: yes
```

Ручной релиз:
```
release_type: manual
release: yes
```

### 15. Монетизация

Три объекта: `consumable_iap_products`, `non_consumable_iap_products`, `sub_products`.
Внутри — параллельные списки (i-й элемент каждого списка относится к i-му продукту).

**Формат матриц локализаций:** `[[...], [...]]`, где **внешний список = локаль**
(в порядке `*_localization_locales`), **внутренний = продукт** (в порядке
`*_ref_names`).

**Доступность по странам (`*_availability_only` / `*_availability_except`) —
взаимоисключающие, указывайте максимум ОДИН:**

| Что указано | Результат |
|---|---|
| ни один ключ (или оба пустые/отсутствуют) | продукт доступен во **всех** странах |
| только `*_availability_only: [ USA, CAN ]` | доступен **только** в указанных странах |
| только `*_availability_except: [ RUS ]` | доступен во **всех, кроме** указанных |

Если указаны оба — приоритет у `only`, `except` игнорируется. Поэтому не указывайте
оба одновременно. Значения — [коды территорий](#коды-территорий).

#### Consumable IAP

| Ключ | Формат | Описание |
|---|---|---|
| `consumable_product_ref_names` | `[a, b]` | Reference names продуктов. |
| `consumable_product_ids` | `[a, b]` | Product IDs. |
| `consumable_product_prices` | `[0.99, 4.99]` | Цены в USD. |
| `consumable_localization_locales` | `[en-US, en-GB]` | Локали локализаций. |
| `consumable_product_localization_names` | `[[..],[..]]` | Имена: `[локаль][продукт]`. |
| `consumable_product_localization_descriptions` | `[[..],[..]]` | Описания. |
| `consumable_product_availability_only` | `[USA, CAN]` | Только эти территории (взаимоискл. с `except`). |
| `consumable_product_availability_except` | `[RUS]` | Все, кроме этих (взаимоискл. с `only`). |
| `consumable_product_screenshot_paths` | `[folder1, folder2]` | Папки со скрином ревью. |

```
consumable_iap_products: {
consumable_product_ref_names: [Coins100, Coins500]
consumable_product_ids: [com.company.app.coins100, com.company.app.coins500]
consumable_product_prices: [0.99, 4.99]
consumable_localization_locales: [en-US, en-GB]
consumable_product_localization_names: [[100 Coins, 500 Coins], [100 Coins UK, 500 Coins UK]]
consumable_product_localization_descriptions: [[Get 100 coins, Get 500 coins], [Get 100 coins, Get 500 coins]]
consumable_product_screenshot_paths: [iap_shot_1, iap_shot_2]
}
```

> Доступность: в примере выше ключи `_availability_only` / `_availability_except`
> не указаны → покупки доступны во **всех** странах. Чтобы ограничить, добавьте
> **один** из них, например `consumable_product_availability_only: [ USA, CAN ]`
> или `consumable_product_availability_except: [ RUS ]`.

#### Non-consumable IAP

Те же ключи с префиксом `non_consumable_` (и `non_consumable_localization_locales`).

```
non_consumable_iap_products: {
non_consumable_product_ref_names: [RemoveAds]
non_consumable_product_ids: [com.company.app.removeads]
non_consumable_product_prices: [2.99]
non_consumable_localization_locales: [en-US]
non_consumable_product_localization_names: [[Remove Ads]]
non_consumable_product_localization_descriptions: [[Remove all ads forever]]
non_consumable_product_screenshot_paths: [iap_removeads]
}
```

#### Подписки

| Ключ | Формат | Описание |
|---|---|---|
| `group_name` | str | Имя группы подписок. |
| `sub_product_ref_names` | `[a, b]` | Reference names. |
| `sub_product_ids` | `[a, b]` | Product IDs. |
| `sub_product_prices` | `[9.99, 99.99]` | Цены USD. |
| `sub_product_durations` | `[ONE_MONTH, ONE_YEAR]` | Длительности (см. справочник). |
| `sub_three_days_trial` | `[0, 2]` | Индексы (0-based) подписок с 3-дневным триалом. |
| `sub_product_localization_locales` | `[en-US]` | Локали. |
| `sub_product_localization_names` | `[[..]]` | Имена: `[локаль][продукт]`. |
| `sub_product_localization_descriptions` | `[[..]]` | Описания. |
| `sub_product_availability_only` | `[USA]` | Только эти территории (взаимоискл. с `except`). |
| `sub_product_availability_except` | `[RUS]` | Все, кроме этих (взаимоискл. с `only`). |
| `sub_product_screenshot_paths` | `[folder]` | Папки со скрином ревью. |

```
sub_products: {
group_name: Premium
sub_product_ref_names: [Monthly, Yearly]
sub_product_ids: [com.company.app.monthly, com.company.app.yearly]
sub_product_prices: [9.99, 99.99]
sub_product_durations: [ONE_MONTH, ONE_YEAR]
sub_three_days_trial: [0]
sub_product_localization_locales: [en-US, en-GB]
sub_product_localization_names: [[Monthly Premium, Yearly Premium], [Monthly UK, Yearly UK]]
sub_product_localization_descriptions: [[Access all features, Save 20% yearly], [Access all, Save 20%]]
sub_product_screenshot_paths: [sub_shot]
}
```

---

## Порядок выполнения

Операции выполняются в фиксированном порядке (пропускаются, если ключей нет):

1. **bundle** — создание bundle id (`create_bundle`)
2. **app** — создание/резолв приложения (всегда)
3. **create_version** — новая версия (`create_new_version`)
4. **app_info** — метаданные по локалям (`app_info`)
5. **content_rights** — content rights (`uses_third_party_content`)
6. **categories** — категории (`primary_category`/`secondary_category`)
7. **distribution** — тексты + скриншоты (`distribution_page`)
8. **app_review_info** — данные для ревью (любой `app_review_info_*`)
9. **copyright** — копирайт (`copyright_text`)
10. **age_ratings** — возрастной рейтинг (`age_ratings_setup`)
11. **pricing** — цена (`pricing_default`)
12. **availability** — доступность + пре-ордер (`app_availability_excepts` / `app_pre_order_enabled`)
12a. **platform_availability** — выключение платформ (`disable_apple_silicon_macos_available` / `disable_apple_vision_pro_available`)
13. **set_last_build** — билд + encryption (`set_last_build`)
14. **data_collection** — App Privacy (`setup_data_collection`)
15. **iap_consumable / iap_nonconsumable / subscriptions** — монетизация
15a. **release_type** — тип релиза версии (`release_type` [+ `release_date`])
16. **release** — отправка на модерацию (`release`)

Ошибка одной операции не останавливает остальные — статус запуска станет `partial`,
детали видны в live-логе.

---

## Справочники констант

### Коды локалей

Используются в `locale` (app_info, distribution_page) и `*_localization_locales`.
Можно указывать как код, так и ассоциативное имя.

| Код | Язык | Код | Язык |
|---|---|---|---|
| `ar-SA` | Arabic | `ko` | Korean |
| `bn-BD` | Bangla | `ms` | Malay |
| `ca` | Catalan | `ml-IN` | Malayalam |
| `zh-Hans` | Chinese Simplified | `mr-IN` | Marathi |
| `zh-Hant` | Chinese Traditional | `no` | Norwegian |
| `hr` | Croatian | `or-IN` | Odia |
| `cs` | Czech | `pl` | Polish |
| `da` | Danish | `pt-BR` | Portuguese (Brazil) |
| `nl-NL` | Dutch | `pt-PT` | Portuguese (Portugal) |
| `en-AU` | English (Australia) | `pa-IN` | Punjabi |
| `en-CA` | English (Canada) | `ro` | Romanian |
| `en-GB` | English (UK) | `ru` | Russian |
| `en-US` | English (US) | `sk` | Slovak |
| `fi` | Finnish | `sl-SI` | Slovenian |
| `fr-FR` | French | `es-MX` | Spanish (Mexico) |
| `fr-CA` | French (Canada) | `es-ES` | Spanish (Spain) |
| `de-DE` | German | `sv` | Swedish |
| `el` | Greek | `ta-IN` | Tamil |
| `gu-IN` | Gujarati | `te-IN` | Telugu |
| `he` | Hebrew | `th` | Thai |
| `hi` | Hindi | `tr` | Turkish |
| `hu` | Hungarian | `uk` | Ukrainian |
| `id` | Indonesian | `ur-PK` | Urdu |
| `it` | Italian | `vi` | Vietnamese |
| `ja` | Japanese | | |
| `kn-IN` | Kannada | | |

> `en-UK` принимается как алиас `en-GB`.

### Коды территорий

Используются в `app_availability_excepts` и `*_availability_only` / `*_availability_except`.
Это **коды стран (ISO-3)**, НЕ локали.

| Код | Страна | Код | Страна | Код | Страна |
|---|---|---|---|---|---|
| `AFG` | Афганистан | `AGO` | Ангола | `AIA` | Ангилья |
| `ALB` | Албания | `ARE` | ОАЭ | `ARG` | Аргентина |
| `ARM` | Армения | `ATG` | Антигуа и Барбуда | `AUS` | Австралия |
| `AUT` | Австрия | `AZE` | Азербайджан | `BEL` | Бельгия |
| `BEN` | Бенин | `BFA` | Буркина-Фасо | `BGR` | Болгария |
| `BHR` | Бахрейн | `BHS` | Багамы | `BIH` | Босния и Герцеговина |
| `BLR` | Беларусь | `BLZ` | Белиз | `BMU` | Бермуды |
| `BOL` | Боливия | `BRA` | Бразилия | `BRB` | Барбадос |
| `BRN` | Бруней | `BTN` | Бутан | `BWA` | Ботсвана |
| `CAN` | Канада | `CHE` | Швейцария | `CHL` | Чили |
| `CHN` | Китай | `CIV` | Кот-д’Ивуар | `CMR` | Камерун |
| `COD` | ДР Конго | `COG` | Республика Конго | `COL` | Колумбия |
| `CPV` | Кабо-Верде | `CRI` | Коста-Рика | `CYM` | Каймановы острова |
| `XKS` | Косово | `CYP` | Кипр | `CZE` | Чехия |
| `DEU` | Германия | `DMA` | Доминика | `DNK` | Дания |
| `DOM` | Доминиканская Респ. | `DZA` | Алжир | `ECU` | Эквадор |
| `EGY` | Египет | `ESP` | Испания | `EST` | Эстония |
| `FIN` | Финляндия | `FJI` | Фиджи | `FRA` | Франция |
| `FSM` | Микронезия | `GAB` | Габон | `GBR` | Великобритания |
| `GEO` | Грузия | `GHA` | Гана | `GMB` | Гамбия |
| `GNB` | Гвинея-Бисау | `GRC` | Греция | `GRD` | Гренада |
| `GTM` | Гватемала | `GUY` | Гайана | `HKG` | Гонконг |
| `HND` | Гондурас | `HRV` | Хорватия | `HUN` | Венгрия |
| `IDN` | Индонезия | `IND` | Индия | `IRL` | Ирландия |
| `IRQ` | Ирак | `ISL` | Исландия | `ISR` | Израиль |
| `ITA` | Италия | `JAM` | Ямайка | `JOR` | Иордания |
| `JPN` | Япония | `KAZ` | Казахстан | `KEN` | Кения |
| `KGZ` | Киргизия | `KHM` | Камбоджа | `KNA` | Сент-Китс и Невис |
| `KOR` | Южная Корея | `KWT` | Кувейт | `LAO` | Лаос |
| `LBN` | Ливан | `LBR` | Либерия | `LBY` | Ливия |
| `LCA` | Сент-Люсия | `LKA` | Шри-Ланка | `LTU` | Литва |
| `LUX` | Люксембург | `LVA` | Латвия | `MAC` | Макао |
| `MAR` | Марокко | `MDA` | Молдова | `MDG` | Мадагаскар |
| `MDV` | Мальдивы | `MEX` | Мексика | `MKD` | Северная Македония |
| `MLI` | Мали | `MLT` | Мальта | `MMR` | Мьянма |
| `MNE` | Черногория | `MNG` | Монголия | `MOZ` | Мозамбик |
| `MRT` | Мавритания | `MSR` | Монтсеррат | `MUS` | Маврикий |
| `MWI` | Малави | `MYS` | Малайзия | `NAM` | Намибия |
| `NER` | Нигер | `NGA` | Нигерия | `NIC` | Никарагуа |
| `NLD` | Нидерланды | `NOR` | Норвегия | `NPL` | Непал |
| `NRU` | Науру | `NZL` | Новая Зеландия | `OMN` | Оман |
| `PAK` | Пакистан | `PAN` | Панама | `PER` | Перу |
| `PHL` | Филиппины | `PLW` | Палау | `PNG` | Папуа — Новая Гвинея |
| `POL` | Польша | `PRT` | Португалия | `PRY` | Парагвай |
| `QAT` | Катар | `ROU` | Румыния | `RUS` | Россия |
| `RWA` | Руанда | `SAU` | Саудовская Аравия | `SEN` | Сенегал |
| `SGP` | Сингапур | `SLB` | Соломоновы острова | `SLE` | Сьерра-Леоне |
| `SLV` | Сальвадор | `SRB` | Сербия | `STP` | Сан-Томе и Принсипи |
| `SUR` | Суринам | `SVK` | Словакия | `SVN` | Словения |
| `SWE` | Швеция | `SWZ` | Эсватини | `SYC` | Сейшелы |
| `TCA` | Тёркс и Кайкос | `TCD` | Чад | `THA` | Таиланд |
| `TJK` | Таджикистан | `TKM` | Туркменистан | `TON` | Тонга |
| `TTO` | Тринидад и Тобаго | `TUN` | Тунис | `TUR` | Турция |
| `TWN` | Тайвань | `TZA` | Танзания | `UGA` | Уганда |
| `UKR` | Украина | `URY` | Уругвай | `USA` | США |
| `UZB` | Узбекистан | `VCT` | Сент-Винсент и Гренадины | `VEN` | Венесуэла |
| `VGB` | Британские Виргинские о-ва | `VNM` | Вьетнам | `VUT` | Вануату |
| `YEM` | Йемен | `ZAF` | ЮАР | `ZMB` | Замбия |
| `ZWE` | Зимбабве | | | | |

### Категории (appCategories)

Значения для `primary_category` / `secondary_category` (ID, заглавными):

```
BOOKS, BUSINESS, DEVELOPER_TOOLS, EDUCATION, ENTERTAINMENT, FINANCE,
FOOD_AND_DRINK, GAMES, GRAPHICS_AND_DESIGN, HEALTH_AND_FITNESS, LIFESTYLE,
MAGAZINES_AND_NEWSPAPERS, MEDICAL, MUSIC, NAVIGATION, NEWS, PHOTO_AND_VIDEO,
PRODUCTIVITY, REFERENCE, SHOPPING, SOCIAL_NETWORKING, SPORTS, STICKERS,
TRAVEL, UTILITIES, WEATHER
```

Подкатегории игр (если нужны): `GAMES_ACTION`, `GAMES_ADVENTURE`, `GAMES_ARCADE`,
`GAMES_BOARD`, `GAMES_CARD`, `GAMES_CASINO`, `GAMES_DICE`, `GAMES_EDUCATIONAL`,
`GAMES_FAMILY`, `GAMES_MUSIC`, `GAMES_PUZZLE`, `GAMES_RACING`, `GAMES_ROLE_PLAYING`,
`GAMES_SIMULATION`, `GAMES_SPORTS`, `GAMES_STRATEGY`, `GAMES_TRIVIA`, `GAMES_WORD`.

> Точный актуальный список отдаёт `GET /v1/appCategories` (тот же, что в UI).

### Длительности подписок

Значения для `sub_product_durations`:

```
ONE_WEEK, ONE_MONTH, TWO_MONTHS, THREE_MONTHS, SIX_MONTHS, ONE_YEAR
```

По умолчанию (если значение неизвестно) — `ONE_MONTH`.

### Платформы

Значения для `platform`: `iOS` (→ IOS), `macOS` (→ MAC_OS), `tvOS` (→ TV_OS).

---

## Полный пример

```
account_name: dev@example.com
account_email: dev@example.com
account_password: MyP@ssw0rd
account_phone_number: 12184033620
proxy_url: http://user:pass@1.2.3.4:8080

create_bundle: yes
create_app: yes
bundle_id: com.company.coolapp
app_name: My Cool App
sku: com.company.coolapp
platform: iOS
create_new_version: yes

privacy_policy_url: https://example.com/privacy
support_url: https://example.com/support
marketing_url: https://example.com

app_info: [
{
locale: en-US
app_name: My Cool App
subtitle: The best app ever
},
{
locale: en-GB
app_name: My Cool App UK
}
]

primary_category: GAMES
secondary_category: ENTERTAINMENT

uses_third_party_content: no

distribution_page: [
{
locale: en-US
description_text: This is a description.
It can be multiline.

Even with empty lines.
keywords: game,fun,arcade,casual
promotional_text: Play now!
screenshots_folder_name: screenshots_en_us
replace_screenshots: yes
},
{
locale: en-GB
description_text: UK description.
keywords: game,fun,uk
screenshots_folder_name: screenshots_en_gb
}
]

app_review_info_signin_required: no
app_review_info_contact_name: John
app_review_info_contact_last_name: Doe
app_review_info_contact_phone_number: +12184033620
app_review_info_contact_email: john@example.com
app_review_info_notes: No login required.

copyright_text: 2026 My Company Inc.

age_ratings_setup: yes
setup_data_collection: yes

pricing_default: yes
app_availability_excepts: []
app_pre_order_enabled: yes
app_pre_order_release_date: 01.12.2026

set_last_build: yes
release: yes

consumable_iap_products: {
consumable_product_ref_names: [Coins100, Coins500]
consumable_product_ids: [com.company.coolapp.coins100, com.company.coolapp.coins500]
consumable_product_prices: [0.99, 4.99]
consumable_localization_locales: [en-US]
consumable_product_localization_names: [[100 Coins, 500 Coins]]
consumable_product_localization_descriptions: [[Get 100 coins, Get 500 coins]]
consumable_product_screenshot_paths: [iap_shot_1, iap_shot_2]
}

sub_products: {
group_name: Premium
sub_product_ref_names: [Monthly, Yearly]
sub_product_ids: [com.company.coolapp.monthly, com.company.coolapp.yearly]
sub_product_prices: [9.99, 99.99]
sub_product_durations: [ONE_MONTH, ONE_YEAR]
sub_three_days_trial: [0]
sub_product_localization_locales: [en-US]
sub_product_localization_names: [[Monthly Premium, Yearly Premium]]
sub_product_localization_descriptions: [[Access all features, Save 20% yearly]]
sub_product_screenshot_paths: [sub_shot]
}
```
