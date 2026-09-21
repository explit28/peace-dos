# Peace.DOS — Disk Operating System `МИР`

[English](#english) | [Русский](#русский)

---

## English

# Disk Operating System `МИР`

A compact disk operating system (OS) designed for Soviet and Russian retro computers based on the **i8080 / KR580VM80A** processor and compatible CPUs (i8085, Z80).

The system is written entirely in assembly language using [Pretty Intel 8080 Assembler](https://svofski.github.io/pretty-8080-assembler/). The entire OS code is optimized to fit into an **8 KB ROM**.

## 📌 Main features
* **Supported platforms:** Radio-86RK, Severnaya Palmira, Apogey, Mikrosha, Partner 01.01, and compatible systems.
* **File interface:** File and directory operations through the popular **CH376** controller.
* **Scripting and automation:** Support for scripts (command files), with a built-in interpreter and text-output template engine.
* **Hardware emulation:** Software implementation of the **I2C** protocol and an **EEPROM programmer** through the i8255 (KR580VV55) parallel port.
* **Graphics:** Viewing monochrome **WBMP** images up to 127×127 pixels.
* **Quasi-multitasking:** Execution of simple background tasks while the system is idle (during keyboard polling).

---

## 🔌 Hardware connection

### CH376 module
The CH376 file module is connected to the computer system bus according to the module pinout. The `INT` and `RST` pins do not need to be connected.

At startup, the OS automatically checks for the CH376 module and an installed storage medium:
* **Device ready:** the `AUTOEXEC.SCP` script is automatically loaded from the storage medium and executed.
* **Device absent/not ready:** the fallback script stored in ROM is executed instead (disk operations are disabled in this case).

### EEPROM programmer (`OUT` command)
The `OUT` command sends data to the port and simultaneously displays a dump of that data. Port output `PC7` emulates the write signal. The circuit is designed for programming **AT28C64**-type devices using the following connection to the VV55:

* `PA0` – `PA7` → `D0` – `D7`
* `PB0` – `PB7` → `A0` – `A7`
* `PC0` – `PC4` → `A8` – `A12`
* `PC7` → `/CE`
* `/OE` → `+5V`
* `/WE` → `GND`

### I2C interface module
The I2C protocol is implemented in software ("bit-banging") through the VV55 parallel interface. To use I2C, a simple interface module must be added to the computer circuit:

![I2C protocol for i8080](https://hub.mos.ru/dni-fx/peace-dos/-/raw/main/86RK_I2C.jpg)

*Note: The pull-up resistor values can be reduced to 4.7 kΩ.*

---

## 💻 Command reference

### Hotkeys
* `F1` — Open help (`HELP`)
* `СТР` — Clear the screen (`CLS`)
* `HOME` — Switch to video mode without line spacing
* `Up Arrow` — Repeat the last command
* `Down Arrow` — Display the current path

### File system and navigation

| Command | Description |
| :--- | :--- |
| `><name.extension>` | Quickly open a file with the specified name `ABC*`. Supported: `.TXT`, `.SCP`, `.WBM`, `.BIN`, `.RK*` |
| `/` | Set the current working path |
| `CAT` | Display the files in the root directory |
| `MCAT <name>` | Create a directory with the specified name |
| `ERASE <name.extension>` | Delete a file or directory |
| `LOAD <address> <name.extension>` | Load a file into RAM at the specified HEX address |
| `SAVE <address> <length> <name.extension>` | Save a memory region to a file (address and length are in HEX) |
| `CALL <name.extension> [L]` | Load a file into RAM and execute it. `L` enables compatibility mode |
| `VIEW <name.extension>` | Load and view a text file |
| `SCP <name.extension>` | Load and interpret a script |
| `WBMP <name.extension>` | Load and display a WBMP image |
| `HELP` | Open the help file |

### Memory and I/O port operations

| Command | Description |
| :--- | :--- |
| `CHKMEM` | Test and verify RAM |
| `SYSINFO` | Display the current system-settings table |
| `DUMP <start> <end>` | Display a HEX memory dump between the specified addresses |
| `READ <address>` | Read a byte from memory, display it on screen, and store it in the variable |
| `WRITE <address> [data...]` | Write an array of HEX data to RAM. If no data is specified, the variable value is written |
| `IN <address> <start> <end>` | Read data from PPI port A into RAM at the specified address (BC port addresses specify start/end) |
| `OUT <address> <end> <start>` | Output data from RAM to PPI port A (BC port addresses specify start/end), while simultaneously displaying a dump |
| `JUMP <address> [L]` | Unconditional jump to the specified HEX address. `L` enables compatibility mode |

### Interface, sound, and variables

| Command | Description |
| :--- | :--- |
| `CLS` | Clear the screen |
| `FLUSH <N>` | Scroll text upward by N lines |
| `CARRIAGE <YYXX>` | Set the cursor to coordinates YY (row), XX (column) |
| `TEXT <address> [!]` | Print a string from the specified HEX address. `!` outputs from the current cursor position |
| `NL` | Carriage return / new line |
| `XCG` | Switch the character generator (change font) |
| `POINTER <address>` | Set the pointer to a string/template at the specified address |
| `LEDON` / `LEDOFF` | Turn the RUS/LAT LED on / off |
| `BEEP <length> <tone>` | Sound signal (parameters in HEX) |
| `PEW <length> <tone>` | Alternative sound signal/effect (parameters in HEX) |
| `DELAY <A>` | Delay execution for A frames |
| `VAR <value>` | Directly write a HEX value to the system variable |
| `MVAR <value>` | Logical AND (masking) of the variable with the specified value |
| `IVAR` | Read a HEX parameter from the command line and store it in the variable |
| `CVAR` | Read a character parameter from the command line and store it in the variable |
| `KEYSCAN` | Wait for a key press and store its code in the variable |
| `IF <value> <command>` | Compare the variable with the specified value. If they match, execute the command |
| `TASK <address>` | Set the address of a background task executed while the system is idle |

### I2C bus control

| Command | Description |
| :--- | :--- |
| `I2CSTART <address>` | Generate a start condition and select an I2C device by HEX address |
| `I2CSTOP` | Generate a stop condition and place the bus in the idle state |
| `I2CTX [data...]` | Transmit data. If no argument is given, the variable value is transmitted |
| `I2CRX [!]` | Receive data into the variable. `!` receives without acknowledgement (NACK) |

### System utilities

| Command | Description |
| :--- | :--- |
| `MONITOR` | Exit the OS to the computer's standard Monitor control program |

---

## 🛠 Programming and OS integration

### Calling OS commands from external programs
External software can use the OS command parser. Place the address of a string containing the command into the `HL` register pair (the string must end with byte `$00`), then execute a `CALL` to the entry point `OS address + 3`.

Example for the Radio-86RK:

```assembly
ORIGIN          equ $0000
OS              equ $E000

ORG ORIGIN
                LXI H, CMD_TITLE  ; Load the command address
                CALL OS + 3       ; Call the OS interpreter
                RET

CMD_TITLE:      db 'TEXT 1000', $00; Call the TEXT command for address $1000

ORG $1000
TXT_TITLE:      db 'HELLO WORLD!!!', $0A, $0D, $00
```

*Interesting feature:* When the `CALL` command is invoked, the current value of the OS internal tick counter is automatically placed in processor register `A`. This is convenient as a seed for pseudo-random number generators (PRNGs).

### Scripts
If a command in an `.SCP` script is preceded by the `@` character, console echo is disabled for that line.

The OS supports **one system variable**. The results of the `READ`, `KEYSCAN`, and `I2CRX` commands are stored in it. The `IF` command compares against this variable, while `WRITE` and `I2CTX`, when called without parameters, use the value stored in it. For example, `WRITE 0000` writes the byte from the variable to address `$0000`.

### Background tasks
Simple background routines are executed automatically during the keyboard-polling loop. If the user does not press a key, the OS continuously executes a `CALL` to the address previously set by the `TASK` command.

---

## 📊 Text template engine

To save memory and make structured data (for example, tables or lists) easier to display, the OS includes a template engine. A template text string must always end with byte `$00`.

### General-purpose control codes
* `$09` — Tab (fixed width, 8 characters)
* `$0A` — Line feed (LF)
* `$0D` — Carriage return (CR)
* `$80` — Take the following byte and display it as HEX
* `$81` — Take the following word and display it as DEC
* `$DF` — Interactive pause (wait for any key press)

### Dynamic macros (used together with the address in `POINTER`)
The construct reads data from memory using offsets relative to the address set by the `POINTER` command.

* `$F0`–`$FF` — Display an ASCII character from the structure at offset `0`..`F`
* `$E0`–`$EF` — Display the HEX value of a byte at offset `0`..`F`
* `$D0`–`$D7` — Display the DEC value of a word at offset `0`..`8`
* `$DB` — Display the value at address `POINTER` in binary format (BIN)
* `$D8` — Display the `POINTER` address itself in HEX format

**Example template for displaying a file-directory entry:**

```assembly
; File name (8+3 bytes), space, attribute (HEX), cluster (HEX), length (DEC), newline
dir_item_tpl: db $F0, $F1, $F2, $F3, $F4, $F5, $F6, $F7, " ", $F8, $F9, $FA, " ", $EB, " ", $ED, $EC, " ", $D7, $0A, $0D, $00*
```

To iterate through entries (for example, database records or files), simply change `POINTER` to the next structure and send the template for output again.

---

## Русский

# Дисковая операционная система «МИР»

Компактная дисковая операционная система (ОС), разработанная для отечественных ретро-ЭВМ на базе процессора **i8080 / КР580ВМ80А** и совместимых с ними (i8085, Z80). 

Система полностью написана на ассемблере в среде [Прекрасный Ассемблер (Pretty Intel 8080 Assembler)](https://svofski.github.io/pretty-8080-assembler/). Весь код ОС оптимизирован и помещается в **ПЗУ объёмом 8 КБ**.

## 📌 Основные возможности
* **Поддерживаемые платформы:** Радио-86РК, Северная Пальмира, Апогей, Микроша, Партнёр 01.01 и аналоги.
* **Файловый интерфейс:** Работа с файлами и каталогами через популярный контроллер **CH376**.
* **Скриптинг и автоматизация:** Поддержка выполнения сценариев (командных файлов), встроенный интерпретатор и шаблонизатор текстового вывода.
* **Аппаратная эмуляция:** Программная реализация протокола **I2C** и программатора **EEPROM** через параллельный порт i8255 (КР580ВВ55).
* **Графика:** Просмотр монохромных изображений формата **WBMP** размером до 127×127 пикселей.
* **Квази-многозадачность:** Выполнение простых фоновых задач во время простоя (опроса клавиатуры).

---

## 🔌 Подключение оборудования

### Модуль CH376
Файловый модуль CH376 подключается к системной шине ЭВМ согласно распиновке модуля. Выводы `INT` и `RST` подключать не нужно.

При запуске ОС автоматически определяет наличие модуля CH376 и установленного носителя информации:
* **Устройство готово:** с носителя автоматически загружается и выполняется сценарий `AUTOEXEC.SCP`.
* **Устройство отсутствует/не готово:** выполняется резервный сценарий из ПЗУ (в этом случае дисковые операции блокируются).

### Программатор EEPROM (Команда OUT)
Команда `OUT` выдает данные в порт и одновременно отображает их дамп. Выход порта `PC7` эмулирует сигнал записи. Схема разработана для прошивки микросхем типа **AT28C64** со следующим подключением к ВВ55:
* `PA0` – `PA7` → `D0` – `D7`
* `PB0` – `PB7` → `A0` – `A7`
* `PC0` – `PC4` → `A8` – `A12`
* `PC7` → `/CE`
* `/OE` → `+5V`
* `/WE` → `GND`

### Модуль сопряжения I2C
Протокол I2C реализован программным методом («ногодрыгом») через параллельный интерфейс ВВ55. Для работы требуется добавить в схему ЭВМ простейший модуль сопряжения:

![Протокол I2C для i8080](https://hub.mos.ru/dni-fx/peace-dos/-/raw/main/86RK_I2C.jpg)
*Примечание: Сопротивление подтягивающих резисторов можно уменьшить до 4.7 кОм.*

---

## 💻 Справочник команд

### Горячие клавиши
* `Ф1` — Вызов справки (`HELP`)
* `СТР` — Очистка экрана (`CLS`)
* `HOME` — Переключение в видеорежим без межстрочных интервалов
* `Стрелка вверх` — Повтор последней команды
* `Стрелка вниз` — Вывод текущего пути

### Файловая система и навигация

| Команда | Описание |
| :--- | :--- |
| `><имя.расширение>` | Быстрое открытие файла с указанным именем `ABC*`. Поддерживаются: `.TXT`, `.SCP`, `.WBM`, `.BIN`, `.RK*` |
| `/` | Установка текущего рабочего пути |
| `CAT` | Просмотр каталога файлов корневой директории |
| `MCAT <имя>` | Создание каталога с указанным именем |
| `ERASE <имя.расширение>` | Удаление файла или каталога |
| `LOAD <адрес> <имя.расширение>` | Загрузка файла в ОЗУ по указанному HEX-адресу |
| `SAVE <адрес> <длина> <имя.расширение>` | Сохранение области памяти в файл (адрес и длина в HEX) |
| `CALL <имя.расширение> [L]` | Загрузка файла в ОЗУ и его запуск. Ключ `L` — режим совместимости |
| `VIEW <имя.расширение>` | Загрузка и просмотр текстового файла |
| `SCP <имя.расширение>` | Загрузка и интерпретация сценария (скрипта) |
| `WBMP <имя.расширение>` | Загрузка и отображение картинки формата WBMP |
| `HELP` | Вызов файла справки |

### Работа с памятью и портами ввода-вывода

| Команда | Описание |
| :--- | :--- |
| `CHKMEM` | Тестирование и проверка оперативной памяти |
| `SYSINFO` | Вывод таблицы текущих системных настроек |
| `DUMP <старт> <конец>` | Просмотр дампа памяти в HEX-формате между адресами |
| `READ <адрес>` | Чтение байта из ячейки памяти, вывод на экран и запись в переменную |
| `WRITE <адрес> [данные...]` | Запись массива HEX-данных в ОЗУ. Если данные не указаны, запишется значение переменной |
| `IN <адрес> <начало> <конец>` | Чтение данных из ППА порт A (адреса на портах BC начало/конец) в ОЗУ по указанному адресу |
| `OUT <адрес> <конец> <начало>`| Вывод данных в ППА порт A из ОЗУ (адреса на портах BC начало/конец) с одновременным выводом дампа |
| `JUMP <адрес> [L]` | Безусловный переход по HEX-адресу. Ключ `L` — режим совместимости |

### Интерфейс, звук и переменные

| Команда | Описание |
| :--- | :--- |
| `CLS` | Очистка экрана |
| `FLUSH <N>` | Прокрутка (скролл) текста вверх на N строк |
| `CARRIAGE <YYXX>` | Установка каретки в координаты YY (строка), XX (столбец) |
| `TEXT <адрес> [!]` | Печать строки с HEX-адреса. Ключ `!` — вывод с текущей позиции курсора |
| `NL` | Перевод каретки (новая строка) |
| `XCG` | Переключение знакогенератора (смена шрифта) |
| `POINTER <адрес>` | Установка указателя на строку/шаблон по адресу |
| `LEDON` / `LEDOFF` | Включение / выключение светодиода РУС/ЛАТ |
| `BEEP <длина> <тон>` | Звуковой сигнал (параметры в HEX) |
| `PEW <длина> <тон>` | Альтернативный звуковой сигнал (эффект, параметры в HEX) |
| `DELAY <A>` | Задержка выполнения на A кадров |
| `VAR <значение>` | Прямая запись HEX-значения в системную переменную |
| `MVAR <значение>` | Логическое И (маскирование) переменной с указанным значением |
| `IVAR` | Чтение HEX-параметра из командной строки и его запись в переменную |
| `CVAR` | Чтение символьного параметра из командной строки и его запись в переменную |
| `KEYSCAN` | Ожидание нажатия клавиши и запись её кода в переменную |
| `IF <значение> <команда>` | Сравнение переменной со значением. При совпадении выполняется команда |
| `TASK <адрес>` | Задание адреса фоновой задачи, выполняемой во время простоя |

### Управление шиной I2C

| Команда | Описание |
| :--- | :--- |
| `I2CSTART <адрес>` | Старт-условие и выбор I2C-устройства по HEX-адресу |
| `I2CSTOP` | Стоп-условие, перевод линии передачи в режим ожидания |
| `I2CTX [данные...]` | Передача данных. Если аргумент пуст, передается значение переменной |
| `I2CRX [!]` | Прием данных в переменную. Ключ `!` — прием без подтверждения (NACK) |

### Системные утилиты

| Команда | Описание |
| :--- | :--- |
| `MONITOR` | Выход из ОС в штатную управляющую программу «Монитор» ЭВМ |

---

## 🛠 Программирование и интеграция с ОС

### Вызов команд ОС из внешних программ
Внешнее ПО может использовать парсер команд ОС. Для этого в регистровую пару `HL` передается адрес строки с командой (строка должна оканчиваться байтом `$00`), после чего выполняется `CALL` на точку входа `Адрес ОС + 3`.

Пример для ЭВМ Радио-86РК:
```assembly
ORIGIN          equ $0000
OS              equ $E000

ORG ORIGIN
                LXI H, CMD_TITLE  ; Загружаем адрес команды
                CALL OS + 3       ; Вызываем интерпретатор ОС
                RET

CMD_TITLE:      db 'TEXT 1000', $00; Вызвать команду TEXT для адреса $1000

ORG $1000
TXT_TITLE:      db 'HELLO WORLD!!!', $0A, $0D, $00
```
*Интересная особенность:* При вызове команды `CALL` в регистр процессора `A` автоматически записывается текущее значение внутреннего счетчика тактов ОС. Это удобно использовать в качестве сида (seed) для генераторов псевдослучайных чисел (ГПСЧ).

### Сценарии (Скрипты)
Если в `.SCP` сценарии перед командой указан символ `@`, то для этой строки отключается вывод эха в консоль.

ОС поддерживает **одну системную переменную**. В неё пишутся результаты команд `READ`, `KEYSCAN` и `I2CRX`. Команда `IF` сравнивает значение этой переменной, а команды `WRITE` и `I2CTX`, вызванные без параметров, берут данные из неё. К примеру, `WRITE 0000` запишет байт из переменной в адрес `$0000`.

### Фоновые задачи
Выполнение простых фоновых подпрограмм происходит автоматически в цикле опроса клавиатуры. Если пользователь не нажимает клавиши, ОС непрерывно выполняет инструкцию `CALL` по адресу, предварительно заданному командой `TASK`.

---

## 📊 Текстовый шаблонизатор

Для экономии памяти и удобного вывода структурированных данных (например, таблиц или списков) в ОС встроен шаблонизатор. Текстовая строка шаблона всегда должна завершаться байтом `$00`.

### Управляющие символы общего назначения:
* `$09` — Табуляция (фиксированная, 8 символов)
* `$0A` — Перевод строки (LF)
* `$0D` — Возврат каретки (CR)
* `$80` — Взять следующий за этим кодом байт и вывести как HEX
* `$81` — Взять следующее за этим кодом слово и вывести как DEC
* `$DF` — Интерактивная пауза (ожидание нажатия любой клавиши)

### Динамические макросы (работают в связке с адресом из `POINTER`):
Конструкция берет данные из памяти со смещением относительно адреса, заданного командой `POINTER`.
* `$F0`–`$FF` — Вывод ASCII-символа из структуры со смещением `0`..`F`
* `$E0`–`$EF` — Вывод HEX-значения байта со смещением `0`  ..`F`
* `$D0`–`$D7` — Вывод DEC-значения слова со смещением `0`..`8`
* `$DB` — Вывод значения по адресу `POINTER` в бинарном виде (BIN)
* `$D8` — Вывод самого адреса `POINTER` в формате HEX

**Пример шаблона для вывода строки каталога файлов:**
```assembly
; Имя файла (8+3 байта), пробел, атрибут (HEX), кластер (HEX), длина (DEC), перенос строки
dir_item_tpl: db $F0, $F1, $F2, $F3, $F4, $F5, $F6, $F7, " ", $F8, $F9, $FA, " ", $EB, " ", $ED, $EC, " ", $D7, $0A, $0D, $00*
```

Для перебора элементов (базы данных или списка файлов) достаточно изменять значение `POINTER` на следующую структуру и заново отправлять шаблон на печать.
