# Curated Agent Files

This is a curated collection of agent files extracted from the [agents repository](https://github.com/pelan05/agents) `/plugins` folder.

## What This Contains

This repository contains a selected list of agent files (`.agent.md` files) synchronized from the main repository's plugin structure. Only agents that exist in this repository are updated when the sync script runs.

## Source

These files are synchronized from the main repository at: `git@github.com:pelan05/agents.git`

The agents are organized in the main repository under `/plugins/<category>/<plugin-name>/agents/` and selected ones are kept in sync here.

## Synchronization Behavior

- **Updates only**: Only agent files that already exist in this repository are updated
- **No deletions**: Existing agents are never deleted, even if removed from the source
- **Selective sync**: The sync script matches `.agent.md` files here with `.md` files in the source repository
- **Manual curation**: New agents must be manually added to this repository before they can be synced

## Usage

Simply reference any agent file from this repository in your VS Code or Claude Code workspace. The files are kept in sync with the source repository using the sync script.

## Maintenance

This repository is updated using the `sync-agents.sh` script in the parent repository. The script:
1. Scans this repository for existing `.agent.md` files
2. Looks for matching `.md` files in the parent repository's `/plugins` folder
3. Updates only the files that have changes
4. Preserves any files that don't have a match in the source