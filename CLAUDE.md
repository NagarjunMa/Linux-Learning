# Linux Learning Repository — Claude Context

## Purpose
Linux systems knowledge base and HRT interview preparation for a software engineer targeting the HRT SysTrade (Systems + Trading Infrastructure) team.

Goals:
- Deep Linux internals for interview and on-the-job use (not surface-level commands)
- Crisis scenario practice for the HRT phone screen and SSH debugging round
- Quick retrieval: jump directly to any subtopic during revision without remembering file locations

## Repository Structure
- Each top-level folder = one topic area
- Each numbered `.md` file inside = one subtopic in learning order
  - Lower numbers = foundational concepts
  - Higher numbers = applied / interview-specific content
- `README.md` — AUTO-GENERATED. Never edit by hand.
- `CHANGELOG.md` — AUTO-GENERATED. Never edit by hand.
- `scripts/update_readme.py` — regenerates README + CHANGELOG from current files

## Existing Topics
- `Linux-Kernel/` — 9 files covering: kernel mental model, process/threads, virtual memory, scheduler, syscalls, networking, I/O, interrupts, trading application tuning
- `HRT-Interview-Prep/` — 6 files covering: HRT role & interview process, command reference, 12 crisis scenarios, low-latency Q&A, Python/Ansible/Salt/Prometheus, culture & phone screen strategy

## How to Add a New Topic
1. Create a new folder: `mkdir TopicName/`
2. Add numbered files starting from `01-Overview.md`
3. First line of each file must be `# Title` (used as the subtopic link in README)
4. Run: `python3 scripts/update_readme.py`
5. Commit: `git add . && git commit -m "feat: add [Topic] notes"`

## How to Add a Subtopic
1. Add a numbered file to an existing folder
2. Run: `python3 scripts/update_readme.py`
3. Commit

## Naming Convention
- Folder: `PascalCase` or `Hyphenated-Name`
- Files: `NN-Descriptive-Name.md` (zero-padded, e.g. `01`, `02`, `10`, `11`)

## Note Format
- `#` heading = title (required, first line)
- `##` subheadings = sections (shown as "Covers" preview in README)
- Code blocks for all commands — these are revision notes, not documentation
- Write for yourself: dense, command-heavy, no hand-holding prose

## What NOT to Do
- Do NOT edit `README.md` or `CHANGELOG.md` directly
- Do NOT create `.md` files in the repo root

## HRT Interview Context
The HRT SysTrade role is "Systems Automation / Python" — not pure sysadmin.
The phone screen tests: Linux commands (HRT-level depth), troubleshooting scenarios, resume deep-dive.
The SSH debugging round: live access to a broken server — systematic triage matters more than memorized commands.
Key mindset: identify the kernel subsystem first, then pick the right tool for that layer.
