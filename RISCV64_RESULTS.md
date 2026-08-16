# riscv64 Guest Support for Proxmox VE 9 — Implementation Results

Companion to `RISCV64_GUEST_SUPPORT_PLAN.md` (the plan) and `ISO_BUILD_SCOPE.md` (ISO scoping).
This document records what was actually built, verified, and found — as distinct from what was
planned.

**Scope as agreed:** x86_64 host running riscv64 guests under TCG software emulation (no KVM),
full-parity effort.

**Status: COMPLETE.** Feature implemented, built, and verified end-to-end on real hardware — a
riscv64 guest boots OpenSBI and our cross-compiled EDK2 UEFI firmware to the Boot Manager, under
TCG, on a Proxmox VE host installed from our ISO.

---

## 1. Source changes (branch `feature/riscv64-guest-support`, one commit per repo)

### qemu-server
| File | Change |
|---|---|
| `QemuServer/Helpers.pm` | Added `riscv64 => '/usr/bin/qemu-system-riscv64'`; new `uses_virt_machine($arch)` helper (returns true for non-x86_64) |
| `QemuServer/CPUConfig.pm` | `riscv64` in arch enum; riscv64 CPU-model table + hardcoded fallback; `get_default_cpu_type()` returns `max` under TCG; `get_cpu_bitness()` returns 64 |
| `QemuServer/Machine.pm` | `riscv64 => 'virt'` default machine |
| `QemuServer/OVMF.pm` | `RISCV_VIRT_CODE.fd` / `RISCV_VIRT_VARS.fd` firmware paths |
| `QemuServer/PCI.pm`, `USB.pm`, `QemuServer.pm` | Replaced ~10 `$arch eq 'aarch64'` checks with `uses_virt_machine($arch)` |
| `QemuServer.pm` | **TPM fix**: `tpm-tis-device` (sysbus) instead of `tpm-tis` (ISA) on virt machines |
| `QemuServer/CPUFlags.pm`, `API2/Qemu.pm`, `API2/Qemu/CPUFlags.pm`, `Drive.pm` | riscv64 branches / description updates |
| `test/Makefile`, `test/cfg2cmd/riscv64/` | 3 new test fixtures + Makefile target |

### pve-qemu
| File | Change |
|---|---|
| `debian/rules` | `riscv64-softmmu` added to `--target-list`; riscv64 machine/CPU-model generation + diff guard |
| `debian/parse-cpu-models.pl` | riscv64 vendor-detection branch (incl. `mips-p8700`); 32-bit cores skip-listed |
| `debian/parse-machines.pl` | `die` → `warn` on empty result (riscv64 `virt` has no versioned variants) |
| `debian/cpu-models-riscv64.json` | **New** golden file, generated from the real built binary |
| `Makefile` | **OpenSBI fix**: purge only `opensbi-riscv32-…`, not `opensbi-riscv.*-…` |

### pve-manager
`Architecture.js`, `OSDefaults.js`, `CPUModelSelector.js`, `Utils.js` — riscv64 in arch selector,
defaults, CPU-model validation, BIOS rendering.

### pve-edk2-firmware
`debian/rules` — `TPM2_ENABLE`/`TPM2_CONFIG_ENABLE` for the RISCV64 target.

---

## 2. Bugs found — all by empirical testing, none hypothetical

| # | Bug | How found | Impact if shipped |
|---|---|---|---|
| 1 | `tpm-tis` doesn't exist on virt machines | Real QEMU device query | TPM VMs fail to start |
| 2 | `parse-cpu-models.pl` dies on `mips-p8700` | Real 11.0.3 `-cpu help` | **pve-qemu build fails** |
| 3 | `parse-machines.pl` dies on riscv64 (no versioned `virt`) | Real build `install` step | **pve-qemu build fails** |
| 4 | `CPUConfig.pm` unguarded JSON load | Self-review | **All VM startups break** on older pve-qemu |
| 5 | 4 uninitialized EDK2 submodules | Real firmware build | Firmware build fails |
| 6 | **OpenSBI blob purged by `riscv.*` glob** | `qm start` on real install | **No riscv64 guest can ever boot** |
| 7 | Identical version strings to stock | Upstream ISO comparison | `apt upgrade` silently strips riscv64 support |
| 8 | `ide-cd` emitted with `bus=ide.N` on virt machines | Booting a real guest ISO | **CD-ROM unusable on riscv64 *and* aarch64** |
| 9 | EDK2 publishes ACPI but not FDT | Booting FreeBSD riscv64 | FDT-based guest OSes cannot boot |

