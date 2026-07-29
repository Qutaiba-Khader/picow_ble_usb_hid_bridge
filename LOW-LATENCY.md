# Low-latency fork

A fork of [shiomachisoft/picow_ble_usb_hid_bridge](https://github.com/shiomachisoft/picow_ble_usb_hid_bridge)
with one goal: cut the input lag. The bridge itself is unchanged — BLE HID device in, USB HID
device out, on a Raspberry Pi Pico W.

## Where the latency actually is

| Stage | Cost | Already optimal upstream? |
| --- | --- | --- |
| Peripheral's own key scan / debounce | 5–20 ms | not ours to fix |
| **BLE connection interval** | **0 → 1 interval** | **no — this is the fix** |
| BLE core 1 → queue → USB core 0 | µs | yes |
| USB IN endpoint poll | 0–1 ms | yes (`bInterval = 1`, `usb_descriptors.c`) |

A key press cannot reach the Pico before the peripheral's next connection event, so the connection
interval **is** the added latency: 0 to one full interval, averaging half of it. Everything else in
this firmware was already at the floor — full-speed USB cannot poll faster than 1 ms, and core 0
drains the report queue as fast as it can spin.

## What was wrong

Stock BTstack asks for **10–30 ms with slave latency 4** (`hci.c` defaults) and, worse, then
**accepts whatever the peripheral asks for, up to 4 seconds**: `l2cap.c` auto-answers a
peripheral's `CONNECTION_PARAMETER_UPDATE_REQUEST`, granting anything inside
`gap_get_connection_parameter_range()`, whose default maximum is 3200 units.

Battery-minded keyboards and TV remotes routinely request 30 ms or more the moment they connect,
and the stock build says yes. So the interval you ran at was whichever one the *peripheral*
preferred — not one anybody chose.

## What changed

All of it in `hog_host_demo.c`, marked `// @@lowlat`, plus one CMake knob. Purely additive —
128 lines added, none removed.

1. **Ask for 7.5 ms on the outgoing connection** — `gap_set_connection_parameters()`, so the link
   *starts* at the target instead of at 10–30 ms. Slave latency 0, so the peripheral is never
   allowed to skip connection events. Initiating scan window is also widened to a 100% duty cycle,
   which shortens connect time.
2. **Narrow the accepted range to that same value** — `gap_set_connection_parameter_range()`. A
   peripheral asking for 30 ms is now denied. A denial is a normal L2CAP response; the link stays
   up on our parameters.
3. **Re-assert from the central side** if a different interval is ever observed, on both
   `GAP_SUBEVENT_LE_CONNECTION_COMPLETE` and `HCI_SUBEVENT_LE_CONNECTION_UPDATE_COMPLETE`. As
   central we own the link — an LL connection update is a command to the peripheral, not a
   request. Bounded to 2 attempts per link so a stubborn peripheral cannot turn it into an endless
   update war on the air.

The negotiated interval is printed over UART on every connect and every parameter change. That
line is the real latency number — everything else is an estimate.

## Which build to flash

| File | Interval | Added by the BLE link | Use when |
| --- | --- | --- | --- |
| `picow_ble_usb_hid_bridge_lowlat_7ms5.uf2` | 7.5 ms | 0–7.5 ms, ~3.75 ms avg | **start here** |
| `picow_ble_usb_hid_bridge_lowlat_15ms.uf2` | 15 ms | 0–15 ms, ~7.5 ms avg | the 7.5 ms build stutters or drops the link |

7.5 ms is the floor the BLE spec allows; there is no faster setting. A peripheral that cannot
service an event every 7.5 ms will stutter or disconnect — that is what the 15 ms build is for.
Shorter intervals also drain the peripheral's battery faster.

Build it yourself with any interval: `cmake -DLOWLAT_CONN_INTERVAL_UNITS=<n>` (units of 1.25 ms).

## Building

`.github/workflows/build.yml` builds both variants headlessly with Pico SDK 2.2.0 — the same
version `src_fw/How to build.txt` pins for the VS Code route.

## Status

**Compiles in CI; not verified on hardware.** The interval logic is straight from the BTstack API
contract, but no peripheral has been measured against it yet. Read the UART line to see what your
device actually negotiated.
