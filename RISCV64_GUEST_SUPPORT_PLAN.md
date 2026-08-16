# Adding riscv64 VM Guest Support to Proxmox VE 9

## Context

Proxmox VE 9 (Debian 13 "trixie") currently ships QEMU-based VM support for
`x86_64` and, as a tech-preview, `aarch64` guests. This plan adds a third
guest architecture, `riscv64`, running under QEMU's `virt` machine.

**Scope decisions (confirmed with user):**
- **Acceleration: TCG only.** The target scenario is an `x86_64` Proxmox
  host running `riscv64` guests. KVM requires matching host/guest
  architecture, so this is pure software emulation (QEMU TCG) — the same
  mechanism PVE already uses today for `aarch64`-on-`x86_64` cross-arch
  emulation (see `get_command_for_arch()` in `qemu-server`, which already
  picks a non-native QEMU binary and skips `/usr/bin/kvm` whenever host and
  guest arch differ). No KVM-on-riscv64-host work is in scope; a future
  follow-up could add that once real riscv64 hardware with KVM matters.
- **Depth: full-parity effort.** Boot, disk/net/serial, display, TPM,
  guest-agent, hotplug (where already supported for aarch64), snapshots,
  backup, and migration are all in scope, phased so each layer is usable
  before the next builds on it.

**Why this is more tractable than it looks:** PVE 9 already has an
"unofficial"/tech-preview `aarch64` guest architecture implemented across
every layer (build, backend, API, UI, tests). Research below shows the
`riscv64` guest architecture is *not* a green-field addition — significant
groundwork already exists upstream:

- `pve-edk2-firmware` (master/trixie) **already builds** a
  `pve-edk2-firmware-riscv` package from `OvmfPkg/RiscVVirt/RiscVVirtQemu.dsc`,
  producing `RISCV_VIRT_CODE.fd` / `RISCV_VIRT_VARS.fd` (32MB each), using
  `gcc-riscv64-linux-gnu` as the cross toolchain. Firmware is done, modulo
  enabling TPM2/secure-boot flags (see TPM section).
- `qemu-server`'s test suite already has a `RISCV_VIRT_VARS` size mock in
  `src/test/run_config2command_tests.pl` (~line 298) — a placeholder someone
  left as a breadcrumb for this exact feature.
