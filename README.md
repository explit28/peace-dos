# Peace.DOS — Disk Operating System `МИР`

[English](#english) | [Русский](#русский)

---

## English

# Disk Operating System `МИР`

The operating system (OS) is designed for computers based on the i8080 processor (Radio-86RK, Severnaya Palmira, Apogey, Mikrosha, Partner 01.01) and compatible systems using processors such as the KR580VM80A, i8085, Z80, and others. It fits into an 8 KB ROM and provides a minimal set of commands for working with the CH376 file interface.<br>
The system includes an interpreter and a template engine, supports script execution, implements the I2C protocol in software through the i8255 (KR580VV55) parallel port, can display WBMP graphics up to 127×127 pixels, and supports working with files and directories.<br>

The CH376 file module is connected to the computer's system bus according to the module pinout. The INT and RST pins do not need to be connected.<br>
At startup, the OS checks for the presence of the CH376 module and storage media connected to it. If the device is ready, the `AUTOEXEC.SCP` script is loaded from the storage medium and executed. If the device is unavailable or not ready, the script stored in ROM is executed instead. In that case, disk operations are unavailable.

The operating system is written entirely in assembly language using Pretty Intel 8080 Assembler: https://svofski.github.io/pretty-8080-assembler/


## Command list

`CHKMEM` - memory test<br/>
`SYSINFO` - display the system settings table<br/>

`CLS` - clear the screen<br/>
`FLUSH N` - scroll text upward by N lines<br/>
`CARRIAGE YYXX` - set the cursor position<br/>
`TEXT ХХХХ` - print a string starting at address ХХХХ<br/>
`NL` - move to a new line<br/>
`XCG` - switch the character generator<br/>
`POINTER XXXX` - set the string pointer to address XXXX<br/>

`LEDON` - turn on the RUS/LAT LED<br/>
`LEDOFF` - turn off the RUS/LAT LED<br/>

`BEEP NNMM` - sound signal, where NN is the duration and MM is the tone<br/>
`PEW NNMM` - sound signal, where NN is the duration and MM is the tone<br/>
`DELAY A` - delay for A frames<br/>

`JUMP ХХХХ` - unconditional jump to address ХХХХ; key `L` enables compatibility mode<br/>
`READ ХХХХ` - read a byte from memory address ХХХХ and display it on screen<br/>
`WRITE XXXX AA BB CC ...` - write a data array to RAM starting at address XXXX<br/>
`IN XXXX YYYY ZZZZ` - read data from a port into RAM starting at address XXXX; port addresses range from YYYY to ZZZZ<br/>
`OUT XXXX YYYY ZZZZ` - output data from RAM addresses XXXX through YYYY to the port; the address starting from ZZZZ is output on PB and PC<br/>
`DUMP AAAA BBBB` - display memory from address AAAA through address BBBB<br/>

`/` - set the current path

`CAT` - list files in the root directory<br/>
`MCAT ABCDE` - create a directory named ABCDE<br/>
`ERASE ABCDE` - delete a file or directory named ABCDE<br/>
`HELP` - open the help file (`F1` key)<br/>
`LOAD XXXX ABC*` - load file ABC* into RAM starting at address XXXX<br/>
`SAVE XXXX YYYY ABC*` - save file ABC* starting at address XXXX with length YYYY<br/>
`CALL ABC*` - load file ABC* into RAM and execute it; key `L` enables compatibility mode<br/>
`VIEW ABC*` - load text file ABC* into RAM and display it<br/>
`SCP ABC*` - load and interpret script ABC*<br/>

`WBMP ABC*` - load and display an image named ABC*<br/>

`VAR X` - store value X in the variable<br/>
`MVAR X` - logical AND with X (masking)<br/>
`IVAR` - read a HEX command-line parameter and store it in the variable<br/>
`CVAR` - read a character command-line parameter and store it in the variable<br/>
`KEYSCAN` - wait for a key press and display the key code<br/>
`IF A B` - compare the variable with A and execute command B if they match<br/>

`I2CSTART AA` - select a device with address AA<br/>
`I2CSTOP` - put the communication line into the idle state<br/>
`I2CTX AA BB CC DD...` - transmit data<br/>
`I2CRX` - receive data; key `!` disables receive acknowledgement<br/>

`TASK XXXX` - execute a task at address XXXX while the system is idle

`MONITOR` - exit to the Monitor control program<br/>

Keys: `F1` - help, `СТР` - clear the screen, `HOME` - video mode without line spacing, `Up Arrow` - last command, `Down Arrow` - current path

## Features

If a command in a script is preceded by the `@` character, command echo is disabled.

When the `CALL` command is invoked, the value of the OS internal counter is placed in processor register A. This is convenient for initializing pseudo-random number generators.

The `OUT` command sends data to the port and simultaneously displays a dump of that data. Port output PC7 emulates the write signal for an external memory IC. The command was designed for programming AT28C64-type memory chips. Connect the chip to the port as follows:

PA0 - PA7 → D0 - D7<br/>
PB0 - PB7 → A0 - A7<br/>
PC0 - PC4 → A8 - A12<br/>
PC7 → /CE<br/>
/OE → +5V<br/>
/WE → GND<br/>

The I2C protocol is implemented by bit-banging the VV55 parallel port. To allow the OS to communicate over I2C, an interface module must be added to the computer circuit:

![I2C protocol for i8080](https://hub.mos.ru/dni-fx/peace-dos/-/raw/main/86RK_I2C.jpg)

The pull-up resistor values can be reduced to 4.7 kΩ.

## Programming in the OS environment

**External programs can execute OS commands.** To do this, place the address of a string containing the command into the HL register pair. Then call `CALL <OS load address> + 3`. For example, for the Radio-86RK:

```asm
ORIGIN          equ $0000
OS              equ $E000

ORG ORIGIN

LXI H, CMD_TITLE
CALL OS + 3
RET

CMD_TITLE:      db 'TEXT 1000', $00;
ORG $1000
TXT_TITLE:      db 'HELLO WORLD!!!', $0A, $0D, $00
```

This allows programs to load files and perform other operations.

**The system includes a template engine for working with text.** This mechanism reduces the length of strings and makes structured-data output more convenient. A text string must always end with the `$00` character. Template control characters:

`$09` - tab, 8 characters<br/>
`$0A` - line feed<br/>
`$0D` - carriage return<br/>
`$80` - output the next byte in HEX format<br/>
`$81` - output the next word in DEC format<br/>
`$DF` - wait for a key press<br/>

The following control characters are used together with `POINTER`, which points to an array of structured data. Such a construction is called a template.

`$F0-$FF` - output the character at POINTER + offset 0..F<br/>
`$E0-EF` - output the HEX value of the byte at POINTER + offset 0..F<br/>
`$DB` - output the value at address POINTER in BIN format<br/>
`$D8` - output POINTER in HEX format<br/>
`$D0-D7` - output the DEC value of the word at POINTER + offset 0..8<br/>

For example, the template for displaying a file directory is the following string:

```asm
db $F0, $F1, $F2, $F3, $F4, $F5, $F6, $F7, " ", $F8, $F9, $FA, " ", $EB, " ", $ED, $EC, " ", $D7, $0A, $0D, $00
```

The first 12 bytes output the file name with extension, followed by the file attribute (HEX), the file's starting cluster (HEX), the file length (DEC), and the end-of-line characters.

To display the next line of the file directory, simply move `POINTER` to the required entry and output the template again. This approach can also be used to display records from a structured database. The OS uses the same mechanism for memory-dump templates and for displaying the lengths of loaded files.

**Scripts have exactly one variable.** The results of `READ`, `KEYSCAN`, and `I2CRX` operations are stored in this variable. The `IF` operator compares against this variable. If `WRITE` or `I2CTX` is used without an argument, the current variable value is used as the argument. For example, `WRITE 0` writes the contents of the variable to memory address zero.

**Simple background tasks are executed while the keyboard is being polled.** If no keyboard key is pressed, a `CALL` is made to the address specified by the `TASK` command.


---

## Русский

# Дисковая операционная система `МИР`

Операционная система (ОС) разработана для ЭВМ с процессором i8080 (Радио-86РК, Северная Пальмира, Апогей, Микроша, Партнёр 01.01) и аналогичных: КР580ВМ80А, i8085, Z80 и другие. Помещается в ПЗУ объёмом 8 кб. Обеспечивает минимальный набор команд для работы с файловым интерфейсом CH376.<br>
Система имеет интерпретатор, шаблонизатор, поддерживает выполнение сценариев, программно эмулирует протокол I2C через параллельный порт i8255 (КР580ВВ55), позволяет просматривать графику формата WBMP в размере до 127х127 пикселей, поддерживает работу с файлами и каталогами.<br>

Файловый модуль CH376 подключается в системную шину ЭВМ согласно распиновки модуля. Выводы INT и RST подключать не нужно.<br>
При запуске ОС определит наличие модуля CH376 и носителя информации в нём. Если устройство готово к работе, с носителя информации будет загружен и выполнен сценарий AUTOEXEC.SCP. Если устройство не готово к работе или отсутствует, будет выполнен сценарий из ПЗУ. В этом случае дисковые операции будут недоступны.

Операционная система полность написана на ассемблере в среде Прекрасный Ассемблер (Pretty Intel 8080 Assembler): https://svofski.github.io/pretty-8080-assembler/


## Список команд:

`CHKMEM` - проверка памяти<br/>
`SYSINFO` - вывод таблицы системных настроек<br/>

`CLS` - очистка экрана<br/>
`FLUSH N` - скролл текста вверх на N строк<br/>
`CARRIAGE YYXX` - установка каретки<br/>
`TEXT ХХХХ` - печать строки с адреса ХХХХ<br/>
`NL` - перевод каретки<br/>
`XCG` - переключение знакогонератора<br/>
`POINTER XXXX` - установка указателя на строку по адресу XXXX<br/>

`LEDON` - включить светодиод РУС/ЛАТ<br/>
`LEDOFF` - выключить светодиод РУС/ЛАТ<br/>

`BEEP NNMM` - звуковой сигнал, где NN - длительность, MM - тон<br/>
`PEW NNMM` - звуковой сигнал, где NN - длительность, MM - тон<br/>
`DELAY A` - задержка на A кадров<br/>

`JUMP ХХХХ` - безусловный переход на адрес ХХХХ, ключ 'L' - режим совместимости<br/>
`READ ХХХХ` - чтение байта из ячейки памяти ХХХХ и вывод на экран<br/>
`WRITE XXXX AA BB CC ...` - запись массива данных в ОЗУ с адреса XXXX<br/>
`IN XXXX YYYY ZZZZ` - чтение данных из порта в ОЗУ с адреса XXXX, адреса на порту с YYYY по ZZZZ<br/>
`OUT XXXX YYYY ZZZZ` - вывод данных в порт из ОЗУ с адреса XXXX по YYYY на PB и PC выводится адрес начиная с ZZZZ<br/>
`DUMP AAAA BBBB` - просмотр памяти с адреса AAAA по адрес BBBB<br/>

`/` - установка текущего пути

`CAT` - каталог файлов корневой директории<br/>
`MCAT ABCDE` - создание каталога с именем ABCDE<br/>
`ERASE ABCDE` - удаление файла или каталога каталога с именем ABCDE<br/>
`HELP` - вызов файла справки (клавиша Ф1)<br/>
`LOAD XXXX ABC*` - загрузка файла ABC* в ОЗУ с адреса XXXX<br/>
`SAVE XXXX YYYY ABC*` - сохранение файла ABC* с адреса XXXX и длиной YYYY<br/>
`CALL ABC*` - загрузка файла ABC* в ОЗУ и вызов, ключ 'L' - режим совместимости<br/>
`VIEW ABC*` - загрузка текстового файла ABC* в ОЗУ и просмотр<br/>
`SCP ABC*` - загрузка и интерпретация сценария ABC*<br/>

`WBMP ABC*` - загрузка и отображение картинки с именем ABC*<br/>

`VAR X` - запись значения X в переменную<br/>
`MVAR X` - логическое И с X (маскирование)<br/>
`IVAR` - чтение HEX параметра командной строки и запись в переменную<br/>
`CVAR` - чтение символьного параметра командной строки и запись в переменную<br/>
`KEYSCAN` - Ожидание нажатия клавиши и вывод кода клавиши<br/>
`IF A B` - сравнение переменной с A и выполнение команды B при условии совпадения<br/>

`I2CSTART AA` - выбрать устройство с адресом AA<br/>
`I2CSTOP` - перевод линии передачи в режим ожидания<br/>
`I2CTX AA BB CC DD...` - передача данных<br/>
`I2CRX` - приём данных, ключ '!' - без подтверждения приёма<br/>

`TASK XXXX` - выполнение задачи по адресу XXXX во время простоя

`MONITOR` - выход в управляющую программу Монитор<br/>

Клавиши: `Ф1` - помощь, `СТР` - очистка экрана, `HOME` - видеорежим без межстрочных интервалов, `стрелка вверх` - последняя команда, `стрелка вниз` - текущий путь

## Особенности

Если в сценарии перед командой стоит символ @, то эхо отключается.

При вызое команды CALL в регистр процессора A записывается значение внутреннего счётчика ОС, это удобно для выставления начальных значений генераторов псевдослучайных чисел.

Команда OUT выдаёт данные в порт и одновременно выводит дамп этих данных. Выход порта PC7 эмулирует сигнал записи во внешнюю микросхему памяти. Команда разработана для прошивки микросхем типа AT28C64. Подключение микросхемы к порту:

PA0 - PA7 → D0 - D7<br/>
PB0 - PB7 → A0 - A7<br/>
PC0 - PC4 → A8 - A12<br/>
PC7 → /CE<br/>
/OE → +5V<br/>
/WE → GND<br/>

Протокол I2C реализован ногодрыгом через параллельный порт ВВ55. Для того, чтобы ОС могла работать по протоколу I2C, необходимо в схему ЭВМ добавить модуль сопряжения:

![Протокол I2C для i8080](https://hub.mos.ru/dni-fx/peace-dos/-/raw/main/86RK_I2C.jpg)

Сопротивление подтягивающих резисторов можно уменьшить до 4.7 кОм.

## Программирование в среде ОС

**Внешние программы могут выполнять команды ОС.** Для этого в регистровую пару HL нужно поместить адрес строки, в которой написана команда. Далее нужно сделать вызов CALL <Адрес размещения ОС> + 3, например для ЭВМ Радио-86РК:<br/>

```asm
ORIGIN          equ $0000
OS              equ $E000

ORG ORIGIN

LXI H, CMD_TITLE
CALL OS + 3
RET

CMD_TITLE:      db 'TEXT 1000', $00;
ORG $1000
TXT_TITLE:      db 'HELLO WORLD!!!', $0A, $0D, $00
```

Таким образом программы могут загружать файлы и выполнять иные операции.

**Для работы с текстами в системе предусмотрен шаблонизатор**. Данный механизм позволяет экономить длину строк, и делает вывод структурированных данных удобным. Текстовая строка всегда должна заканчиваться символом $00. Управляющие символы шаблонизатора:

`$09` - табуляция, 8 символов<br/>
`$0A` - перевод строки<br/>
`$0D` - возврат каретки<br/>
`$80` - вывод следующего байта в формате HEX<br/>
`$81` - вывод следующего слова в формате DEC<br/>
`$DF` - ожидание нажатия клавиши<br/>

Следующие управляющие символы используются вместе с указателем на массив структурированных данных - POINTER. Такая конструкция называется шаблон.

`$F0-$FF` - вывод символа с позиции POINTER + смещение 0..F<br/>
`$E0-EF` - вывод HEX значения байта с позиции POINTER + смещение 0..F<br/>
`$DB` - вывод значения по адресу POINTER в BIN<br/>
`$D8` - вывод POINTER в формате HEX<br/>
`$D0-D7` - вывод DEC значения слова с позиции POINTER + смещение 0..8<br/>

Таким образом, шаблон для вывода каталога файлов будет выглядеть, как строка:

```asm
db $F0, $F1, $F2, $F3, $F4, $F5, $F6, $F7, " ", $F8, $F9, $FA, " ", $EB, " ", $ED, $EC, " ", $D7, $0A, $0D, $00
```

— где первые 12 байт выводят имя файла с расширением, далее атрибут файла (HEX), начальный кластер файла (HEX), длина файла (DEC) и конец строки.

Чтобы вывести на экран следующую строку каталога файлов, достаточно переместить POINTER на нужную запись и снова вызвать вывод шаблона. Так можно делать вывод записей структурированной базы данных. Аналогично в ОС реализованы шаблоны вывода дампа и вывода длины загружаемых файлов.

**Для работы в сценариях есть одна единственная переменная.** В эту переменную записываются результаты операций READ, KEYSCAN и I2CRX. Оператор IF делает сравнение с этой переменной. Если использовать команды WRITE и I2CTX без аргумента, то аргументом становится значение переменной. Например, команда WRITE 0 запишет в нулевую ячейку памяти содержимое переменной.

**Выполнение простых фоновых задач происходит во время опроса клавиатуры.** Если клавиши клавиатуры не нажимаются, происходит вызов CALL по адресу заданному командой TASK.

![Программирование в среде ОС МИР](https://hub.mos.ru/dni-fx/peace-dos/-/raw/main/love.jpg)
