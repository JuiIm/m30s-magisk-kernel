# Test features (branches `wifi-monitor`, `bluetooth`, `pentest-features`)

Everything in this file is **compile tested only**. None of it has run on a
phone yet. `unteth` does not contain any of it.

Each branch contains the one before it:
`wifi-monitor` < `bluetooth` < `pentest-features`.

| section | what | branch |
| --- | --- | --- |
| 1 | phone WiFi: monitor mode, channel hopping, injection | wifi-monitor, more in pentest-features |
| 2 | external USB WiFi adapters | wifi-monitor, more in pentest-features |
| 3 | Bluetooth | bluetooth |
| 4 | USB gadget: mass storage / CD-ROM, more than one instance | pentest-features |
| 5 | Linux chroot and network building blocks, CAN bus, file systems | pentest-features |

## 1. Phone WiFi (Samsung SCSC chip): monitor mode

The driver already had monitor capture code; these branches add a way to use
it without `iw`, a default channel, and an experimental injection path.
Whether the WiFi firmware accepts monitor mode or injected frames is not known.

Turn WiFi off in Android first (also "WiFi scanning" in the location settings).

```
su
echo Y  > /sys/module/scsc_wlan/parameters/monitor_mode
echo 6  > /sys/module/scsc_wlan/parameters/monitor_channel
ip link set wlan0 down
ip link set wlan0 up
dmesg | grep -i monitor | tail
tcpdump -i wlan0 -c 20          # tcpdump: Termux root-repo, or a chroot
```

Back to normal WiFi:

```
echo N > /sys/module/scsc_wlan/parameters/monitor_mode
ip link set wlan0 down
```

then turn WiFi on in Android.

With `iw` the usual way works too (`iw dev wlan0 set type monitor`,
`iw dev wlan0 set channel 11`).

Change the channel while it runs, or let the kernel hop (`pentest-features`):

```
echo 11  > /sys/module/scsc_wlan/parameters/monitor_channel   # retunes at once
echo 250 > /sys/module/scsc_wlan/parameters/monitor_hop_ms    # hop every 250 ms
echo Y   > /sys/module/scsc_wlan/parameters/monitor_hop_5g    # include 5 GHz
echo 0   > /sys/module/scsc_wlan/parameters/monitor_hop_ms    # stop hopping
```

When the firmware gives no radiotap header, the one the driver adds carries
the channel the interface is tuned to.

What the kernel log tells you:

| line | meaning |
| --- | --- |
| `monitor mode started` | the firmware accepted the monitor interface |
| `monitor: the firmware refused to start a monitor interface` | no monitor mode with this firmware |
| `monitor: listening on channel N` / `refused channel N` | channel setup worked / failed |
| `monitor: the firmware delivers frames with / without a radiotap header` | first frame arrived |

Injection (experimental): frames written to the monitor interface are passed
to the firmware. Counters:

```
cat /sys/module/scsc_wlan/parameters/monitor_tx_sent
cat /sys/module/scsc_wlan/parameters/monitor_tx_refused
cat /sys/module/scsc_wlan/parameters/monitor_tx_dropped
```

After 8 refused frames in a row injection switches itself off until the
interface is brought up again. `echo N > .../monitor_inject` turns it off.

## 2. External USB WiFi adapters

Drivers are loadable modules (they are not in `Image`): `ath9k_htc`,
`rt2800usb`, `mt7601u`, `rtl8187`, `88XXau` (RTL8812AU/8821AU/8814AU),
`carl9170`, `rt73usb`, `rt2500usb`, on top of `mac80211`. Their firmware is
built into the kernel image.

Build and collect them:

```
make ... modules
make ... INSTALL_MOD_PATH=<dir> INSTALL_MOD_STRIP=1 modules_install
```

Load order: `ccm`, (`cmac` if it is a module), `mac80211`, then the driver's
own dependencies from `modules.dep`, then the driver. Modules only load on
the exact kernel build they were made with.

In `pentest-features`:

- `rtl8xxxu` is back, with RTL8188EU support (TP-Link TL-WN722N v2/v3) from
  the Kali NetHunter patch set.
- `88x2bu`: Realtek RTL8812BU / RTL8822BU 802.11ac adapters, from the RinCat
  vendor driver (monitor mode and injection).
- mac80211 has the NetHunter injection patch: injected frames keep their
  sequence number and QoS header, and the monitor channel can be changed
  while another interface of the adapter is up.
- The regulatory database is built in (wireless-regdb 2017-12-23), so
  `iw reg set US` (or your country) works without CRDA. Without it every
  adapter is stuck in the restrictive world domain. The phone's own WiFi uses
  its own rules and is not affected.

