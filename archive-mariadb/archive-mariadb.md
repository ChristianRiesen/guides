# Long-Term MariaDB / MySQL Data Archive Guide

## Goal

Decommission a MariaDB or MySQL server while preserving all user data in a form that is:

- Portable across MariaDB and MySQL versions (plain SQL)
- As small as practical (xz compression)
- Self-verifying (multiple independent checksums)
- Resistant to bit rot (par2 recovery data, 20% redundancy)
- Self-describing (carries its own manifest and recovery instructions)
- Restorable years later, even on a different server

Only user data is preserved. System databases (`mysql`, `information_schema`, `performance_schema`, `sys`) are intentionally excluded — they belong to the engine, not to you.

## Tools

All available in Ubuntu 24.04 repositories:

```bash
sudo apt install mariadb-client xz-utils par2 b3sum coreutils
# or for MySQL:
sudo apt install mysql-client xz-utils par2 b3sum coreutils
```

`mariadb-client` provides `mysqldump` and `mysql`. `mysql-client` works equally well — the dump format is interchangeable.

## Archive Layout

A finished archive consists of four files in one directory (substitute your own name, e.g. `dbhost01-mariadb`):

```
<name>-archive.tar              The archive itself (uncompressed tar)
<name>-archive.tar.par2         par2 recovery index
<name>-archive.tar.vol*.par2    par2 recovery volume (20% redundancy)
<name>-archive.tar.sha256       SHA-256 of the .tar (for a quick external check)
```

All four must be kept together. par2 protects the tar from the outside, so even a corrupted tar header can be repaired.

Inside the tar:

```
dump.sql.xz      LZMA2-compressed mysqldump of all user databases
MANIFEST.txt     Human-readable metadata, hashes, and recovery instructions
DATABASES.txt    List of databases included in the dump
SCHEMA.txt       CREATE TABLE statements only (quick reference, no data)
SHA256SUMS       SHA-256 hashes of every file inside the tar
SHA512SUMS       SHA-512 hashes of every file inside the tar
B3SUMS           BLAKE3  hashes of every file inside the tar
```

Three checksum algorithms are stored as defense in depth: if any single hash function is ever found to be weak, the other two still provide a reliable integrity check.

## Creating the Archive

Pick a name and a working directory before you start:

```bash
NAME=dbhost01-mariadb           # base name for the archive
OUT=/where/you/want/the/archive # where the final files land
WORK=$(mktemp -d -t archive-mariadb.XXXXXXXX)
cd "$WORK"
```

Set credentials once so they aren't repeated on every command line (and don't end up in shell history):

```bash
export MYSQL_PWD='your-root-password'
USER=root
HOST=127.0.0.1
PORT=3306
```

Alternatively, put credentials in `~/.my.cnf` with mode `600`:

```ini
[client]
user=root
password=your-root-password
host=127.0.0.1
port=3306
```

### Step 1: Enumerate user databases

System databases must be excluded. Build a list of everything else.

```bash
mysql -u"$USER" -h"$HOST" -P"$PORT" -N -B -e "SHOW DATABASES" \
  | grep -Ev '^(mysql|information_schema|performance_schema|sys)$' \
  > "$WORK/DATABASES.txt"

cat "$WORK/DATABASES.txt"
```

Review the list. If anything looks wrong, stop and fix it now.

### Step 2: Sanity-check the source data

You don't want to archive a broken database. For each user database, run a table check:

```bash
while read -r db; do
  echo "== $db =="
  mysql -u"$USER" -h"$HOST" -P"$PORT" -N -B \
    -e "SELECT CONCAT('CHECK TABLE \`',table_schema,'\`.\`',table_name,'\`;')
        FROM information_schema.tables
        WHERE table_schema='$db' AND table_type='BASE TABLE'" \
  | mysql -u"$USER" -h"$HOST" -P"$PORT" "$db"
done < "$WORK/DATABASES.txt"
```

Every row should report `status: OK`. Investigate anything that doesn't before proceeding.

### Step 3: Dump all user databases to a single SQL file

