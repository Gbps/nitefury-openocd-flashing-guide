# Nitefury II / Acorn CLE OpenOCD Flashing Guide

This guide and repo will help you flash bitstreams to you **[Nitefury II](https://rhsresearch.com/collections/rhs-public/products/nitefury-xilinx-artix-fpga-kit-in-nvme-ssd-form-factor-2280-key-m)** or [**Acorn CLE 215**](https://www.amazon.com/SQRL-CLE-215-Acorn) board using [OpenOCD](https://github.com/openocd-org/openocd) over a JTAG programmer and/or PCIe.

I found it difficult to flash my Nitefury II, so I created this repo as a result of my research into it!

# Boards

- **[Nitefury II](https://rhsresearch.com/collections/rhs-public/products/nitefury-xilinx-artix-fpga-kit-in-nvme-ssd-form-factor-2280-key-m)** - Tested and working. This is the one I have.

- [**Acorn CLE 215**](https://www.amazon.com/SQRL-CLE-215-Acorn) - Untested, but uses relatively the same design as the Nitefury II so it should function the same.
- [**PicoEVB**](https://www.crowdsupply.com/rhs-research/picoevb) - Look [here](https://github.com/RHSResearchLLC/PicoEVB/tree/master/spi-flash-program-openocd) instead.
- [**Litefury**](https://rhsresearch.com/products/litefury) - Not the same FPGA chip used here. However, this could be modified to use this chip by using a proxy bitstream made for this chip. See below.

# Programming Cable

This process uses the [Digilent JTAG-HS3 Programming Cable](https://digilent.com/shop/jtag-hs3-programming-cable/) which uses a [FT232H](https://www.ftdichip.com/old2020/Products/ICs/FT232H.htm) USB->Serial chip. This repo uses OpenOCD and is specifically set up to use this cable and JTAG chip, but there will be a section where you can modify the settings to your own JTAG programmer's settings. 

The Nitefury II comes with a JTAG breakout board which you connect to your FPGA board using the included IO connector. Plug the IO connector into the side of the JTAG breakout board labelled BLK, then connect your JTAG programmer. Your JTAG programmer must be able to connect to a **2x7 Xilinx JTAG Connector.** Do not attempt to use any 2x7 ribbon cables from Amazon, as their connector pitch width will not fit into the slot. It is important to use a cable and programmer specifically intended for Xilinx JTAG.

I highly recommend the [Digilent JTAG-HS3 Programming Cable](https://digilent.com/shop/jtag-hs3-programming-cable/) as this is tested with this setup.

## Using a Different JTAG Programmer
If you do not have the Digilent JTAG-HS3 Programmer, you can modify `flash.cfg` to point to a different programmer's CFG file.

In `flash.cfg`, replace:

`source [find interface/ftdi/digilent_jtag_hs3.cfg]`

With your own board's cfg file.

You can find the name of your programmer cfg here: https://github.com/arduino/OpenOCD/tree/master/tcl/interface/ftdi

# Notes About Flashing (JTAG vs. PCIe)

These boards have two main methods for flashing a bitstream to the device. The first method is flashing over JTAG, which requires hooking up a JTAG setup as seen above. In this guide we will use OpenOCD to communicate with the SPI flash chip over JTAG. While this isn't the fastest mechanism, it is capable of flashing the default project in 2 minutes 30 seconds on my machine, which is much faster than the typical Vivado flash.

**You can also flash the device over PCIe only when a compatible bitstream is active.** A compatible bitstream is one which is specifically built to use its PCIe IP block to expose the internal SPI flash as a PCIe resource to the OS. This allows a usermode flasher application to map the on-board SPI and flash a new bitstream. The Nitefury II project comes with a [spi-loader](https://github.com/RHSResearchLLC/NiteFury-and-LiteFury/tree/master/spi-loader) project which allows flashing from a Linux machine over PCIe.

If all of your designs include the PCIe block diagram as seen in the [spi-loader](https://github.com/RHSResearchLLC/NiteFury-and-LiteFury/tree/master/spi-loader) project, then you can continue to flash new designs over PCIe for the remainder of your development time. If you flash a new bitstream that does not include this functionality, or the functionality stops working, your only fallback is JTAG. It is a good idea to always have JTAG on hand for flashing just in case.

# What is Included in this Repo

In this repo, you will find the following files:

- `bscan_spi_xc7a200t.bit` - This is a bitstream which will act as a "proxy" for OpenOCD to program the flash through the FPGA. OpenOCD will program this bitstream directly to the FPGA over JTAG, but will not save this bitstream to your flash. OpenOCD will then use this new bitstream to easily interface with the flash (which is wired directly to the FPGA, and not to JTAG). This particular bitstream is used in the Nitefury/Acorn. This *must* correspond to the chip on your board. [You can get pre-synthesized bitstreams here.](https://github.com/quartiq/bscan_spi_bitstreams)
- `pcie-chainloader-project0.bin` - This bitstream contains a PCIe IP core which exposes the SPI flash to the OS for programming. Once this bitstream is flashed, you can use [spi-loader](https://github.com/RHSResearchLLC/NiteFury-and-LiteFury/tree/master/spi-loader) over PCIe to program the flash much faster with a new bitstream. [The project file for this bitstream can be found here.](https://github.com/RHSResearchLLC/NiteFury-and-LiteFury/tree/master/Sample-Projects/Project-0/FPGA/Nitefury-II)
- `flash.cfg` - An OpenOCD configuration file (in TCL) that tells it how to program the chip over JTAG. This is configured to flash the pcie-chainloader bitstream. If you have your own bitstream `.bit` or `.bin` file, you can edit the path here.
- `flash-project0.sh` - A shell script that flashes the `pcie-chainloader-project0.bin` bitstream to the board. Copy this and modify it to flash a different bitstream. This will automatically remove drivers which might cause problems running OpenOCD. If you need these drivers, edit the script accordingly.

# How to Flash with JTAG

## == Using Linux ==

This was tested on Ubuntu 22.04 Server running in a VMWare Workstation 16 VM with USB passthrough of the JTAG debugger.

1. Clone this repo.

2. Install openocd (0.11.0 was used in this test)

   - Ubuntu/Debian: `apt install openocd`

3. Ensure the JTAG programmer is connected

   - On Linux, this is accomplished like so:

     - ```
         ❯ lsusb
         Bus 001 Device 007: ID 0403:6014 Future Technology Devices International, Ltd FT232H Single HS USB-UART/FIFO IC
         ```

   - Note: that `0403:6014` is the device vendor ID of your FTDI USB->Serial chip. If this is not `0403:6014`, these instructions will **not** work out of the box for you. See the section **Programming Cable** above.

4. If you want to flash a specific bitstream and do not care to flash the PCIe programmer bitstream, make a copy of `flash.cfg` and `flash-project0.sh` and edit the target bitstream file in `flash.cfg` to point to your new `.bit` or `.bin`. Then, edit your new `flash-project0.sh` script to use that new `.cfg` file instead. Run that script instead.

5. Run `chmod +x ./flash-project0.sh`

6. Run `sudo ./flash-project0.sh`

   - You should hopefully get an output like so:

   - ```
     ❯ ./flash-project0.sh
     Tue Sep 27 03:47:55 AM UTC 2022
     rmmod: ERROR: Module ftdi_sio is not currently loaded
     rmmod: ERROR: Module usbserial is not currently loaded
     Open On-Chip Debugger 0.11.0
     Licensed under GNU GPL v2
     For bug reports, read
             http://openocd.org/doc/doxygen/bugs.html
     Info : auto-selecting first available session transport "jtag". To override use 'transport select <transport>'.
     Info : clock speed 1000 kHz
     Info : JTAG tap: xc7.tap tap/device found: 0x13636093 (mfg: 0x049 (Xilinx), part: 0x3636, ver: 0x1)
     Writing BSCAN_SPI bitstream...
     Info : JTAG tap: xc7.tap tap/device found: 0x13636093 (mfg: 0x049 (Xilinx), part: 0x3636, ver: 0x1)
     Info : Found flash device 'sp s25fl256s' (ID 0x00190201)
     Warn : device needs paging or 4-byte addresses - not implemented
     Programming PCIe Chainloader bitstream to SPI...
     Info : Found flash device 'sp s25fl256s' (ID 0x00190201)
     Warn : device needs paging or 4-byte addresses - not implemented
     Info : Found flash device 'sp s25fl256s' (ID 0x00190201)
     Warn : device needs paging or 4-byte addresses - not implemented
     Info : Found flash device 'sp s25fl256s' (ID 0x00190201)
     Warn : device needs paging or 4-byte addresses - not implemented
     Info : sector 0 took 5 ms
     [...]
     Info : Found flash device 'sp s25fl256s' (ID 0x00190201)
     Warn : device needs paging or 4-byte addresses - not implemented
     Resetting device...
     Done.
     shutdown command invoked
     Tue Sep 27 03:50:21 AM UTC 2022
     ```

   - This will take approximately **3 minutes.** It might not display anything for long periods of time.

If all goes well, the new bitstream is flashed! Congratulations! 

## == Using Windows ==

- Not yet tested, so no instructions here. However, it should be fairly easy to run a similar workflow using the same openocd configuration file and the Windows version of openocd. You can also use a VMWare VM with USB Passthrough, as the above instructions are tested there. Feel free to put in a PR with instructions with running native Windows if you've got them!

# How to Flash with PCIe

Flashing with PCIe is done using the [spi-loader](https://github.com/RHSResearchLLC/NiteFury-and-LiteFury/tree/master/spi-loader) project. Instructions can be found in that repo.

Currently, there is no Windows workflow capable of flashing over PCIe. If your machines are all Windows machines, you will need to use JTAG, either with OpenOCD or Vivado.



