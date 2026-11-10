# PS5 kernel backport status

## Base

- Kernel: **Linux v7.1.7** (`c7ba9d6de43e`); PS5 patchset `792ef83`
- Custom base: `b1acbd5c0cea` (`.config` / `.gitignore`),
  `0649cda23bc6` (`CONFIG_MT7921E=m`)
- Patch-repo HEAD reviewed: `1496086` (#78). Applied: #71, #75, #78 (parts).

Stay on 7.1.7 — newer kernels are unstable here. Only self-contained fixes are
backported.

## Done (newest first)

### 2026-11-10 — Bluetooth and Salina runtime PM (#71, #75, #78)

Backports of `f683ac4` (MT7921), `c5f999d` (IW620), `1496086` (Salina).

- `xhci-pci.c` — probe/remove also match `XHCI_PS5_SALINA_PHY`, holding a
  runtime-PM reference so Salina stays in D0; USB Bluetooth survives runtime
  D3. Both models.
- `btusb.c` — Slim MT7921 `0e8d:3603`: bind interface 0, pair bulk-OUT with
  bulk-IN, skip the ISO setup (`BTUSB_PAIRED_BULK_EP`, `BTUSB_MTK_NO_ISO`).
  Fixes pairing and kexec loss.
- `btusb.c` — fat IW620 `1286:2059`: clears LL-privacy/ext-adv bits and
  answers the repeated link-key request (`ps5_iw620_le`, `ps5_iw620_linkkey`,
  both default on). Fixes LE bonding and reconnects.
  Not taken: 7.3.1 move, PS5 OS pairing importer.
- Remaining IW620 issues are userspace: `JustWorksRepairing=always` and
  `PairableTimeout=0` in BlueZ; reload `btusb` or restart through the PS5 OS
  after a warm reboot with a changed address.

### 2026-10-27 — HDMI hotplug retry backoff (#54)

Backport of `1ec6a4a`. `ps5_hotplug_work` is a delayed work;
`dm_ps5_link_failed()` detects DP connectors with a stream but no active
link. After 3 consecutive failures, hotplug notifications back off 5 s,
doubling to 60 s; the resetup worker skips untrained links. Fields in
`amdgpu_dm.h`; both PS5 works use `system_dfl_wq`.

### 2026-10-27 — SCDC scrambling on 2160p60 exit (#50)

Backport of `7ff8d02`. `hdmi.c` tracks a `scrambling` flag; the FLAVA3 TMDS
setup writes `0x7c00 = 0x00` and waits when the previous mode had scrambling
on. Fixes no signal after switching from 2160p60 (VIC 97) to 1080p60/120.

### 2026-10-22 — HDMI display hotplug (#43)

Backport of `c091037`. `hdmi.c` gets a hotplug callback invoked once the EDID
arrives; FLAVA3 unplug/resume sequences restored. `amdgpu_dm.c` drives
hotplug through `ps5_hotplug_work` (`handle_hpd_irq_helper()`, 10 s window,
500 ms delayed commit to re-apply audio), forcing
`skip_detection_link_training`. Local `channels = 6` kept with a TODO.

### 2026-10-22 — FLAVA3 HDMI audio stability (#42)

Backport of `035d237`. `getHdmiDpAudioN()` returns `0x80000` for FLAVA3;
`enc1_se_setup_dp_audio()` writes it into `DP_SEC_AUD_N`.

### 2026-10-20 — fan detection / Oberon name

Backport of `7f695e4`. Oberon CPU name for models `0x46`–`0x48`; `fan.c` masks
`gddr6_revision = (gddr6_id >> 4) & 0xf`, so the Slim's revision 9 selects the
D1z fan table instead of `Not update`.

### 2026-10-19 — VCN 2.0 hardware video decode/encode

From `8081cd8`, final form `753c793`; gated by `CONFIG_X86_PS5`. IP block
registration, `ps5_vcn` firmware prefix, PS5 reset/start/stop, doorbells off,
plus the firmware blob. Result: H.264/HEVC decode+encode, VP9 decode.

## Evaluated and skipped

| Commit | PR | Subject | Reason |
|---|---|---|---|
| `56922cf` | #33 | Android binder | Waydroid only; unused. |
| `f367cd5` | — | config: 7.2.3 | Version churn; stay on 7.1.7. |
| `0d2ed27` | — | fix bugs, disable xhci | `PCIE_BUS_PERFORMANCE` unneeded; `rdmsrl` rename N/A; would disable working Salina xHCI. |
| `9322b9d` | #36 | MTS Ethernet fixes | Wired already works. |
| `8b7eaea` | #35 | btusb for 7.2.5 | Targets 7.2.5; existing quirks work here. |
| `20a9607` | #44 | VCN patcher | Tooling only; blob already embedded. |
| `670a4c5` | #49 | HDR on FLAVA3 | 174+/21−, overlaps #43/#54; unblocks #63/#67. |
| `5618f62` | #53 | HDMI CEC | New ~970-line driver; standalone. |
| `9bad84f` | #57 | USB-C DP + GFX1013 | 27 files; partly 7.2.3-oriented. |
| `f21f65b` | #58 | PS5 Pro GPU | Pro-only. |
| `730b80e` | #60 | fan hwmon curve | Fights `ps5fan.service`; `target_temp` suffices unless a custom curve/PWM is wanted. |
| `0e91638` | #62 | USB-C DP policy | Needs #57. |
| `a3e6846` | #63 | FRL/4:2:0/ALLM | Only for FRL modes; needs #49. |
| `22c80a1` | #40 | MT7921 Wi-Fi | Covered by `0649cda23bc6`. |
| `53809a3` | #66 | Pro fixes, GPU governor | Pro-only; base-PS5 parts need adaptation. |
| `fdc3eeb` | #67 | FRL defaults/recovery | Builds on #63/#49. |
| `495ea3d` | #68 | Pro CPU name | Pro-only cosmetic. |
| `f683ac4` | #71 | MT7921 + fan probe | Partially taken (MT7921); fan half needs #60. |
| `3c4b1b2` | #72 | USB-C DisplayPort | Needs #57. |
| `5f6b5d3` | #74 | HDMI 8 bpc | Deep-color only; N/A at 1080p60 4:4:4. |
| `c5f999d` | #75 | 7.3.1 + IW620 BT | Partially taken (IW620); 7.3.1 move, importer, RGB range not taken. |
| `7842e18` | #77 | KFD TLB flushes | ROCm only. |
| `1496086` | #78 | sensors/fan/Salina PM | Partially taken (Salina PM); sensors need #66, fan hand-over needs #60. |

## Fan notes

- Firmware-servo controlled; `ps5fan.service` picks a pattern (`on` 1,
  `max` 3, `off` 0). `max` is the default; `on` remains.
- `7f695e4` fixed Slim GDDR6 decoding so the D1z table uploads; before it,
  patterns were no-ops (tools issue #7).
- The probe's 80 °C is overridden by the service's pattern change (86 on
  Slim), so the service writes 75 °C in `ExecStartPost`; edit to 80 for less
  noise. Measured at 75: mean 86.5 → 74.8 °C, peak 94.25 → 80, frametimes
  unchanged.
- Slim idle ~72 °C is below target and holds the fan floor; the fat's larger
  heatsink keeps it near 60. Not a fault.
- #60 would replace this with a 500 ms kernel PWM loop, requiring the service
  to be disabled.

## 7.2.3 regression research

Why 7.1.7 is kept; unrelated to the VCN/fan backports.

- amdgpu/AMD iGPU: ~15 s freezes, FPS drops on 7.2.3; 7.2.2/LTS fine.
  <https://github.com/CachyOS/linux-cachyos/issues/1041>
- `r8169`/iSCSI/VLAN: timeouts, journal aborts, read-only remounts. N/A.
- btrfs large-folio lock leak (7.2-rc): writeback/syncfs stalls.
- Other: btusb HFP/SCO freezes; i915 s2idle black screen.
  <https://www.kernel.org/pub/linux/kernel/v7.x/ChangeLog-7.2>

## Notes

- `tools/testing/selftests/net/ovpn/data64.key` and
  `tools/testing/selftests/sgx/sign_key.pem` are sandbox artifacts — do not
  commit.
- Older batch (#58…#68) stays skipped.

## Pending

1. Build: `make -j$(nproc)`.
2. Boot checks:
   - `dmesg | grep -i "PS5 IW620"` (fat) and `grep -i salina`
   - Salina `power/runtime_status` stays `active`
   - BT: fat BLE pair/reconnect; Slim pair and reboot/kexec
