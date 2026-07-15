# План проекта DPUSB2TypeC KVM

Обновлено: 2026-07-15

## Обозначения

- `[x]` — выполнено и проверено на текущем уровне проекта.
- `[~]` — начато или выполнено частично.
- `[ ]` — не выполнено.
- `BLOCKER` — блокирует производство или следующий этап.

## Что уже сделано

### Архитектура

- [x] Определена общая архитектура KVM/док-станции.
- [x] Выбраны роли двух laptop USB-C портов: Power Source + USB Data UFP + DP Sink/UFP_D.
- [x] Выбран отдельный USB-C PD-вход питания.
- [x] Выбран STM32G474RBT6 как центральный контроллер с четырьмя аппаратными I2C.
- [x] Принята загрузка patch/config PD-контроллеров из Flash MCU без отдельных SPI flash.
- [x] Разделены I2C-шины PD-контроллеров и программируемых преобразователей.
- [x] Выбрана обновлённая архитектура питания: LMR51610Y, TPS26630, BQ25756 и AP2553.
- [x] Выбраны основные видео- и USB-компоненты.

### Схема

- [x] Создан проект EasyEDA `KVM`.
- [x] Созданы 5 листов: `MCU`, `2xUSB_PD`, `USB_DCIN`, `4xUSB2_HUB`, `DP2.1_MUX_REDRIVE`.
- [~] Добавлены STM32G474, TPS26750, TPS65988, TPS65987D, TPS26630, три BQ25756, оставшиеся старые TPS55288 и основные силовые цепи.
- [x] Добавлены USB-C/USB-A/DP-разъёмы и защита TPD4S480/TPD1E0B04.
- [x] Исправлена разводка `I2C1_SCL/SDA` TPS65988 в netlist от 2026-07-11.
- [x] Исправлен выбор выводов `I2C_SYS` STM32: PA6/PA7, pins 23/24.
- [~] Блок USB-C0 собран на TPS26750, TPD4S480 и TPS26630; startup/power-path последовательность описана, но требует финальной проверки.
- [~] Двухпортовый TPS65988 и два laptop power-converter канала собраны частично.
- [~] MCU имеет питание, reset, BOOT0, USB DFU, внешний кварц как DNP-опцию и четыре I2C-шины, но GPIO map не завершён.

### PCB и проверка

- [x] Создан документ PCB1.
- [~] На PCB перенесено 207 компонентов.
- [x] Выполнен API-аудит EasyEDA и зафиксировано текущее состояние.
- [x] Выполнено сравнение схемы, PCB и последнего netlist.
- [x] Проверены локальные даташиты и предварительная доступность критических компонентов LCSC.
- [x] Создана публичная документационная структура проекта.

## Что нужно сделать

> Снимок от 2026-07-15 является текущим. Замечания ниже, перенесённые из аудита 2026-07-11, необходимо повторно сверять с последним netlist перед исправлением.

### Этап 1. Исправить блокирующие ошибки схемы

- [ ] `BLOCKER` TPS26750 pin 20 `POWER_PATH_EN`: убрать соединение с GND; оставить floating, если функция не используется, либо реализовать рекомендованный buffer/power-path circuit.
- [ ] `BLOCKER` TPS26750 pin 21 `NC`: убрать соединение с GND.
- [ ] `BLOCKER` TPS65988 pin 36 `SPI_POCI`: при отсутствии SPI flash заменить pull-up на требуемое соединение с GND.
- [ ] Получить полный список 1 error/20 warnings schematic DRC и закрыть каждое сообщение.
- [ ] Решить назначение `BOOT0_MCU` и `NRST_MCU`: реальные кнопки/тестпады или исключение фиктивных символов из BOM/PCB.
- [ ] Проверить все NC, reserved, thermal pad и unused pins по даташитам.
- [ ] Проверить отсутствие back-power через I2C/GPIO между AON и отключаемыми доменами.

