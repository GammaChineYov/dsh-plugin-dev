# dsh-plugin-dev

<p align="center">
  <a href="README.md">简体中文</a> | <strong>English</strong>
</p>

An Agent Skill for developing, maintaining, and publishing DeepSeek Harness (DSH)
plugins. Tell DSH something like "build a plugin", "modify that plugin", or
"publish the plugin to GitHub", and it will follow a complete
**develop → isolated test → publish** pipeline.

## What problem does it solve

DSH plugins run inside your DSH process. Installing an experimental plugin means
injecting risk into your production environment: style pollution, memory leaks,
conflicts with other plugins, even disrupting daily development. This Skill
**fully separates "experimental validation" from the "production environment"**:

1. **Local compile-only repo**: experimental plugins are only compiled in a
   dedicated cloned repo, never installed into the production environment;
2. **Isolated-environment testing**: deploy to an isolated test server and run
   independent snapshot tests in a docker container (no pollution of the main
   environment), passing the four test categories one by one;
3. **Main-environment install check**: only after container tests pass, install
   into the server's main DSH environment and verify no conflicts with other
   plugins and correct rendering;
4. **Manual verification before installing back**: bridge via SSH tunnel for final
   manual verification, and only **after the user confirms the tests pass** is the
   plugin installed into the local test environment.

## What's inside

- **Full pipeline**: compile → deploy → container snapshot test → main-environment
  install → bridge verification → install back → archive registration;
- **Four test categories**:
  - **Overflow tests (memory)**: abnormal memory growth after loading multiple
    sessions / memory overflow on page refresh / memory overflow on first click
    after page refresh;
  - **Performance tests (lag)**: sending a command message / first click opening a
    long session / first click opening a short session / refreshing the page;
  - **Interaction closed-loop tests**: click / expand / collapse / input / switch
    full-chain closed loops;
  - **Visual-model verification**: whether UI rendering is correct, with
    misalignment / missing / pollution;
- **Development standards**: choosing between dynamic Cordis plugins vs standalone
  bundle packages, the three package-structure essentials, client bundle contract
  (`__ModuleLoader__`, inline styles, slot priority), encoding constraints (no BOM);
- **High-frequency pitfalls**: `inject` declaration, unbounded-loop deadlock,
  React Hooks order, outside-click close, data hygiene, host↔client communication,
  performance intuition;
- **Maintenance flow**: when a plugin breaks, always follow "container isolated
  test → main-environment install test → bridge test → reinstall the production
  package", never patch the local production environment directly;
- **Publishing standards**: official `publish.md`, GitHub essentials
  (LICENSE + README + package.json), fork / Discussion approach since the official
  repo does not accept external PRs.

## Installation

Send this to DSH:

```text
Please install the dsh-plugin-dev skill from https://github.com/GammaChineYov/dsh-plugin-dev
```

For manual install, copy the whole `skills/dsh-plugin-dev/` directory to
`$DSH_HOME/skills/` (default `~/.dsh/skills/`); for project-only use, copy to
`<project root>/.dsh/skills/`. To share with other agents, you can also put it in
`<project root>/.agents/skills/`. The directory watcher makes it effective
immediately, no restart needed.

## Skill / memory separation

This Skill only carries **executable process** (what to do, how to decide). All
**machine-specific facts** (compile repo path, test server address, SSH keys, each
plugin's repo and install state) should live in your own memory/archive area. The
Skill's memory pointers **only give the archive's Chinese display name, never a
specific path** — paths are maintained by your own memory index, and you can find
the matching archive by name. Facts change only in the archive, the Skill never
drifts, and the same Skill is reusable across machines.

## Dependencies

- DeepSeek Harness (DSH); dynamic plugin development needs the
  `cordis-plugin-development` skill, and composition authoring needs the
  `editing-cordis-compositions` skill (both ship with DSH);
- An isolated test server capable of deploying DSH (docker recommended but
  optional, for clean snapshot tests).

License: [MIT](LICENSE)
