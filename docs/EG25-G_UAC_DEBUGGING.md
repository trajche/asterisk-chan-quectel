# Notes from debugging EG25-G UAC voice with chan_quectel

These are the conclusions from a long debugging session with a Quectel
EG25-G modem on a Radxa E25 (aarch64, Armbian/Debian). The goal was
bidirectional voice over the modem's USB Audio Class endpoint. The
findings here are intended to save the next contributor weeks of work.

## TL;DR

`chan_quectel`'s UAC code path does not currently produce working
bidirectional audio against any EG25-G firmware we tried. The audio
problems split into **three independent issues** that all have to be
fixed:

1. **Modem firmware** — only `EG25GGBR07A08M2G_30.202.00.000` is known to
   route uplink audio through the UAC playback endpoint correctly. The
   stock `_A0.300.A0.300` UAC firmware that Quectel ships on request,
   and the newer `_30.203.30.203`, both silently drop uplink. Symptom:
   capture works (caller's voice arrives as real PCM via the UAC capture
   endpoint), but **anything written to the UAC playback endpoint never
   reaches the air interface**, including `AT+VTS` DTMF generated
   internally by the modem.

2. **ALSA tooling** — `aplay`/`arecord` and the MMAP path that
   `chan_quectel` uses produce zero-amplitude or garbled audio against
   the EG25-G UAC card, even on the working firmware. `tinyplay` and
   `tinycap` from <https://github.com/tinyalsa/tinyalsa> work cleanly
   with identical settings (8 kHz S16 LE mono). The Quectel UAC
   Application Note V1.0 §4.2.2 explicitly recommends `tinyalsa` — they
   were not kidding.

3. **URC handling** — the EG25-G running 30.202 firmware does **not**
   send `+CRING` URCs by default for incoming voice calls. The first
   thing it sends is `+CLIP: "<num>",145,...` followed by
   `+QIND: "ccinfo",<idx>,1,4,...`. `chan_quectel` currently keys ring
   detection off `+CRING` exclusively, so it never picks up incoming
   calls on this firmware until either `AT+CRC=1` is sent at init AND
   the modem actually generates +CRING (it does after `AT+CRC=1`), or
   the URC dispatcher learns to treat `+CLIP` and `ccinfo state=4` as
   incoming-call triggers.

## Firmware compatibility matrix

Tested on the same SIM (MTEL MK / Macedonia, CSFB voice, no VoLTE).

