# Chroni2 - Video Processor Unit Reference Guide

**Note: This document is still a DRAFT and not final.**

Chroni2 is the upgraded 16-bit video processor from CLC-88 Compy. It is designed
to help game programmers do common tasks like scrolling, sprites,
video multiplexing and others with the less amount of code.

The name comes from *Chroni, partners in time* (with the CPU).  
It was suggested by Stuart Law from *Cronie, partners in crime*, a popular english saying.

For simplicity, Chroni2 will be named just Chroni in this document.

The main features of Chroni are:
* 2MB of Video RAM
* 256 color palette from a 65.536 color space (RGB565)
* Programmable video / display modes (display lists)
  * 120 columns text mode with 16 colors
  * 8x8 tiles mode with 16 colors per tile
  * 16 and 256 color bitmap modes
* Multiple background layers
* Multi level priority for each layer and sprites
* Hardware coarse and fine scrolling
* Virtual display windows
* Vertical and horizontal blank interrupts
* Sprites

The following content is a Reference Guide. If you are new to 16-bit
computers/console programming, we recommend you to use the Chroni Tutorial (*)
together with this guide.

(*) Not yet released

---
Note: All the examples in this document use 6502 assembly code with MADS macros for simplicity.
You will find *mwa* or *mva* instructions, instead of the *lda/sta* sequences
where applicable.

## Video RAM

There are 2MB RAM for Video RAM (VRAM) which is accessed exclusively by
Chroni. Any access from the CPU must be done through Chroni as it is 
explained later in this document.

There is no memory map for VRAM, Chroni has registers that hold the
addresses for the required content like charsets, tiles or sprites.
Just place the content in any area of the addressable space and then 
point the required Chroni register to that area.

### Addressing

Chroni requires addresses for charsets, tiles, sprites and more, these
addresses are 16-bit, but the addressable space is 20-bit. What Chroni
does is to use those 16-bit addresses as the top 16-bit of the 20-bit
real address.

       Address bit 2 1 1 1 1 1 1 1 1 1 1 0 0 0 0 0 0 0 0 0 0 
                   0 9 8 7 6 5 4 3 2 1 0 9 8 7 6 5 4 3 2 1 0

    16 bit address X X X X X X X X X X X X X X X X X 0 0 0 0

For example, if the charset is set on address $3BFE (16 bit), the real
address in VRAM is $3BFE0 (20 bit).

The addressable space is 20-bit, where each address points to a 16-bit data value.

20-bit gives 1024*1024 addresses, each address holds 2 bytes, so the address
spaces covers 1024*1024*2 bytes, which is 2 Mega Bytes.

### Registers

Chroni registers are mapped to 0x9000 - 0x907F on CPU system memory.

You can find the names declared in asm/6502/os/include/symbols.asm

    00 :  VDLIST      WORD  Display List pointer
    02 :  VLAYER      BYTE  Layer index register (0-3)
    02 :  VLAYER_ST   BYTE  Layer status register
    04 :  VPAL_INDEX  BYTE  Palette index register
    05 :  VPAL_VALUE  BYTE  Palette data register
    06 :  VADDR       WORD  Address register
    08 :  VADDR_AUX   WORD  Address register (auxiliary)
    0a :  VDATA       BYTE  VRAM data read / write
    0b :  VDATA_AUX   BYTE  VRAM data read / write (auxiliary)
    0c :  VCOUNT      WORD  Current scanline count register (read only)
    11 :  WSYNC       BYTE  Any write will halt the CPU until next HBLANK (write only)
    12 :  WSTATUS     BYTE  Chroni status register
    14 :  VSPRITES    WORD  Address of the Sprite definition table
    1a :  VBORDER     WORD  Border color in RGB565 format (obsolete)
    22 :  VLINEINT    WORD  Scanline interrupt register
    24 :  CLOCK_MULT  BYTE  CPU Clock multiplier
    25 :  VRAND       BYTE  Random number generator (read only)
    26 :  VADDRB      WORD  Byte address register (low 16-bit)
    28 :              BYTE  Byte address register (high 1-bit)
    2a :  VADDRB_AUX  WORD  Byte auxiliary address register (low 16-bit)
    2c :              BYTE  Byte auxiliary address register (high 1-bit)
    2e :  VAUOTOINC   BYTE  Autoincrement mode register
    30 :  VADDR_ALIGN WORD  Next 32 byte aligned address available

