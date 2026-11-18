# 36h Collie (12–14 Nov 2026): log analysis

Hub firmware on the car: **v4.7.0** (from the TPMS capture headers). Nodes: **IR_CAN_NODE v5.1.0**.
Hub v4.8.0 exists as a sketch but was never flashed. **There is no node v5.2.0:** `Sauce/IRIS_prompt_IR_CAN_NODE_v5_2_0.md` is only a prompt, written on 12 Nov alongside hub v4.8.0 and never run.

> **Revised 18 Nov after reading node v5.1.0.** Sections 1 and 3 are updated, and section 1b is new. The earlier verdict leaned towards a purely physical cause. The node firmware turns out to have a recovery state it never leaves, which explains the *long* silences. The *trigger* is still something intermittent on the bus.
Times: sys_log is local time (AWST), the data CSV is UTC (+8 h). Every time below is local.

## Sessions

| Boot | Local time | What happened |
|---|---|---|
| 0001–0005 | 12 Nov, up to 21:50 | Bench and pit. No CAN at all (nodes not connected yet) |
| 0006 | 12 Nov 21:50–23:27 | Nodes plugged in one corner at a time. Each brake-node plug-in reset the tyre node on the same corner (tyre `HEALTHY -> INIT`, brake node appears 1 s later) |
| 0007 | 13 Nov 03:31–03:55 | Clean |
| **0008** | **13 Nov 03:56 → 14 Nov 00:23 (20.5 h, no reboot)** | Stint 1 onward. **39 whole-bus drops.** Bus down 10.3 h of 20.5 h. FL brake node lost for good at 18:53:15 |
| 0009 | 14 Nov 00:24–00:43 | Power cycle. **Bus never came up** |
| 0010 | 14 Nov 00:43–05:21 | Power cycle. **Bus silent for the first 3 h 05 min** while the car lapped, then up at 03:49, 1 drop |
| 0011–0013 | 14 Nov 05:45–07:31 | Clean (7 nodes) |
| **0014** | **14 Nov 08:31–15:58** | **9 whole-bus drops.** Bus down 6.6 h of 7.4 h |

## 1. The "brake" dropouts are whole-bus dropouts

Every major dropout is **all live nodes at once, tyres and brakes**, inside the same 10–60 ms window. That held in all 49 events.

Things the logs rule out:

- **Not brake heat or thermal throttling.**
  - Tyre nodes drop at the same moment.
  - The drops went on after the hottest brake node (FL) had died and left the bus (boots 0010 and 0014 had 7 nodes).
  - Brake-node die temperature (TA) at the two captured onsets was 63–114 °C: there is no threshold.
  - Every node comes back at the same instant, whatever its temperature.
- **Not a node crash, watchdog or brownout reset.** A node that reboots reports `INIT` for about 2.5 s. You can see that in boot 0006 when the nodes were hot-plugged. None of the 49 recoveries shows `INIT`: the node MCUs ran straight through every dropout.
- **Not a node jamming the bus.** During the silences the hub's 3 s NORMAL probes saw `rx_err=0` in 2,324 of 2,364 windows. The bus was idle and recessive: nothing transmitting, nothing corrupting.
- **A power cycle did not clear it.** Boots 0009 and 0010 left the bus dead for 3 h 24 min. *(Revised: this does not rule out firmware. If the trigger is still present when the nodes boot, they go bus-off again within seconds and fall straight back into the stuck state in section 1b. The first NORMAL probe in boot 0010, 62 s after power-up, saw `rx_err=129`: errors were on the bus right after the restart.)*
- **Not braking-specific.**
  - 13 Nov 05:19:56: the last frame came as braking ended (109 → 82 km/h).
  - 14 Nov 09:01:49: the last frame came at full throttle (121 km/h and accelerating).

What the recoveries look like:

- **26 of 47 recoveries were a single burst:** about one frame from each node, then silence again 1.5 s later.
- After several of those bursts the hub's error counter jumped to 128–133 in 3 s, which means garbage on the bus.
- The other recoveries held for anything from 10 s to 2 h.

### Verdict (revised)

There are two separate parts:

