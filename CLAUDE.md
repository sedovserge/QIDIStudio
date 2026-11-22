# QIDIStudio Fork — Box Sync Fix

Форк [QIDITECH/QIDIStudio](https://github.com/QIDITECH/QIDIStudio) с фиксами для корректной работы QIDI Box с принтером Q2 на прошивке **1.1.1** и пользовательскими (не-QIDI) RFID-метками.

**Ветка**: `fix/box-02.07.02.60` (предыдущая база: `fix/box-vendor-mapping`, тег `checkpoint/pre-rebase-02.07.02.60`)
**База upstream**: `8e28fc30` = `c58fdc56` (02.07.02.60, 2026-08-06) + PA-фикс `8e28fc30`
**Автор проблемы**: баги в оригинальной студии при работе с не-QIDI филаментами.
**Статус**: на 02.05.01.53 подтверждено на реальном принтере Q2 + QIDI Box со слотами NIT PETG, Filamentarno, KREMEN. Перенос на 02.07.02.60 — ждёт проверки на принтере.

---

## ⚠️ Не обновлять базу дальше `c58fdc56` (02.07.02.60)

Начиная с `c07a89e2` (2026-10-07) upstream читает Box **только** из Klipper-объекта `multi_color_controller` (строки `vendor_str`, `color_rgb_str`, `filament_id` вида `QF…`). Его отдаёт прошивка Q2 **01.01.03.x**. На **1.1.1** этого объекта нет (Moonraker возвращает `{}`), Box отдаётся индексами в `save_variables` — со свежим upstream Box не синхронизируется вообще.

Прошивку не обновляем:
- даунгрейда нет (официально запрещён);
- в 01.01.03.x сопоставление меток зашито в прошивку: `QIDI_Client/resource/filament_config/filament-0.4.json` (56 записей: QIDI, Generic, Bambu, HATCHBOX, Overture, Polymaker) + `LEGACY_RFID_*_MAP` в `box_rfid`; обновление удаляет `box_rfid.*` и перезаписывает `printer_data/config` стоковыми файлами;
- полезных для нас фиксов в 01.01.02.x–01.01.03.x нет.

Пакеты прошивок (для анализа): https://www.qidimaker.com/en/downloads/firmware/Q2

Из upstream после `c58fdc56` переносить только точечные фиксы (`cherry-pick`), не трогающие `QDSDeviceManager`.

---

## TL;DR — что и зачем

Оригинальная студия не могла отправить задание на печать, если в Box стояли филаменты не-QIDI брендов (NIT, Filamentarno, Polymaker и т.д.). Ошибка: **"Not all filaments used in slicing are mapped to the printer"**.

Принтерная часть: на принтере стоковый `klippy/extras/box_rfid.so` заменён на `box_rfid.py`. Стоковый модуль принимает метки только с вендором QIDI (1) / Generic (0), а заменённый пропускает любые вендоры из `[vendor_list]` пользовательского `officiall_filas_list.cfg` (NIT=2 … KREMEN=8). Студийная часть — коммиты ниже.

Причины и фиксы (SHA: старая база 02.05.01.53 → новая 02.07.02.60):

| # | Коммит (старый → новый) | Проблема | Файл |
|---|--------|----------|------|
| 1 | `89d5d93` → `ae5ab63` | vendor/color индексы перекрывались с fila-индексами в `m_filamentConfig` | `QDSDeviceManager.cpp` + `.hpp` |
| 2 | `17504bd` → `4f4bad2` | Guard от затирания пустыми данными + проверки границ + возврат записи в `m_filamentConfig` | `QDSDeviceManager.cpp` |
| 3 | `ee7c83d` + `2d03867` → `970a1d8` + `68d7791` | cfg с принтера через Moonraker (в 02.07 без этого Box вообще не заполняется) | `QDSDeviceManager.cpp` + `.hpp` |
| 4 | `a162f8c` → `6995187` | Шифрование логов мешало диагностике | `Utils.hpp` |

---

## Коммит 1: `89d5d93` — Fix vendor/color mapping

### Проблема

QIDI Box читает RFID-метку и сохраняет три числовых кода в Klipper `save_variables`:
- `filament_slotN` — номер секции `[filaN]` в `officiall_filas_list.cfg`
- `vendor_slotN` — ID из секции `[vendor_list]`
- `color_slotN` — ID из секции `[colordict]`

**Баг в оригинале**: студия хранит все три типа данных в **одном** массиве `m_filamentConfig[100]`:
```cpp
m_filamentConfig[index].vendor        // из vendor_list
m_filamentConfig[index].colorHexCode  // из colordict
m_filamentConfig[index].name/type     // из [filaN]
```

Индексы **перекрываются**: vendor_list занимает 0-7, fila-секции — 1-99, colordict — 1-24. При чтении `vendor_slot = 2` (NIT) студия брала `m_filamentConfig[2].vendor` — но по индексу 2 лежат данные из `[fila2]` (PLA Matte) смешанные с `vendor_list[2]` (NIT). Работало случайно только если vendor_list писался **последним** и перетирал fila-данные в поле vendor.

Второй баг там же — в формировании `tray_info_idx` (`QD_{printer}_{vendor}_{fila}`):
```cpp
std::string test_vendor = slot_vendor == "QIDI" ? "1" : "0";  // упрощение!
```
Любой не-QIDI vendor превращался в `0`, теряя реальный ID.

### Фикс

**`QDSDeviceManager.hpp`**:
- Добавлены **отдельные** массивы `m_vendorNames` и `m_colorHexByIndex` (static-версии `m_general_*` были на базе 02.05; в 02.07 upstream их удалил вместе с `initGeneralData()`)
- В struct `Filament` добавлено поле `vendor_index` (числовой ID vendor из RFID)

**`QDSDeviceManager.cpp`**:
- `initGeneralData()` пишет `vendor_list` и `colordict` **в отдельные массивы** (не в `m_filamentConfig`)
- `updateBoxDataByJson()` читает vendor из `m_vendorNames[vendorIndx]`, color из `m_colorHexByIndex[colorIndex]` + bounds check
- Формирование `QD_` (2 места: cpp:~356 и cpp:~1988) — использует `std::to_string(vendor_index)` вместо `0/1`

### Результат

До: NIT PETG на Q2 → `tray_info_idx = "QD_1_0_51"` → не находит профиль
После: NIT PETG на Q2 → `tray_info_idx = "QD_1_2_51"` → матч с `NIT PETG Basic @Q2 0.4 nozzle.json` ✓

### Как апстримовать при слиянии новой версии

Ищи в `QDSDeviceManager.cpp` паттерны:
- `m_filamentConfig[vendorIndx].vendor` → должно быть `m_vendorNames[vendorIndx]`
- `m_filamentConfig[colorIndex].colorHexCode` → должно быть `m_colorHexByIndex[colorIndex]`
- `slot_vendor == "QIDI" ? "1" : "0"` → должно быть `std::to_string(m_boxData[i].vendor_index)`

---

## Коммит 2: `17504bd` — Guard против затирания данных

### Проблема (регресс после коммита 1)

В `updateFilamentConfig()` при ответе API:
```cpp
std::vector<std::string> vendors;       // локальная, пустая если API не вернул
std::vector<std::string> colorHexCodes; // то же самое
parseToString("vendor_list", vendors);
parseToString("colordict", colorHexCodes);
...
m_vendorNames = vendors;       // ПУСТЫЕ → затирают данные из initGeneralData!
m_colorHexByIndex = colorHexCodes;
```

Также после коммита 1 перестал заполняться `m_filamentConfig[i].vendor`/`colorHexCode`, а `PrinterWebView.cpp:1973,1978` их читает (для SET_COLOR/SET_FILAMENT_VENDOR событий).

### Фикс

```cpp
if (!vendors.empty())    m_vendorNames = vendors;
if (!colorHexCodes.empty()) m_colorHexByIndex = colorHexCodes;
```

Плюс вернули запись в `m_filamentConfig` (dual-write):
```cpp
if (i < vendors.size())       m_filamentConfig[i].vendor = vendors[i];
if (i < colorHexCodes.size()) m_filamentConfig[i].colorHexCode = colorHexCodes[i];
```

В `initGeneralData()` аналогично — пишем и в отдельные массивы, и в `m_filamentConfig`.

### Позже переработано коммитом 3

Этот guard стал неактуален когда полностью переписали `updateFilamentConfig()` на Moonraker+INI. Но логика "не затирать пустыми" осталась уже в helper'е `parseFilamentConfigFromIni()` через bounds check.

---

## Коммит 3: `ee7c83d` + `2d03867` — Скачивание cfg с принтера

### Проблема

Студия использует bundled `resources/profiles/officiall_filas_list.cfg` (50 fila-записей, 2 vendor). Прошивка принтера OTA-обновляется и добавляет новые записи (например `[fila51]` для PETG, `[fila54]` для PETG-CF17, vendor_list 2-7 для NIT/Filamentarno/Polymaker).

Когда Box шлёт `filament_slot = 51`, студия не находит `[fila51]` в своём cfg → `m_filamentConfig[51].type` = `""` → в `DevMappingUtil::ams_filament_mapping()` проверка:
```cpp
if (toLower(filaments[i].type) != toLower(box_filament_infos[j].type)) {
    val.distance = 999999;  // тип не совпадает!
}
```
distance для всех Box-слотов = 999999 → маппинг пустой → ошибка "Not all filaments...".

Существующий `updateFilamentConfig()` пытался тянуть данные с `http://<printer>/api/qidiclient/config/offical_filament_list` — но этот эндпоинт **возвращает 404** на Q2-прошивке. Код молча падал в silent catch.

### Фикс

**`updateFilamentConfig()` переписан**:
- URL: `http://<ip>:7125/server/files/config/officiall_filas_list.cfg` (стандартный Moonraker file API)
- Формат: **INI** (через `boost::property_tree::ini_parser::read_ini(std::istringstream, pt)`), не JSON
- Логика парсинга вынесена в shared helper `parseFilamentConfigFromIni()`, используется также в `initGeneralData()` (DRY)
- `BOOST_LOG_TRIVIAL` логирование (было silent catch)
- Extract host из `m_frp_url` (отрезает `http://` и существующий порт)
- Graceful fallback: если download упал — используем bundled cfg (на базе 02.05; в 02.07 bundled cfg нет — fallback на пути upstream)

**На базе 02.07.02.60** (`970a1d8`): upstream-код `updateFilamentConfig()` сохранён, Moonraker-ветка вставлена в начало async-лямбды до захвата `m_config_mtx`. HTTP и разбор INI — в локальные векторы, затем под мьютексом копирование на место + `flushPendingBoxUpdate()`. Если текст cfg не изменился с прошлой загрузки (`m_lastFilamentCfgBody`), таблицы не трогаются. `parseFilamentConfigFromIni()` возвращает число `[filaN]`; при 0 — fallback.

**Вызов метода** (`2d03867`):
- Раньше вызывался только из `PrinterWebView.cpp:1583` (cloud/network-device path)
- Перенесён в `QDSDeviceManager::onOpen()` (WebSocket connect) — срабатывает **каждый раз** при подключении (старт студии, реконнект). На 02.07 — внутри detached-потока `onOpen` перед `sendSubscribeMessage`, только для локальных устройств (`!is_net_device && !active_p2p`)
- Добавлен public `refreshFilamentConfig()` (сбрасывает `m_is_init_filamentConfig` + вызывает update) — нужен чтобы `QDSDeviceManager` мог форсировать обновление без доступа к приватному полю

### Результат

```
updateFilamentConfig: GET http://192.168.15.154:7125/server/files/config/officiall_filas_list.cfg
updateFilamentConfig: loaded from Moonraker OK
```
Студия автоматически получает актуальные fila-записи с принтера. OTA-обновления прошивки подхватываются без обновления студии.

### Как апстримовать при слиянии новой версии

Проверь:
- `updateFilamentConfig()` — полностью заменён (не JSON API, а Moonraker INI)
- `QDSDeviceManager::onOpen()` — добавлен вызов `dev->refreshFilamentConfig()`
- `QDSDeviceManager::addDevice(...)` — вызова `updateFilamentConfig` там **не должно быть** (убран в пользу onOpen)

---

## Коммит 4: `a162f8c` — Отключение шифрования логов

### Проблема

`LogSink.cpp` шифрует логи AES-256-CBC по умолчанию (`Utils.hpp:374`: `LogEncType enc_type = LOG_ENC_AES_256_CBC;`). Логи записываются как бинарные файлы с суффиксом `_enc`, что мешает диагностике.

### Фикс

Меняем дефолт на `LOG_ENC_NONE`. Логи пишутся plaintext. Существующая функция `DecodeAES256LogFile()` остаётся для расшифровки старых зашифрованных логов если понадобится.

### Примечание

Это удобство разработчика, не функциональный фикс. Можно не переносить при апстриме если лень.

---

## Цепочка данных RFID → Студия (итоговая)

```
RFID-метка (16 байт, Sector 1 Block 0):
  [material_code, color_code, vendor_code, 0...0, Q, I, D, I]
       ↓
box_rfid.py (Klipper) читает байты:
  filament = data[0]  →  save_variable("filament_slot0", 51)
  color    = data[1]  →  save_variable("color_slot0", 2)
  vendor   = data[2]  →  save_variable("vendor_slot0", 2)
       ↓
QDSDeviceManager::onOpen() → фоновый поток → refreshFilamentConfig():
  HTTP GET http://<printer>:7125/server/files/config/officiall_filas_list.cfg
  parse INI → m_filamentConfig[51].type = "PETG"
              m_vendorNames[2] = "NIT"
              m_colorHexByIndex[2] = "#060606"
       ↓
QDSDevice::updateBoxDataByJson() читает save_variables:
  m_boxData[0].filament_idex = 51
  m_boxData[0].type = m_filamentConfig[51].type = "PETG"
  m_boxData[0].vendor = m_vendorNames[2] = "NIT"
  m_boxData[0].vendor_index = 2
  m_boxData[0].colorHexCode = m_colorHexByIndex[2] = "#060606"
       ↓
Формирование tray_info_idx:
  "QD_1_2_51" = "QD_<Q2>_<NIT>_<PETG>"
       ↓
DevMappingUtil::ams_filament_mapping() сравнивает типы:
  "PETG" (slicer) == "PETG" (box) → distance = color distance (не 999999)
  → маппинг успешный → печать отправляется ✓
```

---

## Справочник форматов filament_id

| Формат | Где используется | Пример |
|--------|------------------|--------|
| `QD_{printer}_{vendor}_{fila}` | Системные пресеты Q/X серий + Box sync | `QD_1_2_51` |
| `GF{X}99` | Системные пресеты (наследие BambuStudio) | `GFG99` (PETG), `GFL99` (PLA), `GFB99` (fallback) |
| `P` + MD5(name)[:7] | Пользовательские пресеты | `Pe2d24e4` |
| `ADV` + MD5(type)[:6] | Advanced-профили (тул QidiStudioProfiler) | `ADV6F2944` |

### Генерация P-хеша (`CreatePresetsDialog.cpp`, функция `get_filament_id()`)
```
filament_id = "P" + MD5("<vendor> <type> <serial>")[:7]
```
Вход — имя профиля без `@Printer` суффикса:
- `"eSUN PETG Basic"` → MD5 → `Pe2d24e4`
- `"Polymaker PLA Normal"` → MD5 → `P502b650`

### Printer type mapping в QD_ формате

| Принтер | Код |
|---------|-----|
| X-Plus 4 | 0 |
| Q2 | 1 |
| Q2C | 2 |
| X-Max 4 | 3 |

### Vendor list (из printer's `officiall_filas_list.cfg`)

| ID | Vendor |
|----|--------|
| 0 | Generic |
| 1 | QIDI |
| 2 | NIT |
| 3 | Filamentarno |
| 4 | Polymaker |
| 5 | PIC |
| 6 | Fiberon |
| 7 | FDPlast |
| 8 | KREMEN |

ID 2–8 добавлены пользователем вручную в cfg на принтере (вместе с `[fila51]`…`[fila57]`). Через Moonraker студия подхватывает их автоматически.

---

## Как мерджить новую версию upstream

> Сначала прочитай раздел «⚠️ Не обновлять базу дальше `c58fdc56`». Новая база имеет смысл только если она ещё содержит разбор `save_variables` (проверка: `git show <sha>:src/slic3r/GUI/QDSDeviceManager.cpp | grep -c multi_color_controller` должно быть 0).

1. `git remote add upstream https://github.com/QIDITECH/QIDIStudio && git fetch upstream`
2. Checkpoint: `git tag -a checkpoint/pre-rebase-<ver> <текущая ветка> -m "..."`
3. `git switch -c fix/box-<ver> <текущая ветка>`
4. `git -c merge.conflictStyle=zdiff3 rebase --no-keep-empty --onto <новая_база> <старая_база>` (старая база сейчас `8e28fc30`)
5. Конфликты будут в:
   - `src/slic3r/GUI/QDSDeviceManager.cpp` — `updateBoxDataByJson()`, `updateFilamentConfig()` (`parseFilamentJson`, Moonraker-ветка), `parseFilamentConfigFromIni()`, `onOpen()`, `upBoxInfoToBoxMsg()`
   - `src/slic3r/GUI/QDSDeviceManager.hpp` — `vendor_index`, `m_vendorNames`, `m_colorHexByIndex`, `refreshFilamentConfig()`, `std::atomic<bool> m_is_init_filamentConfig`
   - `src/libslic3r/Utils.hpp` — одна строка про log encryption
6. **Кодировка**: `.cpp/.hpp` в GBK, переводы строк LF. Не открывать и не сохранять их редактором, который перекодирует в UTF-8. После резолва `git diff --stat` не должен показывать тысячи строк.
7. Проверки: `git range-diff <старая_база>..<старая ветка> <новая_база>..HEAD`; `git grep -n 'm_filamentConfig\[vendorIndx\]\|m_filamentConfig\[colorIndex\]\|== "QIDI" ? "1"' -- src` — пусто.
8. CI (Windows-джоба зелёная), затем проверка на принтере (см. «Как проверить»).
9. В `main` — без force push: `git merge -s ours --no-ff <старый main>` на новой ветке, затем `main` fast-forward.

### Точки резолва конфликтов

**Изменения upstream 02.05 → 02.07, влияющие на фикс:**
- `f62d6a62` удалил bundled `resources/profiles/officiall_filas_list.cfg`, `initGeneralData()` и static `m_general_*`. Не возвращать.
- Отложенная обработка Box: если `m_filamentConfig` пуст, `updateBoxDataByJson()` кладёт status в `m_pending_save_variables`; `flushPendingBoxUpdate()` обрабатывает его после загрузки cfg.
- `updateFilamentConfig()`: `std::async` (деструктор future блокирует вызывающего), общая лямбда `parseFilamentJson`, пути p2p / облако (`#if QDT_RELEASE_TO_PUBLIC`, в CI вырезаны) / локальный `/api/qidiclient/...` (на Q2 — 404).
- `QDSPrinterWebView.cpp` сам вызывает `updateFilamentConfig()` после `addDevice`; там же по индексам читаются `m_filamentConfig[i].vendor/colorHexCode` для SET_COLOR / SET_FILAMENT_VENDOR — dual-write нужен.

**Что обязательно сохранить:**
- `updateBoxDataByJson()`: vendor из `m_vendorNames[vendorIndx]`, color из `m_colorHexByIndex[colorIndex]`, `vendor_index = vendorIndx`, проверки границ всех трёх индексов.
- `test_vendor = std::to_string(...vendor_index)` в `updateBoxDataByJson()` и `upBoxInfoToBoxMsg()`.
- Moonraker-ветка в начале async-лямбды `updateFilamentConfig()`, **до** `lock_guard(m_config_mtx)`; upstream-пути остаются fallback'ом.
- `parseFilamentJson`: `if (names.empty()) return false;` и проверки границ.

**Правила потокобезопасности:**
- HTTP — вне `m_config_mtx`; результат применять под мьютексом **копированием на место** (`m_filamentConfig = cfg`, размеры равны → буфер не перевыделяется), **не `swap`**: `swap` освобождает старый буфер, который может читать UI-поток без мьютекса (use-after-free при reconnect).
- Неизменившийся cfg (`m_lastFilamentCfgBody`) не применять повторно.
- Индексы из `save_variables` проверять как `>= 0 && < size()` (`getJsonCurStageToInt` возвращает любое целое, `-1` — только «нет ключа»).
- Не допускать `m_filamentConfig.size() == 0` перед `flushPendingBoxUpdate()` — иначе `updateBoxDataByJson()` повторно возьмёт `m_config_mtx` → deadlock.
- `refreshFilamentConfig()` вызывать только из фонового потока (в `onOpen` — внутри detached-потока до `sendSubscribeMessage`), не из потока websocketpp.

Если upstream добавил свою схему ID (не QD_) — нужно ре-исследовать, см. раздел ⚠️ выше.

---

## Сборка

### Windows (CI)
`.github/workflows/main.yml` — `windows-2022` + VS 2022 + CMake 3.31.6. Артефакт `QIDIStudio_windows` содержит готовый `QIDIStudio.exe`.

### Windows (локально)
```cmd
build_win.bat -s all -d "deps\build\QIDIStudio_dep"
```
Требует: Visual Studio 2019/2022, CMake <4.0, Strawberry Perl, pkgconfiglite.

### Linux (Docker)
```bash
./DockerBuild.sh -d   # зависимости
./DockerBuild.sh -i   # AppImage
```

### Как проверить что фикс работает

1. Запустить студию, подключиться к принтеру
2. В логе `%APPDATA%\QIDIStudio\log\studio_*.log.0` (plaintext после коммита 4) должно появиться:
   ```
   updateFilamentConfig: GET http://<ip>:7125/server/files/config/officiall_filas_list.cfg
   updateFilamentConfig: loaded from Moonraker OK, fila=57
   ```
3. Для **каждого** занятого слота тип, вендор и цвет совпадают с `save_variables` принтера и его cfg. Например, `filament_slot0=51, vendor_slot0=8` → PETG / KREMEN / `QD_1_8_51`.
4. Переподключение (перезагрузка принтера) — в логе снова загрузка cfg, интерфейс не подвисает.
5. Загрузить модель, в Prepare выбрать профиль филамента того же типа что стоит в Box (напр. PETG)
6. Prepare → Send print — должно сработать без ошибки "Not all filaments..."

Ручная проверка эндпоинтов:
```
curl -i http://<ip>/api/qidiclient/config/offical_filament_list          # на Q2 1.1.1: 404
curl http://<ip>:7125/server/files/config/officiall_filas_list.cfg      # cfg принтера
curl "http://<ip>:7125/printer/objects/query?save_variables"           # индексы слотов
curl "http://<ip>:7125/printer/objects/query?multi_color_controller"    # на 1.1.1: {}
```

---

## Известные ограничения

- **Cloud/FRP сценарий**: если принтер доступен только через QIDI cloud (не локально) — Moonraker на порту 7125 недоступен. Moonraker-ветка не сработает, остаются пути upstream. Box sync в таком сценарии всё равно работать не будет (это ограничение прошивки/cloud, не нашего фикса).
- **Нет bundled cfg** (удалён upstream в 02.07): если Moonraker недоступен при подключении, Box остаётся пустым до следующего подключения. Для локального принтера Moonraker — тот же хост, что и WebSocket, так что риск низкий.
- **Гонка читателей**: `QDSPrinterWebView` читает `m_filamentConfig` без мьютекса. Повторная загрузка неизменённого cfg пропускается, изменённый копируется на место без перевыделения — остаётся только гонка на уровне отдельных строк при реальной смене cfg на принтере (так было и в upstream). Полное решение — `shared_ptr`-снимки таблиц.
