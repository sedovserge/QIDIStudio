# QIDIStudio Fork — Fix QIDI Box Vendor Mapping

## Что исправлено

Баг в `QDSDeviceManager.cpp`: при чтении данных RFID-меток из QIDI Box студия неправильно сопоставляла vendor и color с профилями филаментов.

### Корневая причина

QIDI Box читает RFID-метку и сохраняет три числовых кода в Klipper `save_variables`:
- `filament_slotN` — индекс типа филамента (из секций `[filaN]` файла `officiall_filas_list.cfg`)
- `vendor_slotN` — индекс производителя (из секции `[vendor_list]`)
- `color_slotN` — индекс цвета (из секции `[colordict]`)

Студия хранит все данные в одном массиве `m_filamentConfig[100]`, куда пишутся и fila-секции (по индексу fila), и vendor_list (по индексу vendor), и colordict (по индексу color). Эти индексы **перекрываются**: vendor_list занимает 0-7, fila-секции — 1-99.

При чтении `vendor_slot = 2` (NIT) студия брала `m_filamentConfig[2].vendor` — но по индексу 2 лежат данные из `[fila2]` (PLA Matte), а не из `[vendor_list]` ключ 2 (NIT). Работало случайно для некоторых индексов из-за совпадения.

Аналогичный баг с `color_slot` — цвет брался из `m_filamentConfig[colorIndex].colorHexCode`, а не из colordict.

### Второй баг: упрощённый vendor в QD_ ID

При формировании `tray_info_idx` (формат `QD_{printer}_{vendor}_{fila}`) vendor упрощался до `0` (не-QIDI) или `1` (QIDI):
```cpp
std::string test_vendor = slot_vendor == "QIDI" ? "1" : "0";
```
Это теряло реальный vendor ID — NIT (2), Filamentarno (3), Polymaker (4) и т.д. все превращались в `0`.

## Что изменено

### Файлы
- `src/slic3r/GUI/QDSDeviceManager.hpp`
- `src/slic3r/GUI/QDSDeviceManager.cpp`

### Суть правок

1. **Отдельные массивы** `m_vendorNames` и `m_colorHexByIndex` для vendor_list и colordict (+ static-версии `m_general_*`)
2. **`initGeneralData()`** — vendor_list и colordict пишутся в отдельные массивы, не в `m_filamentConfig`
3. **`updateFilamentConfig()`** — API-данные vendor/color тоже в отдельные массивы
4. **`updateBoxDataByJson()`** — vendor берётся из `m_vendorNames[vendorIndx]`, color из `m_colorHexByIndex[colorIndex]`, добавлена проверка границ
5. **Формирование `QD_` ID** (два места) — `std::to_string(vendor_index)` вместо `slot_vendor == "QIDI" ? "1" : "0"`
6. **`Filament::vendor_index`** — новое поле для хранения числового vendor ID

### Результат

До: NIT PETG на Q2 → `QD_1_0_51` (vendor всегда 0 для не-QIDI)
После: NIT PETG на Q2 → `QD_1_2_51` (реальный vendor ID)

## Цепочка данных RFID → Студия

```
RFID-метка (16 байт, Sector 1 Block 0):
  [material_code, color_code, vendor_code, 0...0, Q, I, D, I]
       ↓
box_rfid.py (Klipper модуль) читает байты:
  filament = data[0]  →  save_variable("filament_slot0", 51)
  color    = data[1]  →  save_variable("color_slot0", 1)
  vendor   = data[2]  →  save_variable("vendor_slot0", 2)
       ↓
QDSDeviceManager.cpp читает save_variables через Moonraker API:
  filamentIndex = 51  →  m_filamentConfig[51].name/type (из [fila51])
  vendorIndx    = 2   →  m_vendorNames[2] ("NIT")        ← ИСПРАВЛЕНО
  colorIndex    = 1   →  m_colorHexByIndex[1] ("#FAFAFA") ← ИСПРАВЛЕНО
       ↓
Формирование tray_info_idx:
  "QD_" + printer_type + "_" + vendor_index + "_" + fila_index
  "QD_1_2_51"  ← ИСПРАВЛЕНО (было QD_1_0_51)
       ↓
DevFilaSystem.cpp / setting_id_to_type():
  Ищет системный пресет где filament_id == "QD_1_2_51"
  Находит → матч с профилем
```

## Справочник форматов filament_id

| Формат | Где используется | Пример |
|--------|-----------------|--------|
| `QD_{printer}_{vendor}_{fila}` | Системные пресеты Q/X серий + Box sync | `QD_1_0_51` |
| `GF{X}99` | Системные пресеты (наследие BambuStudio) | `GFG99` (PETG), `GFL99` (PLA) |
| `P` + MD5(name)[:7] | Пользовательские пресеты | `Pe2d24e4` |
| `ADV` + MD5(type)[:6] | Advanced-профили (QidiStudioProfiler) | `ADV6F2944` |

### Генерация P-хеша (пользовательские пресеты)

Исходник: `src/slic3r/GUI/CreatePresetsDialog.cpp`, функция `get_filament_id()`:
```
filament_id = "P" + MD5("<vendor> <type> <serial>")[:7]
```
Входная строка — имя профиля без `@Printer` части:
- `"eSUN PETG Basic"` → MD5 → `Pe2d24e4`
- `"Polymaker PLA Normal"` → MD5 → `P502b650`

### Printer type mapping в QD_ формате

| Принтер | Код |
|---------|-----|
| X-Plus 4 | 0 |
| Q2 | 1 |
| Q2C | 2 |
| X-Max 4 | 3 |

### Vendor list (из officiall_filas_list.cfg)

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

## FAQ

### Массивы m_vendorNames и m_colorHexByIndex — это хардкод?

Нет. Заполняются динамически из двух источников:
1. **`initGeneralData()`** — парсит локальный `officiall_filas_list.cfg` (секции `[vendor_list]` и `[colordict]`)
2. **`updateFilamentConfig()`** — запрашивает API принтера `http://<IP>/api/qidiclient/config/offical_filament_list` и перезаписывает массивы свежими данными

Если на принтере в прошивке добавится новый vendor (напр. ID 8 = "KREMEN") — студия подхватит его автоматически при подключении.

## Статус

- **CI сборка**: успешно пройдена (GitHub Actions, `windows-2022` + `ubuntu-latest` + `macos-15-intel`)
- **Ветка**: `fix/box-vendor-mapping`
- **Коммит**: `89d5d93` (+ `b44a855` trigger CI)

## Сборка

### Windows (CI)
CI настроен в `.github/workflows/main.yml` — собирает на `windows-2022` с VS 2022, CMake 3.31.6.

### Windows (локально)
```cmd
build_win.bat -s all -d "deps\build\QIDIStudio_dep"
```
Требует: Visual Studio 2019/2022, CMake < 4.0, Strawberry Perl, pkgconfiglite.

### Linux (Docker)
```bash
./DockerBuild.sh -d        # собрать зависимости
./DockerBuild.sh -i        # собрать AppImage
```