- `pve-container` (LXC) already fully supports `riscv64` as a container
  architecture (`PVE::LXC::Config` arch enum, ELF-header arch detection in
  `PVE::LXC::Tools`). Confirms `riscv64` is a first-class arch elsewhere in
  the stack — out of scope here (VM guests only, per user's request) but
  useful precedent.
- `pve-manager`'s `FileSelector.js` already includes `riscv64` in its known
  container-template architecture list.

What's missing is the **middle layer**: QEMU isn't built with the
`riscv64-softmmu` target, and `qemu-server`/`pve-manager` don't wire up
`riscv64` as a guest architecture the way they wire up `aarch64`. This plan
closes that gap by mirroring the existing `aarch64` support pattern in each
repo, file by file.

## Repos in scope (already cloned, each on branch `feature/riscv64-guest-support` off `master`/trixie)

| Repo | Path | Role |
|---|---|---|
| `pve-qemu` | `./pve-qemu` | Debian packaging + build flags for the QEMU binaries (`pve-qemu-kvm`) |
| `pve-edk2-firmware` | `./pve-edk2-firmware` | UEFI firmware images (OVMF/AAVMF/RISCV_VIRT) |
| `qemu-server` | `./qemu-server` | Core VM backend: config schema, QEMU command-line/QMP generation, `qm` CLI, `PVE::API2::Qemu` |
| `pve-manager` | `./pve-manager` | WebUI (ExtJS, `www/manager6/**`) and management glue |
| `pve-common` | `./pve-common` | Shared Perl utilities (`PVE::Tools::get_host_arch`, etc.) |
| `pve-guest-common` | `./pve-guest-common` | Shared guest helpers |
| `pve-container` | `./pve-container` | LXC — already has riscv64 support; **no changes planned**, reference only |

Not cloned (no changes anticipated, verify-only): `swtpm` — TPM emulation is
host-side, chardev-socket based, and architecture-independent of the guest;
no swtpm changes are expected. `qemu-guest-agent` is not a separate PVE
repo — it's built from within QEMU's own `qga/` tree but explicitly
*disabled* in `pve-qemu`'s build (`--disable-guest-agent`); the in-guest
agent is a **guest-OS package dependency** (Debian's riscv64 port ships
`qemu-guest-agent`), not something this plan builds. `qemu.git`
(`mirror_qemu`/PVE's QEMU source submodule) needs no patching — QEMU 11's
upstream `virt` riscv64 machine is used as-is.

Proxmox does **not** use GitHub PRs — contributions go to the
`pve-devel`/`pve-user` mailing lists via `git send-email`, reviewed there.
"Feature branch suitable for eventually creating a PR" in each repo gets you
a clean, rebasable branch to `git format-patch`/`git send-email` from later;
treat that as the deliverable, not a GitHub pull request.

## 1. `pve-qemu` — build the riscv64 QEMU target

File: `debian/rules`.

- Add `riscv64-softmmu` to `--target-list=x86_64-softmmu,aarch64-softmmu`
  (line ~49). This is a single build-flag change — QEMU softmmu targets
  don't need a cross-compiler (they emulate the target CPU, but the emulator
  binary itself is host-native), so no new `Build-Depends` needed in
  `debian/control`.
- Mirror the per-arch post-install steps that already exist for aarch64:
  - `machine_file_riscv64 := $(destdir)/usr/share/kvm/machine-versions-riscv64.json`
  - `cpu_models_file_riscv64 := $(destdir)/usr/share/kvm/cpu-models-riscv64.json`
  - `qemu-system-riscv64 -machine help | ./debian/parse-machines.pl > $(machine_file_riscv64)`
  - `qemu-system-riscv64 -cpu help | ./debian/parse-cpu-models.pl > $(cpu_models_file_riscv64)`
  - `diff -u $(cpu_models_file_riscv64) ./debian/cpu-models-riscv64.json` — add the
    new golden file `debian/cpu-models-riscv64.json` (generate once from a
    local build, commit it, same as `cpu-models-aarch64.json`).
- No changes needed to `debian/control`'s `Architecture:`/binary package
  split — `pve-qemu-kvm` already ships every `qemu-system-*` binary in one
  package regardless of host arch (confirmed: aarch64's binary already ships
  this way today for cross-arch use on x86_64 hosts).

## 2. `pve-edk2-firmware` — enable TPM2 for RISC-V firmware

Firmware images already build. One gap: `RISCV64_FLAGS` in `debian/rules`
is just `$(COMMON_FLAGS)` — unlike `AAVMF_FLAGS`/`OVMF_COMMON_FLAGS`, it
doesn't set `-DTPM2_ENABLE=TRUE`. Add that (and `-DTPM2_CONFIG_ENABLE=TRUE`
if the `RiscVVirtQemu.dsc` platform supports it — verify against the EDK2
submodule; if the PCD isn't defined for this platform, this becomes a
"skip, file upstream EDK2 note" item rather than a blocker, since the TPM
device is still usable by the guest OS without firmware-level measured
boot). No secure-boot variant exists yet for RISC-V upstream in EDK2 as of
this QEMU/EDK2 vintage — don't build one; ship only the default
`RISCV_VIRT_CODE.fd`/`RISCV_VIRT_VARS.fd` pair, same shape as aarch64's
`default` (non-`ms`) entry.

Confirm the `pve-edk2-firmware-riscv` binary package (already defined in
`debian/control`) installs to `/usr/share/pve-edk2-firmware/` — same base
dir `qemu-server/src/PVE/QemuServer/OVMF.pm` already uses via `$EDK2_FW_BASE`.

## 3. `qemu-server` — the core wiring

