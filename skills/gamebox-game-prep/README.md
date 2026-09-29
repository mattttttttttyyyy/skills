# kawka.app game prep skill

An agent skill for preparing a self-contained HTML5/WebGL game as a tested ZIP for kawka.app. It covers the root `index.html` package shape, iframe-friendly runtime, optional player saves, and a practical pre-upload check.

## What it helps your agent do

- Export and package the game with `index.html` directly at the ZIP root and all required assets included.
- Fix asset paths, sandbox compatibility, responsive layout, controls, and audio startup.
- Add optional progress, settings, or high-score saves using Kawka's message bridge.
- Check the game in desktop, mobile-sized, and sandboxed browser views, then hand over the final ZIP with a summary of what was checked.

Use it for a new game, an existing project, or a build that needs fixing before upload. It gives your coding agent the packaging, runtime, and save requirements for Kawka.

## Install

From your game project directory, with Node.js/npm available, run the [Skills CLI](https://www.skills.sh/docs/cli):

```bash
npx skills add mattttttttttyyyy/skills --skill gamebox-game-prep
```

Choose your coding agent in the installer. To install only for Codex, add `--agent codex`; to make the skill available across projects, add `--global`.

## Use it

Start a new agent session after installing and ask:

> Use $gamebox-game-prep to prepare this game for kawka.app. Produce a ready-to-upload ZIP, check desktop and mobile play, and add persistent saves if the game needs them.

The agent should return the ZIP, a short controls/device-support summary, and the checks it actually performed. You can then upload the ZIP at [kawka.app/upload](https://kawka.app/upload). Publication follows the site's upload process; the skill does not guarantee acceptance.

## How saves work

Saves are optional and local to this game in the player's browser. They survive a replacement build of the same game, but do not sync between accounts, browsers, or devices and can be cleared with site data. The hosted game uses Kawka's `postMessage` bridge because direct browser storage is unavailable in its sandbox. If saving is unavailable, the game should keep working.

The complete message format and limits are documented in [`SKILL.md`](./SKILL.md).

## Update

```bash
npx skills update gamebox-game-prep
```

The skill is distributed as an open `SKILL.md` from GitHub; it does not require a Kawka-specific npm package. Skills CLI supports direct installation from a GitHub owner/repository and selection by skill name.
