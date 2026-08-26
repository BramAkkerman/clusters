# Claude memory (portable across machines)

This directory holds Claude Code's auto-memory files for the `clusters`
project - feedback, project facts, and reference notes it's accumulated
across sessions (see each file's own frontmatter for its type).

Normally these live at a path derived from the working directory
(`~/.claude/projects/-Users-bram-personal-clusters/memory/`), which isn't
portable - a clone of this repo on another machine wouldn't have them.

**On any new machine, after cloning this repo**, symlink the real path to
this directory so Claude Code picks it up automatically:

```bash
ln -s /path/to/this/clone/.claude-memory ~/.claude/projects/-Users-bram-personal-clusters/memory
```

(Adjust the project slug if the clone lives at a different absolute path -
Claude Code derives it from the working directory path itself, so an
identical clone path keeps this simple.)

Memory writes during a session land here (via the symlink) but only reach
other machines once committed and pushed like any other change in this
repo - there's no automatic sync.