```bash
mysqldump \
    -u"$USER" -h"$HOST" -P"$PORT" \
    --single-transaction \
    --quick \
    --routines \
    --triggers \
    --events \
    --hex-blob \
    --default-character-set=utf8mb4 \
    --add-drop-database \
    --set-gtid-purged=OFF \
    --databases $(cat "$WORK/DATABASES.txt") \
  > "$WORK/dump.sql"
```

Flag-by-flag rationale:

- `--single-transaction` — consistent snapshot of InnoDB tables without locking writers. Required for a hot dump.
- `--quick` — stream rows directly instead of buffering whole tables in RAM. Essential for large tables.
- `--routines --triggers --events` — include stored procedures, functions, triggers, and scheduled events. Triggers are included by default but listed for clarity; the other two are not.
- `--hex-blob` — encode `BINARY`/`BLOB` columns as hexadecimal literals. Survives any text-encoding transform the dump might pass through.
- `--default-character-set=utf8mb4` — full Unicode, including emoji and supplementary planes.
- `--add-drop-database` — emit `DROP DATABASE IF EXISTS` before each `CREATE DATABASE`. Makes restore idempotent.
- `--set-gtid-purged=OFF` — suppress GTID metadata that would otherwise pin the restore to a specific replication topology. Harmless on standalone servers; required for portable restore.
- `--databases <list>` — emit `CREATE DATABASE` and `USE` statements before each database's contents, so restore recreates them as named.

If `mysqldump` errors out on a specific table, dump that database separately with `--ignore-table=db.table` to narrow it down, fix the underlying issue, then redo the full dump.

### Step 4: Generate a schema-only quick reference

A separate schema dump lets future-you understand the structure without restoring gigabytes of data.

```bash
mysqldump \
    -u"$USER" -h"$HOST" -P"$PORT" \
    --no-data \
    --routines \
    --triggers \
    --events \
    --skip-comments \
    --databases $(cat "$WORK/DATABASES.txt") \
  > "$WORK/SCHEMA.txt"
```

### Step 5: Record hashes of the uncompressed dump

These go into the manifest so the dump can be verified after decompression during recovery.

```bash
DUMP_SIZE=$(stat -c%s "$WORK/dump.sql")
DUMP_SHA256=$(sha256sum "$WORK/dump.sql" | awk '{print $1}')
DUMP_SHA512=$(sha512sum "$WORK/dump.sql" | awk '{print $1}')
DUMP_B3=$(b3sum         "$WORK/dump.sql" | awk '{print $1}')
echo "$DUMP_SIZE $DUMP_SHA256"
```

### Step 6: Compress with xz

```bash
xz -9 -e --threads=0 "$WORK/dump.sql"
```

`-9 -e` gives near-optimal LZMA2 compression. SQL dumps compress extremely well, typically 10:1 or better. `--threads=0` uses all available cores. Output: `dump.sql.xz`.

### Step 7: Write the manifest

Create `$WORK/MANIFEST.txt` containing at minimum:

- Archive format version, creation timestamp (UTC ISO 8601), hostname, user
- Source server: hostname, port, server version (`mysql --version` and `SELECT VERSION()`)
- Number of databases dumped, total row count (optional but useful)
- Versions of all tools used (`mysqldump --version`, `xz --version`, etc.)
- The complete list of files inside and outside the tar
- SHA-256, SHA-512, and BLAKE3 of both `dump.sql` (uncompressed) and `dump.sql.xz` (compressed)
- The recovery procedure below, with the actual archive filename substituted in

A minimal template:

```
================================================================
MARIADB / MYSQL DATA ARCHIVE MANIFEST
================================================================

Created (UTC):          2026-05-27T12:34:56Z
Created on host:        <archive-host>
Created by user:        <archive-user>

Source server:          dbhost01.example.com:3306
Source server version:  10.11.6-MariaDB
Databases included:     see DATABASES.txt
System DBs excluded:    mysql, information_schema, performance_schema, sys

Tool versions:
  mysqldump:  <version>
  xz:         <version>
  par2:       <version>
  tar:        <version>
  sha256sum:  <version> (also sha512sum)
  b3sum:      <version>

Hashes of key files:

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

### Step 8: Generate per-file checksums for the tar contents

```bash
cd "$WORK"
sha256sum dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > SHA256SUMS
sha512sum dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > SHA512SUMS
b3sum     dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt > B3SUMS
```

### Step 9: Pack the tar

```bash
cd "$WORK"
tar cf "$OUT/${NAME}-archive.tar" -- \
    dump.sql.xz MANIFEST.txt DATABASES.txt SCHEMA.txt \
    SHA256SUMS SHA512SUMS B3SUMS
