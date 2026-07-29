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

Stock BTstack asks for **10–30 ms with slave latency 4** (`hci.c` defaults) and then **accepts
whatever the peripheral asks for, up to 4 seconds**: `l2cap.c` auto-answers a peripheral's
`CONNECTION_PARAMETER_UPDATE_REQUEST` against `gap_get_connection_parameter_range()`, whose default
maximum is 3200 units.

Battery-minded keyboards and TV remotes routinely request 30 ms or more a few seconds after
connecting, and the stock build says yes. So the interval you ran at was whichever one the
*peripheral* preferred — not one anybody chose.

## What changed

All in `hog_host_demo.c`, marked `// @@lowlat`, plus two CMake knobs.

1. **Ask for 7.5 ms on the outgoing connection** — `gap_set_connection_parameters()`, so the link
   *starts* at the target instead of at 10–30 ms. The initiating scan window is also widened to a
   100% duty cycle, which shortens connect time.
2. **Grant the peripheral's later request, then re-assert the interval from the central side** —
   substituting **slave latency** so the peripheral's power budget is left exactly as it asked for.
   A peripheral asking for a 30 ms interval is not asking for 30 ms of lag; it is asking to keep
   its radio asleep for 30 ms at a time. Interval 7.5 ms + latency 3 gives it the same sleep and us
   a quarter of the lag, because it may still transmit at the very next event once it has a
   keypress. Slave latency costs nothing here — it only delays host→device traffic, and this bridge
   sends none. Bounded to 2 re-asserts per link so it can never become an update war on air.
3. **Log the negotiated interval and the disconnect reason.** That interval line is the real
   latency number; everything else is an estimate.

### What was tried and must not be repeated

The first version narrowed `gap_set_connection_parameter_range()` so a peripheral asking for 30 ms
was **rejected**. That looked correct — a rejection is a legal L2CAP response — and it **broke the
link**: connects, works, drops ~5 s later, forever, on every interval setting.

Many BLE peripherals treat a rejected parameter negotiation as fatal and terminate the connection.
Nordic's `ble_conn_params` module is the common case, and its `FIRST_CONN_PARAMS_UPDATE_DELAY` is
**5000 ms** — exactly when the drops appeared.

**Grant the request, then override it.** Granting keeps the peripheral's state machine happy; the
central-side override is what actually gets the latency. There is a loud comment block in
`hog_host_demo.c` guarding this.

## Which build to flash

| File | Behaviour | Use when |
| --- | --- | --- |
| `..._lowlat_balanced.uf2` | 7.5 ms at connect; if the peripheral raises it, grant, then re-assert 7.5 ms with latency substituted | **start here** |
| `..._lowlat_nodeny.uf2` | 7.5 ms at connect; then accept whatever the peripheral wants, forever | balanced still drops the link |

`nodeny` is the minimal fix and is strictly more permissive than upstream after connect, so it
cannot disconnect for any reason upstream wouldn't. It still gives the full benefit on the many
peripherals that never request a parameter change at all.

If `balanced` drops but `nodeny` holds, the peripheral rejects *forced* updates too, and 7.5 ms is
simply not available with that device.

Build any interval yourself: `cmake -DLOWLAT_CONN_INTERVAL_UNITS=<n> -DLOWLAT_REASSERT=<0|1>`
(units of 1.25 ms).

## Reading the logs

`printf` goes to **UART** (`pico_enable_stdio_uart 1`, USB is the HID device), so seeing these
needs a 3.3 V USB-serial adapter on GP0/GP1 at 115200.

- `Connected: conn interval N units = X.XX ms` — the interval actually in force.
- `Parameters updated: ...` — the peripheral moved it, or our re-assert landed.
- `Disconnected (reason 0xNN)` — **`0x13`** = the peripheral hung up (it disliked something);
  **`0x08`** = connection timeout, the link died on air (range, interference, or an interval the
  peripheral cannot keep up with).

## Building

`.github/workflows/build.yml` builds both variants headlessly with Pico SDK 2.2.0 — the same
version `src_fw/How to build.txt` pins for the VS Code route.

## Status

**Compiles in CI; the interval behaviour is not hardware-verified.** The ~5 s disconnect caused by
the earlier rejection approach was reproduced on real hardware and is fixed; whether any given
peripheral tolerates a forced 7.5 ms update is still unmeasured.
