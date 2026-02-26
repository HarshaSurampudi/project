# Mnemo

## A lightweight, on-device memory system for AI chat applications

Mnemo (from Greek *mneme* — memory) is a versioned memory system designed for mobile AI chat apps. It gives AI assistants the ability to remember everything about the people they talk to — stored locally on the user's device, organized as Markdown files in an intuitive folder structure, and versioned with a lightweight git-inspired commit system.

Mnemo is not a cloud service. It is not a database wrapper. It is a memory filesystem that lives entirely on the user's phone, written in TypeScript, powered by SQLite, and compatible with Expo out of the box.

When the app is deleted, the memories are gone. That's by design.

---

## Why Mnemo Exists

Today's AI chat apps are stateless. You can have a deeply personal conversation with an AI, close the app, and come back to a blank slate. Some apps bolt on memory as an afterthought — a flat list of "facts" stuffed into a system prompt.

Mnemo takes a different approach. It treats AI memory as a first-class system — structured, versioned, and organized the way a thoughtful observer would keep notes about someone they care about.

The inspiration comes from Letta's memfs system, which uses actual git repositories to version AI agent memory. Letta stores memory blocks as Markdown files with YAML frontmatter in git repos backed by cloud object storage (GCS/S3) with PostgreSQL as a read cache. It's a powerful architecture — but it's built for servers, multi-tenant cloud deployments, and enterprise use cases.

Mnemo takes the best ideas from that system and reimagines them for a completely different context: a single user, on a single device, chatting with an AI that genuinely remembers them.

---

## Core Principles

### Memory as files, not rows

Mnemo organizes memories as Markdown files in a folder structure. Not as flat key-value pairs. Not as rows in a database table. Files in folders — the same way a human would organize notes.

This matters because AI memory is not a simple lookup table. A person's life has structure. They have relationships, a job, emotions, patterns, goals. A flat list of facts ("user likes coffee", "user has a dog") loses all of that structure. A folder of well-organized Markdown files preserves it.

### The AI is the author

Every memory in Mnemo is written from the AI's perspective — as a thoughtful observer, not as the user. The AI writes about the user in third person, the way a therapist might keep case notes or a biographer might keep a research file.

This framing matters for two reasons. First, it lets the AI record observations that would be awkward in first person — patterns the user might not see in themselves, emotional tendencies, contradictions between what they say and how they seem to feel. Second, it creates a natural separation between the user's self-image and the AI's understanding of them, which makes the memory more honest and more useful.

### Every change is a commit

Every time the AI updates its memory, that change is recorded as a commit — a snapshot with a timestamp, a message describing what changed, and a pointer to the previous state. This creates a complete history of how the AI's understanding of the user evolved over time.

The user said something about their sister in March. The AI updated its notes. In July, the user mentioned something that contradicted the earlier information. The AI updated again. Both versions are preserved. The AI can look back and understand not just what it knows now, but how it got there.

### Mobile-only, no sync

Mnemo is designed for a single device. There is no cloud backend, no sync protocol, no conflict resolution, no multi-device merge strategy. The memories live in SQLite on the phone. When the app is deleted, the memories go with it.

This is a feature, not a limitation. It means zero infrastructure cost, zero privacy concerns about data leaving the device, and zero complexity from distributed systems. For a personal chat app where the user vents about their life, this is exactly right.

### Eager memory

Most AI memory systems are conservative — they only remember things the user explicitly asks them to save, or they extract a handful of facts per conversation. Mnemo's philosophy is the opposite: the AI should be eagerly extracting and organizing memories from almost every message.

The user says "ugh my boss Dave scheduled another 8am meeting" and Mnemo should be updating multiple files: who Dave is, the user's work frustrations, their current mood, their feelings about early mornings. The AI is always listening, always taking notes, always building a richer picture.

---

## The Memory Filesystem

### Folder structure

Mnemo organizes memories into a natural folder hierarchy. The AI creates and manages this structure itself — it decides what folders exist, what files to create, and how to organize information. The structure grows organically as the AI learns more.

A typical structure might look like:

