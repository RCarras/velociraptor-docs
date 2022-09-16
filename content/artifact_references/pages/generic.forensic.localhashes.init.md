---
title: Generic.Forensic.LocalHashes.Init
hidden: true
tags: [Client Artifact]
---

This artifact creates an SQLite database on the endpoint to hold
local file hashes. These hashes can then be queried quickly.


```yaml
name: Generic.Forensic.LocalHashes.Init
description: |
   This artifact creates an SQLite database on the endpoint to hold
   local file hashes. These hashes can then be queried quickly.

parameters:
  - name: HashDb
    description: Name of the local hash database
    default: hashdb.sqlite

sources:
  - query: |
      LET SQL = "
        CREATE table if not exists hashes(path text, md5 varchar(16), size bigint, timestamp bigint)
        create index if not exists hashidx on hashes(md5)
        create index if not exists pathidx on hashes(path)
        create unique index if not exists uniqueidx on hashes(path, md5)
        "

      LET hash_db <= path_join(components=[dirname(path=tempfile()), HashDb])

      LET _ <= log(message="Will use local hash database " + hash_db)

      // SQL to create the initial database.
      LET _ <= SELECT * FROM foreach(
      row={
          SELECT Line FROM parse_lines(filename=SQL, accessor="data")
          WHERE Line
      }, query={
         SELECT * FROM sqlite(file=hash_db, query=Line)
      })

      SELECT hash_db AS FullPath FROM scope()

```