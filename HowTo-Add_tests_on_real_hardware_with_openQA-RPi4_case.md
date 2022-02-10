
# Add tests on real hardware with openQA - RPi4 with USB-SD-Mux

Here is a small guide to setup hardware and software in an existing openQA setup to test real hardware (headless).
As pre-requires, you need to have an openQA server up and running, with an existing openQA worker.

## Hardware requirements and setup

Here, we will use a Raspberry Pi 4 as a System-Under-Test (SUT), but any Arm system able to boot from uSD card can be used.
We will use an [USB-SD-mux](https://shop.linux-automation.com/usb_sd_mux-D02-R01-V02-C00) to flash the uSD card used to boot the SUT, but other solutuons exists, such as SDWire or SD_Mux.

![alt text](RPi4_USB-SD-Mux.png "Hardware setup diagram")

**Hardware list**:

  * **SUT**: here, Raspberry Pi 4 with a regular 230V/5V power supply (connected to a smart PowerStrip)
  * **Flasher**: [USB-SD-mux](https://shop.linux-automation.com/usb_sd_mux-D02-R01-V02-C00) from Linux Automation
  * **uSD card**: for SUT to flash the image to test
  * **Smart PowerPlug / PowerStrip**: to power on/off the SUT
  * **Ethernet Cable**: to connect SUT to a switch
  * **USB to TTL serial cable** (optional): to grab serial line from SUT

Now you have all the hardware parts, plug them as shown on the diagram above.


## Software setup


### SUT

For the Raspberry Pi 4, please make sure latest firmware is flashed in EEPROM.


### openQA worker setup

You need to setup an openQA worker as usual, but with some additional steps.

First, you need to make sure `icewm-default` and `xterm-console` packages are installed as `generalhw` backend requires them.
Also, you need the software used by your power on/off scripts. Currently the following power on/off are supported:

  * `power_on_kasa.sh`/`power_off_kasa.sh`: requires ` python38-kasa` for `kasa`. Tested with [TP-Link HS100](https://www.tp-link.com/en/home-networking/smart-plug/hs100/)
  * `power_on_tenma.sh`/`power_off_tenma.sh`: requires ` python`. Tested with a Tenma 72-2535 which is a programmable bench power supply.
  * `power_on_off_shelly.sh` : requires `curl`
  * `power_on_off_tuya.sh`: requires `nodejs-common` for `node`. Tested with [Konyks Polyco](https://konyks.com/produit/polyco/).
  * `webhook_ifttt.sh`: requires `curl`. Tested with [Konyks Polyco](https://konyks.com/produit/polyco/) when Konyks products were still supported by IFTTT (before May 2020).

Dependencies for the flash script:

  * `flash_sd.sh` script:
    * requires `usbsdmux` for USB-SD-mux from Linux Automation (currently in use for Rpi 2, 3 and 4 on openqa.opensuse.org)
    * requires `sd-mux-ctrl` for SDWire, SD_Mux, or any other compatible hardware (untested)
  * `flash_usb_otg.sh`: Not covered by this guide as the setup is different. See: [HowTo-Add_tests_on_real_hardware_with_openQA-RPi3_case.md](https://github.com/ggardet/blog/blob/master/HowTo-Add_tests_on_real_hardware_with_openQA-RPi3_case.md) for details.


Finally, in the worker config file `/etc/openqa/workers.ini`, add:

```
[global]
GENERAL_HW_CMD_DIR = /var/lib/openqa/share/tests/opensuse/data/generalhw_scripts
WORKER_HOSTNAME = 192.168.0.30

# Worker 4 specific config
[4]
GENERAL_HW_FLASH_ARGS = usbsdmux localhost:/tmp/4 000000000427 disk/by-id/usb-LinuxAut_sdmux_HS-SD_MMC_000000000427-0:0
GENERAL_HW_FLASH_CMD = flash_sd.sh
GENERAL_HW_POWEROFF_CMD = power_on_off_tuya.sh
GENERAL_HW_POWEROFF_ARGS = <ID> <key> <index> off
GENERAL_HW_POWERON_CMD = power_on_off_tuya.sh
GENERAL_HW_POWERON_ARGS = <ID> <key> <index> on
SUT_IP = 192.168.0.54
WORKER_CLASS = generalhw_RPi4
#GENERAL_HW_SOL_CMD = get_sol.sh
```

You will likely need to update the apparmor profile of `/usr/share/openqa/script/worker`, depending on the scripts you will use to power on/off and to flash the system. An easy way to proceed, is to switch AppArmor to complain mode.


### openQA webUI

On the openQA webUI, you need to create a new machine `RPi4` with the required settings:

```
BACKEND=generalhw
EXCLUDE_MODULES=bootloader_uefi,diskusage,hostname,yast2_lan,console_reboot,gpt_ptable,sshd,ssh_cleanup,pam,yast2_lan_device_settings,journalctl,glibc_locale
SERIALDEV=ttyS0
SSH_XTERM_WAIT_SUT_ALIVE_TIMEOUT=240
TIMEOUT_SCALE=3
VNC_TYPING_LIMIT=7
WORKER_CLASS=generalhw_RPi4
_COMMENT=GENERAL_HW_* are defined in worker config

```

**Now, you can schedule some tests from openQA on this new machine.**
