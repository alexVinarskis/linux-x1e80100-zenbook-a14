### Patches for supporting Asus Zenbook A14 2025, models UX3407QA, UX3407RA.

As changes will be merged upstream, redundant patches will be dropped from this repo, to only maitain diff on top of latest `linux-next`.

## Test setup
Kernel
* `linux-next`
* current patchset tested on `next-20250424`

Initramfs
* Ubuntu's auto-built from kernel installation

HW configurations tested
* UX3407QA
* x1-26-100
* 1920x1200 OLED non-touch 60Hz
* 32GB RAM

## Feature Matrix

* WIP - Something I am already working on
* TBH - Something I could work on, but not started / no sufficient progress was made
* QC  - Something I cannot work on, depends on Qualcomm's upstream support

| Feature                 | Status | Notes                                                                                                        |
| ----------------------- | -----: | ------------------------------------------------------------------------------------------------------------ |
| Battery Charging        |     ✅ |                                                                                                              |
| Battery Info            |     ✅ |                                                                                                              |
| Bluetooth               |     ✅ |                                                                                                              |
| Camera                  | WIP/❌ |                                                                                                              |
| Display                 |     ✅ | Tested low-res OLED panel                                                                                    |
| GPU Acceleration        |        | UX3407RA / X1E-78-100 should work oob. UX3407QA / X1P-42-100, X1-26-100 waiting for GPU support.             |
| Keyboard                |     ✅ |                                                                                                              |
| Microphone              |     ✅ |                                                                                                              |
| NVMe                    |     ✅ |                                                                                                              |
| Speakers                |     ✅ | Require userland configuration, see below.                                                                   |
| Audio Jack              |     ✅ |                                                                                                              |
| Suspend                 |     ✅ | Suspends well, lid switch working. Power drop in sleep isn't best, depends on X1E generic support.           |
| Touchpad                |     ✅ |                                                                                                              |
| TPM                     |     ❌ | Firmware TPM                                                                                                 |
| USB-A 3.0               |     ✅ |                                                                                                              |
| USB-C 3.0               |     ✅ |                                                                                                              |
| USB-C Booting           |     ✅ |                                                                                                              |
| USB-C DP Alt Mode       |     ✅ |                                                                                                              |
| USB-C DP over dock      |     ✅ | Series on the lists, not yet merged                                                                          |
| HDMI                    |    WIP | Parade PS185PDF DP1.4a to HDMI IC                                                                            |
| Wi-Fi                   |     ✅ | UX3407RA with FastConnect 7800 should work oob. UX3407QA requires firmware extraction and patching.          |
| EC                      | WIP/❌ | Similar to out-of-tree EC driver for Lenovo Slim 7x.                                                         |

## WCN688x WiFi

Lower-end UX3407QA has WCN6885 WiFi 6E, as opposed to WiFi 7 on UX3407RA.
At least the WCN6885 is known not to work as is, as it requires both newer firmware, and patched `boards-2.bin`. Without these changes, driver would fail to load:
```bash
$ dmesg -w
...
[ 3190.482952] ath11k_pci 0004:01:00.0: BAR 0 [mem 0x7c400000-0x7c5fffff 64bit]: assigned
[ 3190.485446] ath11k_pci 0004:01:00.0: MSI vectors: 32
[ 3190.485465] ath11k_pci 0004:01:00.0: wcn6855 hw2.1
[ 3190.649874] mhi mhi0: Requested to power ON
[ 3190.649907] mhi mhi0: Power on setup success
[ 3190.732860] mhi mhi0: Wait for device to enter SBL or Mission mode
[ 3191.354711] ath11k_pci 0004:01:00.0: chip_id 0x2 chip_family 0xb board_id 0xff soc_id 0x400c0210
[ 3191.354723] ath11k_pci 0004:01:00.0: fw_version 0x11088190 fw_build_timestamp 2024-10-17 19:57 fw_build_id WLAN.HSP.1.1.c5-00400-QCAHSPSWPL_V1_V2_SILICONZ_WOS-1
[ 3191.412843] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q from ath11k/WCN6855/hw2.1/board-2.bin
[ 3191.412853] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255 from ath11k/WCN6855/hw2.1/board-2.bin
[ 3191.412856] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,qmi-chip-id=2 from ath11k/WCN6855/hw2.1/board-2.bin
[ 3191.412858] ath11k_pci 0004:01:00.0: failed to fetch board.bin from WCN6855/hw2.1
```

### Extract firmware from windows path
```bash
# Find location of driver on Windows partition
WINDOWS_DRIVER="/mnt/c/Windows/System32/DriverStore/FileRepository/qcwlanhsp8380.inf_arm64_ecbde8f5eb2c6dd5/"

# Copy firmware
sudo cp ${WINDOWS_DRIVER}/wlanfw20.mbn /lib/firmware/updates/ath11k/WCN6855/hw2.1/amss.bin
sudo cp ${WINDOWS_DRIVER}/regdb.bin /lib/firmware/updates/ath11k/WCN6855/hw2.1/regdb.bin
sudo cp ${WINDOWS_DRIVER}/m3.bin /lib/firmware/updates/ath11k/WCN6855/hw2.1/m3.bin
```

### Repack `board-2.bin`