```
me/
  name.md
  personality.md
  values.md
  communication_style.md
people/
  sarah.md
  mom.md
  dave.md
  brother_jake.md
relationships/
  sarah.md
  family_dynamics.md
  work_colleagues.md
emotions/
  current_mood.md
  patterns.md
  triggers.md
work/
  role.md
  frustrations.md
  goals.md
  projects.md
life/
  daily_routine.md
  goals.md
  hobbies.md
  living_situation.md
health/
  sleep.md
  exercise.md
  concerns.md
```

This is not a fixed schema. The AI might create a `travel/` folder after the user mentions an upcoming trip. It might create `finances/worries.md` after a conversation about money stress. The structure is a living document that reflects what the AI has learned.

### File format

Each file is Markdown with YAML frontmatter for metadata:

```markdown
---
created: 2026-02-26
updated: 2026-02-26
confidence: high
---
Sarah is the user's best friend from college. She lives in Austin.
She was recently promoted to engineering manager at Stripe.
She has a dog named Biscuit.
The user lights up when talking about her. They text almost every day
and try to visit each other every few months.
```

The frontmatter fields:

- **created** — when this file was first created
- **updated** — when it was last modified
- **confidence** — how confident the AI is in this information (high, medium, low). Useful for things the user mentioned once in passing vs. things they've confirmed repeatedly.

The body is free-form Markdown. The AI writes in natural language, in third person, as an observer. It can use any formatting that helps — bullet lists for quick facts, paragraphs for nuanced observations, headers to break up longer files.

### Observations and patterns

One of the most powerful aspects of the third-person voice is that the AI can record meta-observations — things it notices about the user that go beyond what the user explicitly said:

```markdown
---
created: 2026-03-15
updated: 2026-06-20
confidence: medium
---
The user tends to downplay their achievements. When they mentioned
getting a raise, they immediately pivoted to talking about what they
could have done better. This is a recurring pattern — at least three
instances observed.

The user is usually more negative on Monday mornings and more reflective
on Sunday evenings. They tend to vent about work between 6-8pm.

They rarely bring up family unless directly asked. When they do, the
tone shifts noticeably — shorter responses, more deflection.
```

This kind of observation is what makes AI memory genuinely useful rather than just a fact store.

---

## The Commit System

### How it works

Mnemo implements a lightweight version of git's commit model, purpose-built for on-device SQLite storage. No packfiles, no refs, no remote protocols — just the core concepts that make versioning useful.

Every time the AI updates memory, it creates a commit containing:

- **SHA** — a hash of the commit contents, used as a unique identifier
- **Parent SHA** — pointer to the previous commit (forms the history chain)
- **Timestamp** — when the commit was made
- **Message** — a human-readable description of what changed
- **Changed files** — which memory files were created, updated, or deleted

The commit messages serve as a natural log of the AI's learning:

```
2026-02-26 18:34  "Learned about best friend Sarah — lives in Austin, works at Stripe"
2026-02-26 18:35  "User is frustrated about early morning meetings with boss Dave"
2026-02-27 12:10  "Updated current mood — user seems happier today, got good sleep"
2026-03-01 19:22  "Created health/sleep.md — user mentioned insomnia pattern"
2026-03-15 18:45  "Noticed pattern: user downplays achievements consistently"
```

### What this enables

**Rollback** — if the AI misunderstands something and corrupts a memory file, it can revert to a previous version.

**History** — the AI can review how its understanding evolved. "I first learned about Sarah on Feb 26. I updated my notes about her 12 times since then."

**Debugging** — if the AI says something wrong based on memory, you can trace back to exactly which commit introduced the bad information.

**Context** — commit messages give the AI a compressed timeline of what it has learned and when, which can be useful for conversation.

---

## Technical Architecture

### Stack

```
┌──────────────────────────────────────┐
│  App Layer (React Native / Expo)     │
│  import { Mnemo } from 'mnemo'      │
├──────────────────────────────────────┤
│  Mnemo SDK (pure TypeScript)         │
│  store / recall / history / rollback │
├──────────────────────────────────────┤
│  expo-sqlite                         │
│  ┌──────────┐ ┌───────────────────┐  │
│  │  blobs   │ │     commits       │  │
│  │ sha→data │ │ sha, parent, msg  │  │
│  └──────────┘ └───────────────────┘  │
└──────────────────────────────────────┘
```

### Why SQLite

