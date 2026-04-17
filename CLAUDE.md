# QIDIStudio Fork — Box Sync Fix

Форк [bambulab/QIDIStudio](https://github.com/QIDITECH/QIDIStudio) с фиксами для корректной работы QIDI Box (АСПП) с принтером Q2 и актуальной прошивкой.

**Ветка**: `fix/box-vendor-mapping`
**Автор проблемы**: баги в оригинальной студии при работе с не-QIDI филаментами.
**Статус**: работает, подтверждено на реальном принтере Q2 + QIDI Box со слотами NIT PETG, Filamentarno, KREMEN.

---

## TL;DR — что и зачем

Оригинальная студия не могла отправить задание на печать, если в Box стояли филаменты не-QIDI брендов (NIT, Filamentarno, Polymaker и т.д.). Ошибка: **"Not all filaments used in slicing are mapped to the printer"**.

Причины и фиксы в трёх независимых коммитах:

| # | Коммит | Проблема | Файл |
|---|--------|----------|------|
| 1 | `89d5d93` | vendor/color индексы перекрывались с fila-индексами в `m_filamentConfig` | `QDSDeviceManager.cpp` + `.hpp` |
| 2 | `17504bd` | Guard от затирания локального cfg пустыми данными из API + возврат записи в `m_filamentConfig` | `QDSDeviceManager.cpp` |
| 3 | `ee7c83d` + `2d03867` | Устаревший bundled cfg → скачиваем актуальный с принтера через Moonraker | `QDSDeviceManager.cpp` |
| 4 | `a162f8c` | Шифрование логов мешало диагностике | `Utils.hpp` |

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
- Добавлены **отдельные** массивы `m_vendorNames` и `m_colorHexByIndex` (+ static-версии `m_general_vendorNames`, `m_general_colorHexByIndex`)
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
- Graceful fallback: если download упал — используем bundled cfg

**Вызов метода** (`2d03867`):
- Раньше вызывался только из `PrinterWebView.cpp:1583` (cloud/network-device path)
- Перенесён в `QDSDeviceManager::onOpen()` (WebSocket connect) — срабатывает **каждый раз** при подключении (старт студии, реконнект)
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
QDSDevice::onOpen() вызывает refreshFilamentConfig():
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

Расширяется прошивкой — через Moonraker студия подхватывает автоматически.

---

## Как мерджить новую версию upstream

1. Добавь upstream remote и подтяни: `git remote add upstream https://github.com/QIDITECH/QIDIStudio && git fetch upstream`
2. `git checkout fix/box-vendor-mapping && git rebase upstream/main`
3. Конфликты будут в этих файлах (в порядке важности):
   - `src/slic3r/GUI/QDSDeviceManager.cpp` — основные правки в `updateBoxDataByJson()`, `updateFilamentConfig()`, `initGeneralData()`, `onOpen()`, `addDevice()`
   - `src/slic3r/GUI/QDSDeviceManager.hpp` — новые поля `m_vendorNames`, `m_colorHexByIndex`, `vendor_index`, метод `refreshFilamentConfig()`
   - `src/libslic3r/Utils.hpp` — одна строка про log encryption
4. Проверь что все 4 коммита применены (см. таблицу TL;DR)
5. Запусти CI — должен пройти зелёным на Windows/Linux/macOS

### Точки резолва конфликтов

Если upstream изменил `updateFilamentConfig()` — **сохрани нашу версию** (Moonraker INI), upstream JSON API не работает на Q2.

Если upstream изменил парсинг RFID в `updateBoxDataByJson()` — **обязательно** сохрани:
- Чтение vendor из `m_vendorNames[vendorIndx]` (не `m_filamentConfig`)
- Чтение color из `m_colorHexByIndex[colorIndex]` (не `m_filamentConfig`)
- Запись `vendor_index = vendorIndx`
- Формирование `test_vendor` через `std::to_string(m_boxData[i].vendor_index)`

Если upstream добавил свою схему ID (не QD_) — нужно ре-исследовать. Прошивка Q2 шлёт `tray_info_idx` в формате `QD_*`, см. `QDSDeviceManager.cpp` (строки ~338-343 mapping и ~356 формирование).

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
   updateFilamentConfig: loaded from Moonraker OK
   ```
3. Загрузить модель, в Prepare выбрать профиль филамента того же типа что стоит в Box (напр. PETG)
4. Prepare → Send print — должно сработать без ошибки "Not all filaments..."

---

## Известные ограничения

- **Cloud/FRP сценарий**: если принтер доступен только через QIDI cloud (не локально) — Moonraker на порту 7125 недоступен. Студия упадёт на timeout → fallback на bundled cfg. Box sync в таком сценарии всё равно работать не будет (это ограничение прошивки/cloud, не нашего фикса).
- **Прошивки без fila 51+**: если прошивка старая и в cfg действительно только 50 fila — Box не сможет прислать индекс > 50 в принципе, так что проблема неактуальна.
- **Bundled cfg не обновляется**: `resources/profiles/officiall_filas_list.cfg` в форке остался исходный (50 fila). Его нет смысла обновлять — при коннекте к принтеру подтягиваем актуальный.
