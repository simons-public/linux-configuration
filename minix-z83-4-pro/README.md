# MINIX Z83-4 Pro
Product Page: [minix.com.hk](http://minix.com.hk/products/neo-z83-4-pro)  
Amazon Page: [MINIX NEO Z83-4 Pro](https://amzn.com/B0749NR2KJ)  
Manufacturer: MINIX  
Product Name: Z83-4 Pro  
Version: V1.1  

## Use case
Works great for a simple web-browser/video viewer. Streams video from Netflix/Emby with no issues and supports 4K resolution. Quiet and passively cooled. Uses 12VDC input, so may work good as a carpc. Comes with a VESA mount for mounting on the back of a monitor.

## Specifications
Processor: Intel X5-Z8350 (64-bit)  
GPU: Intel HD Graphics  
Video Connectors: HDMI, Mini DisplayPort  
Memory: 4GB DDR3L  
Storage: 32GB eMMC 5.0  
Ethernet: Realtek RTL8111/8168/8411 PCI Express  
WiFi: 802.11ac Dual-Band  
Bluetooth: 4.2  
Micro SD card reader  
USB: 3x USB 2.0, 1x USB 3.0  

# Notes & Issues on installing Arch Linux

## Booting Install CD
The F11 key is used at boot to enter the UEFI boot menu and boot Arch Linux install media from USB.

Booting using Arch Linux UEFI results in a brief display of systemd starting services, followed by a blank screen. Adding `nomodeset` to kernel parameters allows system to boot into install shell. With `nomodeset` graphics acceleration is disabled, but the correct options can be set in grub after installing.

## Partitioning
The EFI partition is on `/dev/mmcblk0p1`. The other partitions {`mmcblk0p2`, `mmcblk0p3`, `mmcblkp3`} are used by the pre-installed Windows 10 Pro. Deleting p2-p4 and creating a single Linux partition formatted with ext4 is probably the best choice for such a small storage device.

## Bootloader
With `/dev/mmcblk0p1` mounted at `/boot` booting the kernel EFISTUB probably works fine, but GRUB also works and is a little easier to configure. Setting up `/etc/defaults/grub` and running `grub-mkconfig -o /boot/grub/grub.cfg` works with the following options set:
```shell
GRUB_CMDLINE_LINUX_DEFAULT="quiet video=HDMI-A-1:1920x1080@60D"
GRUB_GFXMODE=auto
GRUB_GFXPAYLOAD_LINUX=keep
```

## Video
When using the main HDMI output the onboard Intel video is recognized by the kernel but cannot be setup properly by KMS. The `xrandr` utility reports `HDMI1 disconnected` despite a monitor being connected. Using `xrandr` to manually add modelines and force output on HDMI1 works, but is not persistant. 

The [Linux Documentation](https://github.com/torvalds/linux/blob/master/Documentation/fb/modedb.txt) details a way to force a display to be connected. Adding the kernel option `video=HDMI-A-1:1920x1080@60D` works for forcing the display to be enabled and use digital output, as well as correctly enabling the example resolution (1920x1080) to be detected by Xorg when using the `xf86-video-intel` driver.

## Networking
Ethernet works out of the box. Wifi requires firmware. From the [linuxiumcomau.blogspot.com](http://linuxiumcomau.blogspot.com/2017/12/linux-on-minix-neo-z83-4-and-z83-4-pro.html) website, the firmware can be retrieved from `C:\Windows\System32\drivers\4345r6nvram.txt` on the Windows 10 partition which could be mounted from `/dev/mmcblk0p3`. 

There an easier way with the AUR package [bcm4356a2-firmware](https://aur.archlinux.org/packages/bcm4356a2-firmware/). It uses drivers [from asus](http://dlcdnet.asus.com/pub/ASUS/wireless/USB-BT400/DR_USB_BT400_1201710_Windows.zip), converts the `.hex` files using `hex2hcd` and installs `BCM4356A2-*.hcd` in `/usr/lib/firmware/brcm/`. The drivers should be able to be used by the `brcmfmac` module.

After installing drivers (and `dialog`), wifi-menu works properly, and the created wifi profile can be enabled with `netctl enable PROFILE` and started with `netctl start PROFILE`. This might fail the first time because `wlan0` is up, which can be fixed by running `ip link set wlan0 down` first.

## X11 Setup
The onboard Intel HD graphics works with `xf86-video-intel`. Alternatively, the `xf86-video-fbset` driver can be used with `nomodeset` if video acceleration is not required. Installing `xf86-video-vesa` causes Xorg to crash.

## Firmware
The AUR packages `aic94xx-firmware` and `wd719x-firmware` can be installed to suppress warnings when using `mkinitcpio`, but are not required. The `intel-ucode` package should be installed and will be recognized by `grub-mkconfig` for installing (microcode)[https://wiki.archlinux.org/index.php/microcode] updates.

## Bluetooth
Works out of the box with `bluez` and `bluez-utils`.

## Sound
From (linuxiumcomau.blogspot.com)[http://linuxiumcomau.blogspot.com/2017/12/linux-on-minix-neo-z83-4-and-z83-4-pro.html] ALSA sound can be fixed by adding a definition for the [ALSA Use Case Manager](https://www.systutorials.com/docs/linux/man/1-alsaucm/). Put the `HiFi.conf` file from this repo in the `/usr/share/alsa/ucm/chtrt5645/` directory. PulseAudio sound worked with no additional configuration.

## Powersaving
### Battery
In KDE the power tray icon reports a battery status. This is because of the `axp288` module recognizing [AXP299 Power Management Integrated Circuit](https://datasheetspdf.com/pdf-file/938454/X-Powers/AXP288/1) as having a battery. This can be fixed by blacklisting the `axp288_fuel_gauge` module.  
`/etc/modprobe.d/10-nobattery.conf`:
```shell
# Do not load the axp288_fuel_gauge module since device has no battery
blacklist axp288_fuel_gauge
```

### Resuming
Resuming after sleep puts Xorg back into 1024x768 resolution. A systemd `resume@user.service` can be used to run the appropriate `xrandr` commands to restore resolution.  
`/etc/systemd/system/resume@.service`:
```shell
# Restore resolution after sleeping

[Unit]
Description=User resume restore resolution
After=suspend.target

[Service]
User=%I
Type=oneshot
Environment=DISPLAY=:0
ExecStart=/usr/bin/xrandr --newmode "1920x1080_60.00"  173.00  1920 2048 2248 2576  1080 1083 1088 1120 -hsync +vsync
ExecStart=/usr/bin/xrandr --addmode HDMI1 1920x1080_60.00
ExecStart=/usr/bin/xrandr --output HDMI1 --mode 1920x1080_60.00

[Install]
WantedBy=suspend.target
```
The service can be enabled with `systemctl enable resume@USERNAME.service`

## Managing Storage
Periodic cleaning of the pacman cache can help manage disk storage.  
`/etc/systemd/system/clean-pacman-cache.service`:
```shell
[Unit]
Description=Clean old pkg files from cache
After=multi-user.target

[Service]
Type=oneshot
ExecStart=/bin/sh -c 'yes | pacman -Sc'
```
`/etc/systemd/system/clean-pacman-cache.timer`:
```shell
[Unit]
Description=Clean old pkg files from cache weekly
After=time-sync.target

[Timer]
OnCalendar=weekly
Persistent=true

[Install]
WantedBy=timers.target
```
`systemctl enable clean-pacman-cache.timer`
