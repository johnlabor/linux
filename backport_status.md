# PS5 kernel backport status

## Base

- Kernel: **Linux v7.1.7** (`c7ba9d6de43e`)
- PS5 patchset on top: `792ef83`
- Custom base commits: `b1acbd5c0cea` (`.config` / `.gitignore`),
  `0649cda23bc6` (`CONFIG_MT7921E=m`)
- Last applied patch-repo commit: `1ec6a4a` (#54)
- Patch-repo HEAD reviewed: `495ea3d`

Stay on 7.1.7 — newer kernels are unstable here. Only self-contained fixes are
backported.

## Done (newest first)

### 2026-10-27 — HDMI hotplug retry backoff (#54)

Backport of `1ec6a4a`.

- `amdgpu_dm.c` — `ps5_hotplug_work` is now a delayed work. New
  `dm_ps5_link_failed()` detects DP connectors whose CRTC has a stream but
  whose link is not active; after `PS5_HOTPLUG_RETRIES` (3) consecutive
  failures, new hotplug notifications are delayed 5 s, doubling up to 60 s.
  The resetup worker skips untrained links, and both PS5 works use
  `system_dfl_wq`.
- `amdgpu_dm.h` — delayed-work type and `ps5_hotplug_failures` counter.
- Logs a warning when backing off and an info line when the link recovers.

### 2026-10-27 — SCDC scrambling on 2160p60 exit (#50)

Backport of `7ff8d02`.

- `drivers/ps5/hdmi.c` — added a `scrambling` flag; the FLAVA3 TMDS setup now
  writes `0x7c00 = 0x00` and waits when the previous mode had scrambling on,
  then records `scrambling = vic == 97`.

Fixes no signal after switching down from 2160p60 (VIC 97) to 1080p60/120.
Upstream tested those transitions on an Elgato 4K Pro.

### 2026-10-22 — HDMI display hotplug (#43)

Backport of `c091037`.

- `drivers/ps5/hdmi.c` — `hdmi_set_hotplug_handler()` callback, invoked from
  `hdmi_notification_handler()` once the real EDID arrives; FLAVA3
  unplugged/wait-resolution sequences restored in `hdmiSystemResume()`.
- `amdgpu_dm.c` — PS5 keeps HPD off but now drives hotplug through
  `ps5_hotplug_work`: the work calls `handle_hpd_irq_helper()` for DP
  connectors, opens a 10 s window, and a 500 ms delayed work forces an atomic
  commit so audio is re-applied. Forces `skip_detection_link_training` because
  link training only works during a modeset. Commit-tail setup moved into
  `dm_ps5_hdmi_enable()`.
- `amdgpu_dm.h` — four `amdgpu_display_manager` fields under `CONFIG_X86_PS5`.

Local `channels = 6` (`7370c237`) kept in the helper with a TODO — upstream
derives the count from EDID. `hdmi.c` and `ps5.h` match upstream blobs
(`a4ace5c`, `5f27f6e`).

### 2026-10-22 — FLAVA3 HDMI audio stability (#42)

Backport of `035d237`.

- `drivers/ps5/hdmi.c` — `getHdmiDpAudioN()`: FLAVA3 returns `0x80000`
  (default Naud is choppy, Maud too coarse), others keep the caller's value.
- `include/linux/ps5.h` — prototype.
- `dcn10_stream_encoder.c` — `enc1_se_setup_dp_audio()` writes the helper's
  value into `DP_SEC_AUD_N`.

Post-patch blobs match upstream exactly (`2ee166b`, `322b2a6`, `501f97a`).

### 2026-10-20 — fan detection / Oberon name

Backport of `7f695e4`.

- `arch/x86/kernel/cpu/amd.c` — Oberon name for models `0x46`–`0x48` (cosmetic).
- `drivers/ps5/fan.c` — `gddr6_revision = (gddr6_id >> 4) & 0xf;`

Before: `D-Eiger/Denali Unknown Vendor/Revision 0x1/0xb9` → `Not update`.
Masking yields revision 9 (Samsung D1z), installing the correct fan table.

### 2026-10-19 — VCN 2.0 hardware video decode/encode

From `8081cd8`, final form `753c793`; gated only by `CONFIG_X86_PS5`.

- `amdgpu_discovery.c`, `amdgpu_vcn.c`, `vcn_v2_0.c`: IP block registration,
  `ps5_vcn` firmware prefix, PS5 reset/start/stop, doorbells off.
- `.config` `CONFIG_EXTRA_FIRMWARE`, `.gitignore` negation, firmware blob +
  license in `firmware/` (self-contained repo).

Result: H.264/HEVC decode+encode, VP9 decode. No AV1 (not in VCN 2.0).

## Evaluated and skipped

Reviewed and not backported: does not apply to this tree, is Pro-only, already
covered, or was dropped after evaluation. Items previously marked deferred
(#49, #53, #57) are included — they are not planned at this point.

| Commit | Date · PR | Subject | Reason |
|---|---|---|---|
| `56922cf` | 2026-08-09 · #33 | config: enable Android binder | Only needed to run Waydroid; nothing in the PS5 path uses binder. |
| `f367cd5` | 2026-09-05 · — | config: update to 7.2.3 | Pure version churn to move the base to 7.2.3; we stay on 7.1.7. |
| `0d2ed27` | 2026-09-05 · — | fix bugs, disable xhci | Three unrelated changes: `PCIE_BUS_PERFORMANCE` (not needed), `rdmsrl`→`rdmsrq` (7.2 rename, N/A), and wrapping `xhci_salina_init()` in `#if 0` (would disable the working Salina xHCI path). |
| `9322b9d` | 2026-09-28 · #36 | MTS Ethernet fixes | Wired-only: IMR polarity, multicast hash, IRQ coalescing, `napi_disable()` before DMA stop, RX ring 256 → 1024. Wired networking already works; wireless untouched. |
| `8b7eaea` | 2026-09-29 · #35 | btusb changes for 7.2.5 | Targets 7.2.5 (`BTUSB_BROKEN_EXT_SCAN` BIT(29), IW620 quirks); the existing PS5 quirk works on 7.1.7. Revisit only if BT regressions appear. |
| `20a9607` | 2026-10-22 · #44 | firmware: VCN patcher | Tooling only (Python script + README): generates `ps5_vcn.bin`. No kernel change; blob already embedded. |
| `670a4c5` | 2026-10-22 · #49 | HDR output on FLAVA3 | Not taken: 174+/21− in `amdgpu_dm.c`, `hdmi.c`, `ps5.h`; overlaps the #43/#54 commit-tail and HDMI paths. Would also unblock #63/#67. |
| `5618f62` | 2026-10-24 · #53 | HDMI CEC support | Not taken: new `drivers/ps5/ps5-cec.c` (~970 lines) plus `spcie.c` message routing; standalone feature, not requested. |
| `9bad84f` | 2026-10-27 · #57 | USB-C DP + GFX1013 fixes | Not taken: 27 files, 1521+/134−; USB-C HPD/AUX worker, DCN201/DSC0, GFX1013 GFXOFF/PASID TLB, spcie/MTS/HIDP/xHCI changes; partly 7.2.3-oriented. |
| `f21f65b` | 2026-10-28 · #58 | PS5 Pro GPU support, SMU backend | Pro-only: Viola GPU (`1002:13da`), firmware service/ucode, 64-CU mask, SMU mailbox backend. Base PS5 unaffected; Pro firmware not in this tree. |
| `730b80e` | 2026-10-28 · #60 | ps5: fan: add hwmon curve control | Overlaps `ps5fan.service` (both drive ICC fan service `0x0a`; needs zones in AUTO at probe, fights pattern 1/3) and polls temp/PWM every 500 ms. `--fan on`/`max` already covers cooling. Take only for a tunable curve or hwmon readout, then disable `ps5fan.service`. |
| `0e91638` | 2026-10-29 · #62 | Leave USB-C DP policy off by default | N/A without #57: patches `drivers/ps5/usbc_dp.c`, which only exists after #57. |
| `a3e6846` | 2026-10-29 · #63 | Native FRL, 4:2:0, ALLM, HDMI fixes | Only useful for FRL/high-bandwidth modes (4K60/4K120, high bit depth); at 1080p60 4:4:4 TMDS has ample headroom and ALLM is redundant with a fixed game mode. Calls `sceHdmiSetHdrConfig()`, which needs #49. |
| `22c80a1` | 2026-10-30 · #40 | Enable MT7921 PCIe Wi-Fi | Already covered: local commit `0649cda23bc6` sets `CONFIG_MT7921E=m`, identical change. |
| `53809a3` | 2026-11-01 · #66 | PS5 Pro fixes, GPU governor | Pro parts do not apply. Base-P5 `ps5-gpufreq` + MP1 mailbox locking offer little benefit for 1080p60 desktop/video and need adaptation to our older 490-line `mp1.c`; firmware fan/servo already handles cooling. |
| `fdc3eeb` | 2026-11-02 · #67 | FRL/4:2:0/ALLM on by default, bounded recovery | Builds on #63 and shares the #49 dependency; FRL recovery and 4:2:0 fallback never activate on a 1080p60 4:4:4 TMDS link. |
| `495ea3d` | 2026-11-03 · #68 | Name PS5 Pro (Viola) CPU | Pro-only cosmetic (model `0x4d`); adds nothing without #58/#66. Oberon half already in the tree from `7f695e4`. |

## 7.2.3 regression research

Why 7.1.7 is kept; none of this is caused by the VCN/fan backports.

- **amdgpu / AMD iGPU**: ~15 s freezes and FPS drops on 7.2.3; 7.2.2/LTS fine.
  <https://discuss.cachyos.org/t/kernel-regression-with-amd-igpu-in-7-2-3/35321>
  <https://github.com/CachyOS/linux-cachyos/issues/1041>
- **`r8169` / iSCSI / VLAN**: 180 s timeouts, ext4 journal aborts, read-only
  remounts. Not applicable (PS5 uses MTS).
  <https://discuss.cachyos.org/t/kernel-7-2-3-breaks-iscsi-on-my-system/35437>
  <https://discuss.cachyos.org/t/trouble-with-iscsi-since-kernel-7-2-3/35432>
  <https://discuss.cachyos.org/t/kernel-update-issue-to-7-2-3/35433>
- **btrfs large-folio lock leak** (7.2-rc): writeback/syncfs stalls, possible
  disk hang. <https://freenode.net/article/btrfs-folio-lock-leak-hangs-kvm-guests-on-file-backed-ram>
- Other: btusb HFP/SCO freezes; i915 s2idle black screen.
  Scheduler: <https://lkml.rescloud.iu.edu/2609.1/10670.html>
  Changelog: <https://www.kernel.org/pub/linux/kernel/v7.x/ChangeLog-7.2>

## Notes

- `tools/testing/selftests/net/ovpn/data64.key` and
  `tools/testing/selftests/sgx/sign_key.pem` show as modified but are sandbox
  artifacts (char devices) — do not commit.
- Patch repo HEAD is `495ea3d`; last applied commit is `1ec6a4a` (#54).
- Everything after #54 through HEAD is skipped: #58, #60, #62, #63, #40
  (already covered), #66, #67 and #68. No newer commits remain as candidates.

## Pending

1. Build: `make -j$(nproc)`.
2. Check firmware: `strings vmlinux | grep ps5_vcn`.
3. Boot checks:
   - `dmesg | grep -i vcn` — init, firmware load, no "did not become ready"
   - `dmesg | grep -i fanservo` — `D-Eiger/Denali Samsung D1z`
   - `vainfo` — H.264/HEVC/VP9 decode
   - `nvtop` during playback — DEC/ENC activity
4. VCN failure → revert the three amdgpu files, two config lines, firmware copy.
5. Test HDMI hotplug: replug during playback (video + audio within ~500 ms),
   cold boot with cable in, suspend/resume, multi-channel audio.
