# ESN LAB 2 - 7 segment with avalon interface

to start cygwin from powershell:
```pwsh
& 'C:\intelFPGA\18.1\nios2eds\Nios II Command Shell.bat'
```

to generate bsp:

```bash
nios2-bsp hal ./software/bsp/ ./*.sopcinfo
```

to generate makefile:
```bash
nios2-app-generate-makefile --app-dir ./software/app --bsp-dir ./software/bsp --elf-name maion.elf --src-files ./sofware/app/main.c
```

both (one liner):
```bash
nios2-bsp hal ./software/bsp/ ./*.sopcinfo && nios2-app-generate-makefile --app-dir ./software/app --bsp-dir ./software/bsp --elf-name maion.elf --src-files ./sofware/app/main.c
```

## Introduction

In this lab we have to make a 7 segment driver driven by an AVMM interface to count up.

## System architecture

### 1 * 7 segment version
```mermaid
---
config:
  sankey:
    showValues: false
---
sankey-beta
NIOS2, AVMM, 0.3
AVMM, AV2SEGM, 0.27
AVMM, PIO(button) [unused], 0.03
PIO(button) [unused], btn_i(1), 0.03
AV2SEGM, 7 segment, 0.27
JTAG, NIOS2, 0.1
M10K memory, NIOS2, 0.1
CLK 50M, NIOS2, 0.05
RESET, NIOS2, 0.05
```

```mermaid
flowchart TB
    %% inputs
    CLK_50M[Clock: 50 MHz] --> cpu[NIOS2 CPU]
    btn_rst[Reset Button] --> cpu

    jtag[JTAG Debug Interface] --> cpu
    memory[M10K Memory ❨RAM❩] --> cpu
    
    cpu --> avmm[AVMM Interconnect ❨8 bits wide data, 1 bit wide addr❩]  
    
    avmm --> av2segm[AV2SEGM3 Display Driver]
    av2segm --> seg_display[7-Segment Display ❨Single❩]

    %% Outputs
    seg_display --> seg_out["Segm_Out_Display"]
```


### 3 * 7 segment version
```mermaid
---
config:
  sankey:
    showValues: false
---
sankey-beta
NIOS2, AVMM, 0.3
AVMM, AV2SEGM, 0.27
AVMM, PIO(button) [unused], 0.03
PIO(button) [unused], btn_i(1), 0.03
AV2SEGM, 7 segment 1, 0.09
AV2SEGM, 7 segment 2, 0.09
AV2SEGM, 7 segment 3, 0.0895
JTAG, NIOS2, 0.1
M10K memory, NIOS2, 0.1
CLK 50M, NIOS2, 0.05
RESET, NIOS2, 0.05
```

```mermaid
flowchart TB
    %% Inputs
    CLK_50M[Clock: 50 MHz] --> cpu[NIOS2 CPU]
    btn_rst[Reset Button] --> cpu

    jtag[JTAG Debug Interface] --> cpu
    memory[M10K Memory ❨RAM❩] --> cpu
    
    cpu --> avmm[AVMM Interconnect ❨8 bits wide data, 2 bit wide addr❩]  
    
    avmm --> av2segm[AV2SEGM3 Display Driver]
    av2segm --> seg_ones[7-Segment Display ❨Ones❩]
    av2segm --> seg_tens[7-Segment Display ❨Tens❩]
    av2segm --> seg_hundreds[7-Segment Display ❨Hundreds❩]

    seg_ones --> seg_out_ones[Segm_Out_Ones]
    seg_tens --> seg_out_tens[Segm_Out_Tens]
    seg_hundreds --> seg_out_hundreds[Segm_Out_Hundreds]

```
## Progress

The system is functionnal, both the 1 and 3 7 segment counters have been implmented, backed by C logic from the NIOS over the AVMM bus.

```wavedrom
{ "signal": [
  { "name": "clk", "wave": "p.....|......|......|......|" },
  { "name": "addr", "wave": "x3....|3.....|3.....|3.....|", "data": ["0x0", "0x1", "0x2"] },
  { "name": "data", "wave": "x.=...|.=...|.=...|.=...|", "data": ["ones", "tens", "hundreds"] },
  { "name": "write", "wave": "01....|01....|01....|01....|" }
]}
```

## Conclusion

The AVMM bus widths need to be managed with care as using an 8 bit bus width will cause the default 32 bit IOWR/RD functions from the NIOS2 HAL to fail, to fix this using the 8 bit variants of these functions is require, as follows:

```c
IOWR_8DIRECT(AV2SEGM3_0_BASE, 0x0, ones);
alt_printf("ones: %x\r\n", ones);
IOWR_8DIRECT(AV2SEGM3_0_BASE, 0x1, tens);
alt_printf("tens: %x\r\n", tens);
IOWR_8DIRECT(AV2SEGM3_0_BASE, 0x2, hundreds);
alt_printf("hundreds: %x\r\n", hundreds);
```

The actual AV2SEGM3 ip (the one with 3 7 segments) has been implemented using an address mode set of AVMM registers, where each register corresponds to one 7 segment.