| Firmware | UAC enumerates | Capture | Playback | Notes |
|---|---|---|---|---|
| `EG25GGBR07A07M2G_01.001.01.001` (stock) | ❌ | n/a | n/a | UAC bit off in `AT+QCFG="usbcfg"`. |
| `EG25GGBR07A07M2G_A0.300.A0.300` | ✅ | ✅ real PCM | ❌ silent | The "UAC firmware" Quectel sends on request. Uplink path is broken. AT+VTS also silent. |
| `EG25GGBR07A08M2G_30.203.30.203` | ✅ | ✅ real PCM | ❌ garble | Bytes reach the air uplink but get mangled regardless of format/rate/codec config. |
| **`EG25GGBR07A08M2G_30.202.00.000`** | ✅ | ✅ real PCM | ✅ clean | **The one that works.** Branch `EG25GGBR07A08M2G_30.202.30.202` in the [Biktorgj recovery archive](https://github.com/Biktorgj/quectel_eg25_recovery). |

A user on the Quectel forum independently reports the same finding:
<https://forums.quectel.com/t/eg25-and-uac-for-audio/31663> — they
pinpoint 30.202 as the version that fixed silent uplink for them.

## Required modem AT commands

Once on a working firmware, this is the minimum sequence:

```
AT+QCFG="usbcfg",0x2C7C,0x0125,1,1,1,1,1,0,1   ; persists, enables UAC USB endpoints
AT+CFUN=1,1                                     ; reset to apply usbcfg
; --- volatile, re-apply on every modem boot ---
AT+CVMOD=0                                      ; CSFB voice (only path on networks without VoLTE/IMS)
AT+QPCMV=1,2                                    ; route voice through UAC
AT+CLIP=1                                       ; caller-ID URC
AT+CRC=1                                        ; +CRING URC
AT+QINDCFG="ccinfo",1,1                         ; Quectel ccinfo URC + persist flag
```

Note `AT+CMUT=0` and `AT+QSIDET` return **ERROR** on this firmware —
not supported, ignore them. `AT+QGAINSET` also errors. Default mic gain
(`AT+QMIC?` reports `+QMIC: 4,16384`) is fine.

`AT+QPCMV=0` and `AT+QPCMV=1,1` enumerate as `USB NMEA mode` and
`Debug UART mode` respectively per the docs and don't carry voice.

## Ring detection

Reproducible behaviour on 30.202 firmware, fresh SIM call:

```
+CLIP: "+38970349147",145,,,,0
+QIND: "ccinfo",2,1,4,0,0,"+38970349147",145
```

No `+CRING`, no bare `RING`. With `AT+CRC=1` set, the modem may also
emit `+CRING: VOICE` lines, but that's not guaranteed.

`chan_quectel` must accept any of these as the ring trigger:
- `RING` (bare)
- `+CRING`
- `+CLIP:` (always carries caller ID)
- `+QIND: "ccinfo",<idx>,1,4,...` (Quectel-specific, fires once on incoming)

The reference is [`quectel-daemon.py`](https://github.com/trajche/asterisk-chan-quectel/blob/fix/uac-stream-lifecycle/) — a small Python daemon
that replaced chan_quectel in the production setup; see
`_on_line` for the URC dispatch and `_handle_call` for the answer +
play + record sequence.

## ALSA path notes

What did NOT produce clean audio:
- `aplay -D hw:EG25G` (default MMAP path used by `chan_quectel` — same
  result regardless of `--buffer-size`/`--period-size` tuning)
- `aplay -D plughw:EG25G`
- `cat tone.wav > /dev/snd/pcmC0D0p`
- `chan_quectel`'s own snd_pcm_mmap_writei loop
- `chan_quectel`'s RW-mode build (after this branch's commit
  `fix(uac): RW access + explicit pcm_start + no link`)

What works against the same hardware + firmware + same WAV file:

```sh
tinyplay /var/lib/asterisk/sounds/en/custom/zoidberg-whoop.wav -D 0 -d 0
tinycap /tmp/out.wav -D 0 -d 0 -c 1 -r 8000 -b 16 -t 30
```

We did not isolate which specific ALSA call differs. The recommendation
is to either:
1. Spawn `tinyplay`/`tinycap` subprocesses from chan_quectel for UAC
   audio I/O — ugly but proven to work, and it's what the Quectel App
   Note recommends; or
2. Reverse-engineer what tinyalsa does differently from `aplay` and
   port that into `chan_quectel`'s `pcm.c` directly.

The (1) variant is the path of least resistance for getting a working
patch out the door. The (2) variant is the right thing in the long run
but requires someone to dig through the tinyalsa source and figure out
the exact PCM hw_params dance that the EG25-G accepts.

## Misc dead ends — please don't repeat

- The phantom `+CLCC: 1,1,0,1,0,"",128` (a "data call" that survives
  full modem reset and even physical USB replug) is harmless. It's
  some internal modem signaling session, not a stuck call. Ignore it.
- `AT+QDAI` does NOT need to be set to 5 ("USB Digital Audio") — the
  Quectel UAC App Note never mentions QDAI, and changing it breaks
  capture without fixing playback. Leave it at default.
- `AT+QCFG="amrcodec"` doesn't fix the playback problem on the broken
  firmwares; on the working firmware it doesn't need to be touched.
- VoLTE/IMS (`AT+QCFG="ims"`) doesn't matter for CSFB voice. CSFB
  works fine on networks without VoLTE.
- ModemManager ate the AT port and competed with chan_quectel —
  `systemctl disable --now ModemManager` if it's installed.
- The Radxa E25 brownout-resets when the modem is hot-plugged on a
  weak PSU. Use a 3 A USB-C charger or a powered USB hub. Not a
  software issue.

## Reproduction recipe

A self-contained test that proves whether your firmware has the
playback bug. Run on the host with the modem at `/dev/ttyUSB2` and a
phone you can call FROM:

```python
#!/usr/bin/env python3
import time, subprocess, struct, math, serial
NUMBER = '+1234567890'   # your other phone
s = serial.Serial('/dev/ttyUSB2', 115200, timeout=2)
def at(c, w=1):
    s.write((c+'\r').encode()); time.sleep(w)
    print(c, '=>', s.read(2048).decode(errors='replace').strip())
at('AT+CHUP'); at('AT+CVMOD=0'); at('AT+QPCMV=1,2')
at(f'ATD{NUMBER};')
time.sleep(12)   # wait for answer

# 2-second 440 Hz tone
n=16000
data=b''.join(struct.pack('<h', int(8000*math.sin(2*math.pi*440*i/8000))) for i in range(n))
hdr=(b'RIFF'+struct.pack('<I',36+len(data))+b'WAVEfmt '+
     struct.pack('<IHHIIHH',16,1,1,8000,16000,2,16)+b'data'+struct.pack('<I',len(data)))
with open('/tmp/t.wav','wb') as f: f.write(hdr+data)

subprocess.run(['tinyplay','/tmp/t.wav','-D','0','-d','0'])
at('ATH')
```

Pick up your other phone, listen.
- Clean continuous 440 Hz tone → firmware works, you're on a good build.
- Loud garbled noise → broken firmware (e.g. 30.203 or A0.300). Flash
  30.202.
- Total silence → not even uplink works; could be wrong CVMOD, wrong
  QPCMV, or a non-UAC firmware.

Send the same recipe to Quectel support if you need to escalate; it's
a 30-second test that proves the firmware bug.
