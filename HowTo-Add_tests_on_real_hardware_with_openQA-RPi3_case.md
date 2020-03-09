
# Add tests on real hardware with openQA - RPi3 case

Here is a small guide to setup hardware and software in an existing openQA setup.
As pre-requires, you need to have an openQA server up and running, with an existing openQA worker.

## Hardware requirements and setup

Here, we will use a Raspberry Pi 3 B (or B+) as a System-Under-Test (SUT), but any ARM system able to boot from USB can be used.
We will use an armv7 SabreLite system as a SUT-companion, but any system with a working USB OTG/device port to emulate a USB storage (and a USB host to connect a USB/serial cable) can be used.
The SUT-companion is used to expose USB storage to SUT and grab serial line from SUT.

![alt text](https://paste.opensuse.org/view/raw/13032363 "Hardware setup")

**Hardware list**:

  * **SUT**: here, Raspberry Pi 3 with a regular power supply
  * **SUT-companion** (flasher): here, SabreLite with a regular power supply
  * **USB to TTL serial cable** to grab serial line from SUT
  * **USB/serial cable** to grab serial line from SUT-companion (optionnal, but can help if system is not responding on Network anymore)
  * **USB cable** to connect SUT (USB host) and SUT-companion (USB device)
  * **uSD card** for the Raspberry Pi to hold firmware due to [https://github.com/raspberrypi/firmware/issues/1322](https://github.com/raspberrypi/firmware/issues/1322) - In the end, should only requires bootcode.bin to chain load USB boot.
  * **uSD card** for SUT-companion with JeOS running on it.
  * **Fast external storage** for SUT-companion to hold the image to be exposed on USB (to avoid to kill the uSD card quickly).
  * **2 Smart PowerPlug / PowerStrip**: to power on/off the SUT (and the SUT-companion). I used a [TP-Link HS100](https://www.tp-link.com/en/home-networking/smart-plug/hs100/) with [python3-kasa package](https://build.opensuse.org/package/show/devel:languages:python/python-kasa). I also used a Tenma 72-2535 which is a programmable bench power supply, with [python tenma-serial](https://github.com/kxtells/tenma-serial) software.
  * **2 Ethernet Cables** to connect SUT and SUT-companion to a switch

Now you have all the hardware parts, plug them as shown above on the diagram.


## Software setup

### SUT-companion

Once you have installed an openSUSE image on your device (here, JeOS-sabrelite), you need to configure it.
First, please create ` /var/lib/openqa/share/` folder and mount nfs share from openQA server. See: [http://open.qa/docs/#_configuring_remote_workers](http://open.qa/docs/#_configuring_remote_workers)
That way, you will have access to files used later (scripts, etc.).

Note: Currently, it uses 'root' user on SUT, but we should probably use a specific non-root user, with some additional permissions to mount images on USB to limit security risks.


### SUT

Here, the Raspberry Pi 3 B cannot boot from USB emulation (see [https://github.com/raspberrypi/firmware/issues/1322](https://github.com/raspberrypi/firmware/issues/1322)), so you need to format the uSD card in FAT and copy the firmware files (bootcode.bin, fixup.dat/fixup4.dat and start.elf/start4.elf) from raspberrypi-firmware RPM package.
DO NOT include the configuration files, nor u-boot!

### Worker setup

You need to setup an openQA worker as usual, but with some additional steps.
First, you need to make sure `icewm-default` and `xterm-console` packages are installed as `generalhw` backend requires them.
Then, create SSH keys for user used to run tests (_openqa-worker if run from openQA).
You need to configure a password-less SSH access from openQA worker to SUT-companion. For this, as user used to run tests (_openqa-worker if run from openQA) execute the following command from openQA worker `ssh-copy-id -i ~/.ssh/id_rsa.pub root@<SUT_IP>` (replace <SUT_IP> by the actual SUT IP adress.)
In the worker config file `/etc/openqa/workers.ini`, add:

```
[global]
# https://github.com/os-autoinst/openQA/pull/599
GENERAL_HW_CMD_DIR = /var/lib/openqa/share/tests/opensuse/data/generalhw_scripts

# Worker 1 specific config
[1]
# Config below is SUT/SUT-companion specific and must not be shared across multiple workers
FLASHER_IP = 192.168.0.18
GENERAL_HW_FLASH_ARGS = 192.168.0.18:/tmp/
GENERAL_HW_FLASH_CMD = flash_usb_otg.sh
GENERAL_HW_POWEROFF_CMD = power_off_kasa.sh
GENERAL_HW_POWEROFF_ARGS = 192.168.0.100
GENERAL_HW_POWERON_CMD = power_on_kasa.sh
GENERAL_HW_POWERON_ARGS = 192.168.0.100
GENERAL_HW_SOL_ARGS = 192.168.0.18
GENERAL_HW_SOL_CMD = get_sol_over_SSH.sh
SUT_IP = 192.168.0.44
# keep ability to run qemu test, but add a WORKLER_CLASS specific to the current openQA 'MACHINE'
WORKER_CLASS = qemu_aarch64,generalhw_RPi3B
WORKER_HOSTNAME = 192.168.0.28
```

You will probably need to update the apparmor profile of openQA worker, depending on the scripts you will use to power on/off and to flash the system.

### openQA webUI

On the openQA webUI, you need to create a new machine `RPi3B` with the required settings:

```
BACKEND=generalhw
EXCLUDE_MODULES=bootloader_uefi,diskusage,hostname,yast2_lan,console_reboot,gpt_ptable,sshd,ssh_cleanup,pam,yast2_lan_device_settings
SERIALDEV=ttyS0
SSH_XTERM_WAIT_SUT_ALIVE_TIMEOUT=240
TIMEOUT_SCALE=2
VNC_TYPING_LIMIT=7
WORKER_CLASS=generalhw_RPi3B
_COMMENT=GENERAL_HW_* are defined in worker config.

```

**Now, you can schedule some tests from openQA on this new machine.**