```

The tar is intentionally uncompressed — its contents are already compressed, and an uncompressed tar is the simplest possible container format.

### Step 10: Generate par2 recovery data and external SHA-256

```bash
cd "$OUT"
par2 create -q -q -r20 -n1 "${NAME}-archive.tar"
sha256sum "${NAME}-archive.tar" > "${NAME}-archive.tar.sha256"
```

`-r20` provides 20% redundancy. `-n1` produces a single recovery volume (alongside the small `.par2` index file).

### Step 11: Verify before trusting

Do not delete the source server until the archive has been restored and checked end-to-end. The recovery procedure below should be run against the freshly created archive on a test machine. A useful smoke test after restore:

```bash
# Compare row counts between source and restored copy.
for db in $(cat DATABASES.txt); do
  for tbl in $(mysql -N -B -e "SHOW TABLES" "$db"); do
    src=$(mysql  -h "$SRC_HOST"   -N -B -e "SELECT COUNT(*) FROM \`$tbl\`" "$db")
    dst=$(mysql  -h "$REST_HOST"  -N -B -e "SELECT COUNT(*) FROM \`$tbl\`" "$db")
    [[ "$src" == "$dst" ]] || echo "MISMATCH: $db.$tbl ($src vs $dst)"
  done
done
```

### Step 12: Clean up the working directory

```bash
rm -rf "$WORK"
unset MYSQL_PWD
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

### Step 6: Decompress the dump

```bash
xz -d dump.sql.xz
```

### Step 7: Verify the decompressed dump matches the manifest

```bash
sha256sum dump.sql
```

Compare against the SHA-256 of `dump.sql` listed under "HASHES OF KEY FILES" in `MANIFEST.txt`.

### Step 8: Restore into a target MariaDB/MySQL server

```bash
mysql -u root -p < dump.sql
```

The dump contains `CREATE DATABASE` and `USE` statements for every database, so they are recreated as named. `--add-drop-database` in the dump means any existing database with the same name on the target server will be replaced — be sure that's intended.

### Step 9: Post-restore sanity checks

```bash
mysql -e "SHOW DATABASES"
mysql -e "SELECT COUNT(*) FROM information_schema.tables
          WHERE table_schema NOT IN
          ('mysql','information_schema','performance_schema','sys')"
```

Spot-check a few tables for expected row counts.

## Caveats Worth Knowing

- **Users and privileges are not included.** Grants live in the `mysql` system database, which we deliberately exclude. If you need to preserve users, dump the relevant `mysql.user`, `mysql.db`, etc. tables separately, or use `SHOW GRANTS FOR …` and save the output alongside the archive. For decommissioning, recreating users on the restore target by hand is usually cleaner anyway.
- **Replication state is not preserved.** `--set-gtid-purged=OFF` strips GTID metadata. The archive is for cold restore, not for re-joining a replication topology.
- **Storage engine assumptions.** `--single-transaction` only gives a consistent snapshot for transactional engines (InnoDB, RocksDB). MyISAM and ARIA tables are dumped table-by-table without locking; if writers are active during the dump, those tables may be inconsistent. Either stop writes first, or convert MyISAM tables to InnoDB before dumping.
- **Very large databases.** For multi-terabyte databases, a logical dump can take a long time. Consider a per-database dump loop (one `dump-<db>.sql.xz` per database inside the tar) so a single corrupt table doesn't taint the whole archive.
- **Character set quirks.** `--default-character-set=utf8mb4` covers modern data, but if the source server was configured with `latin1` and stored multibyte text incorrectly (the classic "mojibake" situation), dumping with `utf8mb4` will preserve the broken bytes. Audit suspect tables before archiving.