Критерий завершения: схема проходит ERC/DRC без необъяснённых ошибок, все intentional warnings документированы.

### Этап 2. Завершить незаконченные функциональные блоки

- [ ] `BLOCKER` Полностью подключить TPS65987D (`U4`): питание, CC, VBUS/PP_HV, cable power, I2C, reset, protection и policy.
- [ ] `BLOCKER` Полностью подключить TMUXHS4612 (`U2`): lanes, AUX/DDC, HPD, питание и управляющие входы.
- [ ] `BLOCKER` Полностью подключить PI2DPX2020 (`U10`): 1.8 V, decoupling, I2C/address, enable, AUX/SBU и high-speed lanes.
- [ ] Выбрать и рассчитать отдельный 1.8 V regulator от +5 V для PI2DPX2020.
- [ ] `BLOCKER` Полностью подключить USB2514B (`U9`): питание, crystal/clock, reset, RBIAS, upstream/downstream USB2, straps/SMBus и port power control.
- [ ] `BLOCKER` Полностью подключить FSUSB42 (`U14`) между laptop D+/D− и upstream USB2514B.
- [ ] Завершить USB-A и downstream USB-C цепи питания, OCS/FAULT и MCU override.
- [ ] Завершить HPD path и определить поведение при переключении.
- [ ] Завершить MCU GPIO map: enable, reset, fault, IRQ, HPD, mux select, display и encoder.

Критерий завершения: у каждого функционального узла есть питание, decoupling, управление и полный signal path; неиспользуемые выводы обработаны по даташиту.

### Этап 3. Проверить питание и USB-C PD

- [ ] Составить worst-case power tree и таблицу startup/shutdown sequencing.
- [ ] Зафиксировать допустимые входные профили и проверить весь USB-C0 тракт TPS26750/TPD4S480/TPS26630 для заявленного диапазона до 48 V.
- [ ] Проверить hardware OVP/eFuse между входом и downstream converters.
- [ ] Завершить замену оставшихся TPS55288 на BQ25756 и получить четыре одинаковых управляемых канала.
- [ ] Рассчитать каждый BQ25756: токи ключей/дросселя, shunt, ILIM, MOSFET losses, выходные конденсаторы и thermal margin.
- [ ] Проверить DC-bias всех MLCC, особенно 20 V VBUS rails.
- [ ] Рассчитать бюджет `+3V3_MCU` и `+3V3_SYS`; подтвердить запас LMR51610Y/AP2553.
- [ ] Проверить discharge, reverse current, dead-battery и fault behavior всех Type-C портов.
- [ ] Зафиксировать PDO/APDO tables и алгоритм распределения мощности между портами.

Критерий завершения: power tree работает во всех режимах 5/9/15/20/28/36 V без превышения absolute/recommended limits.

### Этап 4. Синхронизировать PCB со схемой

- [ ] `BLOCKER` Импортировать изменения из актуальной схемы в PCB.
- [ ] Добавить 33 отсутствующие позиции либо документировать осознанные исключения.
- [ ] Удалить/заменить лишние старые `U20` и `U21` после проверки, что их функция больше не нужна.
- [ ] Добиться полного совпадения schematic и PCB netlist.
- [ ] Обновить актуальный BOM после синхронизации.

Критерий завершения: EasyEDA DRC не показывает `PCB and schematic netlist does not match`.

### Этап 5. Stackup и правила high-speed

- [ ] `BLOCKER` Отказаться от текущего двухслойного stackup для финальной DP2.1 платы.
- [ ] Выбрать производимый JLCPCB stackup, ориентировочно 6 или более слоёв.
- [ ] Получить у JLCPCB геометрию controlled-impedance traces для выбранного stackup.
- [ ] Создать net classes для power, USB2, I2C/GPIO, DP AUX и high-speed lanes.
- [ ] Создать все DP/USB differential pairs с правильной полярностью.
- [ ] Задать impedance, width/gap, clearance, via и length/skew constraints.
- [ ] Зафиксировать допустимые polarity swaps и lane mapping в схеме и firmware configuration.

