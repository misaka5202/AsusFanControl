# AsusFanControl

Fan telemetry and fan-mode control for ASUS laptops, as a VirtualSMC plugin.

Split out of [AsusSMC](https://github.com/hieplpvip/AsusSMC) because the fan
data does not come from the EC or from the CPU: it comes from the ASUS platform
firmware, reached through the same WMI/ATK ACPI device that AsusSMC uses for the
keyboard backlight, the ALS and the battery. Keeping it in the SMC layer made
that kext responsible for hardware it does not own, and made the fan code
impossible to use on its own.

## SMC keys

| Key | Meaning |
|---|---|
| `FNum` | number of fans |
| `F0Ac` / `F1Ac` | fan speed, `fpe2` (RPM × 4) |
| `F0Mn` / `F0Mx` / `F1Mn` / `F1Mx` | usable range, constants |
| `F0Id` / `F1Id` | fan descriptor |
| `F0Md` / `F1Md` | `0` = firmware full speed, `2` = automatic |
| `F0Tp` | thermal profile: `0` balanced, `1` performance, `2` silent |

`F0Md` and `F0Tp` are writes. Everything else is read-only.

## Requirements

* macOS 10.11 or newer, x86_64
* [Lilu](https://github.com/acidanthera/Lilu) and
  [VirtualSMC](https://github.com/acidanthera/VirtualSMC) loaded first
* An ASUS laptop whose ACPI exposes the ATK device (`PNP0C14` / `ATK0100`) with
  a `WMNB` method

## Install

1. `AsusFanControl.kext` into `EFI/OC/Kexts`
2. In `config.plist` → `Kernel` → `Add`, add an entry after `VirtualSMC`
3. Reboot

If AsusSMC is also installed, put this one after it. Both kexts match the ATKD
device independently - ACPI devices accept several drivers - and the probe
scores are chosen so they do not displace each other.

## Build

Both SDKs are fetched from the source repositories rather than from release
archives, then the project is built with Xcode:

```sh
git clone --depth 1 https://github.com/acidanthera/MacKernelSDK
# fetch Lilu headers + VirtualSMCSDK as described in .github/workflows/build.yml
xcodebuild -configuration Release -project AsusFanControl.xcodeproj \
  -target AsusFanControl ARCHS=x86_64 ONLY_ACTIVE_ARCH=NO \
  CODE_SIGNING_ALLOWED=NO CODE_SIGN_IDENTITY=""
```

Xcode matters here. A kext target runs `CreateKModInfo`, and the kernel will
not load an extension that lacks the resulting `kmod_info`. A hand-written
`clang` link produces a kext that installs cleanly and then never appears in
`kmutil showloaded` - a silent failure at boot.

## Notes on this hardware

Measured on an ASUS ROG Zephyrus G15 GA503QM (Ryzen 7 5800HS), macOS 26.7.2.

**`TACH` does not return RPM here.** `EC0.TACH()` computes
`0x0041CDB4 / (raw * 2)`; with the counters the EC actually reports (`0x17..0x22`)
that yields 59000-93750 "RPM". The firmware path (`DEVS`/`DSTS`, device
`0x00110013` / `0x00110014`) reports the same counter decoded as `raw * 100`,
the same way `asus-wmi.c` does. `TACH` is only trusted when it returns
something plausible.

**The WMI argument buffer must be 20 bytes.** `ATKD.WMNB` declares a 20-byte
argument buffer on this board generation; sending 8 bytes is rejected with
`AE_AML_BUFFER_LIMIT` and every WMI call fails. The struct here is 24 bytes
(six `uint32`), which satisfies the declaration.

**Thermal profile values.** The DSDT's `If/ElseIf` chain only covers `0`, `1`
and `2`. Any other value leaves the local uninitialised and writes that to the
EC, which was measured to spin both fans to about 5000 RPM with no predictable
result. Out-of-range values are rejected before they reach `DEVS`.

## License

Same terms as AsusSMC. AsusSMC is © 2018-2020 Le Bao Hiep.
