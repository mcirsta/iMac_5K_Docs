# macOS Display-State Capture Runbook (iMacPro1,1 5K tiling)

Date: 2026-06-06
Target machine: iMacPro1,1 (Vega 10 / DCE 12), the 5K panel that **stretches the
left tile under Linux but composites correctly under macOS**.

## Why we capture from macOS

We have proven that every signal Linux delivers to this panel (MSA geometry,
source routing, lane lock, ASSR, DPCD `0x4F1`/`0x107`, link training) is
**byte-identical to a working iMac19,1** — yet the Pro stretches. So the missing
piece is *below* anything Linux can read, in how the **Vega/DCE12 tiled bring-up**
is done.

Crucially:

- **Only macOS produces a working 5K-*tiled* state** on this machine. Firmware
  shows it stretched; systemd-boot dodges tiling with a single 4K mode; Linux
  (amdgpu) attempts the tiled mode and stretches.
- macOS can be in **both** states: a clean cold boot composites correctly
  ("working"), and after the Linux 5K kernel runs the panel is wedged so a warm
  reboot into macOS is also stretched ("wedged").

That gives us a clean **A/B on identical hardware**: capture macOS's display state
in the working state and again in the wedged state, then diff. Whatever changes
between them is what the working tiled bring-up has that the wedged one loses.

## Important: how to document the *visual* state

A normal **screenshot will NOT show the stretch** — `screencapture` records the
framebuffer (which is correct), while the stretch happens *in the panel*. To show
the physical symptom, take a **phone photo of the screen**, not a screenshot.
(This is itself confirmation the fault is panel/TCON-side.)

## What we capture (per state)

| Tool | What it gives |
|---|---|
| `AGDCDiagnose -a` | Apple's graphics display-controller dump: per-display link/timing/stream config (the best single source) |
| `ioreg` | GPU/framebuffer IORegistry tree, incl. tiling links (`IOFBDependentID`), per-framebuffer timing, EDID |
| `system_profiler SPDisplaysDataType` | high-level display/resolution view |
| `log show` (AMD/AGDC) | driver log around the display setup |

---

## Procedure

You will run the **same capture block twice** — once labelled `working`, once
labelled `wedged`. The only thing you change is the `STATE=` line.

### Step 1 — Capture the WORKING state

1. **Fully power off** the machine (hold power ~10 s; a real power cycle, not a
   restart — this clears any wedged panel state).
2. Boot **directly into macOS** (do **not** boot Linux first).
3. Confirm the panel looks **correct** (full, sharp 5K, both halves). Take a
   **phone photo** for the record.
4. Open **Terminal** and run the capture block below with `STATE=working`.

### Step 2 — Capture the WEDGED state

1. From macOS, **reboot into the Linux 5K kernel** and let it reach the stretched
   desktop.
2. **Reboot back into macOS — do NOT power off** (a power-off would clear the
   wedge). 
3. Confirm macOS now shows the **stretched** panel (phone photo for the record).
4. Open **Terminal** and run the same capture block with `STATE=wedged`.

> If after the Linux boot macOS is NOT stretched (i.e. it recovered), note that —
> it's an important data point on its own — and still capture as `wedged`.

---

## The capture block (paste into Terminal)

Run once per state. Change only the first line.

```sh
STATE=working          # <-- set to: working   (first run)
                       #     then:    wedged    (second run)

OUT=~/imacpro_5k_capture
mkdir -p "$OUT"

# --- 1. AGDCDiagnose (best source) ---------------------------------------
AGDC=$(sudo find /System/Library/Extensions -name AGDCDiagnose 2>/dev/null | head -1)
[ -z "$AGDC" ] && AGDC=$(sudo find / -name AGDCDiagnose 2>/dev/null | head -1)
echo "AGDCDiagnose: ${AGDC:-NOT FOUND}"
if [ -n "$AGDC" ]; then
    # run in place; if SIP blocks it, fall back to a copy in /tmp
    sudo "$AGDC" -a > "$OUT/agdc_$STATE.txt" 2>&1 \
      || { sudo cp "$AGDC" /tmp/AGDCDiagnose && sudo /tmp/AGDCDiagnose -a > "$OUT/agdc_$STATE.txt" 2>&1; }
fi

# --- 2. ioreg: full tree + display/framebuffer subtrees -------------------
ioreg -lw0                                  > "$OUT/ioreg_full_$STATE.txt"
ioreg -lw0 -r -c IODisplayConnect 2>/dev/null > "$OUT/ioreg_display_$STATE.txt"
ioreg -lw0 -r -c IOFramebuffer 2>/dev/null    > "$OUT/ioreg_fb_$STATE.txt"
ioreg -lw0 -r -c IOMobileFramebuffer 2>/dev/null >> "$OUT/ioreg_fb_$STATE.txt"

# quick highlight: how macOS links the two tiles + per-fb timing
ioreg -lw0 | grep -iE "IOFBDependent|IOFBCurrentPixel|IOFBConfig|online|tile|stereo|transport|LinkRate|LaneCount|MST" \
                                            > "$OUT/ioreg_tilehints_$STATE.txt"

# --- 3. system_profiler displays -----------------------------------------
system_profiler SPDisplaysDataType          > "$OUT/spdisplays_$STATE.txt" 2>&1

# --- 4. AMD / graphics driver log (last 5 min) ---------------------------
sudo log show --last 5m --info --debug \
  --predicate 'senderImagePath CONTAINS[c] "AMD" OR senderImagePath CONTAINS[c] "AGDC" OR senderImagePath CONTAINS[c] "GraphicsControl" OR senderImagePath CONTAINS[c] "GPUWrangler"' \
                                            > "$OUT/amdlog_$STATE.txt" 2>&1

echo "==> saved $STATE capture to $OUT"
ls -la "$OUT"
```

---

## Step 3 — Package and send

After both runs (`working` and `wedged`) are in `~/imacpro_5k_capture`:

```sh
cd ~ && tar czf imacpro_5k_capture.tar.gz imacpro_5k_capture
echo "send: ~/imacpro_5k_capture.tar.gz"
```

Send `imacpro_5k_capture.tar.gz` plus the two **phone photos** (working vs
wedged screen).

---

## If `AGDCDiagnose` is missing or won't run

It may not exist on every macOS version, or SIP may block it. That's fine —
`ioreg` + `system_profiler` are the reliable fallbacks and still capture the key
data (the tiling links and per-framebuffer timing). Just run the block without
the AGDC section; the rest still produces useful `_working` vs `_wedged` files.

To check availability first:

```sh
sudo find / -name AGDCDiagnose 2>/dev/null
```

---

## What we'll do with it

Diff `working` vs `wedged` for each artifact:

- `agdc_working.txt` vs `agdc_wedged.txt` — the display-controller link/stream/
  timing config that flips.
- `ioreg_tilehints_*` / `ioreg_fb_*` — whether the two tile framebuffers stay
  linked (`IOFBDependentID`) and what per-tile timing/link settings differ.
- `spdisplays_*` — whether macOS still reports one 5120×2880 display.

The first thing that differs between the correct-tiled and wedged states is the
behaviour Linux's `dce120`/Vega path needs to reproduce.

## Notes / caveats

- **Don't power off between the Linux boot and the wedged capture** — a power
  cycle clears the wedge and you'd capture a working state by mistake.
- `system_profiler` may report the **same resolution** in both states (macOS
  still thinks it's driving 5K); the real difference is in the lower-level
  AGDC/ioreg link state, not the reported resolution.
- All commands are **read-only** diagnostics; none change system configuration.
