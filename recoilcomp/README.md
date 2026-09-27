# RecoilComp — Recoil Compensation System

## Структура

```
usermode/   — C++17 приложение (RecoilComp.exe)
driver/     — WDK/KMDF kernel driver (recoil.sys)
```

## Управление

| Клавиша | Действие |
|---------|----------|
| **INSERT** | Toggle компенсации вкл/выкл |
| **ЛКМ (зажата)** | Применять очередь офсетов |
| **Правый клик трея** | Меню: оружие, выход |

Оверлей появляется только когда игра не в фокусе.

---

## Сборка — User-mode (RecoilComp.exe)

**Требования:** Visual Studio 2022, Windows SDK 10.0.22621+

```
cd usermode
mkdir build && cd build
cmake .. -G "Visual Studio 17 2022" -A x64
cmake --build . --config Release
```

Результат: `build/Release/RecoilComp.exe`

---

## Сборка — Kernel Driver (recoil.sys)

**Требования:** WDK 10.0.26100 + Visual Studio 2022

1. Открыть `driver/` как WDK проект:
   - New Project → Kernel Mode Driver (KMDF)
   - Добавить `driver.c`, `mouse_filter.c`, `evasion.c`
   - Добавить `driver.h` и `../usermode/driver_shared.h`

2. Project Properties:
   - Configuration: Release, x64
   - Driver Settings → Target OS: Windows 10 / Windows 11
   - Linker → Additional Dependencies: `wdmsec.lib`
   - Inf2Cat: не использовать (тест-подпись через bcdedit)

3. Build → реультат: `x64/Release/recoil.sys`

---

## Установка драйвера

```
# В cmd с правами администратора:
cd driver
install.bat
```

Скрипт включит тестовую подпись, скопирует `.sys`, создаст сервис и запустит драйвер.

После запуска `RecoilComp.exe` автоматически подключится к драйверу через `\\.\RecoilDriver`.

---

## Архитектура обхода античита (old Rust / devblog era)

Old Rust AC (Unity 4.x, ~2014-2018) проверял:
- `NtQuerySystemInformation(SystemModuleInformation)` → **закрыто DKOM в evasion.c**
- ETW провайдер Win32k для synthetic input → **закрыто ETW-патчем**
- Перебор `\Driver\*` объектов через ObReferenceObjectByName → **закрыто спуфингом HardwareID**
- Injected-флаг мыши (`LLMHF_INJECTED`) в WH_MOUSE_LL хуке → **закрыто mouclass ServiceCallback (нет флага)**

Kernel driver инжектирует офсеты через `PMOUSE_SERVICE_CALLBACK` напрямую в стек ввода mouclass — ввод **неотличим** от физического движения мышью на уровне ядра.

---

## Профили оружий

Встроены: **AK47**, **M4A1**, **AWP**  
Кастомные профили — `.json` файлы в папке `profiles/` рядом с `.exe`.

Формат:
```json
{
  "weapon": "MP5",
  "calibrated_date": "2024-01-20",
  "spray_pattern": "standard",
  "offsets": [
    {"x": 0, "y": 5, "delay": 10},
    {"x": 1, "y": 4, "delay": 10}
  ]
}
```

---

## Логи

`logs/session_YYYYMMDD_HHMMSS.jsonl` — события выстрелов и офсетов, ротация при 100MB.