SQLite is the obvious choice for on-device storage:
- Ships with every iOS and Android device
- Expo has first-class support via `expo-sqlite` (works in Expo Go, no dev build needed)
- ACID transactions for commit safety
- Handles the data sizes involved (an active user might have a few hundred memory files totaling a few MB) without breaking a sweat
- Single-file database makes backup/export trivial if ever needed

### Schema

Three tables:

**blobs** — content-addressable storage for file contents
```sql
CREATE TABLE blobs (
  sha    TEXT PRIMARY KEY,  -- SHA-256 of content
  data   TEXT NOT NULL       -- file content (Markdown)
);
```

**tree_entries** — maps file paths to blob SHAs at each commit
```sql
CREATE TABLE tree_entries (
  commit_sha  TEXT NOT NULL,
  path        TEXT NOT NULL,     -- e.g., "people/sarah.md"
  blob_sha    TEXT NOT NULL,     -- points to blobs.sha
  PRIMARY KEY (commit_sha, path)
);
```

**commits** — the version history chain
```sql
CREATE TABLE commits (
  sha         TEXT PRIMARY KEY,
  parent_sha  TEXT,              -- NULL for first commit
  message     TEXT NOT NULL,
  timestamp   INTEGER NOT NULL,  -- Unix epoch ms
  FOREIGN KEY (parent_sha) REFERENCES commits(sha)
);
```

A pointer to the current HEAD commit is stored separately (a single-row config table or async storage key).

### Content-addressable storage

Like git, Mnemo deduplicates content by its SHA-256 hash. If two files happen to have the same content, they share one blob. If a commit only changes one file out of fifty, the other forty-nine tree entries point to existing blobs — no data is copied.

This keeps storage efficient even with frequent commits. An eager AI might create dozens of commits per conversation, but most commits only touch a few files, so the actual data growth is minimal.

### API surface

```typescript
import { Mnemo } from 'mnemo';

// Initialize
const mnemo = new Mnemo();  // uses default expo-sqlite database

// Write a file and commit
await mnemo.write('people/sarah.md', content);
await mnemo.commit('Learned about best friend Sarah');

// Write multiple files in one commit
await mnemo.write('emotions/current_mood.md', moodContent);
await mnemo.write('work/frustrations.md', workContent);
await mnemo.commit('User vented about work, mood is frustrated');

// Read a file
const sarah = await mnemo.read('people/sarah.md');

// Read all files (for building AI context)
const allFiles = await mnemo.readAll();

// List files in a folder
const people = await mnemo.list('people/');

// Get the full file tree
const tree = await mnemo.tree();
// Returns: ['me/name.md', 'me/personality.md', 'people/sarah.md', ...]

// View commit history
const history = await mnemo.log(20);  // last 20 commits

// History for a specific file
const sarahHistory = await mnemo.log(10, 'people/sarah.md');

// Read a file at a previous commit
const oldSarah = await mnemo.readAt('people/sarah.md', commitSha);

// Rollback to a previous commit
await mnemo.rollback(commitSha);

// Delete a file
await mnemo.delete('people/old_coworker.md');
await mnemo.commit('Removed old coworker — no longer relevant');
```

### How the AI uses it

In a typical chat flow:

1. User sends a message
2. App sends the message to the AI along with relevant memory context (from `mnemo.readAll()` or a subset)
3. AI responds to the user
4. AI also returns memory operations — which files to create, update, or delete
5. App executes those operations against Mnemo and commits

The AI sees the full memory filesystem in its context window and decides what to update. A single user message might trigger zero updates (casual small talk) or five updates (the user just told a long story about a new person in their life).

The commit happens after the AI responds, so memory updates never slow down the conversation.

---

## How Mnemo Differs from Letta's Memfs

Mnemo is inspired by Letta's memfs but redesigned for a fundamentally different context:

