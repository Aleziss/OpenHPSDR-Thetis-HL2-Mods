# OpenHPSDR Thetis HL2 Modifications — VA2CST

This repository contains modifications to [MI0BOT's OpenHPSDR-Thetis](https://github.com/mi0bot/OpenHPSDR-Thetis/releases/tag/v2.10.3.14) (v2.10.3.14), specifically targeting Hermes-Lite 2 users with an LDG automatic antenna tuner connected via the [N2ADR HL2 IO Board](https://github.com/jimahlstrom/HL2IOBoard) using my modded code [HL2IOBoard-LDG-AT1000-ProII](https://github.com/Aleziss/HL2IOBoard-LDG-AT1000-ProII).

> [!NOTE]
> You do not need this modded version of Thetis to run the [HL2IOBoard-LDG-AT1000-ProII](https://github.com/Aleziss/HL2IOBoard-LDG-AT1000-ProII) code to support a LDG tuner. With the standard release of Thetis v2.10.3.14-HL2, you'll be limited to ~8s auto tuning and no physical control from the LDG TUNE button.

## Modifications Overview

Two issues have been identified and resolved in `console.cs`:

### Fix 1 — ATU Timeout Extended (~20 seconds)

**Problem:** When using `CTRL+TUN` to initiate automatic antenna tuning, Thetis was cancelling the tuning process after approximately 7–8 seconds. Some ATUs (such as the LDG AT-1000 Pro II) can require up to 15–18 seconds to complete a tuning cycle, causing Thetis to abort prematurely.

**Root Cause:** Found in `console.cs` in the `AutoTuningHL2()` function. The I/O Board polling loop runs every 40ms and cycles through 12 states (case 0 to case 11). The `AutoTuningHL2()` function is called at cases 1, 4, 7 and 10, meaning it is effectively called every ~160ms. A single constant `TIMEOUT = 50` controlled the tune cycle counter:
```
160ms × 50 = ~8 seconds
```

**Fix Applied:**
```csharp
// BEFORE:
const byte TIMEOUT = 50;

// AFTER:
const byte TIMEOUT = 125;        // VA2CST: Auto tune timeout extended ~20s (was 50 = ~8s)
const byte FAULT_TIMEOUT = 50;   // VA2CST: Fault message display timeout ~8s (separated from TIMEOUT)
```
The `FAULT_TIMEOUT` constant was added to keep the fault message display duration unchanged at ~8 seconds, while the tuning timeout was extended to ~20 seconds.

Fault timeout comparison:
```csharp
// BEFORE:
if (fault_timeout++ >= TIMEOUT)

// AFTER:
if (fault_timeout++ >= FAULT_TIMEOUT)
```

---

### Fix 2 — Hardware TUNE Button Support (without CTRL+TUN)

**Problem:** Thetis only responded to the `0xEE` (RequestRF) command from the IO Board when `CTRL+TUN` had been previously activated in Thetis. Pressing the physical TUNE button on the ATU hardware had no effect.

**Background:** When the physical TUNE button on the LDG AT-1000 Pro II is held >500ms and released, the tuner pulls the KEY line LOW to request RF from Thetis. The Pico on the IO Board monitors the KEY line (GPIO18/In2). Without `CTRL+TUN`, Thetis ignored this hardware request entirely.

**Root Cause:** Thetis reads `REG_INPUT_PINS` from the IO Board and only processes `REG_ANTENNA_TUNER` based on specific input pin states. The KEY line state (In2, bit 2) was never directly used to initiate tuning.

**Fix Applied:**

**Step 1** — Add a private variable near `tune_timeout` and `fault_timeout` declarations:
```csharp
private bool key_was_high = false; // VA2CST: Tracks In2 state for HIGH-to-LOW transition detection
                                   // Prevents false RF triggering if pull-up resistor is missing on J8 In2
```

**Step 2** — In the `UpdateIOBoard()` polling loop (`case 10`), after reading `REG_INPUT_PINS`, Thetis now checks if In2 (KEY line, bit 2) transitioned from HIGH to LOW while in `Idle` state:
```csharp
byte inputPins = (byte)ioBoard.readRegister(IOBoard.Registers.REG_INPUT_PINS);

// VA2CST: Detect hardware TUNE button - In2 (KEY) HIGH-to-LOW transition while idle
// key_was_high ensures In2 was HIGH before going LOW (pull-up resistor required on J8 In2)
if ((inputPins & 0x04) == 0 && auto_tuning == AutoTuneState.Idle && key_was_high)
{
    auto_tuning = AutoTuneState.WaitRF;
    tune_timeout = 0;
    this.Invoke((Action)(() =>
    {
        chkTUN.Checked = true;
        chkTUN_CheckedChanged(this, EventArgs.Empty);
        chkTUN.Text = "AUTO";
    }));
    auto_tuning = AutoTuneState.Tuning;
}
else if ((inputPins & 0x04) == 0 && auto_tuning == AutoTuneState.Tuning)
{
    // VA2CST: Hardware TUNE in progress - keep RF going while KEY is LOW
    AutoTuningHL2(ProtocolEvent.RequestRF);
}

// ... existing fault and antenna tuner code ...

SetupForm.UpdateIOLedStrip(MOX, inputPins);
// VA2CST: Update KEY line state for HIGH-to-LOW transition detection on next cycle
key_was_high = (inputPins & 0x04) != 0;
```

**Step 3** — In `AutoTuningHL2()`, add handling for `RequestRF` when in `Idle` state:
```csharp
case ProtocolEvent.RequestRF:
    switch (auto_tuning)
    {
        case AutoTuneState.Idle:          // VA2CST: Hardware TUNE button pressed
            auto_tuning = AutoTuneState.Tuning;
            tune_timeout = 0;
            this.Invoke((Action)(() =>
            {
                chkTUN.Checked = true;
                chkTUN_CheckedChanged(this, EventArgs.Empty);
                chkTUN.Text = "AUTO";
            }));
            break;
        // ... existing cases unchanged
    }
```

> [!IMPORTANT]
> A **4.7kΩ pull-up resistor to 5V (P3 on IO Board)** is required on J8 In2 (KEY line). Without it, In2 floats LOW at startup and Thetis will not trigger tuning since `key_was_high` will remain `false` until In2 is seen HIGH at least once.

---

## Hardware Requirements

- Hermes-Lite 2 (HL2)
- [N2ADR HL2 IO Board](https://github.com/jimahlstrom/HL2IOBoard)
- LDG AT-1000 Pro II (or compatible Icom AH-4 protocol ATU)
- KEY line connected to J8 pin 2 (GPIO18/In2) with **4.7kΩ pull-up resistor to 5V (P3 on IO Board)**
- START line connected to J6 pin 6 (GPIO22/Out6)

---

## Installation

1. Navigate to your Thetis installation folder:
```
C:\Program Files\OpenHPSDR\Thetis-HL2\
```
2. Rename the original executable as a backup:
```
Thetis.exe → Thetis_original.exe
```
3. Copy the modified `Thetis.exe` from the [Releases](../../releases) page to the installation folder
4. Launch Thetis normally

> [!NOTE]
> Your antivirus may flag an unsigned executable. You may need to add an exception. The source code (`console.cs`) is provided for full transparency.

> [!NOTE]
> It is possible that you will need to reconfigure Thetis (set skins, audio settings, etc) with this thetis.exe file, similar to a new install even though it is not. To be safe, save your database settings from your current working version and reimport it to either Thetis version, they are both the same v2.10.3.14 so it won't cause any import problem.

---

## Video Demonstration

You can see a demonstration of the modded code [HERE](https://youtu.be/9eE7b4Y-6QY?si=WgPXGFOQaXzIzFvE)

---

## Reapplying Changes to a New Version of Thetis

If MI0BOT releases a new version of Thetis, the changes can be reapplied to `console.cs` by:

1. Locating the `AutoTuningHL2()` function
2. Changing `const byte TIMEOUT = 50` to `const byte TIMEOUT = 125` and adding `const byte FAULT_TIMEOUT = 50`
3. Updating `if (fault_timeout++ >= TIMEOUT)` to `if (fault_timeout++ >= FAULT_TIMEOUT)`
4. Adding `private bool key_was_high = false` near the other private variables
5. Adding the `case AutoTuneState.Idle` block in `ProtocolEvent.RequestRF`
6. Adding the In2 detection logic with `key_was_high` in the `UpdateIOBoard()` polling loop
7. Adding `key_was_high = (inputPins & 0x04) != 0` at the end of the `if (status == 0)` block

Refer to the provided `console.cs` for exact implementation details.

---

## Based On

- [mi0bot/OpenHPSDR-Thetis](https://github.com/mi0bot/OpenHPSDR-Thetis) v2.10.3.14
- [jimahlstrom/HL2IOBoard](https://github.com/jimahlstrom/HL2IOBoard)

## Author

Claude Perreault, VA2CST — 2026
