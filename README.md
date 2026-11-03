# Rubydai SwiftUI bonus skill

A [Claude Code](https://claude.com/claude-code) skill for any SwiftUI / iOS project. It collects the "builds fine, still breaks at runtime" lessons so Claude applies them automatically when it works in a SwiftUI project, without being asked.

The skill installs as `rubydai-swift-ui-bonus-skill`.

## Contents

| File | What's in it |
|---|---|
| `SKILL.md` | What Claude reads: when to use the skill, the always-on rules, and which reference to open when |
| `references/silent-bugs-checklist.md` | Silent-bug checklist: the environment, Info.plist permissions, operations that can crash, `String(format:)`, `ForEach` ids, double taps, swallowed errors, view lifecycle, audio, concurrency, navigation, locale, launch timing, **first-use latency (with how to measure it)**, idle heat and battery drain, manual tests before a release |
| `references/api-caching-guide.md` | Caching for backend APIs: cache layers, key design, what must never be cached, the Cloudflare Cache API, KV / R2 / Redis, client-side caching (RAM, disk, `URLCache`, prefetching), abuse prevention, measuring |
| `setup.command` | Installs the skill on a new Mac with a double-click |

Both references stand on their own and are useful without Claude.

## Install on a new Mac

User-level skills live in `~/.claude/skills/<skill-name>/` and apply to every project.

**Option 1: double-click**

1. On the repo page on GitHub, choose **Code → Download ZIP** and unzip it.
2. Right-click `setup.command` → **Open**. The first time, macOS blocks files downloaded from the internet, so use right-click rather than a double-click.

The script:
- checks for git and opens the Command Line Tools installer if it's missing (double-click the script again once it's done);
- clones the skill into `~/Desktop/Swift UI projects/rubydai-swift-ui-bonus-skill`, or pulls if it's already there;
- links it into `~/.claude/skills/rubydai-swift-ui-bonus-skill`;
- adds the "always use this skill" line to `~/.claude/CLAUDE.md` if it's not there yet.

It's safe to run again at any time; anything already in place is skipped. The list of skills and the clone folder are at the top of the file.

**Option 2: clone directly into the skills folder**

```sh
mkdir -p ~/.claude/skills
git clone https://github.com/rubydai-dev/Rubydai-swift-ui-bonus-skill.git ~/.claude/skills/rubydai-swift-ui-bonus-skill
```

Update later with `git -C ~/.claude/skills/rubydai-swift-ui-bonus-skill pull`.

**Option 3: clone elsewhere and symlink** (if you want to edit the skill inside your own projects folder)

```sh
git clone https://github.com/rubydai-dev/Rubydai-swift-ui-bonus-skill.git "<your-folder>/rubydai-swift-ui-bonus-skill"
mkdir -p ~/.claude/skills
ln -s "<your-folder>/rubydai-swift-ui-bonus-skill" ~/.claude/skills/rubydai-swift-ui-bonus-skill
```

Start a new Claude Code session, type `/`, and check that `rubydai-swift-ui-bonus-skill` is in the list of skills.

### Making Claude always use the skill in SwiftUI projects

Claude uses the skill on its own when the task matches the `description` in `SKILL.md`, but not reliably every time. To make sure, add this line to `~/.claude/CLAUDE.md` (`setup.command` does it for you):

```md
- In every SwiftUI / Xcode project, always use the `rubydai-swift-ui-bonus-skill` skill.
```

## Updating the content

- Edit the files in this repo directly, then commit and push.
- If a project has a newer `docs/silent-bugs-checklist.md` or `docs/api-caching-guide.md`, copy it into this repo's `references/`.
- Keep it **generic**: no class names, paths or features specific to one project.
- Write everything in English.
- The `description` in `SKILL.md` decides when Claude uses the skill. Keep it under 1024 characters and list the situations that should trigger it explicitly.
- The skill's folder name and the `name` in `SKILL.md` must stay `rubydai-swift-ui-bonus-skill`: lowercase letters, digits and hyphens only.