# Chroni status register
Writing to the Chroni status register you can enable or disable some features 
of Chroni. Reading the status register you can get some info about the running state
of Chroni.

Following is the info that each bit of the status register holds, together
with the read or write access allowed. Note that bit 1 and 0 are reserved
and you must never write on them.

    bits  7 6 5 4 3 2 1 0
          | | | | | |----- Interrupts enabled    (r/w)
          | | | | |------ Sprites enabled        (r/w)
          | | | |------- Chroni enabled          (r/w)
          | | |-------- 1 if this is an emulator (r) 
          | |---- HBLANK active                  (r)
          |--- VBLANK active                     (r)

On hardware reset, Chroni and Chroni interrupts will be disabled. This
will give you time to prepare the display list and interrupt handlers
before Chroni start using them

This is a simple example to enable Chroni and interrupts on startup

        lda VSTATUS
        ora #(VSTATUS_EN_INTS + VSTATUS_ENABLE)
        sta VSTATUS

### VRAM Access via Chroni VADDR and VDATA registers

You cannot write or read directly to VRAM, only Chroni has access to VRAM.
To access the VRAM you set an address register and then write or read the values
from VRAM using a data register.

For simplicity, let's think we want to write $FF to VRAM address $2a340. The
6502 code for that is (MADS assembler):

    mwa #$2a34 VADDR  ; this is the address we want to read or write
    mva #$ff   VDATA  ; this is a write to said address

The VADDR is a 16 bit register to set the read/write address.
The VDATA is an 8 bit register to read or set the value in that address

By default, the address set via VADDR increments automatically. Each time you
read or write using VDATA, the address increments by 1 byte. You only need
to set VADDR once to write a stream of bytes to VRAM.

Let's say we need this data in VRAM

    address  data
     $4a5f0   $12
     $4a5f1   $38
     $4a5f2   $a5
     $4a5f3   $bb

The code is just:

    mwa #$4a5f VADDR
    mva #$12 VDATA
    mva #$38 VDATA
    mva #$a5 VDATA
    mva #$bb VDATA

it might seem strange at first, but this simplifies crossing page boundaries a lot.
You simply don't need to worry about crossing page boundaries, neither keep
track of current VRAM addresses.

### Auto increment / decrement address register
By default, each time you read or write to VDATA, the internal address
register will be incremented by 1 byte. You can change that behaviour writing
to register VAUOTOINC. The possible values (defined in symbols.asm) are:

        AUTOINC_VADDR_KEEP     = $00  ; disables auto increment / decrement
        AUTOINC_VADDR_INC      = $01  ; set to auto increment
        AUTOINC_VADDR_DEC      = $03  ; set to auto decrement

For example, to turn autoincrement off:

        mva #AUTOINC_VADDR_KEEP VAUOTOINC

### Auxiliary address register
In some cases you may need to access two different VRAM addresses at the same
time, for example when copying from VRAM to VRAM, or when writing char
and attribute values in text mode. To avoid writing the new address each time, 
and keep taking advantage of the autoincrement feature, you can use the auxiliary 
address register.

The following example use both the main address register and the auxiliary
address register to copy 250 bytes of data from VRAM to VRAM. Locations are
pointed by SRC_ADDR and DST_ADDR

        mwa SRC_ADDR VADDR
        mwa DST_ADDR VADDR_AUX
        ldx #250

    copy:
        lda VDATA
        sta VDATA_AUX
        dex
        bne copy
        
### Access to aligned / non aligned VRAM data
As VADDR only used the high 16 bits of the 20 bit address, you can only 
address 16 word aligned addresses (32 byte aligned);

    VADDR = 0 points to byte address 0     (word address 0)
    VADDR = 1 points to byte address 32    (word address 16)
    VADDR = n points to byte address 32*n  (word address 16*n)

To address VRAM locations in between, for example byte address 7 or 
byte address 33 you can use the VADDRB and VADDRB_AUX registers.
Typically, you won't need to access individual random bytes, but if you
need to do so, you can write the full 21 bit address that points to
the specific byte you need to read or write.

The VADDRB and VADDRB_AUX registers are 3 bytes wide, where the first
two bytes are the low 16 bit part and the third byte has the high 5 bit part

