# releases · Delivery package

The files here are **generated from a commit**, not from the working tree. The goal:
anyone can take the delivery and apply it to their copy without copying `.venv`, caches,
or a local database.

## Current delivery: v14 verification + delivery documents

This package is generated above the GitHub baseline `24dc3e6` (`v14`) of
**`AbdElrhmanAhmed7/NovaAPI-v14`** — the first upload that arrived **without a wrapper
directory**. `bash tools/check_layout.sh` passed on the fresh clone, and GitHub ran the
`backend` workflow on the push: **3 jobs, 3 successes**.

* Every gate was re-run from scratch on that tree (new clone, new virtualenv):
  **451 tests · 94% coverage · 15/15 missions · 67/67 operations audited for player
  text (in-process and over a real socket) · 12 offline + 14 live probes · 0 failures**
  (NASA POWER `live`). Evidence: `verification/2026-11-12-v14/`, including
  `github-actions.json` / `.log` for the green CI run.
* The delivery artefacts were missing: `SHA256SUMS.txt` described a patch and a zip
  that were not in the tree. Both are regenerated here from the committed tree.
* The delivery documents pointed at the retired `AbdElrhmanAhmed7/Final` / `11cefe5`
  baseline; they now name `NovaAPI-v14` / `24dc3e6` (`PROJECT_STATUS.md` §9.7 and §13,
  `MIGRATION.md` §10, `verification/README.md`). Earlier sections stay as history.
* The CI path filters now include `*.md` and `tools/**`, because pytest gates the root
  documents and the repo tools — a docs-only push no longer skips CI.

Earlier packages are already merged in this tree: the layout migration (`git mv` out of
`repo/`, fifth occurrence) and the English localization (baseline `a2fd1cc`) — all
player-facing content and documentation are English, the legacy `ar` locale field is
retained as an English alias, and `tests/test_english_content.py` guards against Arabic
text returning to content, API payloads or the console.

## How they are generated

```bash
git commit -m "…"                        # the tree must be clean
bash tools/make_release.sh --base <commit>   # default: origin/main
```

The script refuses to run if there are uncommitted changes (`git status --porcelain`) —
because a delivery package that does not match any commit is not reviewable.

| File | What it is | How to use it |
| --- | --- | --- |
| `nova-backend-final.patch` | This delivery's commit delta above the base (`git format-patch`, `releases/` excluded) | `git am releases/nova-backend-final.patch` on a clean clone of `24dc3e6` |
| `nova-backend-final.zip` | Full tree snapshot (`git archive HEAD`) without `releases/` | Extract into an **empty** folder, then `cd NOVA` and upload **everything inside it** — see the warning below |
| `SHA256SUMS.txt` | SHA-256 hashes of the files generated in the last run | `sha256sum -c releases/SHA256SUMS.txt` |

The patch contains the delivery documents, the CI path filters and the v14 verification evidence. The
release artefacts themselves are packaged separately so the patch does not contain a
copy of itself.

## The snapshot is byte-reproducible

`git archive` stamps the archive with the **commit** id (as the zip comment) and the
commit time (as every member's mtime). Two commits with an *identical tree* therefore
produce different bytes — which quietly broke `SHA256SUMS.txt`: committing the sums
creates a new commit, so the recorded zip hash never matched the zip regenerated from
that commit.

`tools/make_release.sh` now normalises that metadata: fixed timestamp, sorted entries,
no archive comment, and the **tracked file modes** written explicitly (`git archive`
leaves them empty, so a plain `unzip` falls back to its own defaults). The snapshot is a
function of the **tree** alone, so regenerating it after a `releases/`-only commit yields
the same hash and `sha256sum -c SHA256SUMS.txt` keeps passing. The patch is reproducible
for the same reason: `releases/` is excluded from it, so a release commit does not change
it.

## Why the zip is not committed to git

The `.zip` is placed in `.gitignore`: it is a copy of the same commit tree, so storing it
inside git duplicates history and grows the repo for no benefit, and every release can
regenerate it in a second. The **patch** is committed because it is reviewable text in
the diff, and `SHA256SUMS.txt` records the hash of the zip generated alongside it so you
can verify that your copy is identical.

```bash
bash tools/make_release.sh --base 24dc3e6        # the baseline pushed to AbdElrhmanAhmed7/NovaAPI-v14
cd releases && sha256sum -c SHA256SUMS.txt && cd ..
```

> **Note:** all filenames are lowercase (`nova-backend-final.*`) because
> `tools/make_release.sh` generates them that way. The documentation used to say
> `NOVA-backend-final.zip` with uppercase letters — fixed to match the script.

## Applying the patch

On a clean clone of the baseline (`24dc3e6`):

```bash
git am releases/nova-backend-final.patch
bash tools/check_layout.sh
cd NOVA/backend && make install
bash tools/verify.sh          # all gates + campaign + probes
```

After applying, the repository should pass the English-content gates:

```bash
cd NOVA/backend
.venv/bin/pytest tests/test_english_content.py -q
```

## After applying

```bash
cd NOVA/backend && make install
bash tools/verify.sh          # all gates + campaign + probes
git push origin main          # requires your GitHub permissions
```

After pushing, open the **Actions** tab: a workflow named `backend` must appear and run.
If it does not, `.github/workflows/backend.yml` is not at the **root** of the repository —
see [`MIGRATION.md`](../MIGRATION.md).

## ⚠ The zip has a `NOVA/` prefix folder — do not upload the folder itself

`git archive --prefix=NOVA/` wraps the snapshot in a single top-level directory so
that extracting it never scatters files over your current folder. That convenience is
also the trap that produced **five** wrapped deliveries: if you extract the zip and
drag the `NOVA/` folder onto GitHub's upload page, GitHub stores the folder name in
every path, `.github/workflows/backend.yml` ends up below the git root, and CI runs
nothing at all.

```bash
unzip nova-backend-final.zip -d delivery   # → delivery/NOVA/…
cd delivery/NOVA                           # ← go INSIDE the folder
ls .github/workflows/backend.yml           # must exist here
# select everything here (Ctrl+A) and drag that, never the NOVA folder
```

Using git avoids the whole class of mistake:

```bash
git clone https://github.com/AbdElrhmanAhmed7/NovaAPI-v14 && cd NovaAPI-v14
git am /path/to/releases/nova-backend-final.patch
bash tools/check_layout.sh                 # ✓ layout ok
git push origin main
```

After **any** upload or push, `bash tools/check_layout.sh` must say `✓ layout ok`, and
the **Actions** tab must show the `backend` workflow with 3 jobs.
