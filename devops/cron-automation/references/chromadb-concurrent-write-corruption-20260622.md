# ChromaDB Concurrent-Write HNSW Corruption

## Date
2026-06-22

## Problem
Two `ingest_chroma.py --all-pods --batch-size 32` processes writing to the same `chroma_db/` directory simultaneously caused HNSW vector index corruption. One process was launched manually (`bash run-obn-sync.sh`), the other by launchd (`RunAtLoad` triggered on plist bootstrap).

## Symptoms
- `chromadb.PersistentClient(path='chroma_db')` segfaults (exit code 139 / SIGSEGV) on ANY operation that touches collection data:
  - `collection.count()` → SIGSEGV
  - `collection.query(...)` → SIGSEGV
  - `collection.add(...)` → SIGSEGV
- `client.list_collections()` WORKS (returns collection names)
- `sqlite3 chroma_db/chroma.sqlite3 "PRAGMA integrity_check"` returns `ok`
- The SQLite layer is completely fine
- The HNSW index files (`.bin` files in UUID-named subdirectories) are corrupted

## Diagnosis Path
1. Run Python with ChromaDB: segfault before any output
2. Check `sqlite3 integrity_check`: ok
3. Try `list_collections()`: works (12 collections listed)
4. Try `collection.count()`: SIGSEGV
5. Conclusion: SQLite metadata is intact, HNSW native index is corrupted

## Root Cause
ChromaDB's HNSW index uses native (C/Rust) file-backed data structures that are NOT safe for concurrent writes. SQLite has WAL mode and handles concurrency, but the HNSW `.bin` files have no concurrency control. Two processes writing to the same collection's HNSW index simultaneously corrupts the index structure.

## Fix
1. Kill ALL ingestion processes: `ps aux | grep ingest_chroma | grep -v grep | awk '{print $2}' | xargs kill`
2. Restore ChromaDB from backup: `rm -rf chroma_db && cp -r chroma_db_backup_YYYYMMDD chroma_db`
3. Reset state DBs: `rm -f state-*.db /tmp/ob1_chroma_*.db`
4. Re-run with ONLY ONE process: `bash run-obn-sync.sh`
5. Verify: `python3 -c "import chromadb; c=chromadb.PersistentClient(path='chroma_db'); [print(f'{col.name}: {col.count()}') for col in c.list_collections()]"`

## Prevention
- **Never run two `ingest_chroma.py` processes concurrently** — they write to the same ChromaDB
- Before starting a manual sync, check for existing processes: `ps aux | grep ingest_chroma | grep -v grep`
- If a launchd `RunAtLoad` job exists, it will start immediately on bootstrap — either unload it first (`launchctl bootout`) or wait for it to finish before starting a manual run
- The git post-commit hook uses a lockfile (`/tmp/obn-sync.lock`) to prevent concurrent runs, but manual runs bypass this lock
- Consider adding a lockfile check to `run-obn-sync.sh` itself (not just the git hook)

## Backup Strategy
Always backup ChromaDB before operations that might cause concurrent access:
```bash
cp -r chroma_db chroma_db_backup_$(date +%Y%m%d)
```
The backup is the recovery path — there is no "repair" for corrupted HNSW index files.