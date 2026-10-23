# PS5 kernel backport status

## Base

- Kernel: **Linux v7.1.7** (`c7ba9d6de43e`)
- PS5 patchset on top: `792ef83`
- Custom base commits: `b1acbd5c0cea` (`.config` / `.gitignore`),
  `0649cda23bc6` (`CONFIG_MT7921E=m`)

Stay on 7.1.7 — newer kernels are unstable here. Only self-contained fixes are
backported.

## Done (newest first)

### 2026-10-22 21:40 — HDMI display hotplug

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

### 2026-10-22 21:23 — FLAVA3 HDMI audio stability

Backport of `035d237`.

- `drivers/ps5/hdmi.c` — `getHdmiDpAudioN()`: FLAVA3 returns `0x80000`
  (default Naud is choppy, Maud too coarse), others keep the caller's value.
- `include/linux/ps5.h` — prototype.
- `dcn10_stream_encoder.c` — `enc1_se_setup_dp_audio()` writes the helper's
  value into `DP_SEC_AUD_N`.

Post-patch blobs match upstream exactly (`2ee166b`, `322b2a6`, `501f97a`).

### 2026-10-20 00:07 — fan detection / Oberon name

Backport of `7f695e4`.

- `arch/x86/kernel/cpu/amd.c` — Oberon name for models `0x46`–`0x48` (cosmetic).
- `drivers/ps5/fan.c` — `gddr6_revision = (gddr6_id >> 4) & 0xf;`

Before: `D-Eiger/Denali Unknown Vendor/Revision 0x1/0xb9` → `Not update`.
Masking yields revision 9 (Samsung D1z), installing the correct fan table.

### 2026-10-19 22:28 — VCN 2.0 hardware video decode/encode

From `8081cd8`, final form `753c793`; gated only by `CONFIG_X86_PS5`.

- `amdgpu_discovery.c`, `amdgpu_vcn.c`, `vcn_v2_0.c`: IP block registration,
  `ps5_vcn` firmware prefix, PS5 reset/start/stop, doorbells off.
- `.config` `CONFIG_EXTRA_FIRMWARE`, `.gitignore` negation, firmware blob +
  license in `firmware/` (self-contained repo).

Result: H.264/HEVC decode+encode, VP9 decode. No AV1 (not in VCN 2.0).

## Evaluated and skipped

| Commit | Subject | Why skipped |
|---|---|---|
| `56922cf` | config: Android binder | Waydroid only |
| `f367cd5` | config: update to 7.2.3 | 7.2.3 churn |
| `0d2ed27` | fix bugs, disable xhci | mixed; `rdmsrl→rdmsrq` is a 7.2 rename; xhci `#if 0` workaround |
| `9322b9d` | fixes for MTS Ethernet | wired net works; wired only |
| `8b7eaea` | btusb changes for 7.2.5 | wrong base (7.2.5 API) |

`9322b9d` would have fixed IMR polarity, added multicast hash filtering,
switched to dynamic IRQ coalescing, added `napi_disable()` before DMA stop, and
grown the RX ring 256 → 1024.

## 7.2.3 regression research

Why 7.1.7 is kept; none of this is caused by the VCN/fan backports.

- **amdgpu / AMD iGPU**: ~15 s freezes and FPS drops on 7.2.3; 7.2.2/LTS fine.
  <https://discuss.cachyos.org/t/kernel-regression-with-amd-igpu-in-7-2-3/35321>
  <https://github.com/CachyOS/linux-cachyos/issues/1041>
- **`r8169` / iSCSI / VLAN**: 180 s timeouts, ext4 journal aborts, read-only
  remounts. Not applicable (PS5 uses MTS). <https://discuss.cachyos.org/t/kernel-7-2-3-breaks-iscsi-on-my-system/35437>
  <https://discuss.cachyos.org/t/trouble-with-iscsi-since-kernel-7-2-3/35432>
  <https://discuss.cachyos.org/t/kernel-update-issue-to-7-2-3/35433>
- **btrfs large-folio lock leak** (7.2-rc): writeback/syncfs stalls, possible
  disk hang. <https://freenode.net/article/btrfs-folio-lock-leak-hangs-kvm-guests-on-file-backed-ram>
- Other: btusb HFP/SCO freezes; i915 s2idle black screen.
  Scheduler: <https://lkml.rescloud.iu.edu/2609.1/10670.html>
  Changelog: <https://www.kernel.org/pub/linux/kernel/v7.x/ChangeLog-7.2>

## Notes

- VCN and fan are committed; this file may still be untracked.
- `tools/testing/selftests/net/ovpn/data64.key` and
  `tools/testing/selftests/sgx/sign_key.pem` show as modified but are sandbox
  artifacts (char devices) — do not commit.
- No kernel build has been run yet.
- MTS (`9322b9d`) deliberately skipped.
- Patch repo HEAD is `c091037`; all commits through it are now applied.

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