## 3. Bluetooth (`bluetooth` branch)

The kernel Bluetooth stack (BlueZ side: HCI sockets, L2CAP, RFCOMM, BNEP,
HIDP) is built in. Android's own Bluetooth does not use it and is not changed.

### 3a. The phone's own Bluetooth chip as `hci0`

Samsung's driver has a bridge to the kernel Bluetooth stack that was never
enabled. It is enabled here and fixed to compile and not to use freed memory.
It does nothing until you switch it on. The chip has one user at a time:
**turn Bluetooth off in Android first.**

```
su
echo 1 > /sys/module/scsc_bt/parameters/power_control     # hci0 appears
ls /sys/class/bluetooth/
```

Then, from a Linux chroot with BlueZ tools (NetHunter for example):

```
hciconfig hci0 up
hciconfig -a
hcitool scan
btmon                      # trace every HCI packet
hcitool cmd 0x03 0x0003    # send any raw HCI command (this one: reset)
```

Raw HCI access is the Bluetooth counterpart of injection here: any HCI
command, vendor command or ACL packet can be sent to the controller. It does
not give control over the radio below HCI (no arbitrary link-layer packets);
that would need different firmware.

Switch it off again before using Android Bluetooth:

```
hciconfig hci0 down
echo 0 > /sys/module/scsc_bt/parameters/power_control
```

If `hciconfig hci0 up` fails, try the raw mode: the kernel then sends no
commands of its own and only tools that use the HCI user channel work.

```
echo 0 > /sys/module/scsc_bt/parameters/power_control
echo Y > /sys/module/scsc_bt/parameters/raw_device
echo 1 > /sys/module/scsc_bt/parameters/power_control
```

### 3b. External USB Bluetooth adapters and sniffers

Built in: `btusb` (generic USB Bluetooth adapters, with Broadcom and Realtek
setup code), `hci_uart` (H4, for serial adapters via `hciattach`) and
`hci_vhci` (virtual controllers). A plugged adapter shows up as the next
`hciN`. Realtek and some Broadcom adapters want a firmware file that Android
does not ship; plain CSR adapters need none.

USB serial drivers for common BLE sniffer dongles are also built in:
`cdc_acm` (nRF Sniffer, most CDC boards), `cp210x`, `ch341` (FTDI and PL2303
were already there). They show up as `/dev/ttyACM0` or `/dev/ttyUSB0`.

In `pentest-features` more Bluetooth USB adapters that need a firmware download
at plug-in time are covered (`ath3k`, `bcm203x`, `bpa10x`, `bfusb`), the BCSP
UART protocol, and the TP-Link UB500 USB ID.

## 4. USB gadget additions (`pentest-features`)

- The mass storage function is enabled (`CONFIG_USB_CONFIGFS_MASS_STORAGE`):
  the phone can be a USB stick or a CD-ROM drive backed by an image file.
- Samsung's function drivers gave every instance a sysfs device with a fixed
  name and never removed it. A second `mass_storage` or `midi` instance (for
  example in a second gadget) failed, and so did creating one again after
  removing it. Further instances now get `<name>.1`, `<name>.2`, ... and the
  device goes away with the instance.

`gadget.sh` (in the modules package) wraps the common cases:

```
sh gadget.sh hid                    # keyboard + mouse
sh gadget.sh type "hello"           # type through that keyboard
sh gadget.sh cdrom /sdcard/x.iso    # CD-ROM with an image
sh gadget.sh disk  /sdcard/x.img    # USB stick with an image
sh gadget.sh net                    # USB network card (RNDIS + ECM)
sh gadget.sh off                    # back to the phone's normal USB
```

## 5. Linux chroot, networking, CAN, file systems (`pentest-features`)

Built in:

- `SYSVIPC` (System V shared memory, semaphores, message queues). PostgreSQL
  needs it, so Metasploit's database works in a chroot.
- `BINFMT_MISC`.

Modules (in the modules package, `sh load.sh <group>`):

| group | modules |
| --- | --- |
| `net` | bridge (+ br_netfilter), veth, macvlan, 8021q, dummy, rndis_host, lan78xx |
| `can` | can, can-raw, can-bcm, can-gw, vcan, slcan, gs_usb, peak_usb, kvaser_usb, usb_8dev, ems_usb, esd_usb2, mcba_usb |
| `fs` | squashfs, isofs, udf, cifs, ntfs |

## If something goes wrong

Send the output of `dmesg | tail -80` taken right after the failing command.
