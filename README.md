# mbp143-wifi

Wi-Fi for the 15" MacBook Pro of 2017 (MacBookPro14,3) on Linux, with
packages for Ubuntu and for Arch/CachyOS.

The machine's Wi-Fi chip, a Broadcom BCM43602, runs on Linux with the kernel
driver brcmfmac and linux-firmware's firmware for the chip. That firmware
needs a board file: the settings and calibration for the chip on this
particular board. Linux has none for the 14,3, and linux-firmware's firmware
lacks the table that drives this board's front end (the switches and
amplifiers between the chip and its antennas). The result is a weak link,
or none at all on 2.4 GHz, and often no 5 GHz networks in the list. The
board file brcmfmac looks for on this machine,
`brcm/brcmfmac43602-pcie.Apple Inc.-MacBookPro14,3.txt`, is in no
distribution.

This package fixes both, on the machine it is installed on:

- It reads the factory calibration of the machine's own Wi-Fi chip and
  writes the board file from it.
- It writes a firmware: linux-firmware's, with Apple's front-end table for
  this board and the chip setup macOS does for it.
- It gives the machine a MAC address of its own, and moves the connection
  to 5 GHz when the same network is there.

Sound on the same machine (speakers, headset microphone):
[mbp143-audio](https://github.com/dnlbtz/mbp143-audio).

## How it performs

The same MacBookPro14,3, the same spot, the same access point, measured
against a computer on the wired network. Two runs in the order without,
with, with, without; at each load 3 x 10 s of TCP each way. Mbit/s:

| | without | with mbp143-wifi |
|---|---|---|
| 5 GHz signal | -72 to -69 dBm | -59 to -56 dBm |
| 5 GHz upload | 99 / 165 / 182 | 275 / 289 / 301 |
| 5 GHz download | 208 / 225 / 238 | 297 / 325 / 335 |
| 2.4 GHz signal | -81 to -80 dBm | -55 to -53 dBm |
| 2.4 GHz upload | 10 / 16 / 25 | 39 / 54 / 65 |
| 2.4 GHz download | 8 / 13 / 15 | 67 / 79 / 86 |

5 GHz on an 80 MHz channel, 2.4 GHz on a 20 MHz channel; lowest, median
and highest of 12 measurements each (2026-11-18).

"Without" is linux-firmware's firmware with a board file for the
MacBookPro13,3, the best that existed for this machine before. Signal in
dBm: closer to 0 is
stronger, 10 dB is ten times the power.

## Supported hardware

The MacBookPro14,3 only (15", 2017, Touch Bar):

    $ cat /sys/class/dmi/id/product_name
    MacBookPro14,3

The front-end table and the board file template are for this board. The
package checks for the model and for the BCM43602 (PCI `14e4:43ba`) and
changes nothing on any other machine.

Tested with: the Arch package on a MacBookPro14,3 under CachyOS (kernel
7.3); the Ubuntu package's install, upgrade and removal on Ubuntu 26.04,
and its setup on the same MacBookPro14,3. Not yet installed on a 14,3
running Ubuntu.

## Install

Download the package from the
[releases](https://github.com/dnlbtz/mbp143-wifi/releases) and install it:

    sudo apt install ./mbp143-wifi_*.deb                 # Ubuntu 26.04
    sudo pacman -U mbp143-wifi-*.pkg.tar.zst             # Arch/CachyOS

The install reads the Wi-Fi chip, which needs the Wi-Fi driver unloaded:
the connection drops for about 20 seconds and comes back by itself, with
the new firmware and board file. Nothing else to do; a restart is not
needed.

## Uninstall

    sudo apt remove mbp143-wifi                          # Ubuntu 26.04
    sudo pacman -R mbp143-wifi                           # Arch/CachyOS

The firmware and board file made at install are deleted and the Wi-Fi
driver is reloaded. From the next start the chip runs on linux-firmware's
firmware without a board file, as before installing.

## Check that it works

    $ cat /var/lib/mbp143-wifi/sromVars.txt | head -3
    sromrev=12
    subvid=0x106b
    boardtype=0x157
    $ iw dev wlp3s0 link | grep -E 'freq|signal'

The first command shows the values read from the chip. If it says "No such
file", the install could not read the chip and the machine runs on the
fallback (see Troubleshooting). On 5 GHz near the access point the signal
should be in the -50s or -60s dBm.

## What changes on the machine

- `/usr/lib/firmware/updates/brcm/` gets two files written at install: the
  firmware `brcmfmac43602-pcie.bin` and the board file
  `brcmfmac43602-pcie.Apple Inc.-MacBookPro14,3.txt`. The kernel looks in
  `updates/` before linux-firmware's own files, so these two win while they
  are there. Updates of linux-firmware do not touch them.
- `/var/lib/mbp143-wifi/` keeps what was read from the chip: `srom.bin`, the
  raw 2048 bytes, and `sromVars.txt`, the variables the board file takes
  from it as `name=value` lines.
- `/usr/lib/firmware/brcm/brcmfmac43602-pcie.Apple Inc.-MacBookPro14,3.txt`
  is the fallback board file, the MacBookPro13,3's, for when the chip could
  not be read. It is used
  with linux-firmware's own firmware.
- NetworkManager gives the Wi-Fi a stable MAC address per machine and
  network (`/etc/NetworkManager/conf.d/macbookWifiStableMac.conf`). The
  board file carries Broadcom's default address, the same on every 14,3, so
  two of them on one network would collide without it.
- When the Wi-Fi connects on 2.4 GHz and the same network is on 5 GHz at 30 %
  or better, a NetworkManager dispatcher reconnects once, at most every two
  minutes (`/etc/NetworkManager/dispatcher.d/91-wifi-prefer-5ghz`).
- The board file starts the firmware in its table for Germany, revision 2,
  which allows 80 MHz channels on 5 GHz. On a system set to another country
  brcmfmac switches the firmware to that country's table, as it does
  without this package.

## The parts

| file | what it is | what it does |
|---|---|---|
| `firmware/brcmfmac43602-pcie.bin` | linux-firmware's firmware for the BCM43602, version 7.35.177.61 (2015), 635449 bytes, unmodified | The base the setup patches. Carried in the package so the patch always meets the file it was written for. |
| `firmware/femPatch.txt` | 5 changes, 399 bytes | Apple's front-end table for this board (384 bytes) and the branch that loads it; the chip control and PHY register setup macOS does for this board (Bluetooth shares an antenna with Wi-Fi). Every change lists the bytes it expects. |
| `board/template.txt` | the board file, 191 variables | 155 lines `name=@srom`, filled from the chip; 36 written as they stand, each group with the reason in the file's header. |
| `board/sromTable.json` | 227 rows of Broadcom's SROM variable tables | Where in the chip's SROM each of the 155 variables is stored, and how it is written. |
| `board/fixed.txt` | the MacBookPro13,3's board file, set to Germany revision 2 | The fallback. |
| `setup/mbp143-wifi-setup` | Python, standard library only | Reads the SROM, writes the board file and the firmware (`install`), deletes them (`remove`), prints the board file it would write (`show`). |
| `conf/macbookWifiStableMac.conf` | NetworkManager setting | The stable MAC address. |
| `conf/91-wifi-prefer-5ghz` | NetworkManager dispatcher | The move to 5 GHz. |

How the setup reads the chip: with brcmfmac unloaded it maps the chip's
registers through PCI BAR0, as macOS's driver does, reads the 1024 words of
the SROM and checks their CRC. On a 14,3 the SROM is revision 12. Decoded with
Broadcom's full tables it gives the same 322 variables macOS's driver
reads, byte for byte; the package carries only the rows for the variables
the board file needs.

## Troubleshooting

### The install says "keeping linux-firmware's firmware and the fixed board file"

The line before it says why. Common causes: the Wi-Fi driver could not be
unloaded (something holds it), or the machine is not a MacBookPro14,3. Run
the setup again by hand when the cause is gone:

    sudo mbp143-wifi-setup install

### No 5 GHz networks after restarting from macOS

A restart from macOS leaves two of the chip's control bits cleared that a
power-on sets. linux-firmware's own firmware never sets them, so 5 GHz
stays about 20 dB weaker until Linux restarts. The patched firmware sets
them at every channel change. If it still happens, the machine is on the
fallback: check
`/var/lib/mbp143-wifi/sromVars.txt`. Until then, shut down macOS and switch
on, instead of restarting.

### Bluetooth audio stutters while the Wi-Fi is busy on 2.4 GHz

Bluetooth and Wi-Fi share one antenna on this machine. With this package
the Wi-Fi really transmits on 2.4 GHz, and takes air time from Bluetooth.
On 5 GHz they do not compete.

## Build from source

The Ubuntu package (needs `dpkg-deb`):

    packaging/debian/buildDeb

The Arch package, in `packaging/arch`:

    makepkg

`sudo setup/mbp143-wifi-setup install` works from the repository too, with
`MBP143_WIFI_SHARE` set to a directory holding the four files the package
puts in `/usr/share/mbp143-wifi`.

## Credits

The MacBookPro13,3 board file is from
[cristianmiranda's gist](https://gist.github.com/cristianmiranda/6f269797b62076c3414c3baa848dda67),
originally from [kernel bug 193121](https://bugzilla.kernel.org/show_bug.cgi?id=193121).

## Licence

MIT, see `LICENSE`, for everything except the firmware.
`firmware/brcmfmac43602-pcie.bin` is Broadcom's, from linux-firmware, carried
unmodified under Broadcom's licence (`firmware/LICENCE.broadcom_bcm43xx`).
The setup changes it on the machine it is installed on.
