# Letta Memfs System - Research Notes

## What is Memfs?

**Memfs** (Memory Filesystem) is a git-backed versioned memory system for Letta AI agents. It stores agent "memory blocks" (structured text like persona, instructions, notes) as Markdown files in git repositories, providing full version history, diffing, and rollback capabilities.

## Architecture

```
┌─────────────────────────────────────────────────┐
│              REST API / Git HTTP Proxy           │
│  (git_http.py - Smart HTTP protocol endpoints)  │
├─────────────────────────────────────────────────┤
│          GitEnabledBlockManager                  │
│  (block_manager_git.py - orchestrates writes)    │
│  git-first writes → postgres sync as cache       │
├─────────────────────────────────────────────────┤
│              MemfsClient                         │
│  (memfs_client_base.py - block ↔ git ops)        │
├─────────────────────────────────────────────────┤
│             GitOperations                        │
│  (git_operations.py - git CLI wrapper)           │
├─────────────────────────────────────────────────┤
│          StorageBackend (abstract)                │
│  ├── LocalStorageBackend (local filesystem)      │
│  └── [Cloud backend - GCS/S3, enterprise only]   │
└─────────────────────────────────────────────────┘
```

## Key Components

### 1. StorageBackend (`letta/services/memory_repo/storage/base.py`)
Abstract interface for blob storage with methods: `upload_bytes`, `download_bytes`, `exists`, `delete`, `list_files`, `delete_prefix`, `copy`.

### 2. LocalStorageBackend (`letta/services/memory_repo/storage/local.py`)
OSS implementation storing files at `~/.letta/memfs/`. Directory layout:
```
~/.letta/memfs/repository/{org_id}/{agent_id}/repo.git/
```

### 3. GitOperations (`letta/services/memory_repo/git_operations.py`)
Core git engine that:
- **Creates repos** — `git init` in temp dir, commits initial files, uploads `.git/` contents to storage
- **Commits changes** — downloads repo to temp dir → applies file changes → `git commit` → uploads delta back
- **Reads files** — downloads repo, uses `git ls-tree` + `git show` to read at any ref
- **Gets history** — uses `git log` with custom format parsing
- Uses **Redis locks** for concurrent commit safety
- Uses **delta uploads** (mtime-based change detection) for efficiency

### 4. MemfsClient (`letta/services/memory_repo/memfs_client_base.py`)
High-level client translating block operations to git operations:
- Each memory block → `{label}.md` file in the repo
- `create_repo_async()` — initializes repo with agent's blocks
- `get_blocks_async()` — reads all `.md` files at a ref, parses frontmatter
- `update_block_async()` / `create_block_async()` / `delete_block_async()` — commit file changes
- `get_history_async()` — returns commit log
- Block IDs are deterministic UUIDs: `md5(agent_id:label)`

### 5. Block Markdown Format (`letta/services/memory_repo/block_markdown.py`)
Blocks stored as Markdown with YAML frontmatter:
```markdown
---
description: "Who I am and how I approach work"
limit: 20000
---
My name is Memo. I'm a stateful coding assistant...
```
- `description`, `limit` always in frontmatter
- `read_only`, `metadata` only when non-default
- Backward compatible: files without frontmatter treated as value-only

### 6. GitEnabledBlockManager (`letta/services/block_manager_git.py`)
Extends standard `BlockManager`:
- Agents opt in via `git-memory-enabled` tag
- **Write path**: commit to git first (source of truth) → sync to PostgreSQL (cache)
- **Read path**: reads from PostgreSQL for performance
- `enable_git_memory_for_agent()` creates repo and backfills existing blocks
- `get_block_at_commit()` for time-travel queries
- `sync_blocks_from_git()` to rebuild Postgres cache from git

### 7. Git HTTP Proxy (`letta/server/rest_api/routers/v1/git_http.py`)
REST endpoints at `/v1/git/{agent_id}/state.git/...`:
- Proxies Git Smart HTTP protocol (clone/push/pull) to external memfs service
- After `git push`, triggers `_sync_after_push()` to sync blocks to PostgreSQL
- Handles block detachment when `.md` files removed in git

## Data Flow

### Writing a block update:
1. `GitEnabledBlockManager.update_block_async()` checks `git-memory-enabled` tag
2. Calls `MemfsClient.update_block_async()` → serializes block to Markdown
3. `GitOperations.commit()` acquires Redis lock → downloads repo → applies changes → `git commit` → uploads delta
4. Syncs value to PostgreSQL as cache

### Reading blocks:
- Reads from PostgreSQL (fast cache path)
- Historical reads: `get_block_at_commit()` reads from git at specific SHA

### External git push:
- User does `git push` via Smart HTTP
- Letta proxies to memfs service
- `_sync_after_push()` reads HEAD from storage and syncs all `.md` files to PostgreSQL

## Data Models (`letta/schemas/memory_repo.py`)

```python
class MemoryCommit(LettaBase):
    sha: str               # 40-char commit SHA
    parent_sha: str | None # Parent commit SHA
    message: str           # Commit message
    author_type: str       # "agent", "user", "system"
    author_id: str
    author_name: str | None
    timestamp: datetime
    files_changed: List[str]
    additions: int
    deletions: int

class FileChange(LettaBase):
    path: str              # File path in repo
    content: str | None    # New content (None = delete)
    change_type: str       # "add", "modify", "delete"
```

## Configuration

- **`LETTA_MEMFS_SERVICE_URL`** — URL of external memfs service (cloud/enterprise). Enables git HTTP proxy and `GitEnabledBlockManager`
- OSS: local storage at `~/.letta/memfs/` with no external dependencies
- `__init__.py` tries cloud `MemfsClient` first, falls back to local `memfs_client_base.py`

## Key Design Decisions

1. **Git as source of truth** — All memory mutations go through git first; PostgreSQL is cache
2. **File-per-block** — Each memory block is a separate `.md` file, enabling per-block history
3. **Frontmatter metadata** — Block metadata stored in YAML frontmatter
4. **Opt-in per agent** — Only agents tagged with `git-memory-enabled` use git storage
5. **Delta uploads** — Only changed git objects uploaded after commits (mtime-based detection)
6. **Redis locking** — Prevents concurrent commits to same agent's repo
7. **Temp directory workflow** — Repos downloaded to temp dirs for operations, uploaded back (no persistent local state)
8. **Path-based labels** — Hierarchical organization (`system/human`, `system/persona`, `user/preferences`)