Bugs 2, 3, 5, 6, 8, 9 were only discoverable by actually building and running — not by code review.

**Bugs 8 and 9 are open** (found late; see §6). Both are worth noting for their breadth:

- **#8 is pre-existing and not riscv64-specific.** `DriveDevice.pm` emits `bus=ide.$controller`
  unconditionally, but QEMU's virt machine has no IDE bus, so `-device ide-cd,bus=ide.1` fails with
  `Bus 'ide.1' not found`. **aarch64 is affected identically.** It escaped the test suite because
  the cfg2cmd fixtures compare generated command *strings* and never launch QEMU — a real coverage
  limitation worth knowing about. The web UI already hides IDE for virt-machine architectures, so
  this is reachable via the API/CLI but not the GUI. Workaround: use SATA (PVE correctly emits
  `-device ahci,id=ahci0,multifunction=on,bus=pcie.0,addr=0x7`).
- **#9:** QEMU offers ACPI by default, so `PlatformHasAcpiDtDxe` installs the ACPI protocol and
  `FdtClientDxe` never calls `InstallConfigurationTable(&gFdtTableGuid, …)`. FreeBSD riscv64
  requires FDT and dies with `No valid device tree blob found!`. Workaround without rebuilding
  firmware: `args: -machine acpi=off`. Proper fix: build the firmware with `PcdForceNoAcpi=TRUE`.

---

## 3. Build artifacts

All packages carry a local version suffix so they sort above stock and cannot be silently
downgraded by `apt upgrade` (bug 7):

| Package | Version |
|---|---|
| `pve-qemu-kvm` | `11.0.3-2+riscv2` |
| `qemu-server` | `9.2.5+riscv1` |
| `pve-manager` | `9.2.10+riscv1` |
| `pve-edk2-firmware{,-aarch64,-legacy,-ovmf,-riscv}` | `4.2025.05-3+riscv1` |

**ISO:** `pve-riscv64-riscv2.iso`, md5 `b1d57da8a149bb36d55cd890209779bd`, 1,711,276,032 bytes.

---

## 4. ISO build approach — and why it was rewritten

The first pipeline (debootstrap → own squashfs → live-boot → isolinux/GRUB) produced a *booting*
ISO but a broken *installer*, because `live-boot` owns the medium and root union differently than
upstream's custom `init` expects. That surfaced as six separate-looking bugs (empty `/cdrom`,
empty package pool, no bootloader, no X input driver, missing filesystem tooling, an unresolvable
`umount: target is busy` loop) which were all one architectural mismatch.

**Rewritten to derive from upstream's real `proxmox-ve_9.2-1.iso`:** upstream's boot chain, initrd,
custom `init`, GRUB config, and **both squashfs images bit-for-bit unmodified**. Only the package
pool is modified — our 8 packages swapped in, plus 5 dependency backfills, index regenerated.
All six symptoms disappeared at once.

