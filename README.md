# How to Unbrick ASUS G14 by Clearing the UEFI NVRAM (aka CMOS clear)

## tl;dr

- ASUS G14 stores its UEFI (BIOS) settings in its 16MiB SPI Flash ROM (NVRAM region).
- The NVRAM region can be cleared by:
    - Dumping the flash ROM contents with  `flashprog` (using an [opensensor/pico-serprog](https://github.com/opensensor/pico-serprog) or [dword1511/stm32-vserprog](https://github.com/dword1511/stm32-vserprog) flash programmer).
    - In the ROM dump, replacing the NVRAM region with the clean state from an EZFlash update file using `UEFITool` and `UEFITool` NE.
    - Writing the modified NVRAM region back into the Flash ROM.

## Intro

I accidentally bricked my ASUS Zephyrus G14 GA401 (2021) Laptop while playing with [Smokeless_UMAF](https://github.com/DavidS95/Smokeless_UMAF), and it stuck on power-on even without the ROG Logo appearing. Since traditional "CMOS clear" methods (such as long-pressing the power button, cutting off the battery pack for a while, or pressing Ctrl + Home to launch EZFlash) didn't work for me, I had to dump the contents of the Flash ROM chip and write the modified contents back into the chip. 

Thanks to open-source tools for handling SPI Flash ROM and UEFI files, I successfully unbricked my laptop.
This is primarily a memo for myself. Attempting this procedure is at your own risk.

## SPI Flash ROM interfacing

- On my laptop, the SPI Flash ROM (Winbond W74M12JWSIQ, 128Mbit, @1.8V) is located between the M.2 NVME socket and the battery.
    - I directly soldered some wires onto the pins: CS, MISO, GND, MOSI, CLK, VCC.
- As a flash programmer, I tried the following, and both worked:
    - An RP2040 RPi Pico with [opensensor/pico-serprog](https://github.com/opensensor/pico-serprog) firmware.
    - An STM32F103 Blue Pill with [dword1511/stm32-vserprog](https://github.com/dword1511/stm32-vserprog) firmware.
- Also, I used a cheap 3.3V/1.8V level shifter found on my desk, but it turned out to cause read inconsistency (random bit rot), so I just put a 1k/1k voltage divider on CLK and connected MISO directly.
- These serprog-compatible programmers can be controlled by `flashrom` or its fork from the Libreboot team, `flashprog`.
    - `flashrom` can be installed using apt-get, while `flashprog` cannot. However, `flashprog` is seemingly more feature-rich (e.g., supports the `--progress` flag, etc.).
    - I used `flashprog` via an `archilinux` Docker container.
  - References:
    - [Winbond W74M12JW Datasheet](https://www.winbond.com/resource-files/W74M12JW%20RevA.pdf)
    - [Libreboot – Read/write 25XX NOR flash via SPI protocol](https://libreboot.org/docs/install/spi.html)

<img src="https://gist.github.com/user-attachments/assets/09db8637-0936-4d72-bfeb-8037fa0ee1a3" height="300px">

The following is cited from the Libreboot docs. Note in this case VCC is 1.8V.

<img src="https://av.libreboot.org/rpi_pico/soic8_pico_pinouts.jpg" height="300px">


### RP2040 RPi Pico Setup

- Using an RP2040 RPi Pico with [opensensor/pico-serprog](https://github.com/opensensor/pico-serprog) firmware.
- The prebuilt binary can be found at [Release v1.0.0 · opensensor/pico-serprog](https://github.com/opensensor/pico-serprog/releases/tag/v1.0.0). Download the `.uf2` file, power the board while pressing the BOOT button, then copy the file to the `RPI-RP2` drive.
- The pinouts are:
    - GP2: SCK
    - GP3: MOSI
    - GP4: MISO
    - GP5: CS

<img src="https://gist.github.com/user-attachments/assets/ac9ed478-cbbd-43cd-a64e-62211ad6fe25" height="500px">

### STM32F103 Blue Pill Setup

- Using an STM32F103 Blue Pill with [dword1511/stm32-vserprog](https://github.com/dword1511/stm32-vserprog) firmware.
- No prebuilt binary is available, but a detailed guide can be found in the README.
    - Put it in ISP mode by configuring the on-board jumpers as BOOT0=1 and BOOT1=0. Connect PA9 (TX) and PA10 (RX) to a USB Serial adapter. After flashing, make sure BOOT0=0.
- The pinouts are: (ref: [Pull Request #7 · dword1511/stm32-vserprog](https://github.com/dword1511/stm32-vserprog/pull/7))
    - PA6: MISO
    - PA7: MOSI
    - PA4: CS
    - PA5: SCK

```sh
$ git clone --recurse-submodules https://github.com/dword1511/stm32-vserprog.git
$ cd stm32-vserprog
$ git log | head -1
commit c74b4ed5704907f56e6ed1fa59d1874fd8de1bda

$ sudo apt-get install -y stm32flash gcc-arm-none-eabi

$ make BOARD=stm32-bluepill
$ sudo make BOARD=stm32-bluepill flash-uart
```

<img src="https://gist.github.com/user-attachments/assets/78f269fd-bf5a-4241-8998-51ac8d39c9b7" height="500px">

## Clearing NVRAM region

- First, dump the current Flash ROM contents.
```sh
# Start archilinux container. Here ttyACM1 is the flasher, identified from `ls -hl /dev/serial/by-id`
$ docker run --rm -it -v $PWD:/wd -w /wd --device /dev/ttyACM1 archlinux:latest bash

# Install flashprog
$ pacman -Sy flashprog
$ flashprog -V
flashprog v1.4-dirty on Linux 6.8.0-55-generic (x86_64)
flashprog is free software, get the source code at https://flashprog.org
...

# Read and determine the flash
$ flashprog --progress -p serprog:dev=/dev/ttyACM1,spispeed=2M -r g14-bios.flashprog.bin
flashprog v1.4-dirty on Linux 6.8.0-55-generic (x86_64)
flashprog is free software, get the source code at https://flashprog.org

Using clock_gettime for delay loops (clk_id: 1, resolution: 1ns).
serprog: Programmer name is "pico-serprog"
Warning: NAK to query operation buffer size
Found Winbond flash chip "W25Q128.W" (16384 kB, SPI) on serprog.
...
Reading... [==================================================] 100% done.

# Verify the dump
$ flashprog -V --progress -c W25Q128.W -p serprog:dev=/dev/ttyACM1,spispeed=2M -v g14-bios.flashprog.bin
...
Reading... [==================================================] 100% VERIFIED.
```
- Obtain a BIOS update file (a zip archive for EZFlash) from the ASUS website.
```sh
$ unzip GA401QCAS415.zip
  inflating: GA401QCAS.415
```
- Download UEFITool NE binary.
    - https://github.com/LongSoft/UEFITool/releases/tag/A71
- Download UEFI Tool binary.
    - https://github.com/LongSoft/UEFITool/releases/tag/0.28.0
    - UEFITool NE is a successor, but it does not support editing yet. Therefore, I used UEFITool NE for analyzing and extracting the NVRAM file, and then used UEFI Tool to modify the dump file. 
- Open the BIOS update file with `UEFI Tool NE`. 
    - Expand `AMI Aptio capsule > UEFI image > AmiStandardDefaultsVariable > NVRAM`.
    - Right click on `NVRAM` and `Extract as is` to save `.ffs` file.
    - Note the File GUID for NVRAM (e.g. `CEF5B9A3-`...)
    - ![](https://gist.github.com/user-attachments/assets/e3152eba-e08b-449a-bddc-937c70a47f4d)
- Open the dump file with `UEFI Tool`. 
    - Expand `UEFI image > <First Volume UUID>`. Right-click the file with the same UUID (matching the NVRAM File GUID noted earlier).
    - Right-click on `CEF5B9A3-` (or the corresponding UUID) and `Replace as is`. Load the `.ffs` file that you extracted from the BIOS update file.
    - Save the modified file by selecting `File > Save image file`. (`g14-bios.flashprog.replaced.bin`)
    - ![](https://gist.github.com/user-attachments/assets/a60d7913-9a76-457c-bd41-959a68cf0564)
- Now write the modified version to the ROM.
```sh
# Create a layout file to avoid full-flash.
# The content length is guessed from the layout from UEFITool NE.
$ vi flash.layout
$ cat flash.layout
00000000:0001ffff nvram
00020000:00ffffff other

# Write nvram region.
$ flashprog -V -c W25Q128.W --layout flash.layout --image nvram --progress -p serprog:dev=/dev/ttyACM1,spispeed=2M -w g14-bios.flashprog.replaced.bin
...
Using region: "nvram".
Reading old flash chip contents... 
Reading... [==================================================] 100% done.
Erasing and writing flash chip... 
Erasing... [==================================================] 100% , 0x010000-0x01ffff:E
0x000000-0x01ffff:
Writing... [==================================================] 100% W
Erase/write done.
Verifying flash... 
Reading... [==================================================] 100% VERIFIED.
```
- DONE. Remove the flasher and power the laptop. Fingers crossed.
