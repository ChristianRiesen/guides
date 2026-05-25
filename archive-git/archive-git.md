# Long-Term Git Repository Archive Guide

## Goal

Produce a per-repository archive that is:

- As small as practical
- Self-verifying (multiple independent checksums)
- Resistant to bit rot (can repair scattered corruption with par2)
- Self-describing (carries its own manifest and recovery instructions)
- Restorable to a full git repository years later

## Tools

All available in Ubuntu 24.04 repositories:

```bash
sudo apt install git xz-utils par2 b3sum coreutils
```

(`tar`, `sha256sum`, and `sha512sum` come with `coreutils` and are already present.)

## Two Ways to Use This Guide

This guide can be used two ways:

1. **Automated:** run the companion script `archive-git`, which performs every step below, generates the manifest, and verifies the result end-to-end. This is the recommended path.
2. **Manual:** follow the steps yourself, e.g. for understanding what the script does, or for archiving in an environment where you can't run the script.

The end result is identical either way.

## Archive Layout

A finished archive consists of four files in one directory (file names use the repo's basename):

```
<repo>-archive.tar              The archive itself (uncompressed tar)
<repo>-archive.tar.par2         par2 recovery index
<repo>-archive.tar.vol*.par2    par2 recovery volume (20% redundancy)
<repo>-archive.tar.sha256       SHA-256 of the .tar (for a quick external check)
```

All four must be kept together. par2 protects the tar from the outside, so even a corrupted tar header can be repaired.

Inside the tar are six files:

```
repo.bundle.xz   LZMA2-compressed git bundle containing all refs
MANIFEST.txt     Human-readable metadata, hashes, and recovery instructions
REFS.txt         Snapshot of every ref (refname + object id) at archive time
SHA256SUMS       SHA-256 hashes of every file inside the tar
SHA512SUMS       SHA-512 hashes of every file inside the tar
B3SUMS           BLAKE3  hashes of every file inside the tar
```

Three checksum algorithms are stored as defense in depth: if any single hash function is ever found to be weak, the other two still provide a reliable integrity check.

## Creating the Archive

### Option A: Use the script

```bash
cd /where/you/want/the/archive
archive-git /path/to/repo
```

The script performs all the steps below, generates the manifest, and runs the verification at the end. See the script's source for the exact commands.

### Option B: Do it manually

#### Step 1: Check source repository integrity

You don't want to archive pre-existing corruption.

```bash
cd /path/to/repo
git fsck --full --no-dangling
```

#### Step 2: Mirror clone with no hardlinks

Working from a mirror keeps the source repository completely untouched. `--no-hardlinks` ensures the mirror is fully independent even when source and temp directory share a filesystem.

```bash
WORK=$(mktemp -d -t archive-git.XXXXXXXX)
git clone --mirror --no-hardlinks /path/to/repo "$WORK/mirror.git"
```

#### Step 3: Aggressive repack for maximum compression

```bash
git -C "$WORK/mirror.git" config pack.compression 9
git -C "$WORK/mirror.git" repack -adf --depth=250 --window=250
```

#### Step 4: Create the git bundle and capture a ref snapshot

A bundle is git's native single-file format containing every object and ref. No working tree is included. The ref snapshot is used later to verify the restored repository matches the original.

```bash
git -C "$WORK/mirror.git" bundle create "$WORK/repo.bundle" --all
git -C "$WORK/mirror.git" bundle verify "$WORK/repo.bundle"
git -C "$WORK/mirror.git" for-each-ref \
    --format='%(refname) %(objectname)' \
    --sort=refname > "$WORK/REFS.txt"
```

#### Step 5: Record hashes of the uncompressed bundle

These go into the manifest so the bundle can be verified after decompression during recovery.

```bash
sha256sum "$WORK/repo.bundle"
sha512sum "$WORK/repo.bundle"
b3sum     "$WORK/repo.bundle"
```

#### Step 6: Compress with xz

```bash
xz -9 -e --threads=0 "$WORK/repo.bundle"
```

`-9 -e` gives near-optimal LZMA2 compression. `--threads=0` uses all available cores. Output: `repo.bundle.xz`.

#### Step 7: Write the manifest

Create `$WORK/MANIFEST.txt` containing at minimum:

- Archive format version, creation timestamp (UTC ISO 8601), hostname, user
- Source repository name, original path, .git size
- HEAD ref, total refs, commit count, object count
- LFS / submodules detection results
- Versions of all tools used (`git --version`, `xz --version`, etc.)
- The complete list of files inside and outside the tar
- SHA-256, SHA-512, and BLAKE3 of both `repo.bundle` (uncompressed) and `repo.bundle.xz` (compressed)
- The recovery procedure below, with the actual archive filename substituted in

The `archive-git` script generates this file automatically; if writing it by hand, look at a manifest from a script-generated archive for the canonical format.

#### Step 8: Generate per-file checksums for the tar contents

```bash
cd "$WORK"
sha256sum repo.bundle.xz MANIFEST.txt REFS.txt > SHA256SUMS
sha512sum repo.bundle.xz MANIFEST.txt REFS.txt > SHA512SUMS
b3sum     repo.bundle.xz MANIFEST.txt REFS.txt > B3SUMS
```

#### Step 9: Pack the tar

```bash
cd "$WORK"
tar cf /output/path/<repo>-archive.tar -- \
    repo.bundle.xz MANIFEST.txt REFS.txt SHA256SUMS SHA512SUMS B3SUMS
```

The tar is intentionally uncompressed — its contents are already compressed, and an uncompressed tar is the simplest possible container format.

#### Step 10: Generate par2 recovery data and external SHA-256

```bash
cd /output/path
par2 create -q -q -r20 -n1 <repo>-archive.tar
sha256sum <repo>-archive.tar > <repo>-archive.tar.sha256
```

`-r20` provides 20% redundancy. `-n1` produces a single recovery volume (alongside the small `.par2` index file).

#### Step 11: Verify before trusting

Don't rely on `set -e`; actually verify the archive end-to-end before deleting the source. Use the recovery procedure below in a temporary directory. The `archive-git` script does this automatically, including a `diff` between the captured `REFS.txt` and `git for-each-ref` on the restored repository.

## Restoring the Archive

This procedure also appears in the manifest inside every archive, with the actual filename pre-substituted.

### Step 1: Verify the archive integrity using par2

This checks the `.tar` against the recovery data and reports any bit rot or corruption.

```bash
par2 verify <repo>-archive.tar.par2
```

### Step 2: Repair if needed

If `par2 verify` reported damage, repair it. par2 can fix damage up to the redundancy limit (20%) provided enough recovery data survived.

```bash
par2 repair <repo>-archive.tar.par2
```

### Step 3: Quick external check (optional)

```bash
sha256sum -c <repo>-archive.tar.sha256
```

### Step 4: Extract the tar

```bash
tar xf <repo>-archive.tar
```

### Step 5: Verify the extracted files

Any one of these is sufficient; running all three provides defense against any single algorithm being weakened in the future.

```bash
sha256sum -c SHA256SUMS
sha512sum -c SHA512SUMS
b3sum     -c B3SUMS
```

### Step 6: Decompress the bundle

```bash
xz -d repo.bundle.xz
```

### Step 7: Verify the decompressed bundle hash matches the manifest

```bash
sha256sum repo.bundle
```

Compare the output against the SHA-256 of `repo.bundle` listed under "HASHES OF KEY FILES" in `MANIFEST.txt`.

### Step 8: Restore the repository

```bash
git clone repo.bundle restored-repo
cd restored-repo
git fsck --full
```

You now have a complete repository with full history and all refs, identical to the archived original.

## Caveats Worth Knowing

- **git-lfs:** the bundle contains LFS pointer files but not the LFS-tracked blob content (which lives in a separate object store at `.git/lfs/`). If the repository uses LFS, archive that directory separately or run `git lfs fetch --all` before archiving. The `archive-git` script detects this and warns.
- **Submodules:** the archive contains only the parent repository. Each submodule is a separate repository and must be archived individually. The `archive-git` script detects this and warns.
