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

**Done — see §7 for the resulting branches.** Left as-written below since it explains the
reasoning; the "needs rework" framing now describes what *was* done, not what's outstanding.

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

- [x] **Done.** Cloned `proxmox-perltidy` (`git.proxmox.com/git/proxmox-perltidy.git`) for
  its real `perltidyrc`, and `pve-eslint` (own vendored eslint 8.41.0 fork) for JS — Debian's
  packaged `perltidy` (20250105) is too old for the config (`Unknown option:
  cuddled-paren-brace-weld`); installed `Perl::Tidy` 20260808 from CPAN instead, which works.
  Compared `perltidy`'s output on the master-vs-feature-branch diff of every changed `.pm`/`.pl`
  file (tidying both sides and diffing the tidied versions cancels out pre-existing style debt
  in these large legacy files, isolating exactly what this patch would need to reformat). Result:
  every file was already compliant except one multi-line `if (...)` in `USB.pm`, where the style
  guide's "once wrapping is required, the condition goes on its own line" rule wasn't followed —
  fixed. Also fixed a one-character pre-existing style wart (`scalar (@$machines)` →
  `scalar(@$machines)`) in a `pve-qemu` line already being rewritten.
- [x] **Done.** `perlcritic --severity 5` against every changed Perl file: zero violations in
  code actually added by this patch. It did flag several severity-5 issues elsewhere in the same
  large files (a nested named sub, a leading-zero integer literal, two `return undef;`s, three
  subroutine prototypes) — all pre-existing, all outside the diff, left untouched as out-of-scope
  drive-by fixes.
- [~] **Attempted, blocked.** `pve-eslint` needs a webpack build step to produce a runnable
  `bin/` entry point (`Cannot find module './eslint-helpers'` when run directly from the source
  checkout) — not worth chasing further for four small JS files. JS changes are still only
  verified by manual review, not a real linter run.
- [x] `perl -wc`/`perl -c` re-run clean on every file after the commit-splitting work below.

## 4. Known-open bugs — decide disposition before sending

Two bugs from `RISCV64_RESULTS.md` §6 are still unfixed. Each needs an explicit decision,
not silence, since a careful reviewer will find them by testing:

- **Bug 8 — `ide-cd` emits `bus=ide.N` on virt machines** (affects aarch64 too, pre-existing).
  **Decided: option (b)**, file separately. Draft text in
  `submission-drafts/03-bugzilla-known-gaps.txt`.
- **Bug 9 — EDK2 publishes ACPI but not FDT**, breaking FreeBSD riscv64. **Decided: option (b)**,
  file separately. Same draft file.

## 5. Documentation (`pve-docs`)

**Done.** Cloned `git://git.proxmox.com/git/pve-docs.git`, found every `aarch64` mention in
`qm.adoc` (architecture overview, firmware/OVMF, machine type, CPU models — four spots) and
added the riscv64 equivalent alongside each, all under ~80 columns. Deliberately did **not**
touch `pve-system-requirements.adoc` or `pve-faq.adoc`: those describe supported **host**
architectures, and riscv64 is guest-only here — adding it there would overclaim. Pushed as
`daemonhorn/pve-docs`, branch `submit/riscv64-v1`.

## 6. Sending mechanics

**Done** (config) / **drafted** (content) — not sent, sending itself is intentionally left to
you.

`sendemail.to`, `format.subjectprefix` (`qemu-server` / `qemu` / `manager` / `edk2-firmware` /
`docs` — confirmed each against real precedent on lore.proxmox.com, note `qemu` and `manager`
are bare, not `pve-`-prefixed), and `format.signoff` are set as local git config in all five
repos already. SMTP (`[sendemail]` server/port/user) still needs configuring once you have an
address to send from — that's yours to fill in, not something crackable from this environment.

`submission-drafts/` in this repo has the RFC email, both bugzilla reports, and both series'
cover letters, ready to copy/edit/send:

- `00-send-order-and-tpm-fix-note.txt` — the sequencing, plus exact `git format-patch` commands
  and commit ranges for the trickiest part (sending the standalone TPM fix separately from the
  combined series, which shares its qemu-server repo but not that one commit).
- `01-rfc-email.txt`, `02-bugzilla-feature-request.txt`, `03-bugzilla-known-gaps.txt`
- `04-cover-letter-pve-qemu.txt`, `05-cover-letter-combined.txt`

## 7. Submission-ready branches

**Done.** Each repo has a new `submit/riscv64-v1` branch (pushed to its existing GitHub fork,
`pve-docs` fork created fresh) built from `master`, not from `feature/riscv64-guest-support` —
that original branch is left untouched as the as-built/as-tested reference. Diffed every file
in the new branches against the original to confirm zero content drift beyond the two lint fixes
above.

| Repo | Commits | Notes |
|---|---|---|
| `qemu-server` | 6 (split from 1) | TPM fix isolated as commit 2/6 — designed to be cherry-picked and sent standalone first |
| `pve-qemu` | 4 (split from 1) | OpenSBI purge fix isolated as commit 1/4 |
| `pve-manager` | 1 (unchanged count) | changelog hunk stripped, sign-off added |
| `pve-edk2-firmware` | 1 (unchanged count) | changelog hunk stripped, sign-off added |
| `pve-docs` | 1 (new) | riscv64 guest documentation |

All debian/changelog hunks and the fake `support@proxmox.com` maintainer identity are gone from
every commit. **AI-attribution trailer decision (§2.1): applied a default** — kept
`Co-authored-by: Claude Opus 5 <noreply@anthropic.com>` (it's on Proxmox's recognized trailer
list), dropped `Claude-Session:` (nonstandard, leaks an internal URL with no meaning to an
outside reviewer). This was flagged as your call, not mine — override before sending if you'd
rather keep, change, or drop the trailer entirely; it's a one-line `git commit --amend` per repo
since nothing has been mailed yet.

Full test suite re-run on the split `qemu-server` branch (102 default-arch + 3 aarch64 fixtures)
to confirm the split introduced no regressions: all pass.

## 8. Sequenced checklist

1. [ ] File the bugzilla enhancement request (draft ready); get the `#NNNN`.
2. [ ] Send the RFC/heads-up email to pve-devel (draft ready) — wait for signal before sending
   patches, per Proxmox's own "coordinate first" guidance.
3. [ ] Send the Harmony CLA to `office@proxmox.com`.
4. [x] Decide the AI-attribution trailer question — applied a default (§2.1/§7), override if
   you'd rather not.
5. [x] Strip the `debian/changelog` hunks and fake maintainer identity from all commits.
6. [x] Split `qemu-server` and `pve-qemu` into reviewable commit sequences; `Signed-off-by`
   added throughout.
7. [x] Ran `proxmox-perltidy` + `perlcritic -5`; fixed the one real finding. `eslint` attempted,
   blocked on `pve-eslint`'s own build step — JS still only manually reviewed.
8. [x] Decided disposition of bugs 8 and 9 — file separately (draft reports ready).
9. [x] Added the `pve-docs` patch.
10. [ ] Send `qemu-server`'s standalone TPM fix first (commands in
    `submission-drafts/00-send-order-and-tpm-fix-note.txt`).
11. [ ] Send the `pve-qemu` (`qemu`-prefixed) series once the TPM fix has landed or at least
    been acked.
12. [ ] Send the combined `edk2-firmware/qemu-server/manager/docs` series, referencing the
    merged/pending `qemu` series.
13. [ ] Track review threads on lore.proxmox.com; respond with `-v2`+ as needed.
