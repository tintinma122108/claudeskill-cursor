# Claude Skills/cursor小板正

A collection of reusable Claude Code skills for frontend developers and designers.
根据前端实现来回切回cursor描述修改的对象及修改的要点，达咩！
打开cursor，看见页面，哪里想改点哪里，让你的页面还原板板正正的。
安装了这个skill，点击任何想要修改的部分，直接带回到cursor。

## What is a Claude Skill?

A skill is a `.md` file that tells Claude Code how to perform a specific task. Once installed, you invoke it with `/skill-name` in any Claude Code conversation.

## Installation

### Install a single skill

```bash
curl -o ~/.claude/commands/setup-inspector.md \
  https://raw.githubusercontent.com/tintinma122108/claudeskill-cursor/main/setup-inspector.md
```

Replace `YOUR_USERNAME` with your GitHub username, and `setup-inspector` with the skill you want.

### Install all skills

```bash
mkdir -p ~/.claude/commands && \
  curl -s https://api.github.com/repos/tintinma122108/claudeskill-cursor/contents \
  | grep '"name"' | grep '\.md' | grep -v README \
  | sed 's/.*"name": "\(.*\)".*/\1/' \
  | xargs -I{} curl -s -o ~/.claude/commands/{} \
    https://raw.githubusercontent.com/tintinma122108/claudeskill-cursor/main/{}
```

---

## Skills

### `/setup-inspector`

Configure **click-to-editor** for any React project: click a UI element in the browser → Cursor automatically opens the corresponding component file.

**What it does:**
- Auto-detects your framework and bundler (React + Vite, Next.js, CRA, Vue + Vite)
- Installs the necessary packages
- Configures your dev server and entry file
- Runs a 5-step verification flow to catch common issues
- Includes a bug case library from real-world troubleshooting

**Usage:** Run `/setup-inspector` in any frontend project.

**Install:**
```bash
curl -o ~/.claude/commands/setup-inspector.md \
  https://raw.githubusercontent.com/tintinma122108/claudeskill-cursor/main/setup-inspector.md
```

---

## Contributing

Found a bug or added support for a new framework? PRs are welcome.

When submitting a fix, please also update the **Bug Case Library** section inside the relevant skill file so the knowledge is preserved for everyone.
