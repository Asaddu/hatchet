# Asaddu Team Development Workflow for Hatchet Fork

## Overview

This document explains how the Asaddu team works on the Hatchet fork with BMAD Method tools while contributing clean code upstream.

## Repository Structure

- **Upstream**: `https://github.com/hatchet-dev/hatchet` (original project)
- **Our Fork**: `git@github.com:Asaddu/hatchet.git` (our team repository)
- **Main Branch**: `main` - Tracks upstream, stays clean (NO BMAD files)
- **Team Branch**: `asaddu-dev` - Contains BMAD tools and team documentation

## Initial Setup (One-time per developer)

### 1. Clone the Asaddu fork with team branch
```bash
# Clone directly to the team development branch
git clone -b asaddu-dev git@github.com:Asaddu/hatchet.git
cd hatchet

# Add upstream remote for syncing
git remote add upstream https://github.com/hatchet-dev/hatchet.git
```

### 2. Verify remotes
```bash
git remote -v
# Should show:
# origin    git@github.com:Asaddu/hatchet.git (fetch)
# origin    git@github.com:Asaddu/hatchet.git (push)
# upstream  https://github.com/hatchet-dev/hatchet.git (fetch)
# upstream  https://github.com/hatchet-dev/hatchet.git (push)
```

## Branch Strategy

### `main` branch
- **Purpose**: Clean branch for upstream contributions
- **Contains**: Only Hatchet code, no BMAD files
- **Used for**: Creating PRs to upstream Hatchet

### `asaddu-dev` branch
- **Purpose**: Team development with BMAD Method
- **Contains**: All Hatchet code + BMAD tools + team docs
- **Includes**:
  - `.bmad-core/` - BMAD core framework
  - `.bmad-infrastructure-devops/` - Infrastructure tools
  - `.claude/` - Claude Code configurations
  - `web-bundles/` - BMAD web agent bundles
  - `ASADDU-TEAM-WORKFLOW.md` - This documentation
  - Project planning docs from BMAD workflow

## Workflows

### A. Daily Development (Using BMAD)

```bash
# Always work on asaddu-dev for regular development
git checkout asaddu-dev
git pull origin asaddu-dev

# Use BMAD Method for planning
# Create your PRDs, architecture docs, etc.
# Work with the AI agents

# Commit team work
git add .
git commit -m "feat: implement payment processing"
git push origin asaddu-dev
```

### B. Contributing to Upstream (e.g., Rust SDK)

```bash
# 1. Sync main with upstream
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

# 2. Create feature branch from clean main
git checkout -b feature/rust-sdk

# 3. Develop the Rust SDK
# Work in /sdks/rust/
# Follow Hatchet's patterns from other SDKs

# 4. Push to our fork
git push origin feature/rust-sdk

# 5. Create PR from GitHub
# FROM: Asaddu/hatchet:feature/rust-sdk
# TO: hatchet-dev/hatchet:main
```

### C. Syncing Team Branch with Upstream Changes

```bash
# Periodically sync asaddu-dev with upstream
git checkout main
git fetch upstream
git merge upstream/main
git push origin main

git checkout asaddu-dev
git merge main
# Resolve any conflicts, keeping BMAD files
git push origin asaddu-dev
```

## Rust SDK Development Plan

### 1. Research Phase (on asaddu-dev)
```bash
# Use BMAD to document existing SDKs
@architect *document-project
# Focus on /sdks/ directory structure
```

### 2. Planning Phase (on asaddu-dev)
```bash
# Create PRD for Rust SDK
@pm *create-brownfield-epic
# Title: "Add Rust SDK for Hatchet"
```

### 3. Implementation Phase (on feature branch)
```bash
# Switch to clean branch
git checkout main
git pull upstream main
git checkout -b feature/rust-sdk

# Create SDK structure
mkdir -p sdks/rust/src
# Implement following patterns from Go/Python/TypeScript SDKs
```

### 4. Testing Phase
Follow Hatchet's CONTRIBUTING.md:
- Set up local environment
- Run `task start-db` and `task setup`
- Test Rust SDK against local instance

### 5. Contribution Phase
- Push to `origin feature/rust-sdk`
- Create PR to upstream
- PR will NOT contain any BMAD files

## File Structure Example

```
hatchet/
├── .bmad-core/                    # Only in asaddu-dev branch
├── .bmad-infrastructure-devops/   # Only in asaddu-dev branch
├── .claude/                       # Only in asaddu-dev branch
├── web-bundles/                   # Only in asaddu-dev branch
├── ASADDU-TEAM-WORKFLOW.md        # Only in asaddu-dev branch
├── docs/
│   ├── prd.md                     # Team planning docs (asaddu-dev only)
│   ├── architecture.md            # Team planning docs (asaddu-dev only)
├── sdks/
│   ├── go/
│   ├── python/
│   ├── typescript/
│   └── rust/                      # New SDK - will go upstream
│       ├── Cargo.toml
│       ├── src/
│       └── examples/
└── [rest of Hatchet files]
```

## Important Notes

1. **Never merge asaddu-dev into main** - This would contaminate the clean branch
2. **Always merge main into asaddu-dev** - This brings upstream changes to team branch
3. **Create feature branches from main** for upstream contributions
4. **Create feature branches from asaddu-dev** for internal features

## Quick Reference Commands

```bash
# Start team work
git checkout asaddu-dev
git pull origin asaddu-dev

# Start upstream contribution
git checkout main
git pull upstream main
git checkout -b feature/new-feature

# Sync with upstream
git fetch upstream
git checkout main
git merge upstream/main
git push origin main
git checkout asaddu-dev
git merge main

# Check which branch you're on
git branch --show-current

# See all branches
git branch -a
```

## BMAD Method Integration

When working on `asaddu-dev`:
1. Use BMAD agents for planning and development
2. Follow the workflows in `.bmad-core/user-guide.md`
3. Store planning artifacts in `docs/`
4. Commit all BMAD-generated documentation

When working on feature branches for upstream:
1. Focus only on the specific feature (e.g., Rust SDK)
2. Follow Hatchet's existing patterns
3. Don't reference BMAD tools in code or comments
4. Ensure all tests pass per CONTRIBUTING.md

## Team Collaboration

- All team members work on `asaddu-dev` by default
- Share BMAD planning docs through commits to `asaddu-dev`
- Coordinate upstream contributions through GitHub issues
- Use PR reviews on our fork before submitting upstream

## Questions?

- Reach out in team Slack/Discord
- Check `.bmad-core/user-guide.md` for BMAD Method details
- Refer to Hatchet's CONTRIBUTING.md for upstream requirements