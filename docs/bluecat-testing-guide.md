# bluecat — hardware testing guide

Test order goes easiest → hardest. Stop at the first failure and capture the output (see "Reporting back").

## 0. Setup (once)

```bash
cd /Users/wero1414/Sniffle/python_cli
python3 -m pip install pyserial    # if not already
chmod +x bluecat.py
```

Set these for your bench (substitute your values):

```bash
export PORT=/dev/cu.usbmodem11201        # your CatSniffer port (ls /dev/cu.usbmodem*)
export BAUD=921600                       # CatSniffer V3 = 921600 (the _1M firmware). DO NOT omit.
export LED=BE:96:24:00:07:DA             # your ELK-BLEDOM strip (a PUBLIC address)
```

> **Baud rate matters.** CatSniffer V3 runs the CC1352 UART at 921600 (the `_1M` build). `bluecat` defaults to 2 Mbaud, so you MUST pass `-b $BAUD` on every command or it won't talk to the sniffer.

**Firmware:** Tests 1–3 (scan/connect) work with any Sniffle firmware on the board. Tests 4–5 (hijack/follow) need the **hijack firmware** flashed (`fw/sniffle.hex`, the build we made). Flash it before those.

Smoke check (no hardware needed):
```bash
./bluecat.py --help && ./bluecat.py scan --help && ./bluecat.py fuzz --help
```
Expect three help screens, no errors.

---

## 1. Scan — list nearby devices  (passive, easiest)

```bash
./bluecat.py scan -s $PORT -b $BAUD --time 10 | tee /tmp/bc_scan.txt
```
**Expect:** a table of nearby BLE devices (your phone, the ELK-BLEDOM strip, neighbours) with MAC, name, RSSI, addr type, posture `UNKNOWN`, #svcs. Sorted by RSSI.
**PASS:** the ELK-BLEDOM appears in the list.
**If it fails:** "Sniffle device not found" → wrong `-s`/`-b`. Empty table → nothing advertising nearby, or wrong channel (`-c 37/38/39`).

---

## 2. Scan + probe — classify open vs encrypted  (active)