- **Trigger:** something intermittent and borderline on the bus. Its cause is not identified yet; see section 1b for the right-hand-side error storm.
- **Duration:** a node firmware recovery path with no exit (section 1b). It turns each short trigger into minutes or hours of silence.

### Original verdict (18 Nov, before reading v5.1.0)

This is kept for reference. Power starvation and the hub connector remain possible *triggers*, but they are no longer the leading explanation.

The cause is **physical, on a path that all 8 nodes share with the hub**: intermittent, and unaffected by node temperature or by rebooting. There are two candidates, and both fit everything above:

1. **Node supply.** The nodes have no regulator of their own. They run on the hub's 3.3 V buck output (MAX25223), sent down the loom on CAN connector pin 1 (schematic sheet 5; Iris TechnicalSpec §2.7).
   - At the corners, after loom, splice and M8 connector resistance, the voltage can sag into the window where the ESP32-C3 keeps running but the TCAN3413 transceiver drops out on undervoltage.
   - That gives a silent, clean bus, no reboots, everything stopping and starting together, and bursts that collapse: bus traffic draws more transceiver current and pulls the rail back down.
2. **An intermittent CAN_H/CAN_L contact between the hub and the loom** (hub M8 connector or the first splice). The nodes would keep ACKing each other while the hub hears nothing, and chattering contacts would explain the error-counter spikes.

**Neither candidate is a software bug, and the logs cannot tell them apart.** One cheap test can (section 4).

## 1b. What node v5.1.0 adds (`Sauce/IR_CAN_NODE_v5_1_0.txt`)

**N1. A recovery state with no exit.** This is the main finding.

How bus-off is handled:

1. `startBusOffRecovery()` calls `twai_initiate_recovery()` and sets `canRecovering = true`.
2. While `canRecovering` is set, **every transmission is skipped**: TOBJ, RAWIR and OTA replies (loop line ~1805, `send()` line ~1375).
3. `canStatusWatchdog()` claims a 30 s last-resort reinstall, but the check sits only in `case TWAI_STATE_BUS_OFF`.
4. After `twai_initiate_recovery()` the driver is in `TWAI_STATE_RECOVERING`, which falls through to `default: // in progress, nothing to do`.

So if recovery does not complete, nothing ever times out. The node stays powered, feeds its watchdog, never reboots, and sends nothing until the TWAI controller finally counts 128 × 11 recessive bits. That matches the log signature exactly:

- every node silent at once;
- no `INIT` at recovery;
- a clean, idle bus as seen by the hub;
- every node resuming within about 10 ms, because they all finish recovering on the same bus at the same moment;
- after a recovery, often one frame from each node, then straight back into bus-off.

What the code can't explain on its own is why recovery stalls for minutes or hours on a bus the hub sees as idle. Recovery needs a clean recessive level *at each node's own RX pin*. Local noise, a ground or common-mode offset at the corners, or a transceiver in a bad state would all prevent it. **A timeout plus driver reinstall makes this path finite whatever the reason.** If the long silences shrink to seconds, the firmware latch is confirmed.

**N2. The health byte is the node's TX error counter, not a sensor fault.**

- `healthCheck()` sets CRITICAL when TEC > 200 or there are 10 consecutive TX-queue failures, and DEGRADED when TEC > 100 or there are more than 3. The low nibble is `consecutiveFails`.
- Every hub log line reading `health ... -> CRITICAL (node fail count 0)` therefore means **that node's TEC was above 200, a few errors short of bus-off (256).**
- Boot 0008, 07:27 to 08:04 on 13 Nov: a **37-minute error storm on the right-hand side**. BRAKE_FR, TIRE_FR and BRAKE_RR repeatedly went above TEC 100 and 200 while the bus stayed up. The left-hand nodes stayed quiet.
- This is the "borderline" fault in action: a real, localised source of errors that did not always tip the whole bus over.
- Only 4 of the 20 drop onsets in boot 0008 had a TEC warning in the 2 minutes before. The health byte is only sampled every 5 s, and TEC can go from 0 to bus-off in milliseconds.

**N3. Every TOBJ frame carries a sequence number in byte 3, and the hub throws it away.**

`sequenceNumber` increments on each frame queued for sending. The hub's view of it at stale and at recovery splits the open question **with the current v5.1.0 nodes, no node change needed**:

