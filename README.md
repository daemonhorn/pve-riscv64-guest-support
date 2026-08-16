# riscv64 VM Guest Support for Proxmox VE 9

Adds **riscv64 as a supported VM guest architecture** to Proxmox VE 9, running under
QEMU/TCG software emulation on x86_64 hosts (no KVM required).

Verified end to end: a riscv64 guest boots **OpenSBI v1.7** and cross-compiled
**EDK2 UEFI firmware** to the boot manager, on a Proxmox VE host installed from a
custom ISO built with these changes.

## Component repositories

| Repo | Change |
|---|---|
| [qemu-server](https://github.com/daemonhorn/qemu-server) | Core backend: arch support, `uses_virt_machine()`, sysbus TPM fix, CPU models, firmware paths, 3 test fixtures |
| [pve-qemu](https://github.com/daemonhorn/pve-qemu) | Builds `riscv64-softmmu`; CPU/machine metadata parsers; ships the riscv64 OpenSBI blob |
| [pve-manager](https://github.com/daemonhorn/pve-manager) | Web UI: architecture selector and riscv64 defaults |
| [pve-edk2-firmware](https://github.com/daemonhorn/pve-edk2-firmware) | TPM2 enabled for the RiscVVirt platform |

All four carry a single commit on branch `feature/riscv64-guest-support`.

## Documents

- **[RISCV64_RESULTS.md](RISCV64_RESULTS.md)** — what was built and verified, the seven bugs
  found, and remaining gaps. Start here.
- **[RISCV64_GUEST_SUPPORT_PLAN.md](RISCV64_GUEST_SUPPORT_PLAN.md)** — the original
  implementation plan.
- **[ISO_BUILD_SCOPE.md](ISO_BUILD_SCOPE.md)** — scoping analysis for building a release ISO.

## Notable findings

Four of the seven bugs were only discoverable by actually building and running, not by
code review:

- `parse-cpu-models.pl` died on `mips-p8700`, breaking the `pve-qemu` build.
- `parse-machines.pl` died on riscv64, whose `virt` machine has no versioned variants.
- Four EDK2 submodules were uninitialized, breaking the firmware build.
- The `opensbi-riscv.*` purge glob stripped the riscv64 OpenSBI blob, so **no riscv64
  guest could boot at all** — found only after a full ISO install onto real hardware.

A fifth is worth noting for existing users: on QEMU's `virt` machines the TPM is the
sysbus `tpm-tis-device`, not the ISA `tpm-tis`. This affected **aarch64** as well, so the
fix is not riscv64-specific.

## Status

Feature-complete and verified (firmware boot, both BIOS and UEFI installs, full
command-generation test suite). Not yet verified: live migration, backup/restore, and
guest-agent against a *running* riscv64 guest OS — these need a guest OS installed, not
just firmware.

Upstream contribution note: Proxmox accepts patches via its mailing list
(`git send-email`), not pull requests. These commits would want splitting into smaller
reviewable pieces before submission.
