# Upstream Submission Plan — riscv64 Guest Support

Companion to `RISCV64_GUEST_SUPPORT_PLAN.md` (the plan) and `RISCV64_RESULTS.md` (what was
built and verified). This document covers what's left before the patches can actually be
mailed to `pve-devel@lists.proxmox.com` and accepted.

Proxmox does not take GitHub PRs — patches are reviewed on a public mailing list via
`git send-email`. That process, and everything below, is sourced from
[Developer Documentation](https://pve.proxmox.com/wiki/Developer_Documentation), the
[Perl Style Guide](https://pve.proxmox.com/wiki/Perl_Style_Guide), and precedent from
[lore.proxmox.com](https://lore.proxmox.com/pve-devel/).

**Bottom line: the code is verified and working, but not yet submission-shaped.** Nothing
found below invalidates the implementation — every item is process/hygiene, not a redesign.

---

## 1. Blocking legal/process prerequisites

| # | Item | Status | Action |
|---|---|---|---|
| 1 | **Harmony CLA** on file with Proxmox | Unknown — no record of this being sent | Download the individual-contributor PDF from Proxmox's agreements page, sign, email to `office@proxmox.com`. Required before any patch can be *committed*, not before it can be *sent* — but should be started now since it can take a review cycle. |
| 2 | **Bugzilla entry** for the feature | None exists — checked `bugzilla.proxmox.com`, no `riscv64` hits | File an enhancement request ("Support riscv64 as a VM guest architecture") on bugzilla.proxmox.com. Gives the series a `#NNNN` for `close #NNNN:` in the lead commit, and is where reviewers/other users will find and discuss it. |
| 3 | **Coordinate before sending** | Not yet done | Proxmox's own guidance: "coordinate your efforts before starting development... to have a common view of the problem and solution." Development already happened, so this becomes a **heads-up/RFC email** to pve-devel *before* the formal series — short, links the bugzilla entry, states scope (TCG-only, x86_64 host, no KVM) and asks whether that scope is acceptable before a 4-repo series lands in reviewers' inboxes. This is the single highest-leverage step: it's much cheaper to learn "we don't want TCG-only riscv64" now than after formatting five patch series. |

## 2. Commit hygiene — every commit needs rework before sending

Inspected all four `feature/riscv64-guest-support` commits against the style guide. Two
findings apply to all four; the rest are per-repo.

### 2.1 Applies to all four repos

- **No `Signed-off-by` trailer.** Required on every commit (`git commit -s`, or
  `git config format.signoff true`). None of the four have it. Must be added — this is a DCO
  attestation, not optional.
- **`Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>` and `Claude-Session: ...`
  trailers.** `Co-authored-by` is on Proxmox's recognized trailer list, so the attribution
  itself isn't the problem — but Proxmox has published no policy on AI-assisted
  contributions, and `Claude-Session:` is a nonstandard trailer that leaks an internal
  session URL with no meaning to an outside reviewer. **This needs your decision**, not mine:
  keep as-is, keep `Co-authored-by` and drop `Claude-Session`, or drop both. Whichever you
  pick, apply it uniformly across all four before formatting patches.
- **Debian changelog stanzas use a fake identity and a local version.** Each repo's
  `debian/changelog` got a new top stanza like:
  ```
  qemu-server (9.2.5+riscv1) trixie; urgency=medium
   * Add riscv64 guest support: ...
   -- Proxmox riscv64 support <support@proxmox.com>  Sat, 15 Aug 2026 ...
  ```
  `support@proxmox.com` as a maintainer identity is an impersonation risk in a real patch —
  drop it. The `+riscv1`/`+riscv2` local-version suffix was correct for the sandboxed test
  build (kept it from colliding with/being downgraded by stock packages) but has no place
  upstream, where Proxmox's own release process owns versioning. **Recommendation:** drop
  the changelog hunk from all four patches entirely. Upstream maintainers add the changelog
  entry themselves at release time (visible in existing history — most feature commits in
  `git log` don't touch `debian/changelog`); let the commit message carry the description
  instead.

### 2.2 Per-repo: split into reviewable commits

Proxmox reviewers expect one logical change per commit. All four are currently one big
commit each. Concretely:

**`qemu-server`** (19 files, 346 insertions) — split into:
1. `helpers: add uses_virt_machine() and use it in Machine.pm` (mechanical, low risk)
2. **`fix: use sysbus TPM (tpm-tis-device) on virt-machine architectures`** — this is a
   real bug affecting **aarch64 today**, independent of riscv64. Strongly consider sending
   this alone, immediately, as its own one-line patch with its own bugzilla bug — it's an
   easy, uncontroversial aarch64 fix that doesn't need to wait on the rest of the series,
   and getting it reviewed separately de-risks the bigger series.
2b. Replace the remaining `$arch eq 'aarch64'` call sites (PCI.pm, USB.pm, QemuServer.pm)
   with `uses_virt_machine($arch)` — can ride with #1 or stand alone.
3. `cpuconfig: add riscv64 CPU models, default type, and bitness` — includes the
   `-f $cpu_models_riscv64_file` guard; call out in the commit body *why* the guard exists
   (breaks nothing on hosts with an older pve-qemu-kvm) since it's the kind of defensive
   check a reviewer will ask about if unexplained.
4. `ovmf/machine: add riscv64 firmware paths and default machine`
5. `test: add riscv64 cfg2cmd fixtures`
   (5 replaces the single `riscv64: add support for riscv64 VM guests` commit with a small
   series — use `--cover-letter` to explain how they fit together.)

**`pve-qemu`** (6 files) — split into:
1. **`fix: purge only the riscv32 OpenSBI blob, not riscv64's`** — standalone bug fix.
   Currently folded into the target-list commit, but it's the most severe bug found
   ("no riscv64 guest can ever boot") and reads as an obvious, self-contained fix once
   separated from the larger build-system change.
2. `debian/parse-cpu-models.pl: recognize riscv64 CPU models`
3. `debian/parse-machines.pl: don't die when a target has no versioned machines`
4. `debian/rules: build riscv64-softmmu and generate its machine/CPU-model metadata`
   (needs 2 and 3 to land first, or the build step it adds will fail immediately — order
   matters within this series)

**`pve-manager`** — this one is plausibly fine as a single commit (it's already a coherent
UI-layer change), but drop the changelog hunk per §2.1 and double check `CPUModelSelector.js`
against `eslint`/the JS style guide (see §3) before sending.

**`pve-edk2-firmware`** — single commit is fine; it's already a minimal, single-purpose
change (two PCD flags).

### 2.3 Subject-line / series-grouping convention

Checked recent multi-repo feature submissions on lore.proxmox.com for precedent — TDX and
AMD SEV-SNP support both went out as **one combined series across firmware + qemu-server +
manager**, e.g.:

```
[PATCH edk2-firmware/qemu-server/manager v3 0/9] Add support for Intel TDX
```

`pve-qemu` was not part of those combined series because those features didn't need new
QEMU build targets — ours does. Recommended grouping, in send order:

1. **`qemu-server`** standalone: `[PATCH qemu-server] fix: use tpm-tis-device on virt machines`
   (the aarch64 bug fix from §2.2, sent first and independently)
2. **`pve-qemu`** standalone series: `[PATCH qemu 0/4] build riscv64-softmmu target`
   (uses bare `qemu` prefix — confirmed current convention, not `pve-qemu`)
3. **`edk2-firmware/qemu-server/manager`** combined series once #2 is merged/acked:
   `[PATCH edk2-firmware/qemu-server/manager 0/N] add riscv64 guest support`, cover letter
   explicitly notes the dependency on the `qemu` series from step 2.

This mirrors real precedent and lets reviewers evaluate the foundational QEMU build change
on its own before the larger guest-integration series.

## 3. Code-quality gates to run before sending

Proxmox requires `proxmox-perltidy` + `perlcritic` (severity 5) for Perl, and `eslint` +
the JS style guide for pve-manager. **Neither is installed in this environment and neither
is in the currently configured apt sources** (checked; `apt-cache search` returns nothing).
Before formatting patches:

- Source `proxmox-perltidy` and its `.perltidyrc` (it's a thin wrapper — check
  `pve-common`/`pve-doc-generator` or ask on-list for the current package name) and run it
  over every changed `.pm`/`.pl` file. None of this session's Perl was ever run through it.
- Install `perlcritic` (`libperl-critic-perl` from Debian) and run at `--severity 5`
  against the same files.
- Run `eslint` against `www/manager6/{qemu/Architecture.js,qemu/OSDefaults.js,
  form/CPUModelSelector.js,Utils.js}` — these were only checked by manual brace-balance
  counting this session, never by a real linter or in a browser.
- `perl -wc` each changed `.pm` as a cheap first pass (already done during implementation,
  but re-run after any commit-splitting edits change file contents).

## 4. Known-open bugs — decide disposition before sending

Two bugs from `RISCV64_RESULTS.md` §6 are still unfixed. Each needs an explicit decision,
not silence, since a careful reviewer will find them by testing:

- **Bug 8 — `ide-cd` emits `bus=ide.N` on virt machines** (affects aarch64 too, pre-existing).
  Options: (a) fix it and fold into the qemu-server series as another standalone commit,
  (b) file it as its own bugzilla bug and mention it in the riscv64 series' cover letter as
  a known, separately-tracked limitation so reviewers don't think it was missed. Given it's
  pre-existing and aarch64-shared, (b) keeps the riscv64 series scoped and lets the fix be
  reviewed on its own merits.
- **Bug 9 — EDK2 publishes ACPI but not FDT**, breaking FreeBSD riscv64. Needs
  `PcdForceNoAcpi=TRUE` in the firmware build — untried. Same choice as above; given it only
  affects non-Linux guests and has a documented workaround (`args: -machine acpi=off`),
  filing it separately rather than blocking the series on a firmware rebuild seems right,
  but flag it explicitly in the cover letter either way.

## 5. Documentation (`pve-docs`)

Not yet touched — no `pve-docs` clone exists in this workspace (the `riscv64-docs/`
directory here is a stray second clone of our own `pve-riscv64-guest-support` GitHub repo,
not Proxmox's actual docs repo). `arch: <string>` and the CPU-model/machine documentation in
`pve-docs` likely need a riscv64 mention alongside the existing aarch64 one. Clone
`git://git.proxmox.com/git/pve-docs.git`, find where aarch64 is documented, and add the
equivalent riscv64 coverage (AsciiDoc, ~80-100 col lines per the style guide) as a fifth,
low-risk patch — good candidate to fold into the combined series in §2.3 step 3.

## 6. Sending mechanics

Per-repo, before formatting:

```bash
git config --local sendemail.to pve-devel@lists.proxmox.com
git config --local format.subjectprefix 'PATCH <repo-prefix>'  # qemu-server / qemu / manager / edk2-firmware
git config --local format.signoff true
```

Format and send (example for the qemu-server sub-series once split per §2.2):

```bash
git format-patch -s -o outgoing/ --cover-letter master..feature/riscv64-guest-support
# edit outgoing/0000-cover-letter.patch: summary, scope, bugzilla link, test coverage,
# explicit note on the two open bugs from §4 and their tracking status
git send-email --to=pve-devel@lists.proxmox.com outgoing/*.patch
```

SMTP needs configuring once (`~/.gitconfig`, `[sendemail]` block — server/port/user for
whichever address the CLA was signed under). Use `-v2`, `-v3`, ... on any resend after
review feedback, and always resend the *complete* series at the new version, with a
"changes since v1" summary in the cover letter.

## 7. Sequenced checklist

1. [ ] File the bugzilla enhancement request; get the `#NNNN`.
2. [ ] Send the RFC/heads-up email to pve-devel (scope + bugzilla link) — wait for signal
   before proceeding, per Proxmox's own "coordinate first" guidance.
3. [ ] Send the Harmony CLA to `office@proxmox.com` (parallel with step 2 — has its own lead
   time).
4. [ ] Decide the AI-attribution trailer question (§2.1) and apply uniformly.
5. [ ] Strip the `debian/changelog` hunks and fake maintainer identity from all four commits.
6. [ ] Split `qemu-server` and `pve-qemu` into the reviewable commit sequences in §2.2;
   re-add `Signed-off-by` throughout.
7. [ ] Source and run `proxmox-perltidy` + `perlcritic -5` (Perl) and `eslint` (JS); fix
   anything they flag.
8. [ ] Decide and document disposition of bugs 8 and 9 (§4).
9. [ ] Add the `pve-docs` patch (§5).
10. [ ] Send `qemu-server`'s standalone TPM fix first.
11. [ ] Send the `pve-qemu` (`qemu`-prefixed) series once the TPM fix has landed or at least
    been acked.
12. [ ] Send the combined `edk2-firmware/qemu-server/manager` (+docs) series, cover letter
    referencing the merged/pending `qemu` series.
13. [ ] Track review threads on lore.proxmox.com; respond with `-v2`+ as needed.
