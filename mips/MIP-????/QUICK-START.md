# MPS Quick Start Guide

Get started creating a Midnight Problem Statement in 5 minutes.

## What is an MPS?

A **Midnight Problem Statement (MPS)** is a formal document that describes a challenge in the Midnight ecosystem **before** proposing solutions. Think of it as a detailed bug report or feature request that helps the community understand and solve ecosystem-wide problems.

## When to Write an MPS

**Write an MPS when:**

- You've identified a technical limitation affecting multiple users
- Current tools/approaches are insufficient for a use case
- A gap exists in the ecosystem
- Performance, privacy, or usability issues need solving
- You want to solicit solution proposals from the community

**Don't write an MPS when:**

- You have a specific solution in mind (write a MIP instead)
- Problem is specific to your deployment (file an issue)
- Problem is already documented in another MPS
- You're reporting a bug (use GitHub issues)

## 5-Minute MPS Creation

### Step 1: Create Directory (30 seconds)

```bash
mkdir -p MPS-????
cd MPS-????
```

### Step 2: Copy Template (30 seconds)

```bash
cp ../MPS-TEMPLATE.md README.md
```

### Step 3: Fill in Metadata (2 minutes)

Open `README.md` and update the YAML header:

```yaml
---
MPS: ?                           # Leave as ? until assigned
Title: Your Problem Title        # Short, descriptive
Category: Indexer                # Pick one: Node, Ledger, Indexer, SDK, Wallet, etc.
Status: Open                     # Usually "Open" for new MPSs
Authors:
  - Your Name <you@example.com>  # Your contact info
Proposed Solutions: []           # Empty for new problems
Discussions:
  - https://github.com/midnight/discussions/XXX  # Link to discussion
Created: 2026-03-08             # Today's date (YYYY-MM-DD)
License: CC-BY-4.0              # Standard documentation license
---
```

### Step 4: Write Core Sections (2 minutes)

Focus on these three sections first:

**Abstract** (2-3 sentences):
```markdown
## Abstract

[What is the goal?] [What technical obstacle prevents it?]
```

**Problem** (1 paragraph):
```markdown
## Problem

Current state: [What exists today]
Limitation: [What doesn't work]
Impact: [Why it matters]
```

**One Use Case** (structured example):
```markdown
## Use Cases

### UC1: [User Scenario Title]

**Scenario**: [Concrete example of what user tries to do]

**Current Approach**: [How they try to solve it today]

**Limitations**: [Why current approach fails]

**Desired Outcome**: [What success looks like]
```

### Step 5: Add Goals & Questions (1 minute)

**Goals** (3-5 items):
```markdown
## Goals

1. [Most important objective with measurable criteria]
2. [Second priority]
3. [Additional goal]

### Non-Goals
- [What's explicitly out of scope]
```

**Open Questions** (3-5 items):
```markdown
## Open Questions

1. [Question about trade-offs or design approach]
2. [Question about edge cases]
3. [Question about compatibility]
```

## Example: Minimal MPS

Here's a complete minimal MPS (can be written in 5 minutes):