Let's say we need to access the byte at address $1C745 and write $A5 on it.
The code for that will be:

    mwa #$C7A5 VADDRB
    mva #$1C   VADDRB+2
    mva #$A5   VDATA

Note that the VADDRB and VADDRB_AUX registers are just another way to access
the internal Chroni Address register, and it is subject to the same auto increment
and auto decrement features that VADDR and VADDR_AUX has.

#### Trick to access non 16 word aligned data in VRAM
Another way to access non-aligned data in VRAM is to use a simple trick.
The autoincrement register goes byte by byte, so you can set the address as
16 word aligned, then make the address register increment by one until you
reach the required address:

        mwa #TARGET_ADDRESS VADDR
        lda VDATA   ; skip 1 byte
        lda VDATA   ; this is the first non-aligned byte

After reading VDATA, the address register will point to the next
byte after TARGET_ADDRESS, thus you will get the first non-aligned byte.

## Colors and Palettes

There is a global 256 color palette in RGB565 format (16 bit per entry).
This palette defines the global 256 color scheme available at once,
taken from a 64K color space.

The palette is a 256 16 bit RGB565 colors array stored inside Chroni.

To define a color within the palette you use the VPAL_INDEX and VPAL_VALUE
registers. The index is a value between 0-255 to define which color index
you want to access, then you write 2 bytes on the value register to define
the 16-bit RGB565 color. Less significant byte goes first. 

The following code set the color index 5 to RGB656 #FF3A

        mva #5  VPAL_INDEX
        mva #3A VPAL_VALUE
        mva #FF VPAL_VALUE       

The VPAL_INDEX register supports autoincrement, so you can define several
color entries with only one write to the VPAL_INDEX register.

The following code sets the first 32 colors of the palette from te data
on address "palette"

        mva #0 VPAL_INDEX
        ldx #0
    set_color:
        lda palette, x
        sta VPAL_VALUE
        inx
        cpx #64
        bne set_color

### 16 color "sub palettes"
Depending on the video mode, colors are not accessed directly, but through
smaller 16 color palettes or sub palettes, taken from the whole 256 color palette.

For example, a tile can use up to 16 colors, but which ones from the 256
colors available? The tile definition includes a color index which selects
a group of 16 colors within the 256 colors available. For example:

        tile 0, color index 0 : uses color from 0 to 15
        tile 2, color index 1 : uses color from 16 to 31
        tile 5, color index 4 : uses color from 64 to 79

So, for each 16-bit color pixel, text, tile or sprite on the screen you can
select which 16 color palette to use. 

## Display lists

The display list is a processing instruction set for Chroni. To draw a screen, Chroni
will read each entry in sequence to know which video mode to use and for how many
scanlines long.

You can place a playlist anywhere on memory, then use the VDLIST register to set
the starting address.  The following example uploads a playlist to VRAM and then 
sets the VDLIST register to point to that address.

        mwa #dlist_on_vram VADDR
        ldx #0

    upload_dl:
        lda display_list, x
        sta VDATA
        inx
        cpx #display_list_size
        bne upload_dl

        mwa #dlist_on_vram VDLIST

Now, which data you need to have in display_list to create a screen? You need...

### Display list instructions

A typical display list instruction contains a video mode and a number of scanlines 
you want to use for that video mode. For example, a text mode has 8 scanlines for
each row, so for a 24 rows text mode you need to specify 24*8 = 192 scanlines.

The minimal instruction size is 16-bit (2 bytes), but one instruction can grow bigger
than that depending on the features you want to enable. All instructions will always
be word aligned.

The simplest display list will contain 2 instructions: One defining the video mode and 
another one marking the End of the List. After this End of List marker Chroni will
stop creating an image and will start the vertical blank process.

More complex display lists can create screens with mixed content easily. You can
mix video modes, scrolling and non-scrolling areas, multiple scrolling ares (parallax)
and more.

For example, you can create a screen with a score panel at the top
and a playfield at the bottom, like this one:

        |-----------------------------|
        |            score            |
        |-----------------------------|
        |                             |
        |                             |
        |          playfield          |
        |                             |
        |                             |
        |-----------------------------|

You can also make that playfield scrollable without affecting the score
section. Display lists makes it very easy to create this kind of screens
without using much CPU code.

### Display list instruction specification

