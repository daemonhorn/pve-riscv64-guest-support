# Scope: Building a Real Release ISO with riscv64 Guest Support

Status: scoping only, nothing in this document has been executed. Researched against the
real `pve-installer` and `proxmox-ve` repos (cloned into this working directory at
`pve-installer/` and `proxmox-ve/`) plus the working open-source `pve-iso-builder`
reference implementation, since Proxmox's actual internal ISO-assembly pipeline is not
published in its public git repos.

## 1. Key clarification that shrinks the scope

The ISO installs a normal **x86_64 PVE host**. Our feature adds the ability for that
x86_64 host to run **riscv64 guests** via QEMU/TCG — it does not turn PVE into something
that runs *on* riscv64 hardware. So the ISO itself needs:

- no riscv64 kernel
- no riscv64 debootstrap target
- no riscv64 boot chain for the installer

It's a completely standard x86_64 PVE 9 ISO, just with four packages substituted for
ones built from our patched source. This is a much narrower job than "build a riscv64
Proxmox ISO."

## 2. What an ISO actually is (researched, not assumed)

`pve-installer` (cloned, inspected) builds the **installer wizard** — `proxinstall`,
`proxmox-tui-installer`, `proxmox-low-level-installer`, `proxmox-auto-installer`, etc.
This is a `.deb` that gets *installed into* the ISO's live environment; grepping its
Makefile and `debian/rules` for `squashfs`/`debootstrap`/`xorriso`/`isolinux` returns
nothing — it is not the tool that assembles the ISO.

`proxmox-ve` (cloned, inspected) is a tiny meta-package: its `Depends:` is just
`pve-manager`, `pve-qemu-kvm`, `qemu-server`, `proxmox-default-kernel`, and a handful of
base-system packages. The real transitive closure (dozens of `libpve-*-perl` packages,
`librados2-perl`, ceph tooling, etc., confirmed by reading `pve-manager`'s own
`Depends:`) comes in through `pve-manager`, not through anything we touched.

The actual ISO-assembly pipeline Proxmox runs internally isn't in the public repos I
could reach. The mechanics are well documented by the working open-source
`pve-iso-builder` project, which is a legitimate stand-in for producing a real bootable
ISO:

1. `debootstrap --arch=amd64 <codename>` a minimal Debian rootfs → snapshot as
   `pve-base.squashfs`.
2. Bind-mount `/proc`, `/sys`, `/dev`; install kernel, GRUB, zfs/btrfs/xfs tooling,
   `proxmox-ve` (and its full closure) from a **local apt package pool** so install works
   offline from the ISO.
3. Overlay a writable layer on `pve-base.squashfs`, install the installer wizard +
   `openssh-client` + `spice-vdagent`, snapshot as `pve-installer.squashfs`.
4. `grub-mkimage` for UEFI (`efi.img`, FAT16, El Torito), classic isolinux/MBR chain for
   BIOS.
5. `xorriso` produces the final hybrid (BIOS+UEFI) ISO 9660 image.

## 3. Dependency-closure analysis: rebuild vs. reuse

We changed exactly four source packages this session: `pve-qemu` (→ `pve-qemu-kvm`),
`qemu-server`, `pve-manager`, `pve-edk2-firmware`. None of the four changes:

- bump a shared-library SONAME other packages link against (pve-qemu's libs are
  private/internal to the binary; nothing outside the package loads them),
- change a Perl/JS API surface other packages call into (qemu-server and pve-manager
  changes are additive — new `riscv64` branches alongside existing `x86_64`/`aarch64`
  ones),
- remove or rename anything pve-edk2-firmware already shipped (only adds new firmware
  files + a new `pve-edk2-firmware-riscv64` binary, mirroring the existing `-aarch64`
  package).

So every other package in the closure — kernel, corosync, pve-cluster, ceph, the ~40
`libpve-*-perl` packages, everything — can be pulled **unmodified, at its current
version**, straight from `download.proxmox.com`'s `pve-no-subscription` repo, exactly as
we already did for the build sysroot in this session. Zero rebuild needed outside the
four repos already on `feature/riscv64-guest-support`.

## 4. Rebuild scope — the actual new work

| Package | What's needed | Risk | Time | Blocker |
|---|---|---|---|---|
| `pve-qemu-kvm` | Full `--target-list=x86_64-softmmu,aarch64-softmmu,riscv64-softmmu` build (this session only built the riscv64-only spike) | Low — process already proven end to end | ~30-90 min compile (3x the single-target build) | `dh_shlibdeps` needs a real `dpkg` status database; our unprivileged `apt-get download`+`dpkg-deb -x` sysroot doesn't register one, so `dpkg-shlibdeps` can't find dependency info for locally-extracted libs (confirmed this session — build/install succeed, only the final `binary-arch` shlibdeps step fails) |
| `qemu-server` | Straightforward `dh`-based Perl package build | Low | Low — no compiled objects | Should work in the same unprivileged sandbox; not yet attempted |
| `pve-manager` | Straightforward `dh`-based package build (JS/Perl, `Architecture: all`) | Low | Low | Should work unprivileged; not yet attempted |
| `pve-edk2-firmware` (riscv64 firmware images) | New cross-compilation environment: `gcc-riscv64-linux-gnu`, EDK2 BaseTools, the `edk2`/`edk2-platforms` submodules | Medium-high — untested in this session, different toolchain than the QEMU build | Comparable to the QEMU build, unknown until attempted | Not yet attempted at all |

## 5. ISO assembly — the step requiring real root

Everything in this session's build work used an unprivileged trick: a user namespace
(`unshare --user --map-root-user --mount`) plus an OverlayFS mount to make extracted
`.deb` contents visible at real system paths, without ever needing genuine root. That
technique is sufficient for *compiling and running* a binary. It is not realistically
sufficient for `debootstrap`-ing a full rootfs and running `dpkg -i`/`apt-get install`
against hundreds of packages with real maintainer scripts (postinst/postrm, `update-*`
triggers, systemd unit registration, etc.) — that class of operation needs a genuine
chroot with real root, or a real container (Docker/Podman/systemd-nspawn) with root
inside it.

This is a hard blocker in the current sandbox specifically, not in the ISO-build concept
generally. Options:

- A real root shell / VM (a private Proxmox VE host reachable over SSH as root has real
  root and is Debian-based, so it's technically capable — but it also runs other live
  VMs, so using it for a multi-package apt/dpkg build run needs an explicit go-ahead
  given the resource and disruption footprint).
- Any other machine/VM/container where you can grant real root for a throwaway build
  environment.

## 6. Recommended path if you want to proceed

1. Rebuild `qemu-server` and `pve-manager` as real `.deb`s in the existing unprivileged
   sandbox — low risk, likely works as-is, no new blockers expected.
2. Attempt the full 3-target `pve-qemu-kvm` build and see whether `dh_shlibdeps` can be
   worked around cheaply (e.g. a minimal fake `dpkg` admin dir seeded with `Provides`/
   `shlibs` entries for the handful of libraries actually missing metadata) before
   concluding it needs real root.
3. Attempt the `pve-edk2-firmware` riscv64 cross-build — this is the least-proven part
   and worth an early go/no-go check.
4. Only once 1-3 produce real `.deb`s: stand up a genuine-root environment (your call on
   where) and run the debootstrap/squashfs/xorriso assembly, substituting our four
   `.deb`s into the local package pool ahead of everything else pulled unmodified from
   Proxmox's real repo.

Steps 1-3 can continue in this sandbox without any new access. Step 4 needs a decision
from you on where the privileged build happens.
