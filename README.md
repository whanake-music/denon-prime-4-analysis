# Denon DJ Prime 4 - Research / Analysis

Tēnā koutou! This document is where you can find all my notes on reverse-engineering the Prime 4's protocols - particularly with the screens.

I'm currently working on a mapping to support the Prime 4 in Mixxx, which is going pretty well so far. However, there currently isn't a way to control the screens on the Prime 4 using MIDI messages. It should definitely be possible to control the screens somehow, as both Serato and Virtual DJ can do it. My goal is to try and figure out how it's done, and document my findings here so we don't have to worry about relying on closed-source solutions that will inevitably run out of support.

## TODO list
- Main 10" touch-screen

## Helpful Resources

[This GitHub repo by TheKikGen](https://github.com/TheKikGen/MPC-LiveXplore) contains a lot of useful information about various pieces of DJ hardware that run some form of Engine DJ OS.

[This page in Virtual DJ's manual](https://www.virtualdj.com/manuals/hardware/denon/prime4/layout/screen.html) states that the Prime 4 first downloads a custom Virtual DJ skin from the internet when Virtual DJ is launched with the Prime 4 connected for the first time. This means the skin must be in the Prime 4's filesystem somewhere after download, and should be accessible on an SSH-enabled Engine DJ OS image for further analysis.

[This GitHub repo by icedream](https://github.com/icedream/denon-prime4) explains how to extract the Engine OS software and enable SSH on it, so you can poke around on the Prime 4's filesystem as much as you like.

[This wiki page on the Mixxx GitHub repo](https://github.com/mixxxdj/mixxx/wiki/Reverse%20engineering) explains how to use Wireshark to sniff packets being sent to and from DJ hardware (e.g. the Prime 4), particularly when using proprietary DJ software on a Windows VM. (Yep, we're gonna MITM some DJ gear!)

## How to MITM the Prime 4 using Wireshark

Admittedly, I had a bit of trouble figuring out how to do this, so I'm collating all the necessary information here to simplify the process for anyone else wanting to do this. For context: I'm using the Wireshark GUI on Arch Linux in conjunction with Virtual DJ and MIDI-OX running in a Windows virtual machine.

### Instructions
- Install Wireshark (`sudo pacman -S wireshark-cli wireshark-qt`)
- Enable USB monitoring in the kernel (`sudo modprobe usbmon`)
- Run Wireshark with sudo priveleges (`sudo wireshark`)
- List connected USB devices (`lsusb`)
- Observe the `Bus 00` and `Device 00` prefixes before the `Numark PRIME 4` devices. Mine says `Bus 002 Device 006`.
- Select the appropriate `usbmon` filter in Wireshark based on the `Bus` number you see. For me, this is `usbmon2`. Double-clicking will make Wireshark start capturing packets immediately.
- Launch/use any relevant application (e.g. Virtual DJ) to capture its packets to and from the Prime 4.
- Stop capturing by clicking the red square in the top-left of Wireshark's GUI.
- Add the following search filter: `usb.src == "x.y.1" or usb.dst == "x.y.1"` where `x` and `y` represent `Bus 00x` and `Device 00y` from the `lsusb` output.

## SysEx Messages

Here, you can find all the information I could gather on the Prime 4's SysEx messages. There isn't a reference manual explaining these, so I'm finding them as I go.

- `f0 00 02 0b 7f 08 60 00 04 04 01 02 00 f7` - Returns the position of all the knobs and faders on the Prime 4.

#### Send specific RGB values to multi-coloured LEDs
- `f0` - SysEx header
- `00 02 0b` - Denon ID
- `7f` - Send message to main control surface
- `08 03 00` - Command to send RGB colour value
- `05` - Number of SysEx bytes left to send before the `f7` footer
- `<channel>` - MIDI channel number of LED minus 1
- `<index>` - Index of LED
- `<red>` - Value between `00` and `7f`
- `<green>` - Value between `00` and `7f`
- `<blue>` - Value between `00` and `7f`
- `f7` - SysEx footer

### Jog Wheel Displays
- Unlock left display for SysEx control - `f0 00 02 0b 10 08 10 00 00 f7`
- Unlock right display for SysEx control - `f0 00 02 0b 30 08 10 00 00 f7`

#### Change colour of jog wheel display element
- `f0` - SysEx header
- `00 02 0b` - Denon ID
- `10 or 30` - Control the left or right screen respectively
- `08 0b 00` - Command for changing colour of element
- `09` - Number of SysEx bytes left to send before the `f7` footer
- `<element>` - Element to change (See list below for possible values)
- `0* 0*` - Opacity level (between `00 00` and `0f 0f`)
- `0* 0*` - Red level (Same format as above)
- `0* 0*` - Green level (Same format as above)
- `0* 0*` - Blue level (Same format as above)
- `f7` - SysEx footer

#### Possible elements to change
- `00` - Album Art (Cannot change colour, only opacity)
- `01` - Engine DJ OS Logo
- `02` - Platter Position Ring
- `03` - Platter Position Indicator
- `04` - Slip Position Ring
- `05` - Slip Position Indicator
- `06` - Track Progress Ring
- `07` - Track Progress Indicator
- `08` - Text

#### Messages used in Virtual DJ
- Show '3' on left wheel screen - `f0 00 02 0b 10 08 0a 00 04 01 02 02 10 f7`
- Clear left wheel text - `f0 00 02 0b 10 08 0a 00 04 01 00 01 00 f7`

#### Change foreground text
- `f0` - SysEx header
- `00 02 0b` - Denon ID
- `10 or 30` - Control the left or right screen respectively
- `08 0a 00` - Command for changing text
- `04` - Number of SysEx bytes left to send before the `f7` footer
- `<visible elements A>` - See table below
- `<visible elements B>` - See table below
- `<text to display>` - See list below for possible values
- `f7` - SysEx footer

#### <visible elements A>
| Value        | Slip Ring | Slip Indicator |
|--------------|-----------|----------------|
| `0*` or `4*` |           |                |
| `1*` or `5*` | X         |                |
| `2*` or `6*` |           | X              |
| `3*` or `7*` | X         | X              |
| Value        | Background Image | Engine DJ OS Logo | Wheel Position Ring |
|--------------|------------------|-------------------|---------------------|
| `*0` or `*8` |                  |                   |                     |
| `*1` or `*9` | X                |                   |                     |
| `*2` or `*a` |                  | X                 |                     |
| `*3` or `*b` | X                | X                 |                     |
| `*4` or `*c` |                  |                   | X                   |
| `*5` or `*d` | X                |                   | X                   |
| `*6` or `*e` |                  | X                 | X                   |
| `*7` or `*f` |                  | X                 | X                   |

#### <visible elements B>
| Value        | Overlay Graphic | Text | Beat Jump Symbol |
|--------------|-----------------|------|------------------|
| `*0` or `*1` |                 |      |                  |
| `*2` or `*3` |                 | X    |                  |
| `*4` or `*5` | X               |      |                  |
| `*6` or `*7` | X               | X    |                  |
| `*8` or `*9` |                 |      | X                |
| `*a` or `*b` |                 | X    | X                |
| `*c` or `*d` | X               |      | X                |
| `*e` or `*f` | X               | X    | X                |
| Value                        | Loop Symbol |
|------------------------------|-------------|
| `0*` or `2*` or `4*` or `6*` |             |
| `1*` or `3*` or `5*` or `7*` | X           |

#### <text to display>
- `00 00` - 1/64
- `00 01` - 1/32
- `00 02` - 1/16
- `00 03` - 1/8
- `00 04` - 1/4
- `00 05` - 1/2
- `00 06` - 1
- `00 07` - 2
- `00 08` - 4
- `00 09` - 8
- `00 0a` - 16
- `00 0b` - 32
- `00 0c` - 64
- `00 0d` - --
- `00 0e` - A
- `00 0f` - B
- `01 00` - 3
- `01 01` - 6
- `01 02` - 12
- `01 03` - C
- `01 04` - D

### FX OLED Screens

#### Messages used in Serato
- Sending `f0 7e 00 06 01 f7` to the Prime 4 returns a long message. Not sure what this means yet.

#### Send progress bars to OLED screens
- `f0` - SysEx header
- `00 02 0b` - Denon ID
- `00 08 09 00` - Command to send progress bar
- `09` - Number of SysEx bytes left to send before the `f7` footer
- `<screen index>` - Select screen to send message to (value between 0x00 and 0x07)
- `<bar width>` - Width of total bar to draw (value between 0x00 and 0x7f)
- `<vertical size>` - Height of total bar to draw (value between 0x00 and 0x03)
- `01` - (Haven't figured out this one yet)
- `<primary brightness>` - Brightness of filled bar section (0x00 = black, 0x01 = white, 0x02 = light grey, 0x03 = dark grey) [Needs more investigation]
- `<background brightness>` Brightness of unfilled bar section (same format as `<primary brightness>`)
- `<vertical position>` - Vertical position of bar from top to bottom (0x00 to 0x03 respectively)
- `<progress A>` - Where to start filling in the bar (between 0x00 (left side) and `<bar width>` (right side))
- `<progress B>` - Where to stop filling in the bar (same format as `<progress A>`)
- `f7` - SysEx footer

#### Send text to OLED screens
- `f0` - SysEx header
- `00 02 0b` - Denon ID
- `00 08 08 00` - Command to send text
- `<message length>` - Number of SysEx bytes left to send before the `f7` footer
- `<screen index>` - Select screen to send message to (value between 0x00 and 0x07)
- `<text size>` - Size of text to be drawn on screen (value between 0x00 and 0x03)
- `<text alignment>` - Align text to the left (0x00), centre (0x01), or right (0x02) of the screen
- `<vertical position>` - Vertical position of text from top to bottom (0x00 to 0x03 respectively)
- `<text bytes>` - See list of values below
- `f7` - SysEx footer

#### Text values for OLED FX screens
- `00` - End text
- `01 to 20` - [Space]
- `21` - !
- `22` - "
- `23` - #
- `24` - $
- `25` - %
- `26` - &
- `27` - '
- `28` - (
- `29` - )
- `2a` - *
- `2b` - +
- `2c` - ,
- `2d` - -
- `2e` - .
- `2f` - /
- `30` - 0
- `31` - 1
- `32` - 2
- `33` - 3
- `34` - 4
- `35` - 5
- `36` - 6
- `37` - 7
- `38` - 8
- `39` - 9
- `3a` - :
- `3b` - ;
- `3c` - <
- `3d` - =
- `3e` - >
- `3f` - ?
- `40` - @
- `41` - A
- `42` - B
- `43` - C
- `44` - D
- `45` - E
- `46` - F
- `47` - G
- `48` - H
- `49` - I
- `4a` - J
- `4b` - K
- `4c` - L
- `4d` - M
- `4e` - N
- `4f` - O
- `50` - P
- `51` - Q
- `52` - R
- `53` - S
- `54` - T
- `55` - U
- `56` - V
- `57` - W
- `58` - X
- `59` - Y
- `5a` - Z
- `5b` - [
- `5c` - \
- `5d` - ]
- `5e` - ^
- `5f` - _
- `60` - `
- `61` - a
- `62` - b
- `63` - c
- `64` - d
- `65` - e
- `66` - f
- `67` - g
- `68` - h
- `69` - i
- `6a` - j
- `6b` - k
- `6c` - l
- `6d` - m
- `6e` - n
- `6f` - o
- `70` - p
- `71` - q
- `72` - r
- `73` - s
- `74` - t
- `75` - u
- `76` - v
- `77` - w
- `78` - x
- `79` - y
- `7a` - z
- `7b` - {
- `7c` - |
- `7d` - }
- `7e` - ~
- `7f` - [Space]

## VU Meters

After some packet-sniffing of the Prime 4 with Virtual DJ, I have found the MIDI messages that control the VU meters of the 4 channels.

`b0` / `b1` / `b2` / `b3`

This first MIDI byte is a Control Change (`CC`) byte that determines which mixer strip's VU meter to control.

`0a`

This second MIDI byte is the address of the VU meters themselves.

The third MIDI byte controls the 'volume' displayed by the VU meters. Below is a visual representation of what each value shows on the VU meters.

- `^` = Blue LED (OFF)
- `&` = Blue LED (ON)
- `=` = White LED (OFF)
- `#` = White LED (ON)
- `*` = Green LED (OFF)
- `$` = Green LED (ON)

- `00 - ^===******` 0
- `01 - ^===*****$` 1
- `02 - ^===****$*` 2
- `03 - ^===***$**` 3
- `04 - ^===**$***` 4
- `05 - ^===*$****` 5
- `06 - ^===$*****` 6
- `07 - ^==#******` 7
- `08 - ^=#=******` 8
- `09 - ^#==******` 9
- `0A - &===******` 10
- `0B - ^===******` 11
- `0C - ^===******` 12
- `0D - ^===******` 13
- `0E - ^===*****$` 14
- `0F - ^===*****$` 15
- `10 - ^===****$$` 16
- `11 - ^===***$*$` 17
- `12 - ^===**$**$` 18
- `13 - ^===*$***$` 19
- `14 - ^===$****$` 20
- `15 - ^==#*****$` 21
- `16 - ^=#=*****$` 22
- `17 - ^#==*****$` 23
- `18 - &===*****$` 24
- `19 - ^===*****$` 25
- `1A - ^===*****$` 26
- `1B - ^===****$$` 27
- `1C - ^===****$$` 28
- `1D - ^===****$$` 29
- `1E - ^===***$$$` 30
- `1F - ^===**$*$$` 31
- `20 - ^===*$**$$` 32
- `21 - ^===$***$$` 33
- `22 - ^==#****$$` 34
- `23 - ^=#=****$$` 35
- `24 - ^#==****$$` 36
- `25 - &===****$$` 37
- `26 - ^===****$$` 38
- `27 - ^===****$$` 39
- `28 - ^===***$$$` 40
- `29 - ^===***$$$` 41
- `2A - ^===***$$$` 42
- `2B - ^===***$$$` 43
- `2C - ^===**$$$$` 44
- `2D - ^===*$*$$$` 45
- `2E - ^===$**$$$` 46
- `2F - ^==#***$$$` 47
- `30 - ^=#=***$$$` 48
- `31 - ^#==***$$$` 49
- `32 - &===***$$$` 50
- `33 - ^===***$$$` 51
- `34 - ^===***$$$` 52
- `35 - ^===**$$$$` 53
- `36 - ^===**$$$$` 54
- `37 - ^===**$$$$` 55
- `38 - ^===**$$$$` 56
- `39 - ^===**$$$$` 57
- `3A - ^===*$$$$$` 58
- `3B - ^===$*$$$$` 59
- `3C - ^==#**$$$$` 60
- `3D - ^=#=**$$$$` 61
- `3E - ^#==**$$$$` 62
- `3F - &===**$$$$` 63
- `40 - ^===**$$$$` 64
- `41 - ^===**$$$$` 65
- `42 - ^===*$$$$$` 66
- `43 - ^===*$$$$$` 67
- `44 - ^===*$$$$$` 68
- `45 - ^===*$$$$$` 69
- `46 - ^===*$$$$$` 70
- `47 - ^===*$$$$$` 71
- `48 - ^===$$$$$$` 72
- `49 - ^==#*$$$$$` 73
- `4A - ^=#=*$$$$$` 74
- `4B - ^#==*$$$$$` 75
- `4C - &===*$$$$$` 76
- `4D - ^===*$$$$$` 77
- `4E - ^===*$$$$$` 78
- `4F - ^===$$$$$$` 79
- `50 - ^===$$$$$$` 80
- `51 - ^===$$$$$$` 81
- `52 - ^===$$$$$$` 82
- `53 - ^===$$$$$$` 83
- `54 - ^===$$$$$$` 84
- `55 - ^===$$$$$$` 85
- `56 - ^==#$$$$$$` 86
- `57 - ^=#=$$$$$$` 87
- `58 - ^#==$$$$$$` 88
- `59 - &===$$$$$$` 89
- `5A - ^===$$$$$$` 90
- `5B - ^===$$$$$$` 91
- `5C - ^==#$$$$$$` 92
- `5D - ^==#$$$$$$` 93
- `5E - ^==#$$$$$$` 94
- `5F - ^==#$$$$$$` 95
- `60 - ^==#$$$$$$` 96
- `61 - ^==#$$$$$$` 97
- `62 - ^==#$$$$$$` 98
- `63 - ^==#$$$$$$` 99
- `64 - ^=##$$$$$$` 100
- `65 - ^#=#$$$$$$` 101
- `66 - &==#$$$$$$` 102
- `67 - ^==#$$$$$$` 103
- `68 - ^==#$$$$$$` 104
- `69 - ^=##$$$$$$` 105
- `6A - ^=##$$$$$$` 106
- `6B - ^=##$$$$$$` 107
- `6C - ^=##$$$$$$` 108
- `6D - ^=##$$$$$$` 109
- `6E - ^=##$$$$$$` 110
- `6F - ^=##$$$$$$` 111
- `70 - ^=##$$$$$$` 112
- `71 - ^=##$$$$$$` 113
- `72 - ^###$$$$$$` 114
- `73 - &=##$$$$$$` 115
- `74 - ^=##$$$$$$` 116
- `75 - ^###$$$$$$` 117
- `76 - ^###$$$$$$` 118
- `77 - ^###$$$$$$` 119
- `78 - ^###$$$$$$` 120
- `79 - ^###$$$$$$` 121
- `7A - ^###$$$$$$` 122
- `7B - ^###$$$$$$` 123
- `7C - ^###$$$$$$` 124
- `7D - ^###$$$$$$` 125
- `7E - ^###$$$$$$` 126
- `7F - &###$$$$$$` 127

NOTE: As far as I'm aware, the Master L + R channels are not controlled by MIDI. They are instead controlled by the audio sent through the Prime 4's soundcard when in Computer Mode. As long as your audio is configured to go through the Prime 4, the Master L + R meters should behave as expected automatically.