- **Small jump** across a dropout: the nodes stopped sending. That is N1, stuck in recovery.
- **Jump of about 50 × the gap in seconds**: the nodes kept sending and the hub stopped hearing them. That is the hub side or the CAN pair at the hub end.

**N4. No other give-up paths found.**

- The I2C recovery is bounded at about 60 ms.
- The task watchdog is 2 s with panic.
- TX uses a 5 ms timeout.
- The RX hardware filter means the node only ever receives its own OTA IDs.

## 2. Did the brakes come back at full temperature mid-stint?

**The logs cannot prove it either way.** The data CSV had already stopped recording before every recovery (bug S1 below), so no temperature was logged at any recovery.

What the sys_log does show:

- The bus came back mid-session and stayed up: for example 13 Nov 07:06:19 (44 min), 10:50:33 (34 min) and 14:34:09 (2 h 9 min).
- At every recovery the nodes **had not rebooted** and all came back within about 10 ms.

So seeing them come back already hot is expected: they were running and measuring the whole time and only lost the bus. **That fits your conclusion that this is not thermal throttling.**

## 3. Software issues found, in priority order

**S1 (critical): the data CSV dies silently.** Logging stops while the hub keeps running, the sys_log and TPMS capture keep writing to the same card, and nothing is logged about it:

| Log | CSV stopped | Hub ran until | Data lost |
|---|---|---|---|
| log_0008 | 05:26:35 | 00:23 next day | 19 h |
| log_0010 | 03:28:25 (at 115 km/h) | 05:21 | 2 h |
| log_0014 | 10:27:28 | 15:58 | 5.5 h |

- `sdLoggerTask` never checks what `println()` or `flush()` return.
- The most likely mechanism is a single failed write leaving the file object in a permanent error state (FatFs is sticky per file). The SD card shares its SPI bus with the TFT at 25 MHz, and the worst-case display time grew from 414 ms to 649 ms.
- **Fix:**
  - Check the bytes written and the flush result.
  - On failure, write a sys_log ERROR, close the file and reopen it as `log_NNNN_b.csv`.
  - Count reopens, and put rows written and the last good flush time in vitals.

**S2: don't reinstall the driver on the first frame back.** In 25 of those 26 one-burst recoveries the bus died within one frame of `Frame received in LISTEN_ONLY -> promoting to NORMAL`, a driver uninstall and reinstall 10 ms after the first frame. The logs can't show whether that caused the deaths or only coincided with them, but the change is cheap and safe:

- Only promote after about 500 ms of continuous frames from at least 2 nodes.
- Better still, stay in LISTEN_ONLY for telemetry altogether. With 7–8 nodes ACKing each other the hub's ACK is redundant, and NORMAL is only needed for node firmware jobs and probes.

That takes the hub out of the list of suspects. 2,190 driver reinstalls in boot 0008 alone is a lot of churn.

**S3 (revised): there is no node v5.2.0.**

- Hub v4.8.0's `GET_HEALTH` and bit-rate features need node v5.2.0. On v5.1.0 nodes they would get UNKNOWN_CMD.
- **Node v5.1.1, minimal and with no protocol change:**
  - Put a timeout on any recovery that stays pending, `RECOVERING` included: about 1–2 s, then a driver reinstall.
  - Retry forever with backoff and per-role jitter (fixes N1).
- **Hub:**
  - Log each node's sequence number at stale and at recovery (N3).
  - Log health changes as "TX error counter > 100 / > 200" instead of "node fail count" (N2).
  - Carry S1 and S2.
- **Leave the 250k bit-rate idea alone until this is solved.** It changes a variable mid-investigation.

**S4 (revised): health lines are bus errors, not sensor errors.** See N2. Earlier I read BRAKE_FR's 61 health changes as hot-sensor read failures. They are the node's own TX error counter going above 100 and 200. Keep them, label them correctly, and rate-limit them to one summary a minute.

**S5: brakes sit at 440.67 °C, the MLX saturation ceiling, for most of every hot stint.** Everything above about 380 °C is clipped. RAWIR extension stays off until it is calibrated (see the RAWIR notes).

**S6: vitals lines are cut at 220 characters**, so the `sd=`/`sys=`/`drops=` fields are lost. This is already fixed in v4.8.0.

