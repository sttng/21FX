- From: https://byuu.org/projects/21fx 
- Archive.org: https://archive.is/2020.03.22-085708/https://byuu.org/projects/21fx

# 21fx
## 21fx

21fx is an expansion port device for the SNES designed for emulation research.
The hardware was designed by defparam, and the firmware and register interface was designed by byuu.
It consists of two components: an expansion bridge to connect to the SNES expansion port and convert the interface to a DB25 data port, a stereo audio input jack, and a monaural audio output jack; and a hook which contains the actual logic and interfaces between the expansion port and the SNES.
It works by monitoring the SNES address bus and overriding the SNES reset vector address to jump into a custom 124-byte boot loader. From here, an SNES program can be uploaded into memory and then executed for analyzing either the SNES console itself, or any cartridge currently connected to the SNES. Said program can communicate bidirectionally with a host PC over a USB-TTY interface at speeds of around 1 MiB per second (the IPLROM interface is designed for size constraints over speed, and can achieve a transfer rate of 175 KiB per second. A second-stage program can expand upon this with a faster transfer routine.)
Unfortunately, due to the non-standard expansion port used by the SNES, it is not possible to mass produce these devices. One would have to hand-modify an appropriate card edge connector and mount it to the PCB themselves in order to create their own from the open source hardware project files.

## Links

[GitHub Project Page](https://github.com/defparam/21FX) 
 
## IPLROM

```
//21fx IPLROM

//jump: $216678 (uint24)offset $00
//save: $216678 (uint24)offset $01 (uint16)length $00 snes->link[length]
//load: $216678 (uint24)offset $01 (uint16)length $01 link->snes[length] $00
//exec: $216678 (uint24)offset $01 (uint16)length $01 link->snes[length] $01

//NTSC clock rate: 21477272 * 1324 / 1364
//116 clocks per iteration of inner load/save loops
//maximum throughput: 179719 bytes/second

//IPLROM is re-entrant
//all registers are destroyed; HDMA and FastROM are disabled
//if link is disconnected; original reset vector will be invoked
//if link is connected; $4377-$437b is destroyed (HDMA channel 7)

//note 1: length of zero is equivalent to 65536
//note 2: offset cannot cross bank boundaries; use one transfer per bank

architecture wdc65816
base $2184

function boot {
  sei; clc; xce
  rep #$30; lda #$2100; tad
  tax; stz $0c,x
  sep #$20; ldx #$437b; txs

  handshake:
    //connected?
    lda $fe; bne +
    jmp ($fffc); +

    //signature (24-bit)
    jsr read; cmp #$21; bne handshake
    jsr read; cmp #$66; bne handshake
    jsr read; cmp #$78; bne handshake

  //offset (24-bit)
  jsr read; pha; pha; plb  //bank
  jsr read; pha; xba       //hi
  jsr read; pha; tax       //lo

  //execute?
  jsr read; beq exec

  //length (16-bit)
  jsr read; xba  //hi
  jsr read; tay  //lo

  //load?
  jsr read; beq save

  load:
    -; bit $fe; bpl -
    lda $ff; sta $0000,x
    inx; dey; bne load
    jsr read; beq boot

  exec:
    php; rti

  save:
    -; bit $fe; bvc -
    lda $0000,x; sta $ff
    inx; dey; bne save
    bra boot
}

function read {
  -; bit $fe; bpl -
  lda $ff; rts
}
```