```bash
$ mkdir -p /tmp/qca

# Identify and copy board config for you platform
# - It will be `.elf` file of ~59KB
# - Match based on device name, in this case `UX3407Q`
% cp ${WINDOWS_DRIVER}/bdwlan_wcn685x_2p1_nfa725a_UX3407Q.elf /tmp/qca/

# Copy and decompress current `board-2.bin`
$ cp /usr/lib/firmware/ath11k/WCN6855/hw2.0/board-2.bin.zst /tmp/qca/board-2_original.bin.zst
$ zstd -d /tmp/qca/board-2_original.bin.zst

# Unpack `board-2.bin`
# This requires QCA tool, available at:
# https://github.com/qca/qca-swiss-army-knife/blob/master/tools/scripts/ath11k/ath11k-bdencoder

$ ./ath11k-bdencoder -e /tmp/qca/board-2_original.bin

# Patch to add required entry, matching target name
# Inspect `board-2.json`. Compare to error in dmesg, eg:
$ dmesg | grep "failed to fetch board data for"
[   13.546264] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q from ath11k/WCN6855/hw2.1/board-2.bin
[   13.546273] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255 from ath11k/WCN6855/hw2.1/board-2.bin
[   13.546275] ath11k_pci 0004:01:00.0: failed to fetch board data for bus=pci,qmi-chip-id=2 from ath11k/WCN6855/hw2.1/board-2.bin

# Extract 1st entry which is the most precise match:
# "bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q"

# Add apropriate entry to `board-2.json`:
             {
                 "names": [
                     "bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=9509,qmi-chip-id=2,qmi-board-id=255"
                 ],
                 "data": "bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=9509,qmi-chip-id=2,qmi-board-id=255.bin"
             },
+            {
+                "names": [
+                    "bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q"
+                ],
+                "data": "bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q.bin"
+            },
             {
                 "names": [

# Rename copied-out elf file to the board name format
$ mv /tmp/qca/bdwlan_wcn685x_2p1_nfa725a_UX3407Q.elf /tmp/qca/"bus=pci,vendor=17cb,device=1103,subsystem-vendor=14cd,subsystem-device=950a,qmi-chip-id=2,qmi-board-id=255,variant=UX3407Q.bin"

# Repack `board-2.bin`
$ ./ath11k-bdencoder -c /tmp/qca/board-2.json

# Finally, compress the new `board-2.bin` and copy it to destination
$ zstd /tmp/qca/board-2.bin
$ sudo cp /tmp/qca/board-2.bin.zst /lib/firmware/updates/ath11k/WCN6855/hw2.1/board-2.bin.zst
```

### Reload kernel module
```bash
$ sudo modprobe -r ath11k_pci ath11k
$ sudo modprobe ath11k_pci

# You should now see the wifi driver to start normally
$ dmesg -w
...
[ 3228.406613] ath11k_pci 0004:01:00.0: BAR 0 [mem 0x7c400000-0x7c5fffff 64bit]: assigned
[ 3228.409524] ath11k_pci 0004:01:00.0: MSI vectors: 32
[ 3228.409545] ath11k_pci 0004:01:00.0: wcn6855 hw2.1
[ 3228.577327] mhi mhi0: Requested to power ON
[ 3228.577364] mhi mhi0: Power on setup success
[ 3228.924764] mhi mhi0: Wait for device to enter SBL or Mission mode
[ 3229.547159] ath11k_pci 0004:01:00.0: chip_id 0x2 chip_family 0xb board_id 0xff soc_id 0x400c0210
[ 3229.547171] ath11k_pci 0004:01:00.0: fw_version 0x11088190 fw_build_timestamp 2024-10-17 19:57 fw_build_id WLAN.HSP.1.1.c5-00400-QCAHSPSWPL_V1_V2_SILICONZ_WOS-1
[ 3229.866703] ath11k_pci 0004:01:00.0 wlP4p1s0: renamed from wlan0
[ 3229.874853] ath11k_pci 0004:01:00.0: Failed to set the requested Country regulatory setting
[ 3229.874861] ath11k_pci 0004:01:00.0: failed to process regulatory info -22
[ 3229.875182] ath11k_pci 0004:01:00.0: Failed to set the requested Country regulatory setting
[ 3229.875183] ath11k_pci 0004:01:00.0: failed to process regulatory info -22
[ 3236.596237] wlP4p1s0: authenticate with 50:64:2b:5f:e3:ba (local address=8c:3b:4a:a6:fa:f3)
[ 3236.596250] wlP4p1s0: send auth to 50:64:2b:5f:e3:ba (try 1/3)
[ 3236.601170] wlP4p1s0: authenticated
```

## Audio configuration  

Besides device tree changes in kernel, two things are required to get audio working: alsa configuration and toplogy firmware.

### Audioreach-topology
* Download latest sources with Asus Zenbook A14 support from https://github.com/linux-msm/audioreach-topology/
* Build via
```bash
cmake .
cmake --build .
```
* Install compiled binary to firmware path via
```bash
sudo cp qcom/x1e80100/ASUSTeK/zenbook-a14/X1E80100-ASUS-Zenbook-A14-tplg.bin /lib/firmware/updates/qcom/x1e80100/X1E80100-ASUS-Zenbook-A14-tplg.bin
```

### Alsa configuration
* Download lastet configuration with Asus Zenbook A14 support from https://github.com/alsa-project/alsa-ucm-conf
* Follow instructions in `README.md` to unpack

Reboot to apply changes. You should now have:
* Working x2 speakers
* Working x2 microphones
* Working audio jack with microphone

Audio over HDMI and USB Type-C DP alt mode is not yet supported.