**S7: timing headroom.**

- Worst-case aggregator time went from 45 ms to 179 ms, and display from 414 ms to 649 ms, over long boots.
- No CAN frames were missed (`CAN_RX_MISSED` stayed at 0), but this is the same SPI-bus contention that is the main suspect for S1.

## 4. Hardware actions

- **H1, the decisive test:** put a second CAN listener at the corner splice furthest from the hub (a USB-CAN dongle, or a spare ESP32 in LISTEN_ONLY logging to SD).
  - It keeps seeing frames while Sentinel goes silent: it's the hub-side CAN pair or connector.
  - It goes silent too: the nodes stopped driving the bus, so it's the supply.
- **H2:** on the bench, feed the node loom from a variable supply. Find the voltage where the TCAN3413s drop out while the C3s keep running. Then measure the real 3.3 V at a far corner with the car hot and the bus busy.
- **H3:** wiggle and heat test the hub M8 connector and every splice with the bus live.
- **H4 (on hold):** sending 12 V or 5 V down the loom and regulating locally on each node is good practice, but don't point the finger at power until H1 and the sequence-number logging have spoken.
- **H6:** inspect the right-hand-side branch (FR corner and RR brake): connectors, splices, shield and ground, and routing near the ignition or alternator. It produced the 37-minute TEC storm in boot 0008.
- **H5:** the FL brake node died alone at 18:53:15 on 13 Nov and stayed dead through three power cycles. It ran hottest (TA 113 °C). Bench it.

## 5. What worked well

- **TPMS:**
  - About 5,670 CRC-valid frames (FL 1,721, FR 1,749, RL 685, RR 1,516).
  - Pressure valid on 85–96 % of CSV rows across the long stints.
  - All four wheels tracked all weekend.
- **The hub itself:**
  - Boot 0008 ran 20.5 h straight with no crash or reset.
  - The CAN state machine never made things worse: it recovered on its own about 47 times, never jammed the bus, and showed `---` instead of frozen values.
- **GNSS:** steady 10 Hz, 19–22 satellites, PDOP around 1.1, worst epoch gap 130–188 ms.
- **The sys_log:** it flushed every line and kept going through everything. It is the only reason this analysis was possible.
- **Tyre temperatures:** clean and plausible whenever the bus was up.
- **The WiFi drop at +84 s on every boot is the TPMS RF survey switching WiFi off on purpose**, not a fault.

## 6. Fix release (written 18 Nov 2026, not yet bench tested)

**IR_CAN_NODE v5.3.0** is built on v5.1.0. There is no v5.2.0.

- **Recovery is bounded in both stuck states.** If recovery is still pending (`BUS_OFF` or `RECOVERING`) after 1 s plus backoff plus per-node jitter, the node reinstalls the driver. It retries forever. This is fix N1.
- **TX watchdog:**
  - no successful transmission for 2 s: driver reinstall;
  - none for 30 s: restart;
  - after 3 such restarts in a row (a lone node with no ACK partner), the restart interval stretches to 5 min.
- **A full TX queue** is cleared after 5 failed sends in a row.
- **Black box:** an RTC record of the node's last state, NVS lifetime counters and chip temperature, readable with `GET_HEALTH` pages 0–4. Hub v4.8.0+ already reads these.
- **Left out on purpose:** `SET_BITRATE` and bit-rate auto-detect.

**BTTM v4.9.0** is built on v4.8.0, which was never flashed, so v4.9.0 also brings all of v4.8.0's diagnostics.

- **S1:** a CSV write failure is now detected (short write, or the file size on the card not growing), logged, and the log reopened as `log_NNNN_b.csv`. An `sdstat` line goes to the sys_log every 10 min.
- **S2:** the hub only switches back to NORMAL after 500 ms of frames from at least 2 channels. A single-burst return is logged as `frames for N ms ... then silence -> not promoted`.
- **N3:** node sequence numbers are logged at every stale and recovery, with a verdict, and missed frames per channel go into `canstat`.
- **N2:** health lines now say "node TEC > 100 / > 200".
- **Bit rate:** the switch and the rate hunt are disabled, and the bus is pinned to 500k.