Each display list entry is a 16 bit value. The basic entry is this:

    F E D C B A 9 8 7 6 5 4 3 2 1 0
    | | | | 0 0 0 + + + + + + + + + ---> number of scalines
    | | | | + + + ---------------------> reserved
    | + + + ---------------------------> video mode
    +----------------------------------> scroll enabled

#### Scanlines
You are free to specify the number of scanlines per video mode, even if
the video mode has a fixed number of scanlines per row, like text modes
or tile mode which have 8 scanlines per row, you can specify any arbitrary
number like 6 or 12. Using this method you can grow or shrink a section
of the screen to create perspective effects.

A good example of this kind of effects is found on the game Coryoon
for the PC Engine (https://www.youtube.com/watch?v=SQySsSVTyjQ)

#### Video Modes
The video mode value specifies which one of the text/tiled/bitmap mode you want
to use. At this stage of development, these are the valid video modes:

    ID | Type     | Colors | Display                   |  Row Height
    -----------------------------------------------------------------
     0 | Blank    |    1   | Uses the background color |  1 scan
     1 | Text     |   16   | 120 Chars per row         |  8 scans
     2 | Tiles    |   16   | 60 Tiles per row          |  8 scans
     3 | Bitmap   |   16   | 480 pixels per row        |  1 scan
     4 | Bitmap   |  256   | 480 pixels per row        |  1 scan
     F | END      |    0   | End of display list       |  0 scan

Video modes 0x0 and 0xF are special modes:
- Video mode 0x0 is blank screen, the border/background color is used
- Video mode 0xF is the end of the playlist / screen

The video mode 0 is useful for plain backgrounds without any content, just color.
A smart use of this mode in some games is to modify the color for each scanline
creating sunrise / sunset effect and more.

#### Char/Attributes/Tile addresses 

If the entry defines a video mode with data to be displayed (0x1-0xE), the 
following entry is the VRAM address of the screen buffer. The interpretation
of this buffer depends on the video mode: text, tiles or bitmap.

This address is a 16-bit value as any other Chroni register, so it is 
the top 16 bits of a 20 bit VRAM address. For example if this address is
set as 0x2154, it turns into VRAM address 0x21540.

The following example is a text video mode with 240 scanlines. The screen buffer
where char values are stored start at address $28200

        10F0 : Text Mode 1, 240 scanlines ($F0)
        2820 : VRAM address of the screen data

#### Char attributes

Each char in text mode have a background and text color, selectable from 16
colors. They are the first 16 colors of the 256 color palette.

For each char value there will be an attribute value which defines the 
background and text color for that char. One attribute value is just one byte
where the top 4 bits define the background color and the lower 4 bits define
the text color. For example the attribute value $A5 define that the 
background color is $A (color 10) and the text color is $5 (color 5).

The attribute data is the same size as the screen data, because each char
requires exactly one attribute value.

The address of the attribute data is set just after the address of the screen data.
We can extend our example for this, for example if the attribute data is at 
address $42000, the display list is:

        10F0 : Text Mode 1, 240 scanlines ($F0)
        2820 : VRAM address of the screen data
        4200 : VRAM address of the attribute data

#### Char / Tile patterns

Text and tile video modes not only require chars or tile values, they also
require the charset or tile definition which are the patterns bits that define 
the graphics of each character or tile. For example the char 'A' could
be defined as the following pixel or bit pattern:

        0 0 0 0 0 0 0 0   => $00
        0 0 0 1 1 0 0 0   => $18
        0 1 1 1 1 1 1 0   => $7E
        0 1 1 0 0 1 1 0   => $66
        0 1 1 1 1 1 1 0   => $7E
        0 1 1 0 0 1 1 0   => $66
        0 0 0 0 0 0 0 0   => $00

That pattern definition resides in a certain VRAM area and that address must
be set on Chroni. Most computers defined a global location for the entire screen,
but Chroni allows to set the pattern address at any part of the screen. Each
time you define a text or tile mode, you need to declare the pattern data address.

So, the previous example was still incomplete, it was missing the pattern address. Let's
fix that declaring that the pattern address is at location $84600

        10F0 : Text Mode 1, 240 scanlines ($F0)
        2820 : VRAM address of the screen data
        4200 : VRAM address of the attribute data
        8460 : VRAM address of the charset patterns

Note that this makes it really easy to combine multiple charset or tile patterns
on a single screen, without using any CPU code.

Finally, if the display list is complete, we can add an End of Screen instruction.
According to the mode instruction we can use $0F00, but for simplicity, you
can use just $FFFF. 

Here is the whole display list for this text mode example:

        10F0 : Text Mode 1, 240 scanlines ($F0)
        2820 : VRAM address of the screen data
        4200 : VRAM address of the attribute data
        8460 : VRAM address of the charset patterns
        FFFF : End of Display List

Now a few more examples:

This is an example of a tiled video mode with 240 scanlines. The tile data
starts at VRAM address 0x28200 whilst the tile patterns start at 0x84600

        20F0 : Mode 2, 240 scanlines (0xf0)
        2820 : VRAM address of the tiles data
        8460 : VRAM address of the tile patterns
        FFFF : End of display list

This is an example of a text video mode with 192 scanlines. The char data
starts on VRAM address 0x0800, attribute data starts on VRAM address 0x20000
and charset patterns data start at 0x30000

        10C0 : Mode 1, 192 scanlines (0xc0)
        0800 : VRAM address of the char data
        2000 : VRAM address of the attribute data
        3000 : VRAM address of the charset patterns data
        FFFF : End of display list

### Scrolling

If the scroll bit is set, there will be three more entries: The virtual screen
size, the visible portion of that screen and the fine scrolling values.

Note that all values, except for fine scrolling, are measured in 8 pixels. 
So for example a value of 3 is 24 pixels. Now, chars and tiles are
8x8 pixels size, so you can also think that a width of 3 is just 3 chars,
or a height of 8 is just 8 tiles, etc.

#### Virtual window size
The scrolling technique uses a virtual screen that is larger than the visible
screen. While other systems have a fixed virtual screen size, Chroni let
the developer chose a virtual screen size.  For example a tiled mode of 
60x34 visible tiles can have a virtual screen of 64x40 tiles. 

    
    <-------- 64 tiles (virtual width) ------->
    +------------------------------------------+  +
    |                                          |  | 
    |                                          |  |
    |     +-------------------------------+    |  |
    |     |                               |    |  | virtual
    |     |         Visible Area          |    |  | height
    |     |                               |    |  |
    |     +-------------------------------+    |  |
    |     <--- 60 tiles (visible width) -->    |  |
    |                                          |  |
    |              Virtual window              |  |
    |                                          |  |
    +------------------------------------------+  +

The first scrolling value is the size of the scrolling window (or virtual window). 
This should be larger than the screen size

    F E D C B A 9 8 7 6 5 4 3 2 1 0
    | | | | | | | | | | | | | | | |
    | | | | | | | | + + + + + + + + ----> window width   
    + + + + + + + + --------------------> window height 

For the example above with a virtual window of 64x40 tiles, 
this value will be $4028.

#### Visible area position

Then comes the current position into the visible area:

    F E D C B A 9 8 7 6 5 4 3 2 1 0
    | | | | | | | | | | | | | | | |
    | | | | | | | | + + + + + + + + ----> left position 
    + + + + + + + + --------------------> top  position

Again, top and left are measured in 8 pixels.

Following is the same virtual window diagram with
the left and top of the visible area set.

    <------------ (virtual width) ------------->
    +------------------------------------------+  +
    |                                          |  | 
    |        top                               |  |
    |   left +----------------------------+    |  |
    |        |                            |    |  | virtual
    |        |       Visible Area         |    |  | height
    |        |                            |    |  |
    |        +----------------------------+    |  |
    |        <----- (visible width) ------>    |  |
    |                                          |  |
    |                                          |  |
    |              Virtual window              |  |
    |                                          |  |
    +------------------------------------------+  +

#### Virtual window wrap

Note that the virtual window wraps, so if the width is 132 and the left position is 129
the first displayed element will be from memory position 129, the next one will
be 130, and the next one will be 131, reaching the window width. The next
displayed element will be element 0, then 1, and so on.
The same applies for vertical positions.

This is just that example using left = 129 and width = 130

    position  001 | 002 | 003 | 004 | 005 | 006 | ...
    address   129 | 130 | 131 | 001 | 002 | 003 | ...

#### Fine scrolling position.

The top and left values are measured in char / tile elements, so incrementing the value
by 1 will move the screen 8 pixels. To move the screen by 1 pixel you can use
the fine scrolling position.

    F E D C B A 9 8 7 6 5 4 3 2 1 0
    | | | | | | | | | | | | | | | |
    | | | | | | | | + + + + + + + + ----> fine horizontal position 
    + + + + + + + + --------------------> fine vertical position

#### Full display list with scrolling

Let's finish with a display list for a tiled mode with scrolling enabled.
The virtual screen size will be 160x64.

    A01F : scrolling tiles mode 0x2 with 270 scanlines
    A200 : tiles at VRAM address 0xA2000
    A600 : tile patterns at VRAM address 0xA6000
    A040 : scrolling window is 160x64 bytes
    0312 : current position is left=3, top=18
    0102 : fine scrolling is x=1, y=2
    FFFF : end of display list

#### Tip: Parallax Scrolling

Note that scrolling data is part of the display list, so you can declare
multiple scroll sections in one screen.

## Display list examples

A simple text mode with attributes, 240 scanlines height (30 chars)

    10F0 : mode 0x1 with 240 scanlines
    8200 : chars at VRAM address 0x82000
    9200 : attributes at VRAM address 0x92000
    9800 : charset patterns at VRAM address 0x98000
    FFFF : end of display list
    
An Atari like text mode, with 24 empty lines at the top, then 192 scanlines (24 chars)

    0018 : 24 empty scanlines
    10C0 : mode 0x1 with 192 scanlines
    8200 : chars at VRAM address 0x82000
    9200 : attributes at VRAM address 0x92000
    9800 : charset patterns at VRAM address 0x98000
    FFFF : end of display list

A mixed video mode, 3 lines of text at the top, tiles at the bottom

    1018 : mode 0x1 with 24 scanlines (3*8)
    8200 : chars at VRAM address 0x82000
    9200 : attributes at VRAM address 0x92000
    9800 : charset patterns at VRAM address 0x98000
    20C0 : mode 0x2 with 192 scanlines
    A200 : tiles at VRAM address 0xA2000
    A600 : tile patterns at VRAM address 0xA6000
    FFFF : end of display list

Another mixed video mode, 3 lines of fixed tiles at the top, then a playfield
with scrolling tiles at the bottom

    2018 : fixed tiles mode 0x2 with 24 scanlines (3*8)
    8200 : tiles at VRAM address 0x82000
    8800 : tile patterns at VRAM address 0x88000
    A0C0 : scrolling tiles mode 0x2 with 192 scanlines
    A200 : tiles at VRAM address 0xA2000
    A600 : tile patterns at VRAM address 0xA6000
    A040 : scrolling window is 160x64 bytes
    0312 : current position is left=3, top=18
    0102 : fine scrolling is x=1, y=2
    FFFF : end of display list

### Tiles

For tiles, the screen data has 16-bit per tile, the value contains the 16 color
sub-palette for that tile and the offset address of the tile pattern. The offset
is based on the tile pattern address.

The 256 color palette is divided in 16 sub-palettes of 16 colors each. The
top 4 bits of the tile value are the sub-palette index.  The lower 12
bits are the VRAM offset of the tile patterns.

    bits FEDCBA9876543210
         ||||------------ sub-palette
             |||||||||||| vram address (high 12 bits)

#### Tile patterns

Each tile is 8x8 pixels where each pixel can have any of the 16 colors of the
sub-palette, so each pixel takes 4 bits. As each row is 8 pixels, you need
4*8 = 32 bit for each row (4 bytes / 2 words), and the whole 8 rows of the
tiles require 4*8 bytes or 2*8 words, which is 32 bytes or 16 words.

If you check Chroni addressing, each tile starts exactly at each VADDR address,
that's why at some point this document says you may never need to access random
bytes in VRAM.

Now, the tile pattern value is only 12 bit and not 16, that's why you can
specify a pattern base address for tiles to complete the 20 bit required
to access the full VRAM.

##### Pixels in tile patterns

Each tile pattern is read as 16 bit values, where the pixels are stored
starting with the lowest bits. For example is the value $3F8A is read, then
pixel colors for that value will be $A, $8, $F and $3.

### Bitmap modes

Bitmap modes only need a screen data address, they don't use patterns. There
are 16 and 256 color bitmap modes, requiring 4 or 8 bit respectively.
Again, bitmap modes are read as 16 bit values, where the pixels are stored
starting with the lowes bits.  For example is the value $2C1F is read, in 16 color 
mode, the pixel colors for that value will be $F, $1, $C and $2. The same value
in 256 color mode will have the pixel colors as $1F and $2C.