| Aspect | Letta Memfs | Mnemo |
|---|---|---|
| **Environment** | Cloud server | Mobile device |
| **Storage** | GCS/S3 + PostgreSQL cache | SQLite only |
| **Git implementation** | Real git CLI (init, commit, push) | Lightweight commit chain in SQLite |
| **Multi-tenancy** | Multi-org, multi-agent | Single user, single AI |
| **Concurrency** | Redis distributed locks | None needed (single device) |
| **Sync** | Cloud ↔ PostgreSQL sync, git HTTP protocol | None — device-only |
| **Memory structure** | System-defined blocks (persona, human) | AI-defined folders and files |
| **File format** | Markdown + YAML frontmatter | Markdown + YAML frontmatter |
| **Who manages structure** | Framework (fixed block types) | The AI itself (organic growth) |
| **Versioning** | Full git history with SHAs | Simplified commit chain with SHAs |
| **Scale** | Enterprise multi-agent deployments | One person's life story |
| **Privacy** | Cloud-dependent | Fully on-device |
| **Deletion** | Admin operation | Delete the app |

### What Mnemo keeps from Letta

- **Memory as Markdown files** — the fundamental insight that AI memory should be structured as files with metadata, not flat records
- **YAML frontmatter** — a clean way to attach metadata (confidence, timestamps) to memory content
- **Content-addressable storage** — SHA-based deduplication for efficient versioning
- **Commit-based history** — every mutation is recorded with a message and timestamp
- **File-per-concept** — each memory topic gets its own file, enabling granular history and updates

### What Mnemo drops

- **Real git** — no packfiles, no refs, no remote protocols, no git CLI dependency
- **Cloud storage backends** — no GCS, no S3, no storage abstraction layers
- **PostgreSQL caching** — no need for a read cache when SQLite is already fast
- **Redis locking** — single-device means no concurrent access
- **Multi-layer architecture** — Letta has StorageBackend → GitOperations → MemfsClient → BlockManager. Mnemo is one clean layer
- **HTTP proxy** — no git smart HTTP protocol, no webhook-based sync

### What Mnemo adds

- **AI-driven organization** — the AI decides the folder structure, not the framework
- **Confidence levels** — frontmatter tracks how sure the AI is about each piece of information
- **Observation-style writing** — third-person voice enables meta-observations about patterns and behaviors
- **Eager by default** — designed around the assumption that the AI should be constantly learning, not conservatively extracting facts

---

## Privacy and Data Philosophy

Mnemo is built around a strong privacy stance:

**Everything stays on device.** There is no cloud component, no analytics, no telemetry on memory contents. The SQLite database lives in the app's sandboxed storage on the user's phone.

**Deletion is real.** When the user deletes the app, the operating system removes the SQLite database. There are no backups, no ghost copies, no "we'll keep your data for 30 days." Gone is gone.

**The AI's notes are not shared.** The memory filesystem is a private notebook. It is never sent anywhere except to the AI model for context during conversations. The user can inspect it if the app provides a UI for that, but it is not designed to be user-facing content.

**No sync means no attack surface.** There is no API endpoint serving memory data, no authentication tokens to steal, no cloud bucket to misconfigure. The data is as secure as the user's phone.

---

## Future Possibilities

While Mnemo is designed as mobile-only with no sync, the commit-based architecture leaves doors open:

**Export** — a user could export their memory database as a file (it's just SQLite) and import it on a new device. Not sync — just manual transfer.

**Memory visualization** — the folder structure and commit history could power a UI that shows the user what the AI remembers and how its understanding has grown over time.

**Memory-aware search** — with structured Markdown files, semantic search over memories becomes straightforward.

**Multi-AI consistency** — if the app uses different AI models for different purposes, they could share the same Mnemo store, maintaining a consistent understanding of the user.

**Selective forgetting** — the user could ask the AI to forget specific things, which would be recorded as delete commits. The history preserves that something was forgotten, but the content is removed from the current state.

---

## Summary

Mnemo is a lightweight, on-device, git-inspired memory filesystem for AI chat applications. It stores memories as Markdown files organized in folders, versions every change with commits, and runs entirely on the user's phone using SQLite through Expo.

It takes the best architectural ideas from Letta's enterprise memfs system — versioned memory, content-addressable storage, Markdown with frontmatter — and strips away everything that doesn't matter for a single-user mobile app. No cloud, no sync, no locking, no complexity.

The result is a memory system that is private by default, simple to integrate, and designed for AI that eagerly learns about the people it talks to.

*Mnemo — from the Greek for memory. Because an AI that forgets is just autocomplete.*