Feasibility rested on two verified facts: `.pve-cd-id.txt` is an opaque UUID (not a content
checksum, so substitution doesn't invalidate it), and our packages live **only** in
`/proxmox/packages/`, never in the squashfs images.

**Dependency skew:** our git-HEAD packages need newer deps than upstream's 9.2-1 pool. A cascade
analyzer converged in 2 rounds (5 packages, then 0 new): `libpve-common-perl`, `libpve-storage-perl`,
`proxmox-mini-journalreader`, `proxmox-widget-toolkit`, `pve-ha-manager`.

### Build assertions (`preflight4.sh`), each regression-tested against the real bug it catches
- Embedded-EFI search target exists in the staged tree *(catches the stale `efi.img` regression)*
- Shipped `pve-qemu-kvm` contains `qemu-system-riscv64` *(catches a stock package leaking in)*
- Shipped `pve-qemu-kvm` contains `opensbi-riscv64-generic-fw_dynamic.bin` *(catches bug 6)*
- riscv32 blob still purged *(catches over-correction)*
- Per-package expected version suffix; no stock or stale variants
- `Packages` index count matches, and post-dates every `.deb`

---

## 5. Verification performed

**Unit / command-generation:** 3 riscv64 fixtures (default host, native-riscv64 host, reverse
cross-arch) — parity with aarch64's 3, plus riscv64 uniquely covers TPM and UART. Full suite of
102 default-arch + 3 aarch64 + 3 riscv64 fixtures passes against a real installed Perl stack.

**Package-level:** `dpkg-deb -c` diffs vs upstream stock show *exactly* the intended additions —
`pve-qemu-kvm` 577 vs 574 files (adds `qemu-system-riscv64` + 2 JSON files), `qemu-server` and
`pve-manager` file lists **identical** to stock. Nothing upstream ships went missing.

**Real-hardware install** (a private Proxmox VE 9 host, nested VMs 400/401):
- **BIOS/SeaBIOS**: full install → reboot → working PVE ✅
- **UEFI/OVMF**: full install → reboot → working PVE ✅
- Installed systems confirm: our `pve-manager 9.2.10`, our `qemu-system-riscv64` (QEMU 11.0.3),
  both 32 MiB RISC-V firmware volumes, all custom packages `ii`.

**Feature validation** — `qm create --arch riscv64` accepted and persisted; `qm showcmd` on a real
installed system emits every changed code path correctly:

```
/usr/bin/qemu-system-riscv64 ... -cpu max ... -machine 'accel=tcg,type=virt+pve0'
-device 'tpm-tis-device,tpmdev=tpmdev'      # sysbus TPM fix
-device 'virtio-gpu,id=vga,bus=pcie.0,...'  # no legacy VGA
-device 'usb-ehci,...' 'usb-kbd,...'        # uses_virt_machine()
```

vTPM manufactured successfully (TPM 2.0, RSA-2048 + ECC EK, PCR banks) before QEMU reached
firmware load — proving the whole swtpm → chardev → `tpm-tis-device` path.

---

## 5a. Final proof — riscv64 guest boots real firmware

On VM 401's installed PVE, after upgrading to `pve-qemu-kvm 11.0.3-2+riscv2`, `qm start` on an
`arch: riscv64` guest produced, verbatim:

```
OpenSBI v1.7
Platform Name             : riscv-virtio,qemu
Platform HART Count       : 1
Platform Timer Device     : aclint-mtimer @ 10000000Hz
Platform Console Device   : uart8250
Firmware Base             : 0x80000000
Runtime SBI Version       : 3
```

Switching to OVMF (`--bios ovmf --efidisk0 …,efitype=4m`) seeded a 32.0 MiB efidisk — exactly
`RISCV_VIRT_VARS.fd`'s size — and `showcmd` confirmed `OVMF.pm` emitting
`/usr/share/pve-edk2-firmware//RISCV_VIRT_CODE.fd`. That boots too:

```
RISC-V EDK2 firmware version 4.2025.05-3
BdsDxe: failed to load Boot0001 "UEFI QEMU QEMU HARDDISK "
        from PciRoot(0x0)/Pci(0x5,0x0)/Scsi(0x0,0x0): Not Found
BdsDxe: No bootable option or device was found.
```

Version `4.2025.05-3` is precisely the package cross-compiled this session with
`gcc-riscv64-linux-gnu`. EDK2 initialized, ran BdsDxe, and enumerated the virtio-scsi controller at
`Pci(0x5,0x0)` — matching `addr=0x5` in `showcmd`, independently confirming the `PCI.pm` bus
assignment. The only failure is an empty disk with no OS, which is expected.

**Full verified chain: OpenSBI → EDK2 UEFI → BdsDxe → Boot Manager, riscv64, TCG, our packages.**

Independently reproduced on the **BIOS/SeaBIOS** path (VM 400, clean full install from the final
ISO rather than an in-place upgrade). `qm start` produced zero error output and the guest reported
the same OpenSBI v1.7 banner, plus its full SBI extension set
(`time,rfnc,ipi,base,hsm,srst,pmu,dbcn,fwft,legacy,dbtr,sse`). Its EDK2 self-identified as
**`4.2025.05-3+riscv1`** — carrying our local version suffix, confirming the ISO-installed
firmware package rather than any stock artifact. (VM 401 reported plain `4.2025.05-3` because it
upgraded only `pve-qemu-kvm` in place, leaving the pre-suffix firmware installed — the difference
is expected and consistent.)

So both firmware paths are confirmed by two independent routes: full-ISO install (BIOS) and
in-place package upgrade (UEFI).

---

## 5b. A real riscv64 OS boots — FreeBSD 15.1

FreeBSD 15.1-RELEASE riscv64 boots to a root shell on a riscv64 guest, via the full chain
OpenSBI v1.7 → our EDK2 → FreeBSD EFI loader → kernel:

```
FreeBSD 15.1-RELEASE releng/15.1-n283562-96841ea08dcf GENERIC riscv
EFI Firmware: Proxmox distribution of EDK II
real memory  = 2147090432 (2047 MB)
FreeBSD/SMP: Multiprocessor System Detected: 2 CPUs
ahci0: <Intel ICH9 AHCI SATA controller> ... AHCI v1.00 with 6 1.5Gbps ports
ada0: <QEMU HARDDISK 2.5+> ATA-7 SATA device ... 16384MB
vtnet0: ... status: active
```

`sysctl -n hw.machine_arch` → **`riscv64`**. Kernel, disk (AHCI), and network (virtio-net) all
functional. Requires `args: -machine acpi=off` pending the bug-9 firmware fix.

**The disk install was not completed.** Not a riscv64 defect: `bsdinstall`'s dialog TUI cannot
initialize over this serial link (`ncurses: cannot initialize terminal type`), and the fallback to
a scripted install is blocked by lossy serial input — at 0.3 s/character, `ls /usr/freebsd-dist`
still arrived as `ls /us/free\x00sd-di`. The guest console is "Video Primary, Serial Secondary" and
root login on `ttyu0` is refused (console marked insecure). The recommended path is a
`mini-memstick` with an `/etc/installerconfig` baked in, avoiding interactive input entirely.

Two media findings from the attempt: FreeBSD 15.1 riscv64 GENERIC attaches no `cd0` (only `pass0`),
so it cannot mount its own `disc1.iso` install media; and while `virtio_pci` binds, there is **no
`vtscsi0` child** — the virtio-scsi driver is absent, though virtio-net works. Hence memstick-on-SATA.

---

## 6. Known gaps / follow-up

- **Bug 8 (open, affects aarch64 too):** `ide-cd` is emitted with `bus=ide.N` on virt machines,
  which have no IDE bus. Needs a decision on correct behaviour — reject `ide*` at the API layer for
  virt-machine architectures (matching what the web UI already does), or transparently map it to
  the AHCI/SATA controller. Fixing it also means the cfg2cmd suite should gain a case that would
  have caught it.
- **Bug 9 (open):** rebuild `pve-edk2-firmware` with `PcdForceNoAcpi=TRUE` so FDT-based guests
  (FreeBSD, some Linux configs) boot without needing `-machine acpi=off` per VM.
- **Not verified:** live migration, backup/restore, guest-agent against a *fully installed* riscv64
  guest OS; cluster operations. A booted FreeBSD riscv64 kernel is proven; completing an unattended
  install (mini-memstick + `installerconfig`) is the prerequisite for the rest.
- **Unexplained non-fatal EDK2 error.** The riscv64 EDK2 firmware emits
  `ERROR: C40000002:V03051002` before BdsDxe on both test systems. Firmware continues normally and
  reaches the Boot Manager, so it is not blocking — but the root cause was not chased and is worth
  investigating before this is considered production-ready.
- **Reinstall hygiene:** a partial disk wipe between installs is *not* sufficient — stale LVM
  metadata beyond the wiped region causes `unable to initialize physical volume /dev/sdaN`. Use
  `blkdiscard` on the whole volume. (Verified during testing that nested-guest LVM never leaked to
  the host VG.)
- **Installer hostname auto-fill** picks up the build/test network's reverse DNS — on the test LAN
  it defaulted to the *production* host's FQDN. Worth overriding in ISO defaults.
- **JS lint skipped** during `pve-manager` builds (`BIOME=true`); Proxmox's real biome config isn't
  published. A skip, not a pass.
- **`filesystem.module` manifest** not implemented (moot under the upstream-derived pipeline).
- **Upstream contribution:** Proxmox takes mailing-list patches (`git send-email`), not PRs. The
  changes are staged as uncommitted working-tree diffs across 4 repos, ready to be split into
  reviewable commits.

---

## 7. Reproducing

```bash
# Packages (each repo uses `make deb`, NOT bare dpkg-buildpackage)
cd pve-qemu && make deb            # ~20 min, 3 QEMU targets
cd qemu-server && make deb
cd pve-manager && BIOME=true make deb
cd pve-edk2-firmware && make deb   # ~30 min, multi-target EDK2 cross-build

# Unit tests
cd qemu-server/src/test && perl -I../ ./run_config2command_tests.pl cfg2cmd/riscv64
```

ISO assembly scripts live under the session scratch dir (`build_from_upstream.sh`,
`preflight4.sh`); they derive from an extracted upstream ISO and are not part of any repo.