Критерий завершения: все high-speed сети имеют явные правила, а stackup подтверждён изготовителем.

### Этап 6. Placement и routing

- [ ] Зафиксировать размеры платы, положение разъёмов и ограничения корпуса.
- [ ] Проверить footprint/pin numbering каждого разъёма по официальному drawing и физическому образцу.
- [ ] Разместить ESD/OVP максимально близко к разъёмам.
- [ ] Разместить decoupling по рекомендациям даташитов.
- [ ] Разместить buck-boost power stages с минимальными hot loops.
- [ ] Развести DP2.1/USB high-speed channel с непрерывными return paths и минимальным числом переходов.
- [ ] Развести USB2, I2C, clocks и control signals.
- [ ] Развести power planes, GND, thermal copper и stitching vias.
- [ ] Настроить copper pours и провести repour.

Критерий завершения: 0 unrouted connections и нет нарушений PCB rules.

### Этап 7. Инженерная проверка

- [ ] Полный PCB DRC.
- [ ] Независимая schematic review.
- [ ] Signal-integrity review DP2.1, AUX/HPD и USB2.
- [ ] Power-integrity и voltage-drop review.
- [ ] Thermal review converters, MOSFET, shunts, connectors и PD controllers.
- [ ] EMC/ESD review и план pre-compliance testing.
- [ ] Mechanical/3D collision review.
- [ ] Проверка test points и factory programming/recovery interface.

Критерий завершения: все блокирующие замечания закрыты или документированно приняты.

### Этап 8. Подготовка JLCPCB PCB+PCBA

- [ ] Повторно проверить stock, MOQ и lifecycle каждого LCSC part.
- [ ] Найти производственные замены для отсутствующих TMUXHS4612 и GT-USB-7013C либо оформить consigned parts.
- [ ] Сократить количество уникальных Extended Parts там, где это не ухудшает проект.
- [ ] Экспортировать Gerber/drill и проверить Gerber viewer.
- [ ] Экспортировать BOM и CPL с совпадающими designators.
- [ ] Проверить rotations, layer и package orientation в JLCPCB viewer.
- [ ] Подготовить assembly drawing и список DNP.
- [ ] Заказать сначала малую ревизию прототипа.

Критерий завершения: JLCPCB BOM/CPL validation проходит без unmatched/conflicting parts.

### Этап 9. Bring-up и firmware

- [ ] Реализовать безопасный MCU startup, watchdog и fault logging.
- [ ] Реализовать загрузку patch/config TPS26750/TPS65988/TPS65987D с timeout/retry/CRC.
- [ ] Реализовать BQ25756 configuration и safe output sequencing по отдельным I2C-шинам.
- [ ] Реализовать PD budget manager и port policy.
- [ ] Реализовать KVM switching: USB mux, DP mux/redriver и HPD.
- [ ] Реализовать display/encoder UI.
- [ ] Подготовить factory test firmware и bring-up checklist.
- [ ] Проверить USB DFU и SWD recovery.

## Условия готовности к первому заказу

Плата может считаться готовой к первому прототипу только когда одновременно выполнены следующие условия:

- [ ] Схема завершена и проверена по даташитам.
- [ ] Schematic и PCB netlist совпадают.
- [ ] PCB имеет 0 unrouted connections.
- [ ] PCB DRC не содержит необъяснённых ошибок.
- [ ] Выбран и подтверждён многослойный controlled-impedance stackup.
- [ ] High-speed rules и пары настроены.
- [ ] BOM/CPL валидируются JLCPCB.
- [ ] Проведены power, thermal, SI, EMC/ESD и mechanical reviews.
- [ ] Есть bring-up plan, current-limited power-up и точки измерения.
