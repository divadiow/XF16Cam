# Flashing and first start

[Project overview](../README.md) · [Hardware guide](hardware.md) ·
[Developer notes](../project/example/xf16cam/readme.md)

Flashing replaces the original 1 MiB image. Read this page once before applying
power; a backup and a verified hardware identity make recovery much easier.

## 1. Confirm the target

Only continue when the board matches the photographs in the
[hardware guide](hardware.md) and the main device is marked `XF16`. Beken, Taixin,
XR872AT, or other A9 variants are incompatible.

The expected external flash is 1 MiB/8 Mbit (though I think I've seen older devices with 2MiB). Keep a verified factory dump if at
all possible. Community experiments found that in-circuit SPI clips sometimes
reported changing IDs such as `13 13`, `8E 80 29`, and `C7 40 14`; compare
multiple reads and do not trust a dump that is not repeatable. An off-board read
may be necessary when the rest of the PCB loads the programmer.

## 2. Choose the correct image

A successful GitHub Actions artifact contains two similarly named firmware
images:

| File | Use |
| --- | --- |
| `xf16cam-xr872-v<version>.img` | Complete image for the first serial flash or UART recovery |
| `xf16cam-xr872-v<version>-ota.img` | Compressed update accepted by an already-running XF16Cam web console |

Do not upload the complete serial image through the web updater. Do not write
the compressed OTA file at flash offset zero.

Until tagged releases are available, select a successful run from the
[XF16Cam build workflow](https://github.com/divadiow/XF16Cam/actions/workflows/build-xf16cam.yml)
and download its `xf16cam-xr872-...` artifact. GitHub may require sign-in and
retains these artifacts for 14 days. `SHA256SUMS` in the artifact lets you check
that the download is intact.

## 3. UART connection

The console is 115200 baud, 8 data bits, no parity, one stop bit, and no flow
control. At PCB level, use **3.3 V UART logic**:

| Camera | USB–UART adapter |
| --- | --- |
| PB0 / TX | RX |
| PB1 / RX | TX |
| GND | GND |

Power the camera through its normal regulated 5 V USB input and share ground
with the adapter. Never put 5 V logic on PB0 or PB1. Desolder the pouch
battery while wiring or flashing, especially if its condition is unknown.

On the photographed A9 revision, UART is routed through the micro-USB data path,
so a USB breakout can avoid soldering directly to the PCB:

| Known cable contact | USB–UART adapter / supply |
| --- | --- |
| Green, D− | TX at 3.3 V logic |
| White, D+ | RX at 3.3 V logic |
| Black, GND | Common ground |
| Red, VBUS | Regulated 5 V camera power |

Cheap cable colours and board revisions can differ; verify the contacts before
applying power. Treat this connector as UART, not normal USB data, and do not
connect the data pair to a PC USB host and a UART adapter at the same time. The
[original adapter wiring and photographs](https://www.elektroda.com/rtvforum/topic4074636-90.html#21528571)
show the confirmed revision.

## 4. Back up and write with PhoenixMC

The XRADIO `PhoenixMC` tools are included in `tools/`. Both the factory
application and XF16Cam can enter the BootROM through the serial `upgrade`
command (assuming factory app or original Runtop app is running), so ordinarily there should be no need to ground PB02 and PB03 as bootstrapping.

For the conservative first pass in the PhoenixMC GUI:

1. Close every terminal using the port, select the camera's COM port, and start
   at 115200 baud.
2. Open the debug/read controls and wait for `Open comm OK` so the address and
   length fields become available.
3. Read from address `0` for length `100000` (hex). The result must be exactly
   1,048,576 bytes.
4. Read it a second time and compare SHA-256 hashes. Keep both verified factory
   backups somewhere outside this checkout.
5. Select the complete `xf16cam-xr872-v<version>.img`, choose **Update**, and
   wait for a successful write result before removing power.
6. Reboot the camera normally.

The `100000` length is intentional for this 1 MiB target; generic XRadio guides
often show `200000` for 2 MiB devices. Use the complete image, never the
`-ota.img` file.

PhoenixMC sends `upgrade`, then synchronisation bytes; a receptive application
answers and reboots into the XR BootROM. XF16Cam deliberately keeps its console
enabled and the handoff available even if the camera sensor, Wi-Fi, or storage
has failed. The [protocol investigation](https://www.elektroda.com/rtvforum/topic4074636-150.html#21534886)
and [console-enable finding](https://www.elektroda.com/rtvforum/topic4074636-150.html#21536774)
explain why this works after a full power cycle.

If flash is blank/corrupt and no application can receive `upgrade`, software
handoff is impossible. Do not guess catalogue XR872ET strap pins: the XF16 uses
a different package. Restoring the verified 1 MiB image to the desoldered flash
is the conservative recovery route when UART entry has been lost. The
[long-running XF16 investigation](https://www.elektroda.com/rtvforum/topic4074636.html)
documents the BootROM experiments and pitfalls of unstable in-circuit SPI
writes.

## 5. First boot

With no saved configuration, XF16Cam starts:

- SSID: `XF16CAM`
- Security: open (no Wi-Fi password)
- Camera address: `192.168.4.1/24`
- One DHCP lease: `192.168.4.100`
- Default media mode: RTSP
- Default video resolution: 320 × 240

Join `XF16CAM` and open `http://192.168.4.1/`. Windows or a phone may warn that
the network has no Internet; remain connected. Use the Network page to scan,
select an SSID, enter its password, and save. The camera reboots and joins that
network. If it cannot connect within about 20 seconds, it returns to the setup
AP.

The console remains available in either AP or STA mode. To watch in a browser,
select **Browser video** on the Live page; changing between Browser and RTSP
persists the choice and reboots. For RTSP use:

```text
rtsp://<camera-ip>:8554/stream
```

## Serial recovery commands

At the 115200-baud console:

```text
wifi ap
wifi sta <ssid> <password>
upgrade
```

`wifi ap` restores setup-AP mode. `wifi sta` saves credentials and reboots.
`upgrade` immediately hands control to the UART BootROM. PhoenixMC normally
sends it automatically. For a manual handoff, issue `upgrade` in the terminal,
close the terminal so it releases the COM port, and then connect PhoenixMC.

Holding the PA20 button for 3 seconds is the no-terminal route back to the open
setup AP.

## Web OTA

Once XF16Cam is running:

1. Download the matching `-ota.img` file.
2. Open **System → Firmware update** in the web console.
3. Choose the OTA image, keep USB power connected, and select **Install update**.
4. Wait for verification and reboot, then reload the page at the same AP or STA
   address.

The upload is streamed into a 372 KiB staging partition and is not selected for
boot until the SDK image structure and MD5 check pass. A bad or interrupted
upload normally leaves the current firmware bootable. OTA is not signed or
encrypted, and the web endpoint has no authentication; use it only on a trusted
network.

## Troubleshooting

- **No serial text:** confirm 115200 8N1, common ground, crossed TX/RX, and that
  the adapter uses 3.3 V logic. Check the opposite micro-USB data contact if the
  board revision routes the pair differently.
- **Uploader cannot synchronise:** close the terminal, reboot normally, and let
  the uploader send `upgrade` again. If the application is alive, issuing
  `upgrade` in the console first is a useful diagnostic.
- **No `XF16CAM` AP:** hold PA20 for 3 seconds. If boot never reaches the XF16Cam
  version line, capture the UART log and use serial/SPI recovery.
- **No camera sensor:** this should not block management services. The web page
  should report that no supported sensor was detected.
- **OTA rejected:** verify that the filename ends in `-ota.img`, that it came
  from the same target workflow, and that the upload was not interrupted.
