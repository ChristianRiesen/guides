# Long-Term PostgreSQL Data Archive Guide

## Goal

Decommission a PostgreSQL server while preserving all user data in a form that is:

- Portable across PostgreSQL versions (plain SQL)
- As small as practical (xz compression)
- Self-verifying (multiple independent checksums)
- Resistant to bit rot (par2 recovery data, 20% redundancy)
- Self-describing (carries its own manifest and recovery instructions)
- Restorable years later, even on a different server

Only user data is preserved. Template databases (`template0`, `template1`) and the empty default `postgres` database are excluded — they belong to the engine, not to you.

## Tools

All available in Ubuntu 24.04 repositories:

```bash
sudo apt install postgresql-client xz-utils par2 b3sum coreutils
```

`postgresql-client` provides `psql`, `pg_dump`, and `pg_dumpall`. If the source server runs a newer PostgreSQL than your client, install the matching client version from the [PostgreSQL APT repository](https://www.postgresql.org/download/linux/ubuntu/) — using a client older than the server is unsupported and can silently produce a bad dump.

## Archive Layout

A finished archive consists of four files in one directory (substitute your own name, e.g. `dbhost01-postgres`):

```
<name>-archive.tar              The archive itself (uncompressed tar)
<name>-archive.tar.par2         par2 recovery index
<name>-archive.tar.vol*.par2    par2 recovery volume (20% redundancy)
<name>-archive.tar.sha256       SHA-256 of the .tar (for a quick external check)
```

All four must be kept together. par2 protects the tar from the outside, so even a corrupted tar header can be repaired.

Inside the tar:

```
globals.sql.xz   LZMA2-compressed dump of roles, tablespaces, and other cluster-wide objects
dump.sql.xz      LZMA2-compressed pg_dump of all user databases (single concatenated SQL stream)
MANIFEST.txt     Human-readable metadata, hashes, and recovery instructions
DATABASES.txt    List of databases included in the dump
SCHEMA.txt       Schema-only dump (no data) for quick reference
SHA256SUMS       SHA-256 hashes of every file inside the tar
SHA512SUMS       SHA-512 hashes of every file inside the tar
B3SUMS           BLAKE3  hashes of every file inside the tar
```

Three checksum algorithms are stored as defense in depth: if any single hash function is ever found to be weak, the other two still provide a reliable integrity check.

### Why plain SQL and not `-Fc` (custom format)?

`pg_dump -Fc` is smaller and supports parallel restore and selective extraction, but it is a binary format whose long-term stability depends on `pg_restore`. For an archive intended to outlive the toolchain that created it, plain SQL — which any future PostgreSQL can read, and which a human can open in a text editor — is the safer choice. The 20-year-from-now scenario is the design target.

## Creating the Archive

Pick a name and a working directory before you start:

```bash
NAME=dbhost01-postgres          # base name for the archive
OUT=/where/you/want/the/archive # where the final files land
WORK=$(mktemp -d -t archive-postgres.XXXXXXXX)
cd "$WORK"
```

Set connection parameters. The cleanest way is via standard libpq environment variables or a `~/.pgpass` file:

```bash
export PGHOST=127.0.0.1
export PGPORT=5432
export PGUSER=postgres
# Password from ~/.pgpass (mode 600):
#   127.0.0.1:5432:*:postgres:<your-password>
```

Verify connectivity before dumping anything:

```bash
psql -l
```

### Step 1: Enumerate user databases

Templates and the default `postgres` database are excluded. Build a list of everything else.

```bash
psql -At -c "
  SELECT datname
  FROM pg_database
  WHERE datistemplate = false
    AND datname <> 'postgres'
  ORDER BY datname;
" > "$WORK/DATABASES.txt"

cat "$WORK/DATABASES.txt"
```

Review the list. If `postgres` itself contains real data (it sometimes does on legacy servers), add it back manually.

### Step 2: Sanity-check the source data

Run a basic integrity check on each user database. `amcheck` (if available) is more thorough; at minimum, confirm no database is unreachable.

```bash
while read -r db; do
  echo "== $db =="
  psql -d "$db" -At -c "SELECT 'connected to ' || current_database();"
  psql -d "$db" -At -c "SELECT count(*) FROM pg_class WHERE relkind='r';" \
    | awk '{print "  user tables: " $1}'
done < "$WORK/DATABASES.txt"
```

If you have the `amcheck` extension installed, also run:

```bash
while read -r db; do
  psql -d "$db" -At -c "
    DO \$\$
    DECLARE r record;
    BEGIN
      FOR r IN SELECT oid::regclass AS rel
               FROM pg_class
               WHERE relkind='i' AND relam=(SELECT oid FROM pg_am WHERE amname='btree')
      LOOP
        PERFORM bt_index_check(r.rel);
      END LOOP;
    END
    \$\$;
  "
done < "$WORK/DATABASES.txt"
```

### Step 3: Dump cluster-wide globals (roles, tablespaces)

`pg_dumpall --globals-only` captures CREATE ROLE statements and tablespace definitions that live outside individual databases.

```bash
pg_dumpall \
    --globals-only \
    --no-role-passwords \
  > "$WORK/globals.sql"
```

`--no-role-passwords` omits MD5/SCRAM password hashes. This is the safe default — if the archive ever leaks, the password hashes don't. If you genuinely need to restore login credentials, drop the flag and treat the archive as secret material.

### Step 4: Dump all user databases to a single SQL file

```bash
> "$WORK/dump.sql"
while read -r db; do
  echo "-- ============================================================" >> "$WORK/dump.sql"
  echo "-- database: $db"                                                  >> "$WORK/dump.sql"
  echo "-- ============================================================" >> "$WORK/dump.sql"
  pg_dump \
      --format=plain \
      --create \
      --clean \
      --if-exists \
      --no-owner \
      --no-privileges \
      --quote-all-identifiers \
      --encoding=UTF8 \
      --serializable-deferrable \
      --dbname="$db" \
    >> "$WORK/dump.sql"
done < "$WORK/DATABASES.txt"
```

Flag-by-flag rationale:

- `--format=plain` — readable SQL, restorable by `psql` with no extra tools. Critical for long-term durability.
- `--create` — emit `CREATE DATABASE` so the restore recreates the database by name.
- `--clean --if-exists` — emit `DROP … IF EXISTS` before each `CREATE`. Makes restore idempotent without erroring on first run.
- `--no-owner --no-privileges` — strip ownership and GRANT statements. They reference roles that may not exist on the restore target, and they're rarely useful for a cold archive. If you do need them, drop both flags (and make sure `globals.sql` is restored first so the roles exist).
- `--quote-all-identifiers` — bulletproof against future reserved-word changes in SQL standards.
- `--encoding=UTF8` — force UTF-8 regardless of the source database's encoding.
- `--serializable-deferrable` — take the dump in a serializable read-only transaction, blocking until it can see a consistent snapshot without conflict. The safest consistency mode for a hot dump; on an idle server it's essentially free.

If a specific database fails to dump, exclude it from `DATABASES.txt` temporarily, investigate, and redo.

### Step 5: Generate a schema-only quick reference

A separate schema dump lets future-you understand the structure without restoring gigabytes of data.

```bash
> "$WORK/SCHEMA.txt"
while read -r db; do
  echo "-- database: $db" >> "$WORK/SCHEMA.txt"
  pg_dump \
      --schema-only \
      --no-owner \
      --no-privileges \
      --quote-all-identifiers \
      --dbname="$db" \
    >> "$WORK/SCHEMA.txt"
done < "$WORK/DATABASES.txt"
```

### Step 6: Record hashes of the uncompressed dumps

These go into the manifest so the dumps can be verified after decompression during recovery.

```bash
for f in globals.sql dump.sql; do
  echo "=== $f ==="
  echo "size:    $(stat -c%s "$WORK/$f")"
  echo "SHA-256: $(sha256sum "$WORK/$f" | awk '{print $1}')"
  echo "SHA-512: $(sha512sum "$WORK/$f" | awk '{print $1}')"
  echo "BLAKE3:  $(b3sum     "$WORK/$f" | awk '{print $1}')"
done
```

Copy this output into the manifest you'll write in step 8.

### Step 7: Compress with xz

```bash
xz -9 -e --threads=0 "$WORK/globals.sql" "$WORK/dump.sql"
```

`-9 -e` gives near-optimal LZMA2 compression. SQL dumps compress extremely well, typically 10:1 or better. `--threads=0` uses all available cores. Output: `globals.sql.xz` and `dump.sql.xz`.

### Step 8: Write the manifest

Create `$WORK/MANIFEST.txt` containing at minimum:

- Archive format version, creation timestamp (UTC ISO 8601), hostname, user
- Source server: hostname, port, server version (`SELECT version();`)
- Number of databases dumped
- Versions of all tools used (`pg_dump --version`, `xz --version`, etc.)
- The complete list of files inside and outside the tar
- SHA-256, SHA-512, and BLAKE3 of both `dump.sql`/`globals.sql` (uncompressed) and `dump.sql.xz`/`globals.sql.xz` (compressed)
- The recovery procedure below, with the actual archive filename substituted in

A minimal template:

```
================================================================
POSTGRESQL DATA ARCHIVE MANIFEST
================================================================

Created (UTC):          2026-05-27T12:34:56Z
Created on host:        <archive-host>
Created by user:        <archive-user>

Source server:          dbhost01.example.com:5432
Source server version:  PostgreSQL 15.4 on x86_64-pc-linux-gnu
Databases included:     see DATABASES.txt
Excluded:               template0, template1, postgres
Role passwords:         omitted (--no-role-passwords)

Tool versions:
  pg_dump:    <version>
  pg_dumpall: <version>
  xz:         <version>
  par2:       <version>
  tar:        <version>
  sha256sum:  <version> (also sha512sum)
  b3sum:      <version>

Hashes of key files:

  globals.sql (uncompressed)
    size:    <bytes>
    SHA-256: <hash>
    SHA-512: <hash>
    BLAKE3:  <hash>

  globals.sql.xz (compressed, as stored inside the tar)
    size:    <bytes>
    SHA-256: <hash>
    SHA-512: <hash>
    BLAKE3:  <hash>

  dump.sql (uncompressed)
    size:    <bytes>
    SHA-256: <hash>
    SHA-512: <hash>
    BLAKE3:  <hash>

  dump.sql.xz (compressed, as stored inside the tar)
    size:    <bytes>
    SHA-256: <hash>
    SHA-512: <hash>
    BLAKE3:  <hash>

(see RECOVERY INSTRUCTIONS section, copied verbatim from the
recovery section of this guide, with the actual filename
substituted in)
```

### Step 9: Generate per-file checksums for the tar contents

```bash
cd "$WORK"
sha256sum globals.sql.xz dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > SHA256SUMS
sha512sum globals.sql.xz dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > SHA512SUMS
b3sum     globals.sql.xz dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > B3SUMS
```

### Step 10: Pack the tar

```bash
cd "$WORK"
tar cf "$OUT/${NAME}-archive.tar" -- \
    globals.sql.xz dump.sql.xz \
    MANIFEST.txt DATABASES.txt SCHEMA.txt \
    SHA256SUMS SHA512SUMS B3SUMS
```

The tar is intentionally uncompressed — its contents are already compressed, and an uncompressed tar is the simplest possible container format.

### Step 11: Generate par2 recovery data and external SHA-256

```bash
cd "$OUT"
par2 create -q -q -r20 -n1 "${NAME}-archive.tar"
sha256sum "${NAME}-archive.tar" > "${NAME}-archive.tar.sha256"
```

`-r20` provides 20% redundancy. `-n1` produces a single recovery volume (alongside the small `.par2` index file).

### Step 12: Verify before trusting

Do not delete the source server until the archive has been restored and checked end-to-end. The recovery procedure below should be run against the freshly created archive on a test machine. A useful smoke test after restore:

```bash
# Compare row counts between source and restored copy.
while read -r db; do
  for tbl in $(psql -d "$db" -At -c "
    SELECT quote_ident(schemaname)||'.'||quote_ident(tablename)
    FROM pg_tables
    WHERE schemaname NOT IN ('pg_catalog','information_schema')"); do
    src=$(psql -h "$SRC_HOST"  -d "$db" -At -c "SELECT count(*) FROM $tbl")
    dst=$(psql -h "$REST_HOST" -d "$db" -At -c "SELECT count(*) FROM $tbl")
    [[ "$src" == "$dst" ]] || echo "MISMATCH: $db.$tbl ($src vs $dst)"
  done
done < DATABASES.txt
```

### Step 13: Clean up the working directory

```bash
rm -rf "$WORK"
```

## Restoring the Archive

This procedure also belongs in the manifest inside every archive, with the actual filename pre-substituted.

### Step 1: Verify the archive integrity using par2

```bash
par2 verify <name>-archive.tar.par2
```

### Step 2: Repair if needed

```bash
par2 repair <name>-archive.tar.par2
```

par2 can fix damage up to the redundancy limit (20%) provided enough recovery data survived.

### Step 3: Quick external check (optional)

```bash
sha256sum -c <name>-archive.tar.sha256
```

### Step 4: Extract the tar

```bash
tar xf <name>-archive.tar
```

### Step 5: Verify the extracted files

Any one of these is sufficient; running all three provides defense against any single algorithm being weakened in the future.

```bash
sha256sum -c SHA256SUMS
sha512sum -c SHA512SUMS
b3sum     -c B3SUMS
```

### Step 6: Decompress the dumps

```bash
xz -d globals.sql.xz dump.sql.xz
```

### Step 7: Verify the decompressed dumps match the manifest

```bash
sha256sum globals.sql dump.sql
```

Compare against the SHA-256 values listed under "HASHES OF KEY FILES" in `MANIFEST.txt`.

### Step 8: Restore globals first, then databases

Globals must be restored before the per-database dumps so any role references can resolve. Run as the PostgreSQL superuser:

```bash
psql -U postgres -d postgres -f globals.sql
psql -U postgres -d postgres -f dump.sql
```

The `dump.sql` file contains `CREATE DATABASE`, `DROP DATABASE IF EXISTS`, and per-database `\connect` directives, so a single `psql -d postgres -f dump.sql` is sufficient — `psql` will follow the `\connect` lines into each newly created database.

If you used `--no-role-passwords` (the default in this guide), set passwords manually after restore:

```bash
psql -U postgres -c "ALTER ROLE someuser WITH PASSWORD '...'"
```

### Step 9: Post-restore sanity checks

```bash
psql -l
psql -d postgres -c "
  SELECT datname, pg_size_pretty(pg_database_size(datname))
  FROM pg_database
  WHERE datistemplate = false
  ORDER BY datname;"
```

Spot-check a few tables for expected row counts.

## Caveats Worth Knowing

- **Role passwords are not included by default.** `--no-role-passwords` omits them so the archive is safe to store without being treated as a secret. If you genuinely need credentials preserved, drop the flag and protect the archive accordingly.
- **Ownership and privileges are stripped.** `--no-owner --no-privileges` produces a clean archive that restores under whatever role runs `psql`. If you need original ownership preserved, drop both flags — but be aware the restore target must already have the same roles, which means restoring `globals.sql` first.
- **Replication state is not preserved.** The archive is for cold restore, not for re-joining a streaming-replication or logical-replication topology.
- **Extensions.** `CREATE EXTENSION` statements are included in the dump, but the extension binaries themselves are not. Before restoring, make sure every extension referenced in the dump is installed on the target server (`SELECT * FROM pg_available_extensions` on the source is a good pre-archive check).
- **Large objects.** `pg_dump` includes large objects (`pg_largeobject`) by default in plain format when dumping a single database. If you have a database that uses LOs heavily, confirm they survived the round-trip in your verification step.
- **Very large databases.** For multi-terabyte databases, a logical dump can take many hours. Consider a per-database file inside the tar (`dump-<db>.sql.xz`) so a single corrupted table doesn't taint the whole archive, and so partial restores are possible.
- **Custom collations and locales.** A dump restored on a different OS or glibc version may sort text differently than the source. Indexes built on `text` columns may need `REINDEX` after restore. For most archived-and-forgotten datasets this doesn't matter, but it bites if you compare sort order across the restore boundary.
