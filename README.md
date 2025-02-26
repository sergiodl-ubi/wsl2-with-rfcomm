# wsl2-with-rfcomm
Instructions for enabling Bluetooth's Classic RFCOMM port on WSL2's Ubuntu from internal or external USB's bluetooth adapters.

## Setup

* Ubuntu 24.04 base
* WSL2-Linux-Kernel
  * linux-msft-wsl-6.6.y branch
* Dell Inspiron 3505
  * Qualcomm Bluetooth Wifi Combo: QCA9377 802.11ac
    * [Qualcomm Atheros Bluetooth Device: 0cf3:e009](https://linux-hardware.org/?id=usb:0cf3-e009)
 
## Prerequisites 

* WSL2 installed and running
* usbipd
  * [Windows Knowledge Database](https://learn.microsoft.com/en-us/windows/wsl/connect-usb)
  * [Project's Github](https://github.com/dorssel/usbipd-win)
    * Find and bind (share) your USB Bluetooth adapter according to instructions.

## Procedure

### Build a custom kernel with BTQCA driver enabled and built-in firmware

1. The default kernel has no USB Bluetooth device's complete compatibility enabled. I proceeded to embed the drivers into the kernel, but having them compiled as modules might work too.
2. Install the packages needed to build the kernel as per [WSL2-Linux-Kernel readme](https://github.com/microsoft/WSL2-Linux-Kernel).
  * Other tutorials might have not included `libncurses-dev` for `make menuconfig` or `cpio`, so try to follow updated instructions.
3. Install linux firmware package and zstd compressing tool: "sudo apt install linux-firmware zstd"
  * For Qualcomm's QCA9377 chipset, if you don't include the firmware binaries, you can see in `dmesg` that the kernel will fail to load both "qca/rampatch_usb_00000302.bin" and "qca/nvm_usb_00000302.bin" with a -2 error ([POSIX ENOENT: No such file or directory](https://en.wikipedia.org/wiki/Errno.h)).
  * If you are using a different chipset, build your kernel first with bluetooth capability and then check in dmesg logs for the needed firmware. You will need to rebuild the kernel later to included the needed firmware. Or be lucky to find someone else on internet that already knows which binaries are related to your adapter.
  * Go to `/usr/bin/firmware` and unpack the needed binaries:
    * sudo zstd -d rampatch_usb_00000302.bin.zst
    * sudo zstd -d nvm_usb_00000302.bin.zst 
3. Clone to your WSL2 Ubuntu instance the kernel sources: `$ git clone --branch linux-msft-wsl-6.6.y https://github.com/microsoft/WSL2-Linux-Kernel --progress --depth 1`
  * You can experiment with other kernels but check driver compatibility first.
4. Enter the kernel's directory and edit the configuration: `$ make menuconfig KCONF_CONFIG=Microsoft/config-wsl`. You can search for options with the `/` key.
5. For Qualcomm's Atheros driver (btqca) it seems it needs at least these modules enabled:
  * CONFIG_BT_QCA=m
  * CONFIG_BT_HCIUART=m
  * CONFIG_BT_HCIUART_QCA=y
  * But I further baked in the RFCOMM module. 
    * CONFIG_BT=y
    * CONFIG_BT_RFCOMM=y
  * You will need to built-in the drivers binaries, so set the following string:
    * CONFIG_EXTRA_FIRMWARE="qca/rampatch_usb_00000302.bin qca/nvm_usb_00000302.bin"
  * And modify the firmware dir since it's different on this distribution:
    * CONFIG_EXTRA_FIRMWARE_DIR="/usr/lib/firmware"
6. Make the kernel: `$ make -j$(nproc) KCONFIG_CONFIG=Microsoft/config-wsl`.
7. Install modules and headers: `$ sudo make modules_install headers_install`.
8. Copy the new kernel into a known location: `$ cp arch/x86/boot/bzImage /mnt/c/Users/YourWinUserName/`
9. Create a custom configuration for your wsl: `$ vim /mnt/c/Users/YourWinUserName/.wslconfig`
```
    [wsl2]
    kernel=C:\\Users\\YourWinUserName\\bzImage
```
10. Open an administrator Window's console (Powershell or cmd) and shutdown your wsl2: `> wsl --shutdown`

12. Start again your WSL2 Ubuntu terminal. And monitor your system's logs at the moment you attach your Bluetooth adapter to wsl: `$ dmesg -Tw`
13. On the Window's console attach your Bluetooth adapter with usbipd: `> usbipd attach --wsl --busid=ADAPTERID`
  * BUSID's can be seen with `usbipd list`.
  * Remember to first bind the adapter so it can be shared with WSL2. Read the current instructions from the developer since they may change.
13. If the dmesg log shows that the adapter has been currectly identified and loaded, you can proceed to enable the RFCOMM port.

### Enable RFCOMM serial service

14. Clone the following bluetooth simulator `$ git clone https://github.com/ubi-naist/BluetoothSerialTest`
15. Copy or add the changes in `BluetoothSerialTest/systemd/system/bluetooth.target.wants/bluetooth.service` to your local `/etc/systemd/system/bluetooth.target.wants/bluetooth.service` systemd service file.
16. Copy `BluetoothSerialTest/systemd/system/rfcomm.service` to `/etc/systemd/system/`
17. Reload systemd daemon: `$ sudo systemctl daemon-reload`
18. Restart your bluetooth service: `$ sudo systemctl restart bluetooth.service`
19. Enable rfcomm service if you want it to automatically start: `$ sudo systemctl enable rfcomm.service`
20. Start rfcomm service: `$ sudo systemctl start rfcomm.service`
21. Confirm rfcomm service has started successfully: `$ systemctl status rfcomm`
22. You can test the port by using applications such as "Serial Bluetooth Terminal"
  * For this you will first pair your phone with your computer:
    * First run `$ bluetoothctl` and write `[bluetooth]# discoverable yes`
    * Try to connect from your phone, on success it will prompt you to confirm the pairing code on the console.
      * If it fails to connect, you can also try to pair from `bluetoothctl` side: set `scan on` copy the mac address of your phone and pair with `pair MACADDRESS`.
  * On the Serial Bluetooth Terminal app, go to Devices and select the paired computer. It should connect and the prompt from `bluetoothctl` should change from "[bluetooth]#" to "[yourphonealias]#" and show a notification saying that you have connected successfully.
23. Usually only after a client has successfully connected to your host, the "/dev/rfcomm0" serial port will actually appear and be available for communication.
24. This communication can be tested with "Serial Bluetooth Terminal" and the script on the `BluetoothSerialTest` repository through the `$ python3 blueserialtest.py` script after properly setting up the environment.
     