```markdown
---
MPS: ?
Title: Wallet Sync Performance for Mobile Devices
Category: Indexer
Status: Open
Authors:
  - Alice Developer <alice@example.com>
Proposed Solutions: []
Discussions:
  - https://discord.com/channels/midnight/discussion-thread
Created: 2026-03-08
License: CC-BY-4.0
---

## Abstract

Mobile wallets need to synchronize shielded transaction history to display balances. Current indexer queries require scanning all commitments, resulting in 10+ minute sync times and excessive mobile data usage. This prevents practical mobile wallet adoption.

## Problem

Midnight wallets must scan blockchain commitments to identify relevant transactions through trial decryption. The indexer provides commitment data via GraphQL, but queries return all commitments in a block range. For a wallet inactive for 1 month, this means downloading and processing 432,000 blocks with ~4.3M commitments, consuming 8-12 minutes and 50+ MB of mobile data.

Mobile users expect sync times under 10 seconds and minimal data transfer. Current approach is unusable on mobile networks.

## Use Cases

### UC1: User Opens Mobile Wallet After 1 Week

**Scenario**: User installed Midnight mobile wallet, received payments, then didn't open app for 1 week. They open app to check balance.

**Current Approach**: Wallet queries indexer for all commitments in last 100,800 blocks, downloads 12MB of data, performs trial decryption locally, taking 2-3 minutes.

**Limitations**: User waits 2-3 minutes staring at loading spinner. On metered connection, consumes significant data quota. Battery drain from crypto operations.

**Desired Outcome**: Balance appears within 5 seconds with <1MB data transfer.

## Goals

1. Wallet sync completes in <10 seconds for 1 week of missed blocks
2. Data transfer <1MB for typical sync operation
3. Maintain privacy (no leakage of wallet ownership)
4. Backward compatible with existing wallet SDK

### Non-Goals
- Light client consensus verification (separate concern)
- Historical transaction search (focus on recent sync)

## Open Questions

1. Can we index commitments by encrypted metadata without privacy leakage?
2. What trade-offs exist between query privacy and performance?
3. Should wallets maintain local checkpoint state for incremental sync?
4. How do we handle chain reorganizations during sync?

## Copyright

This MPS is licensed under CC-BY-4.0.
```

## Next Steps After Writing

### 1. Get Feedback (Before Submitting)

Post your draft to community channels:

- **GitHub Discussions**: For technical discussion
- **Discord #development**: For quick feedback
- **Community Calls**: Present for synchronous discussion

Iterate based on feedback. Add more use cases, refine goals, expand open questions.

### 2. Submit PR

When ready for formal review:

```bash
git add MPS-????/
git commit -m "feat: add MPS-???? for [problem title]"
git push origin your-branch-name
gh pr create --title "MPS-????: [Your Problem Title]"
```

### 3. Review Process

- Maintainers review for clarity and completeness
- Community provides feedback
- You iterate based on comments
- Once approved, MPS number is assigned and merged

### 4. After Merge

- Your MPS gets a permanent number (e.g., `MPS-0042`)
- Community can propose MIPs (solution proposals) referencing your MPS
- You can also propose your own MIP to solve it
- MPS status changes to "Solved" when a complete solution is implemented

## Tips for Success

### Make It Concrete

❌ "Wallets are slow"
✅ "Wallet sync takes 10 minutes for 1 month of inactivity"

### Show Real Impact

❌ "This would be nice to have"
✅ "Mobile users uninstall app due to sync times"

### Ask Hard Questions

❌ "How do we make it faster?"
✅ "What privacy guarantees can we maintain while enabling prefix queries?"

### Define Success

❌ "Better performance"
✅ "Sync time <10 seconds for 90th percentile of users"

### Stay Focused

✅ One clear problem with concrete scope
❌ Multiple loosely related problems in one MPS

## Common Pitfalls

### ❌ Solution Disguised as Problem

Don't write: "We need to implement Bloom filters for commitment indexing"

✅ Instead: "Commitment queries are too slow for mobile wallets (10min sync time)"

### ❌ Too Vague

Don't write: "Midnight needs better indexing"

✅ Instead: "Wallet sync for 1 month inactive wallet takes 10 minutes on mobile"

### ❌ Too Specific

Don't write: "My deployment needs custom indexer API for my specific use case"

✅ Instead: "DApps need real-time notification of relevant transactions"

### ❌ Missing Use Cases

Don't write only abstract problem description.

✅ Always include 2-3 concrete scenarios with Current/Limitations/Desired

## Resources

- **Full Template**: See `MPS-TEMPLATE.md` for detailed guidance
- **Checklist**: Use `SUBMISSION-CHECKLIST.md` before submitting
- **Process**: Read `README.md` for full MPS process

---

**Ready to start?** Copy `MPS-TEMPLATE.md` and begin writing!

## Attribution

This MPS process is derived from [CIP-9999: Cardano Problem Statements](https://cips.cardano.org/cip/CIP-9999), licensed under CC-BY-4.0. Adapted for the Midnight ecosystem.

## License

This document is licensed under CC-BY-4.0 (Creative Commons Attribution 4.0 International).

**Version**: 1.0.0
**Last Updated**: 2026-03-09
