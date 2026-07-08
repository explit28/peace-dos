Fork of the original project by Dmitry Ivanov https://hub.mos.ru/dni-fx/peace-dos

# MIR Disk Operating System

The operating system (OS) was developed for computers with the i8080 processor, such as Radio-86RK, Severnaya Palmira, Apogey, Mikrosha, Partner 01.01, and similar systems: KR580VM80A, i8085, Z80, and others. It fits into an 8 KB ROM. It provides a minimal command set for working with the CH376 file interface.

The system includes an interpreter and a template engine, supports script execution, emulates the I2C protocol in software through the i8255 parallel port (KR580VV55), allows viewing WBMP graphics up to 127×127 pixels, and supports working with files and directories.

The CH376 file module is connected to the computer system bus according to the module pinout. The INT and RST pins do not need to be connected.

At startup, the OS detects whether the CH376 module and a storage medium are present. If the device is ready, the script `AUTOEXEC.SCP` will be loaded from the storage medium and executed. If the device is not ready or is absent, the script from ROM will be executed. In that case, disk operations will not be available.

The operating system is written entirely in assembly language using Pretty Intel 8080 Assembler:
https://svofski.github.io/pretty-8080-assembler/

## Command list

`CHKMEM` — memory test
`SYSINFO` — display the system settings table

`CLS` — clear the screen
`FLUSH N` — scroll text upward by N lines
`CARRIAGE YYXX` — set the cursor position
`TEXT XXXX` — print a string starting from address XXXX
`NL` — carriage return / new line
`XCG` — switch character generator
`POINTER XXXX` — set the pointer to a string at address XXXX

`LEDON` — turn on the RUS/LAT LED
`LEDOFF` — turn off the RUS/LAT LED

`BEEP NNMM` — sound signal, where NN is duration and MM is tone
`PEW NNMM` — sound signal, where NN is duration and MM is tone
`DELAY A` — delay for A frames

`JUMP XXXX` — unconditional jump to address XXXX; key `L` enables compatibility mode
`READ XXXX` — read a byte from memory cell XXXX and display it on the screen
`WRITE XXXX AA BB CC ...` — write an array of data to RAM starting at address XXXX
`IN XXXX YYYY ZZZZ` — read data from a port into RAM starting at address XXXX; port addresses from YYYY to ZZZZ
`OUT XXXX YYYY ZZZZ` — output data from RAM to a port, from address XXXX to YYYY; the address starting from ZZZZ is output on PB and PC
`DUMP AAAA BBBB` — view memory from address AAAA to address BBBB

`/` — set the current path

`CAT` — list files in the root directory
`MCAT ABCDE` — create a directory named ABCDE
`ERASE ABCDE` — delete a file or directory named ABCDE
`HELP` — open the help file, F1 key
`LOAD XXXX ABC*` — load file ABC* into RAM at address XXXX
`SAVE XXXX YYYY ABC*` — save file ABC* from address XXXX with length YYYY
`CALL ABC*` — load file ABC* into RAM and call it; key `L` enables compatibility mode
`VIEW ABC*` — load text file ABC* into RAM and view it
`SCP ABC*` — load and interpret script ABC*

`WBMP ABC*` — load and display an image named ABC*

`PVAR` — output the character from the variable to the screen
`KEYSCAN` — wait for a key press and output the key code
`IF A B` — compare the variable with A and execute command B if they match

`I2CSTART AA` — select a device with address AA
`I2CSTOP` — switch the transmission line to idle mode
`I2CTX AA BB CC DD...` — transmit data
`I2CRX` — receive data; key `!` means receive without acknowledgement

`MONITOR` — exit to the Monitor control program

## Keys

F1 — help
СТР — clear screen
HOME — video mode without line spacing
Up arrow — previous command
Down arrow — current path

## Special features

If a command in a script is preceded by the `@` symbol, echo is disabled.

The `OUT` command outputs data to the port and simultaneously displays a dump of that data. The PC7 port output emulates the write signal for an external memory chip. The command was designed for programming chips such as the AT28C64.

Chip connection to the port:

```text
PA0 - PA7 → D0 - D7
PB0 - PB7 → A0 - A7
PC0 - PC4 → A8 - A12
PC7 → /CE
/OE → +5V
/WE → GND
```

## Programming in the OS environment

External programs can execute OS commands. To do this, the HL register pair must contain the address of the string containing the command. Then call:

```asm
CALL <OS location address> + 3
```

For example, for the Radio-86RK computer:

```asm
ORIGIN equ $0000
OS equ $E000

ORG ORIGIN

LXI H, CMD_TITLE
CALL OS + 3
RET

CMD_TITLE: db 'TEXT 1000', $00;

ORG $1000
TXT_TITLE: db 'HELLO WORLD!!!', $0A, $0D, $00
```

Thus, programs can load files and perform other operations.

For working with text, the system includes a template engine. This mechanism allows reducing string length and makes output of structured data convenient. A text string must always end with the `$00` character.

Template engine control characters:

```text
$09 — tabulation, 8 characters
$0A — line feed
$0D — carriage return
$80 — output the next byte in HEX format
$81 — output the next word in DEC format
$DF — wait for a key press
```

The following control characters are used together with the pointer to an array of structured data — `POINTER`. Such a construction is called a template.

```text
$F0-$FF — output a character from POINTER + offset 0..F
$E0-$EF — output the HEX value of a byte from POINTER + offset 0..F
$DB — output the value at address POINTER in BIN format
$D8 — output POINTER in HEX format
$D0-$D7 — output the DEC value of a word from POINTER + offset 0..8
```

Thus, a template for displaying a file directory will look like this string:

```asm
db $F0, $F1, $F2, $F3, $F4, $F5, $F6, $F7, " ", $F8, $F9, $FA, " ", $EB, " ", $ED, $EC, " ", $D7, $0A, $0D, $00
```

Here, the first 12 bytes output the filename with extension, followed by the file attribute in HEX, the initial file cluster in HEX, the file length in DEC, and the end of the line.

To display the next file directory entry on the screen, it is enough to move `POINTER` to the required entry and call the template output again. This makes it possible to output records from a structured database. Similarly, the OS implements templates for dump output and for displaying the length of loaded files.

For scripts, there is only one variable. The results of the `READ`, `KEYSCAN`, and `I2CRX` operations are written into this variable. The `IF` operator compares against this variable. The `PVAR` operator outputs the literal character value of the variable to the command line. For example, this is convenient after receiving data with the `I2CRX` operator when it is necessary to read and display a text string from a device.
