# PS5 kernel backport status

## Base

- Kernel: **Linux v7.1.7** (`c7ba9d6de43e`, tag `v7.1.7`)
- PS5 patchset applied on top: `ps5-linux-patches` commit **`792ef83`**
- Custom base commits: `b1acbd5c0cea` (track `.config` / `.gitignore`),
  `0649cda23bc6` (enable `CONFIG_MT7921E=m`)

The plan is to stay on 7.1.7: kernels above 7.1.7 are unstable/unnecessary. Only
selected, self-contained fixes are backported.

## Done (newest first)

### 2026-10-20 00:07 — fan detection / Oberon name

Backport of `7f695e4` "CFI-1116A: fix fan detection, set Oberon name (#34)".

- `arch/x86/kernel/cpu/amd.c` — Oberon name for models `0x46`–`0x48` (cosmetic).
- `drivers/ps5/fan.c` — `gddr6_revision = (gddr6_id >> 4) & 0xf;`

Functional part: before the fix this console logged
`D-Eiger/Denali Unknown Vendor/Revision 0x1/0xb9` → `Not update`. Masking gives
revision 9 (Samsung D1z), so the correct fan/autoservo table is installed.

### 2026-10-19 22:28 — VCN 2.0 hardware video decode/encode

Backport of `8081cd8` (initial), final form from `753c793` (gated by
`IS_ENABLED(CONFIG_X86_PS5)` only; no `device == 0x13fb` checks).

Files changed: `amdgpu_discovery.c`, `amdgpu_vcn.c`, `vcn_v2_0.c` (IP block
registration, `ps5_vcn` firmware prefix, PS5 reset/start/stop sequences,
doorbells off), `.config` (`CONFIG_EXTRA_FIRMWARE`), `.gitignore` negation, and
the firmware blob + license copied into `firmware/` so the repo is
self-contained.

Result: H.264/HEVC decode+encode, VP9 decode. No AV1 (unsupported on VCN 2.0).

## Evaluated and skipped (chronological)

Commits between `792ef83` (7.1.7 baseline) and the VCN commits:

| Commit | Subject | Decision |
|---|---|---|
| `56922cf` | config: enable Android binder for waydroid | skipped — only needed for Waydroid |
| `f367cd5` | config: update to 7.2.3 | skipped — 7.2.3 version churn, not for 7.1.7 |
| `0d2ed27` | linux: fix some bugs and disable xhci | skipped — mixed; `rdmsrl→rdmsrq` is a 7.2 API rename (N/A on 7.1.7); xhci `#if 0` is a workaround; PCIe `PCIE_BUS_PERFORMANCE` not needed |
| `9322b9d` | linux: fixes for mts (Ethernet) | skipped — wired networking works fine (see summary below) |
| `8b7eaea` | btusb changes for 7.2.5 | skipped — 7.2.5 API compatibility, wrong base |

### `9322b9d` MTS details (skipped)

What it would have changed in `drivers/net/phy/mts/` (`mts.h`, `mts_main.c`;
`mts_phy.c`, `Kconfig`, `Makefile` unchanged):

- Fix IMR polarity: register `0x54` is an interrupt **mask**, not enable;
  writes proper `GBE_IMR_NORMAL` / `GBE_IMR_ALL` values and clears ISR at init.
- Add multicast hash filtering (previously asked-for MC addresses were only logged).
- Dynamic IRQ coalescing/moderating instead of a hardcoded `GBE_COAL_VAL`,
  including the timer interrupt bit.
- `napi_disable()` and interrupt/coalescing shutdown before stopping DMA.
- RX ring 256 → 1024 descriptors, RX restart state, clear MIB drop/CRC counters.

Reason to skip: wired Ethernet is fine on this machine; not needed for the current goal.
This is **wired only** — it does not touch wireless (`mt7921e`).

## 7.2.3 regression research (why 7.1.7 is kept)

Reports consistent with the severe disk-read/stability problems seen after a 7.2.3
upgrade. None are caused by the VCN or fan backports.

- **amdgpu / AMD iGPU**: random ~15s freezes, large FPS drops on 7.2.3; rollback
  to 7.2.2/LTS fixes it.
  <https://discuss.cachyos.org/t/kernel-regression-with-amd-igpu-in-7-2-3/35321>
  <https://github.com/CachyOS/linux-cachyos/issues/1041>
- **`r8169` / iSCSI / VLAN**: 7.2.3 broke iSCSI with 180s timeouts, ext4 journal
  aborts, read-only remounts. Not applicable to PS5 (uses MTS, not `r8169`).
  <https://discuss.cachyos.org/t/kernel-7-2-3-breaks-iscsi-on-my-system/35437>
  <https://discuss.cachyos.org/t/trouble-with-iscsi-since-kernel-7-2-3/35432>
  <https://discuss.cachyos.org/t/kernel-update-issue-to-7-2-3/35433>
- **btrfs large-folio lock leak** (7.2-rc line): writeback/syncfs stalls behind a
  permanently locked folio; possible local disk hang if using btrfs.
  <https://freenode.net/article/btrfs-folio-lock-leak-hangs-kvm-guests-on-file-backed-ram>
- **Other 7.2.3 reports**: btusb HFP/SCO hard freezes; i915 s2idle black screen.
- **Scheduler-related 7.2 regression**: <https://lkml.rescloud.iu.edu/2609.1/10670.html>
- **Linux 7.2 changelog**: <https://www.kernel.org/pub/linux/kernel/v7.x/ChangeLog-7.2>

## Notes for next session

- VCN and fan changes were committed by the user; `git log` has the hashes.
  This status file itself may still be untracked and needs committing.
- `git status` may show `tools/testing/selftests/net/ovpn/data64.key` and
  `tools/testing/selftests/sgx/sign_key.pem` as modified; these are sandbox
  artifacts (char devices), **not** real changes. Do not commit them.
- No kernel build has been run. The user asked not to build until they say so.
- MTS (`9322b9d`) was deliberately skipped; no MTS files were modified.

## Pending

1. Build the kernel: `make -j$(nproc)` (not yet run).
2. Verify the firmware is embedded: `strings vmlinux | grep ps5_vcn`.
3. Boot checks:
   - `sudo dmesg | grep -i vcn` — VCN init, firmware load, no "did not become ready"
   - `sudo dmesg | grep -i fanservo` — should now show `D-Eiger/Denali Samsung D1z`
   - `vainfo` — H.264/HEVC/VP9 decode entrypoints
   - Firefox `about:support` — `Hardware Video Decoding: Enabled`
   - `nvtop` while playing video — DEC/ENC activity
4. If VCN fails at runtime, revert the three amdgpu files plus the two config lines
   and the firmware copy; 3D/display behavior is unaffected either way.
