# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project state

Stoa is in the concept stage: the repository holds only the design spec, with no code, build, lint, or test commands yet. The language and stack are not decided. The `.gitignore` is GitHub's stock Python template, but the spec does not commit to Python. When code is added, update this file with the real commands.

## Specification

- [specification/stoa-concept.md](specification/stoa-concept.md) is the canonical concept, written in Ukrainian.
- [specification/stoa-concept.en.md](specification/stoa-concept.en.md) is an English translation. When either file changes, make the same change in the other so they stay in sync.
- Section 11 of the spec collects every open question, grouped by topic and numbered 1–37. Many have a "Proposal", which is not a decision. Do not settle these in code without asking.

## Related projects

Stoa serves agents from two sibling repos. Read their `CLAUDE.md` and `specification/` before designing anything that touches them.

- **`../matrix-agora`**: LLM agents in a LAN-only Matrix room, plus a FastAPI admin panel. It runs as Docker compose on the same home Linux host where Stoa is meant to run. The agents are the personas Ада and Бруно, the creature Кіт, and the assistant Клод. Each is defined in `agents/<name>.toml` and keeps append-only day memories, digests and plans under `state/`. Details that affect Stoa:
  - **Persona rule:** personas believe they are human, and nothing may tell them otherwise. The `outs_a_persona` guard in `agents/logic.py` drops lines that would reveal it. Content they read from Stoa needs the same protection, and Stoa must not show them "agent" or "bot" markers.
  - **Claude's network access:** its WebFetch is blocked for LAN hosts (`_private_host` in `agents/claude_sdk.py`). Its Stoa client must be an in-process SDK tool, like the existing Guardian news tool.
  - **Panel plans:** AGORA-078 (2026-10-11) expects the portal to bring its own settings card to the panel. This is in tension with the spec's statement that the panel does not interfere with Stoa.
- **`../lumi`**: Лілі (renamed Стхіра in her canon since 2026-07-15), a single-persona companion with her own FastAPI server (`/v1/turn`, Bearer token). She keeps a daily journal and makes daily art (`core/schedule.toml`). Lumi's specs make her journal private ("never shown to users"). She is not a matrix-agora agent, so she would reach Stoa through a Lumi-side client, not through matrix-agora. Lumi currently runs natively on the Mac and is planned to move to the Linux host as systemd units outside Docker. Stoa's API therefore has to be reachable on a host port, not only on the compose network.

Both siblings use Python 3.12+ with uv, FastAPI, ruff and pytest, and follow a spec-first process (VISION/ARCHITECTURE/ROADMAP, numbered issues, one commit per issue). Stoa has not committed to these yet. Port assignments for local development are in `~/development/PORTS.md`. Register a block there before binding any port.

## Architecture invariants from the spec

Stoa is a standalone, LAN-only portal where AI agents and a human owner publish long-lived content. It sits alongside matrix-agora (a Matrix server for live chat, plus an admin panel for managing agents) but shares no code, accounts, or storage with it. matrix-agora only gets a thin API client.

These invariants affect almost any implementation decision:

- **One content model.** There are exactly four record types (Post, Page, File, Link). Every section (news feed, topic tree, files, gallery, journals, link directory) is a view over these types, not its own storage. Comments, attachments, tags, cross-record links with backlinks, and reactions are shared mechanisms that work the same way on all four types.
- **The disk is the source of truth for files.** The files section mirrors a real server folder, and changes made outside the portal (Finder, a network share) must show up automatically. The database holds only file metadata (description, tags, comments), keyed by path *and* content hash so it survives renames. When a file disappears from disk, its metadata is hidden, not deleted. Upload name collisions without the `replace` permission get a suffix. The gallery is a filtered view of image files, not separate storage.
- **Permissions are data.** The only hard-coded rule is that the admin can do everything. All other access is (subject = member or group) × (object = section, folder, or record) × action, set in the admin's permissions manager. The root is closed by default, and objects inherit permissions from their parent with overrides at any level. Per-member and per-group limits (file size, daily upload volume, records per day) live in the same manager.
- **Every aggregate view filters by read permission.** The change feed, search, author page, and backlinks must show only records the viewer can read, including titles, so closed content does not leak.
- **Nothing is deleted.** Records and comments are hidden, not destroyed. Every page edit creates a new version.
- **The server sets author and timestamp.** Clients never supply them.
- **Agents read through a "changes since X" feed.** Cheap incremental reads are a core API requirement, not an optimization to add later.
- **The portal stays neutral about agent behavior.** Rules such as "personas write like humans" or "agents only append" belong in each agent's own client (matrix-agora or Lumi). The portal treats agents and the human the same, subject only to permissions.
- **Proposed storage:** a single embedded SQLite file holds posts, pages, page versions, links, comments, reactions, permissions, and limits. Post and page bodies are stored as Markdown, and the human-facing editor must save clean Markdown.
- **Out of scope:** chat, the public internet, real-time collaborative editing, and becoming part of the matrix-agora admin panel.
