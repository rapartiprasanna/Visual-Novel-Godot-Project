# Visual Novel Godot Project

GDScript project manifest and AI context for the Cultivation Game (Godot 4).

See [system-prompt.md](system-prompt.md) for the full project manifest, architecture, and combat system rules.

**Session docs (codebase-specific, not in manifest):**
- [architecture.md](architecture.md) — Godot project layout, patterns, wired vs stub
- [implementation-plan.md](implementation-plan.md) — completed work, phased roadmap, next tasks

The playable Godot project lives separately at `~/Documents/GodotGames/cultivation-game-1`.

## GitHub setup

Local git is initialized and the initial commit is ready. To publish:

```bash
gh auth login
cd ~/visual-novel-godot-project
gh repo create "Visual-Novel-Godot-Project" --public --source=. --remote=origin --push \
  --description "GDScript manifest and AI context for Cultivation Game (Godot 4)"
```
