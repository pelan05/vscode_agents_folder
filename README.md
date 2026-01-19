# Curated Agent Files & Skills

This is a curated collection of agent files and skill modules extracted from the [agents repository](https://github.com/pelan05/agents) `/plugins` folder.

## What This Contains

### 📁 Agents Folder
- **Active agents**: Curated list of actively maintained agent files in the `/agents` folder
- **Disabled agents**: Additional agents available in the `/disabled` folder for optional activation
- Only agents that exist in the `/agents` folder are updated when the sync script runs

### 🎯 Skills Folder
- **Specialized knowledge modules**: 120+ skill files providing domain-specific expertise
- Skills are extracted from `/plugins/*/skills/SKILL.md` and renamed to `<folder-name>.skill.md`
- Examples: `stripe-integration.skill.md`, `kubernetes-manifest.skill.md`, `python-async-patterns.skill.md`
- All skills are automatically synchronized and updated from the source repository

## Source

These files are synchronized from the main repository at: `git@github.com:pelan05/agents.git`

The agents are organized in the main repository under `/plugins/<category>/<plugin-name>/agents/` and selected ones are kept in sync here.

## Synchronization Behavior

### Active Agents (`/agents`)
- **Updates only**: Only agent files that already exist in the `/agents` folder are updated
- **No deletions**: Existing agents are never deleted, even if removed from the source
- **Selective sync**: The sync script matches `.agent.md` files here with `.md` files in the source repository
- **Manual curation**: New agents must be manually moved from `/disabled` to `/agents` for active maintenance

### Disabled Agents (`/disabled`)
- **Auto-discovery**: All new agents from the source repository are automatically added here
- **Ready for activation**: Move any agent from `/disabled` to `/agents` to include it in regular updates
- **Model names updated**: All agents have updated model references (e.g., `Claude Sonnet 4.5`)

### Skills (`/skills`)
- **Automatic sync**: All SKILL.md files are automatically discovered and synchronized
- **Renamed for clarity**: Files are renamed from `SKILL.md` to `<folder-name>.skill.md`
- **Always up-to-date**: Skills are updated every time the sync script runs

## Usage

### Agents
Reference any agent file from the `/agents` or `/disabled` folders in your VS Code or Claude Code workspace.

**Activating a disabled agent:**
```bash
# Move an agent from disabled to agents folder
mv disabled/my-agent.agent.md agents/

# Next sync will keep it updated
```

### Skills
Reference skill files to provide specialized domain knowledge to your agents. Skills work alongside agents to provide deeper expertise in specific areas.

```markdown
# In your agent file
skills:
  - stripe-integration
  - python-async-patterns
```

### Copying over to repo .github folder
Copy the contents of this repository to your local .github folder using the command below:

```bash
curl -L https://github.com/pelan05/vscode_agents_folder/archive/refs/heads/main.tar.gz | tar -xz --strip-components=1 && \
rm -f .gitignore README.md
```

## Maintenance

This repository is updated using the `sync-agents.sh` script in the parent repository. The script:

1. **Updates active agents**: Scans `/agents` folder and updates matching files from source
2. **Discovers new agents**: Finds all agents in source and adds new ones to `/disabled` folder
3. **Syncs skills**: Extracts all SKILL.md files and adds/updates them in `/skills` folder
4. **Updates model names**: Converts model references to full names (e.g., `Claude Sonnet 4.5`)
5. **Includes personal additions**: Syncs custom agents from `/personal_additions` folder

**To run the sync:**
```bash
./sync-agents.sh
```

The script provides detailed output showing what was updated, added, or skipped.