This is where most of the work is. Every site below currently branches on
`$arch eq 'aarch64'`; each needs a `riscv64` case (mostly: extend a hash
table, occasionally: add a boolean condition alongside aarch64's).

**`src/PVE/QemuServer/Helpers.pm`**
- `$arch_to_qemu_binary`: add `riscv64 => '/usr/bin/qemu-system-riscv64'`.
- `get_command_for_arch()` / `get_vm_arch()` need no logic changes — they're
  already arch-generic; they just consume the map above.

**`src/PVE/QemuServer/CPUConfig.pm`**
- `pve-qm-cpu-arch` standard option `enum => [qw(x86_64 aarch64)]` (~line 32):
  add `riscv64`. This is the single schema choke-point that unlocks
  `arch: riscv64` in the VM config, API, and `qm` CLI everywhere else in the
  stack (`get_standard_option('pve-qm-cpu-arch', ...)` is reused across the
  API).
- `$builtin_models_by_arch`: add `riscv64 => {}` (empty, like aarch64 — rely
  on the dynamically-loaded `cpu-models-riscv64.json`).
- `initialize_cpu_models()`: add a third `$cpu_models_riscv64_file =
  '/usr/share/kvm/cpu-models-riscv64.json'` load, mirroring the aarch64
  block (~lines 118-131).
- `get_default_cpu_type($arch, $kvm)`: since KVM is out of scope for
  riscv64, this always takes the "no kvm" path. Recommend
  `$cputype = 'max'` for `$arch eq 'riscv64'` (QEMU riscv64's `max` CPU
  exposes the newest ratified extensions under TCG — the direct riscv64
  analogue of aarch64's non-KVM `'max'` fallback).
- `get_cpu_bitness()`: add `return 64 if $arch eq 'riscv64';`.
- `get_cpu_options()`: the existing `$arch eq 'x86_64'` guards (hyperv
  enlightenments, CPU vendor property, kvm-off flags) already fall through
  safely for any non-x86_64 arch — verify no change needed, just confirm by
  running the test suite.
- `print_cpu_device()` (CPU hotplug): currently hard-`die`s for anything but
  x86_64 — leave as-is; riscv64 CPU hotplug is out of scope (matches
  aarch64's current limitation, not a regression).

**`src/PVE/QemuServer/Machine.pm`**
- `%machine_for_arch` (or equivalent, ~line 125): add `riscv64 => 'virt'`.
  Everything else in this file (`machine_base_type`, the `virt(?:-...)`
  regex, version pinning) is already arch-generic string matching — no
  further changes expected. Update the stale comment at ~line 483
  ("virt machines are normally ARM64 ones") since it'll no longer be true.

**`src/PVE/QemuServer/OVMF.pm`**
- Add a `riscv64` entry to the `$OVMF` table:
  ```perl
  riscv64 => {
      default => [
          "$EDK2_FW_BASE/RISCV_VIRT_CODE.fd", "$EDK2_FW_BASE/RISCV_VIRT_VARS.fd",
      ],
  },
  ```
- `get_ovmf_files()`'s arch-dispatch `if`/`elsif` chain: riscv64 needs no
  special case (no `pre-enrolled-keys`/secure-boot handling exists for it
  yet, same as aarch64 without secure boot firmware installed) — it'll
  correctly fall through to `type = 'default'`.
- Everything downstream (`print_ovmf_commandline`, `-blockdev` generation)
  is already arch-generic once `$OVMF->{riscv64}` exists.

**`src/PVE/QemuServer/PCI.pm`**
- Line ~298: `die "aarch64 cannot use IDE devices\n" if $arch eq 'aarch64' && ...`
  — riscv64's `virt` machine also has no IDE controller; extend the
  condition (or generalize to a shared `arch_has_no_ide($arch)`-style helper
  — see cross-cutting note below).
- Line ~304-306: PCI bus-slot special-casing for aarch64's `virt` PCIe root
  — riscv64's `virt` machine also uses a generic PCIe host bridge; extend
  the same way.

**`src/PVE/QemuServer/USB.pm`**
- Lines ~136, ~155: `$arch eq 'aarch64'` gates for using `usb-ehci`
  (input-device bus) and skipping the legacy `piix3-usb-uhci`/`pve-usb.cfg`
  path. riscv64 guests will always be modern Linux (`ostype: l26`), so in
  practice `$use_qemu_xhci` will usually be true and these legacy branches
  moot — but extend the conditions to `riscv64` for correctness/symmetry
  with aarch64 rather than relying on that being always true.

**`src/PVE/QemuServer.pm`**
- `print_tabletdevice_full()` (~line 1189): `if ($q35 || $arch eq 'aarch64')`
  → also true for riscv64 (uses `ehci` bus for the input tablet).
- `print_keyboarddevice_full()` (~line 1201): `return if $arch ne 'aarch64';`
  — riscv64's `virt` machine also has no PS/2 keyboard, needs the same
  `usb-kbd` injection; change to a helper that returns true for both.
- `$vga_map_aarch64` (~line 1487) "QEMU builds only the non-VGA variants of
  the virtio GPU for aarch64": **verify against the actual riscv64 QEMU
  build** whether `virtio-vga`/`virtio-vga-gl` are built for riscv64 or only
  the non-VGA `virtio-gpu`/`virtio-gpu-gl` variants (this is a
  compile-time QEMU config question, not a PVE one — check
  `qemu-system-riscv64 -device help` after step 1). If it matches aarch64
  (non-VGA only), add `riscv64` to reuse `$vga_map_aarch64` (rename to
  something arch-neutral like `$vga_map_no_legacy_vga`).
- Serial console (~line 3336-3339): "On aarch64, serial0 is the UART
  device" special-case — riscv64's `virt` machine also exposes a UART
  (16550-compatible) as the primary console the same way; extend the
  `$arch eq 'aarch64' && $i == 0` condition.
- `add_tpm_device()` / `get_tpm_paths()` (~lines 2847-2865): **see TPM
  section below — this is a pre-existing gap, not something to just copy.**
- Lines ~4744-4818 (keyboard/tablet hotunplug on machine-type transition):
  same `$arch eq 'aarch64'` pattern as above, extend identically.

**Cross-cutting recommendation:** by this point there will be 8-10 call
sites doing `$arch eq 'aarch64' || $arch eq 'riscv64'`. Introduce one helper
in `Helpers.pm`, e.g. `sub uses_virt_machine($arch) { return $arch ne
'x86_64'; }` (or an explicit allow-list if a 4th arch later needs different
treatment), and use it at each site instead of duplicating the OR chain.
This is a refactor *of the aarch64 code*, so do it as its own preparatory
commit before the riscv64-specific commits, to keep the diff reviewable.

**`src/PVE/API2/Qemu.pm`**
- Line ~1517: `vmgenid` auto-generation skipped for `aarch64` (ACPI-based
  mechanism, not meaningful without full ACPI). riscv64 firmware/ACPI
  support is newer/partial too — add `riscv64` to this exclusion.
- `efitype` (`src/PVE/QemuServer/Drive.pm` ~line 513): the 2m/4m distinction
  is "ignored for arch=aarch64" since aarch64 only ships one firmware
  variant — same true for riscv64 once `$OVMF->{riscv64}` only has a
  `default` key; update the description string to mention riscv64 too.

**`src/PVE/API2/Qemu/Machine.pm`** (~line 67): reads
`/usr/share/kvm/machine-versions-$arch.json` with `$arch` as a live
variable, not a hardcoded per-arch dispatch — **confirmed already
arch-generic**. No code change needed here; it starts working the moment
§1 ships `machine-versions-riscv64.json` and the CPUConfig.pm enum (above)
allows `riscv64`.

**`src/query-machine-capabilities/query-machine-capabilities.c`**:
checked — this is a host-side-only C helper (compiled once for the PVE
*host's* native arch via `#ifdef __x86_64__`/`__aarch64__`, run as a
systemd service to detect host CPU features like AMD SEV/Intel TDX/ARM
crypto extensions for confidential-VM and CPU-passthrough support). It has
nothing to do with guest architecture and needs **no changes** for
riscv64-guest-on-x86_64-host support — the host stays x86_64 throughout
this plan's scope. (The changelog line "improve cross-architecture support
for query-machine-capabilities" refers to commit `c3cf6795`, already
merged, which guards the SEV/TDX *host* checks to `__x86_64__` only — this
is prior art confirming the file's structure, not open work.)

**Cross-arch KVM-forcing already exists and is generic** — verified, not
new work: `Helpers.pm::get_command_for_arch()` already returns the plain
`qemu-system-$arch` binary (not `/usr/bin/kvm`) whenever `$arch` doesn't
match `get_host_arch()`, and `QemuServer.pm`'s `$kvm //= 1 if
is_native_arch($arch);` (~line 3159) only defaults KVM on for native-arch
VMs. `pve-manager`'s `CreateWizard.js` (~lines 313-314, 349) independently
forces `kv.kvm = 0` client-side whenever
`!PVE.qemu.Architecture.isHostArchitecture(kv.arch, nodename)`. This is the
exact mechanism the aarch64-on-x86_64 tech preview already relies on — it
requires zero new logic for riscv64, only the `Architecture.js` table
entries (§5) and CPUConfig enum (§3) to light it up. Note it's a
*default*, not a hard guard: a user can still explicitly pass `kvm=1` with
a foreign arch via the API and hit a QEMU startup failure — pre-existing
behavior for aarch64 today, not a riscv64-specific gap to fix.

**`src/PVE/API2/Qemu/CPUFlags.pm`**
- `query_available_cpu_flags()`: currently documented as returning an empty
  list for `aarch64`; confirm/extend the same for `riscv64` (no per-flag
  CPUID-style introspection for RISC-V the way x86 has) — or check if
  RISC-V TCG "max" CPU exposes queryable extension flags in QEMU 11 and
  return those instead of an empty list, if easy.

**Confirmed no changes needed** (verified arch-agnostic already):
`RNG.pm`, `Network.pm`, `Drive.pm` (besides the doc string above),
`Blockdev.pm`, `Cfg2Cmd.pm`, `QemuMigrate.pm`, `VZDump/QemuServer.pm`,
`pve-common`'s `get_host_arch`/`get_host_dpkg_arch`. Migration and backup
already work per-VM regardless of guest arch since they operate on disk
images and QMP, not CPU semantics.

## 4. TPM — cross-cutting device-model fix (affects aarch64 too)

`QemuServer.pm::add_tpm_device()` currently emits `-device tpm-tis,...`
unconditionally, with **no arch branch at all**. `tpm-tis` is the ISA/LPC
TPM device, which doesn't exist as a valid device on the `virt` machine
(no ISA bus) — reading the code, this looks like a **latent bug for
aarch64 today**, but it hasn't been reproduced yet. First step is to
confirm it, not assume it: once §1's `pve-qemu` build produces
`qemu-system-aarch64`/`qemu-system-riscv64`, run
`qemu-system-aarch64 -M virt -device tpm-tis,tpmdev=x -tpmdev
emulator,id=x,chardev=y -chardev socket,id=y,path=/tmp/x 2>&1` and confirm
it actually errors (wrong bus) before writing a fix — QEMU may alias or
warn rather than hard-fail, which would change the urgency/framing of this
as a "fix aarch64" commit. Once reproduced, fix by branching on
machine/arch and using the sysbus variant `tpm-tis-device` for any
non-x86_64 (`virt`-based) machine:
```perl
my $tpm_device = $arch eq 'x86_64' ? 'tpm-tis' : 'tpm-tis-device';
push @$devices, "-device", "$tpm_device,tpmdev=tpmdev";
```
Verify `tpm-tis-device` is actually registered for riscv64's `virt` machine
in QEMU 11 (`qemu-system-riscv64 -device help | grep tpm`) — if the RISC-V
`virt` board doesn't wire up the sysbus TPM MMIO region the way `virt`-arm
does, this may need a QEMU-side board patch or fall back to `tpm-crb`
depending on what the `virt-riscv` board exposes. Treat this as a
**prerequisite spike**: confirm TPM device availability on riscv64 QEMU
before committing to full TPM support in the phasing below; if unsupported
upstream, TPM becomes a "not yet" item for riscv64 specifically (still fix
the aarch64 ISA/sysbus bug regardless, since it's broken independent of
this project).

No changes needed in `swtpm` — it's a host-side socket/CUSE TPM emulator
that only ever runs as an x86_64 host process; the chardev socket protocol
between QEMU and swtpm is architecture-independent.

**RISC-V boot-chain spike (no aarch64 analogue — don't assume "mirror
aarch64" holds here):** `virt`-aarch64 boots straight from pflash/EFI with
no earlier firmware stage. `virt`-riscv64 boots through OpenSBI first
(QEMU's built-in `-bios default` fw_dynamic payload runs in M-mode), which
then hands off to whatever is next — EDK2 UEFI (`RISCV_VIRT_CODE.fd` via
pflash) in our case. PVE's non-CVM `-blockdev`/`-drive if=pflash` path
(`OVMF.pm::print_ovmf_commandline`) never passes an explicit `-bios`, so it
implicitly relies on QEMU's per-machine default. Verify by hand, once §1
builds `qemu-system-riscv64`, that `-M virt -pflash RISCV_VIRT_CODE.fd
-pflash RISCV_VIRT_VARS.fd` (no `-bios` flag) actually boots through OpenSBI
into the EDK2 payload as expected, before trusting any `cfg2cmd/riscv64/`
golden-file command line as correct — the fixture only proves the *command
line* matches, not that QEMU actually boots with it.

## 5. `pve-manager` — WebUI

**`www/manager6/qemu/Architecture.js`** is the single source of truth for
UI arch behavior — its own comment says "To add a new architecture, add the
respective entry in the defaults/renderers and selection." Add a `riscv64`
entry to every table:
```js
selection: [..., ['riscv64', gettext('RISC-V (64-bit)')]],
kvmOSTypes: { riscv64: { bases: ['Linux', 'Other'], ostypes: ['l26', 'other'] } },
defaultProcessorModel: { riscv64: 'max' }, // NOT 'host' like aarch64: aarch64's
    // default assumes native+KVM; riscv64 in this plan is TCG-only (no
    // matching riscv64 host), so 'host' is invalid and 'max' (all
    // TCG-emulatable extensions) is the correct analogue — must match
    // get_default_cpu_type('riscv64', 0) in CPUConfig.pm (§3) exactly, or
    // the wizard creates VMs the backend then rejects.
defaultMachines: { riscv64: 'virt' },
defaultCDDrive: { riscv64: ['scsi', 2] },
allowedScsiHw: { riscv64: ['virtio-scsi-pci', 'virtio-scsi-single'] },
allowedMachines: { riscv64: ['__default__'] },
allowedBusses: { riscv64: ['sata', 'virtio', 'scsi', 'unused'] },
allowedFirmware: { riscv64: ['ovmf'] },
```
Add a `riscv64` case to `render_vcpu_architecture()`.

**`www/manager6/form/CPUModelSelector.js`** (~line 112-118): the validator
special-cases `aarch64` to only accept `host`/`max`/`cortex-a53`/`cortex-a57`
when KVM is on. Since riscv64 is TCG-only in this plan, add a `riscv64`
branch that accepts `max` (and any named riscv64 cores QEMU exposes, e.g.
`sifive-u54`, `thead-c906` — pull the actual list from
`cpu-models-riscv64.json` once built in step 1).

**`www/manager6/qemu/OSDefaults.js`** (~lines 57, 87): add a `riscv64` entry
to both the `busType`/`networkCard`/`busPriority` defaults block and the
`architectures` bus-priority override block, mirroring aarch64's ("no ide,
ovmf can't boot from sata" reasoning applies identically to riscv64's
`virt` machine).

**`www/manager6/Utils.js`** (~line 547-550): `render_qemu_bios()` picks
`'OVMF (UEFI)'` as the implied default BIOS for non-x86_64; already
arch-generic via the `arch === 'aarch64'` check — extend to riscv64 (or
generalize to `arch !== 'x86_64'`, consistent with the `Helpers.pm` helper
recommended above).

**`www/manager6/form/FileSelector.js`**: already includes `riscv64` in the
container-template `known` arch list — no change needed here (that's the
LXC path); double check there's no equivalent **QEMU disk-image/appliance**
arch filter elsewhere that needs the same addition (search for other
`templateArch`-like filters scoped to `content=='iso'`/`vztmpl'` handling).

No `PVE::API2::*` changes expected in `pve-manager` itself — VM API
validation lives in `qemu-server`'s `PVE::API2::Qemu`, which `pve-manager`
consumes as-is.

## 6. `pve-guest-common` / `pve-common`

No changes identified — `PVE::Tools::get_host_arch()` /
`get_host_dpkg_arch()` are already architecture-generic (return whatever
`uname -m`/`dpkg --print-architecture` report), and nothing in
`pve-guest-common`'s shared helpers branches on CPU architecture. Include
both repos in the review pass at the end in case something was missed, but
don't allocate implementation time up front.

## 7. Guest agent

No PVE code changes. `qemu-server`'s `Agent.pm` talks to the in-guest
`qemu-guest-agent` over the arch-independent virtio-serial/vsock channel —
nothing there branches on guest CPU arch today and nothing needs to.
Verification is a **guest-OS-side** concern: confirm Debian's riscv64 port
(or whichever guest distro is used for testing) has a `qemu-guest-agent`
package, install it in the test VM, and confirm `qm agent <vmid> ping`
works — this is a test-plan item, not an implementation item.

## 8. `qm` CLI

**Verified, not just expected:** `grep -n "arch" src/PVE/CLI/qm.pm` returns
zero hits — `src/PVE/CLI/qm.pm` has no arch-conditional code at all,
including `qm terminal` (operates purely on the configured `serialN:
socket` device, unrelated to the QEMU-internal UART-as-primary-console
wiring at `QemuServer.pm:3336-3339`), `qm monitor`, `qm showcmd`, `qm
importovf`, and `qm cloudinit`. `qm` is a thin `PVE::CLIHandler` wrapper
generated from `PVE::API2::Qemu`'s schema — once the `pve-qm-cpu-arch` enum
(step 3) includes `riscv64`, `qm create/set --arch riscv64` and `qm cpu ...`
validation, help text, and bash completion all pick it up automatically. No
code changes needed here. Still run `qm create 8006 --arch riscv64 ...`
against a build with the schema changes as an end-to-end sanity check
(step 4 of the verification plan) — confirm validation/error messages read
sensibly, since that exercises the schema wiring rather than `qm.pm` itself.

## 9. Unit tests

`qemu-server/src/test/cfg2cmd/aarch64/` is the exact template to mirror —
it already contains `simple-arm.conf` / `simple-arm.conf.cmd` (golden QEMU
command line), `simple-arm-host.conf` (native aarch64 host), and
`simple-x86-on-arm-host.conf` (cross-arch: aarch64 host, x86_64 guest — the
mirror image of our TCG scenario). Add `cfg2cmd/riscv64/`:
- `simple-riscv.conf` / `.cmd` — `arch: riscv64` guest on the default
  (x86_64) test host, analogous to `simple-arm.conf` but exercising the
  cross-arch/TCG path directly (no `HOST_ARCH:` override needed, since
  x86_64 is already the test-harness default).
- Optionally `simple-riscv-on-x86-host.conf` if the harness's naming
  convention wants the host explicit — check `run_config2command_tests.pl`'s
  `HOST_ARCH:` comment-directive handling (~line 298 area, where the
  existing `RISCV_VIRT_VARS` mock already lives) to match existing naming.
- Cover: default machine (`virt`), OVMF firmware pflash lines
  (`RISCV_VIRT_CODE.fd`/`RISCV_VIRT_VARS.fd`), default CPU (`max`), serial
  console UART special-case, scsi-only disk bus, and (once the TPM spike
  above resolves) a `tpmstate0` fixture using `tpm-tis-device`.
- Extend `run_pci_addr_checks.pl`/`run_pci_reservation_tests.pl` with a
  riscv64 case if they're parameterized by arch (check current aarch64
  coverage there first — they may already iterate all known archs
  generically, in which case adding `riscv64` to the arch enum in step 3 is
  sufficient and no test file edits are needed).
- No JS/webui test suite exists to extend in `pve-manager` beyond manual
  verification (matches current aarch64 UI testing practice — confirm no
  regression in this assumption before skipping it).

## Suggested commit/PR sequencing (per repo, roughly in this order)

Sequencing note: the TPM device-model fix (§4) and the OpenSBI→EDK2 boot
chain (§4's boot-chain spike) both depend on having a working
`qemu-system-riscv64` binary to test against. Building it (originally step
2) must happen *before* either spike is resolved and *before* the TPM fix
is committed as anything more than a hypothesis — otherwise the fix lands
ahead of the evidence that it's the right one. Order below reflects that;
only the arch-classification helper refactor is independent enough to go
first.

1. `qemu-server`: prep refactor only — introduce the arch-classification
   helper in `Helpers.pm` (no behavior change yet, purely reviewable
   independent of riscv64).
2. `pve-qemu`: add `riscv64-softmmu` target + golden files. This unblocks
   local testing of everything downstream, including both spikes below.
3. `pve-edk2-firmware`: enable TPM2 flags for the RISC-V build (can land in
   parallel with #2).
4. **Run both spikes** against the step-2 build: (a) confirm/characterize
   the `tpm-tis` ISA-device bug on `qemu-system-aarch64 -M virt` and check
   `tpm-tis-device` registration on `qemu-system-riscv64 -M virt` (§4); (b)
   confirm the OpenSBI→EDK2 boot handoff actually boots with no explicit
   `-bios` flag (§4). Only proceed to #5 once both have a concrete answer.
5. `qemu-server`: fix `add_tpm_device()` for aarch64 + riscv64 together,
   using the step-1 helper and the step-4 spike results.
6. `qemu-server`: the rest of the `riscv64` wiring (CPUConfig, Machine,
   OVMF, PCI, USB, QemuServer.pm device/console dispatch, API2 exclusions)
   — one commit per module, using the step-1 helper.
7. `qemu-server`: test fixtures (`cfg2cmd/riscv64/**`), including a TPM
   fixture now that #5 has landed.
8. `pve-manager`: `Architecture.js` + the four dependent UI files.
9. Final pass over `pve-common`/`pve-guest-common` to confirm no gaps.

## Verification plan

1. **Build**: `pve-qemu` — `dpkg-buildpackage` (or `make deb`) with the new
   target-list; confirm `qemu-system-riscv64` and
   `/usr/share/kvm/{machine-versions,cpu-models}-riscv64.json` are produced.
   `pve-edk2-firmware` — confirm `RISCV_VIRT_CODE.fd`/`RISCV_VIRT_VARS.fd`
   land in the `pve-edk2-firmware-riscv` package.
2. **Unit tests**: `qemu-server`: `prove src/test/run_config2command_tests.pl`
   (add the new fixtures first) plus the full existing suite, to catch
   regressions from the `Helpers.pm` refactor.
3. **Perl syntax/lint**: standard `perlcritic`/`perl -c` pass PVE already
   runs in CI for each touched `.pm`.
4. **End-to-end smoke test** (needs the built packages installed on a test
   x86_64 PVE 9 host): `qm create <id> --arch riscv64 --memory 1024 --net0
   virtio,bridge=vmbr0 --scsi0 <storage>:8,size=4G --ostype l26`, attach a
   riscv64 Debian/Fedora cloud image or netinst ISO, `qm start <id>`, confirm
   boot to a shell over the serial console (`qm terminal <id>`), confirm
   network via DHCP, confirm `qm agent <id> ping` once guest-agent is
   installed in-guest, test an EFI-disk-backed boot, and (pending the TPM
   spike) a `tpmstate0`-attached VM.
5. **WebUI smoke test**: create a VM via the wizard selecting Architecture
   = RISC-V (64-bit), confirm firmware/bus/CPU-model options correctly
   narrow to the riscv64-allowed set at every wizard step, confirm the
   Hardware panel renders correctly post-creation.
6. **Migration/backup smoke test**: snapshot and `vzdump` a running riscv64
   VM, restore it, confirm it boots.

## Known limitations / explicitly out of scope

- KVM acceleration for riscv64 (requires a riscv64 host with the H
  extension) — noted as a possible future follow-up, not built here.
- CPU hotplug for riscv64 (matches aarch64's existing limitation).
- UEFI Secure Boot for riscv64 (no upstream EDK2 secure-boot platform file
  for RISC-V yet, unlike aarch64/x86_64).
- SPICE/virtual-display beyond `virtio-gpu` framebuffer (no legacy VGA path
  exists on `virt`, same as aarch64).
- LXC container riscv64 support — already exists, untouched by this plan.