Make sure the strip is **not** connected to your phone (so it's connectable).
```bash
./bluecat.py scan -s $PORT -b $BAUD --probe --time 8 | tee /tmp/bc_probe.txt
```
**Expect:** same table, but the `posture` column is filled by briefly connecting to each device: the ELK-BLEDOM should show `OPEN`; phones/locks/etc. that require pairing show `ENCRYPTED_*`.
**PASS:** ELK-BLEDOM shows `OPEN` (highlighted as the alert).
**Note:** slower (connects to each device one at a time); some devices won't accept a connection and stay `UNKNOWN`.

---

## 3. Connect + enumerate + REPL  (core, no hijack)

Disconnect the phone from the strip first. ELK-BLEDOM uses a **public** address, so pass `--public`.
```bash
./bluecat.py --mac $LED --connect --public -s $PORT -b $BAUD | tee /tmp/bc_connect.txt
```
**Expect:**
1. `[+] in CENTRAL — posture: OPEN`
2. The GATT tree: services `0x1800`, `0x1801`, `0xFFF0`, with the control char value handle `0x000E`.
3. A `bluecat>` prompt.

In the REPL, try:
```
read 0x0003                       # GAP Device Name -> "ELK-BLEDOM..."
write 0x000e 7e070503ff000010ef   # RED
write 0x000e 7e07050300ff0010ef   # GREEN
write 0x000e 7e0705030000ff10ef   # BLUE
enum                              # re-print the tree
sub 0x0011                        # subscribe to a CCCD handle (if the strip has a notify char)
info
quit
```
**PASS:** the strip changes colour on the writes, and `enum`/`read` return sensible data.
**If it fails:** can't reach CENTRAL → the strip is still connected to the phone (only allows one link) or it's not connectable; wrong address type (try without `--public`).

---

## 4. Hijack + enumerate  (needs hijack firmware flashed)

Flash `fw/sniffle.hex` first. **Connect the phone to the strip and keep it connected.**
```bash
./bluecat.py --mac $LED --hijack --public -s $PORT -b $BAUD | tee /tmp/bc_hijack.txt
```
**Expect:** sniff → "Tracking… stable" → fire hijack → `in CENTRAL` → **the phone loses control** → GATT tree → REPL (same commands as Test 3).
**PASS:** phone drops the connection, you can drive the strip's colour from the REPL.
**Known limitation:** this rides the firmware takeover we were still debugging (run #5). If the strip's connection renegotiated to a short interval / long timeout, the takeover can drop immediately and the phone keeps control. If that happens, capture `/tmp/bc_hijack.txt` and the `HJ END:` line.

---

## 5. Follow — hijack the first connection seen  (no MAC)

Flash hijack firmware. Start bluecat, THEN connect the phone to any unencrypted device.
```bash
./bluecat.py --public -s $PORT -b $BAUD | tee /tmp/bc_follow.txt
```
(`--public` only matters if the caught device is public; harmless otherwise.)
**Expect:** `[*] follow mode: catching first connection on ch 37...`, then it catches a connection, reports the caught address, hijacks, enumerates.
**PASS:** it latches and takes over a connection without you specifying a MAC.
**Note:** no 3-channel hopping in follow mode, so it only catches connections that start on ch 37 — retry, or try `-c 38`/`-c 39`.

---

## 6. Fuzz  (most complex)

Disconnect the phone (use connect mode so crash-resume works).

**Value fuzzing** (mutate writes to the control characteristic):
```bash
./bluecat.py fuzz --mac $LED --connect --public --mode values \
  --handle 0x000e --seed 7e070503ff000010ef -o /tmp/fuzz_values.jsonl -s $PORT -b $BAUD
```
**Handle sweep** (write a payload to every handle in a range):
```bash
./bluecat.py fuzz --mac $LED --connect --public --mode sweep \
  --seed 00 --start 0x0001 --end 0x00ff -o /tmp/fuzz_sweep.jsonl -s $PORT -b $BAUD
```
**ATT-opcode** and **LL-control** fuzzing:
```bash
./bluecat.py fuzz --mac $LED --connect --public --mode opcodes -o /tmp/fuzz_op.jsonl -s $PORT -b $BAUD
./bluecat.py fuzz --mac $LED --connect --public --mode ll      -o /tmp/fuzz_ll.jsonl  -s $PORT -b $BAUD
```
**Expect:** each run sends its inputs, writes a JSONL line per input to the `-o` file, and prints a summary `{tested, crashes, anomalies}`. On a crash (device stops responding / link drops) in connect mode it reconnects and continues.
**PASS:** the run completes, the `.jsonl` file has one line per input, and the summary prints. Watch the strip — a crash/reset is a finding (check the `crashed:true` lines).
**Note:** crash-resume only works in `--connect` mode (you own the link); `--hijack`/follow can't auto-reconnect.

---

## 7. PCAP capture (any access mode)

Add `-o file.pcap` to any access command (Tests 3–5), e.g.:
```bash
./bluecat.py --mac $LED --connect --public -o /tmp/bc.pcap -s $PORT -b $BAUD
```
**PASS:** `/tmp/bc.pcap` opens in Wireshark and shows the session's link-layer packets. (Use this with crackle/Wireshark + LTK for encrypted targets.)

---

## Reporting back

For any failure, send me:
- The command you ran and the captured `/tmp/bc_*.txt` (you `tee`'d them).
- For hijack issues: the `HJ ...` / `HJ END:` debug lines.
- For fuzz crashes: the `crashed:true` lines from the `.jsonl`.

## Quick troubleshooting
| Symptom | Likely cause |
|---|---|
| "Sniffle device not found" | wrong `-s` port, or `-b 921600` missing |
| Garbage / no response | wrong baud (must be `921600` for CatSniffer V3) |
| Can't reach CENTRAL on `--connect` | target already connected to phone, or needs `--public` toggled |
| Hijack drops instantly, phone keeps control | run-#5 firmware issue (renegotiated conn params) |
| LED doesn't change | wrong handle/format — use `0x000e` + `7e0705 03 RRGGBB 10ef` |
| Hijack target reported "encrypted" | expected — bluecat won't hijack encrypted links |
