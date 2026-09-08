# Bubblewrap Sandbox for AI Coding Agents

A small Bubblewrap-based sandbox for running AI coding agents such as Claude Code and Codex with access to the **current project only**, instead of exposing the entire home directory.

The main goal is simple:

* the current project is available read-write;
* the real home directory is hidden;
* the agent gets a separate persistent home directory;
* system binaries and libraries are available read-only;
* selected development tools can be exposed explicitly;
* network access remains available.

This is intended as a practical safety layer for local AI coding agents, not as a replacement for a VM or container runtime.

## How it works

Suppose you run the sandbox from:

```text
/home/user/dev/my-project
```

Inside the sandbox, the agent still sees:

```text
/home/user/dev/my-project
```

but `/home/user` is not your real home directory.

Instead, it is backed by:

```text
~/.local/share/agent-sandbox/home
```

The current project is then mounted back on top of this fake home:

```text
real ~/.local/share/agent-sandbox/home -> sandbox /home/user

real ~/dev/my-project -> sandbox /home/user/dev/my-project
```

As a result, the agent can work normally in the current project but cannot access sibling projects or sensitive directories such as:

```text
~/.ssh
~/.aws
~/Documents
~/dev/other-project
```

unless they are explicitly mounted.

## 1. Install Bubblewrap

### Ubuntu

```bash
sudo apt update
sudo apt install bubblewrap

bwrap --version
```


### Fedora

```bash
sudo dnf install bubblewrap

bwrap --version
```


## 2. Create the sandbox directory

```bash
mkdir -p ~/.local/share/agent-sandbox/home
```

Create the launcher file:

```bash
touch ~/.local/share/agent-sandbox/sandbox
```

## 3. Sandbox script

Start with:

```bash
#!/usr/bin/env bash

set -euo pipefail

SANDBOX_ROOT="$HOME/.local/share/agent-sandbox"
REAL_HOME="$HOME"
PROJECT="$(realpath "$PWD")"

mkdir -p "$SANDBOX_ROOT/home"

exec bwrap \
    --die-with-parent \
    --unshare-all \
    --share-net \
    --dev /dev \
    --proc /proc \
    --tmpfs /tmp \
    --ro-bind /usr /usr \
    --ro-bind /bin /bin \
    --ro-bind /lib /lib \
    --ro-bind /lib64 /lib64 \
    --ro-bind /etc/ssl/certs /etc/ssl/certs \
    --ro-bind /etc/resolv.conf /etc/resolv.conf \
    --ro-bind /etc/passwd /etc/passwd \
    --ro-bind /etc/group /etc/group \
    --bind "$SANDBOX_ROOT/home" "$REAL_HOME" \
    --bind "$PROJECT" "$PROJECT" \
    --chdir "$PROJECT" \
    "$@"
```

Make it executable:

```bash
chmod +x ~/.local/share/agent-sandbox/sandbox
```

Test it:

```bash
~/.local/share/agent-sandbox/sandbox bash
```

## 4. Verify filesystem isolation

Inside the sandbox:

```bash
pwd
ls ~
ls ..
```

The current project should be visible. Verify that sensitive or unrelated host directories are hidden:

```bash
ls ~/.ssh
ls ~/dev/other-project  # No such file or directory
```

Verify that the current project remains writable:

```bash
touch sandbox-test
rm sandbox-test

exit
```


## 5. Python and uv

If the project uses a virtual environment created with `uv`, `.venv/bin/python` may be an absolute symlink to a uv-managed Python installation. Check it outside the sandbox:

```bash
ls -l .venv/bin/python
readlink -f .venv/bin/python

# example: .venv/bin/python -> /home/user/.local/share/uv/python/cpython-3.13.2-linux-x86_64-gnu/bin/python3.13
```

Because the real home directory is hidden, this symlink will be broken inside the sandbox. To expose uv-managed Python installations read-only, add the following **after the fake HOME bind**:

```bash
--ro-bind "$REAL_HOME/.local/share/uv/python" \
          "$REAL_HOME/.local/share/uv/python" \
```

The order matters.

```bash
--bind "$SANDBOX_ROOT/home" "$REAL_HOME" \  # first creates the fake home
```

Then selective mounts expose individual resources inside it. The relevant part should therefore look like:

```bash
--bind "$SANDBOX_ROOT/home" "$REAL_HOME" \
--ro-bind "$REAL_HOME/.local/share/uv/python" \
          "$REAL_HOME/.local/share/uv/python" \
```

Test inside the sandbox:

```bash
.venv/bin/python --version
```

System Python remains available independently, for example:

```bash
python3 --version
```

## 6. Claude Code

First determine where Claude Code is installed:

```bash
command -v claude
ls -l "$(command -v claude)"
readlink -f "$(command -v claude)"

# example: ~/.local/bin/claude -> ~/.local/share/claude/versions/2.x.x
```

In that case, expose both the launcher and its installation directory read-only. Add these **after the fake HOME bind**:

```bash
--ro-bind "$REAL_HOME/.local/bin/claude" \
          "$REAL_HOME/.local/bin/claude" \
--ro-bind "$REAL_HOME/.local/share/claude" \
          "$REAL_HOME/.local/share/claude" \
```

Do not blindly expose your real `~/.claude` directory, instead, let Claude use the sandbox home: `~/.local/share/agent-sandbox/home/.claude`. From inside the sandbox, this appears normally as `~/.claude`. Launch Claude:

```bash
~/.local/share/agent-sandbox/sandbox claude
```

Complete Claude's initial setup and authentication inside the sandbox. This gives Claude separate sandbox credentials, sessions and runtime state without exposing the real Claude profile.

### Migrating existing Claude settings

If you already have customized Claude configuration, copy only the configuration you actually need instead of the entire `~/.claude` directory. For example:

```bash
cp ~/.claude/CLAUDE.md \
   ~/.local/share/agent-sandbox/home/.claude/

cp ~/.claude/RTK.md \
   ~/.local/share/agent-sandbox/home/.claude/
```

Other candidates include: `settings.json`, `keybindings.json`, `hooks/`, etc. Avoid copying runtime/history data unless required:

```text
.credentials.json
history.jsonl
projects/
sessions/
file-history/
shell-snapshots/
session-env/
cache/
debug/
```

It is generally cleaner to authenticate Claude separately inside the sandbox.

## 7. Codex

If Codex is installed system-wide, returns, for example:

```bash
command -v codex

>> /usr/bin/codex
```
then no additional mount is necessary because `/usr` is already exposed read-only. Launch it with:

```bash
~/.local/share/agent-sandbox/sandbox codex
```

Complete authentication inside the sandbox if required. Codex will then keep its sandbox-specific state under `~/.local/share/agent-sandbox/home/.codex`, while seeing it internally as `~/.codex`. If Codex is installed somewhere under the real home directory instead, selectively expose its installation files in the same way as Claude Code.

## 8. Optional: RTK

If you use RTK, first locate it:

```bash
command -v rtk
readlink -f "$(command -v rtk)"
```

For an installation such as `~/.local/bin/rtk` add to sandbox launcher after the fake HOME bind:

```text
--ro-bind "$REAL_HOME/.local/bin/rtk" \
          "$REAL_HOME/.local/bin/rtk" \
```

Verify inside the sandbox:

```bash
rtk --version
```

Initialize RTK for Claude inside the sandbox:

```bash
rtk init -g
```

This creates sandbox-specific RTK configuration instead of modifying your real home directory.

For Codex:

```bash
rtk init -g --codex
```

The resulting files live physically under the sandbox home, for example:

```text
~/.local/share/agent-sandbox/home/.config/rtk/
~/.local/share/agent-sandbox/home/.codex/
```

## 9. Example full script

A setup with uv-managed Python, Claude Code and RTK may look like this:

```bash
#!/usr/bin/env bash

set -euo pipefail

SANDBOX_ROOT="$HOME/.local/share/agent-sandbox"
REAL_HOME="$HOME"
PROJECT="$(realpath "$PWD")"

mkdir -p "$SANDBOX_ROOT/home"

exec bwrap \
    --die-with-parent \
    --unshare-all \
    --share-net \
    --dev /dev \
    --proc /proc \
    --tmpfs /tmp \
    --ro-bind /usr /usr \
    --ro-bind /bin /bin \
    --ro-bind /lib /lib \
    --ro-bind /lib64 /lib64 \
    --ro-bind /etc/ssl/certs /etc/ssl/certs \
    --ro-bind /etc/resolv.conf /etc/resolv.conf \
    --ro-bind /etc/passwd /etc/passwd \
    --ro-bind /etc/group /etc/group \
    --bind "$SANDBOX_ROOT/home" "$REAL_HOME" \
    --ro-bind "$REAL_HOME/.local/share/uv/python" \
              "$REAL_HOME/.local/share/uv/python" \
    --ro-bind "$REAL_HOME/.local/bin/claude" \
              "$REAL_HOME/.local/bin/claude" \
    --ro-bind "$REAL_HOME/.local/share/claude" \
              "$REAL_HOME/.local/share/claude" \
    --ro-bind "$REAL_HOME/.local/bin/rtk" \
              "$REAL_HOME/.local/bin/rtk" \
    --bind "$PROJECT" "$PROJECT" \
    --chdir "$PROJECT" \
    "$@"
```

The Claude and RTK paths in this example are installation-specific. Check them with `command -v`, `ls -l` and `readlink -f` before copying this configuration.

## 10. Zsh / Oh My Zsh aliases

To avoid typing the full launcher path, add aliases to `~/.zshrc`. For example:

```bash
alias sclaude="$HOME/.local/share/agent-sandbox/sandbox claude"
alias scodex="$HOME/.local/share/agent-sandbox/sandbox codex"
alias sbash="$HOME/.local/share/agent-sandbox/sandbox bash"
```

Reload the configuration:

```bash
source ~/.zshrc
```

Now enter any project and run:

```bash
cd ~/dev/my-project
sclaude  # or scodex
```

For debugging:

```bash
sbash
```

The directory from which the command is started becomes the sandbox's writable project directory. Keeping the sandboxed and unsandboxed commands separate is useful `claude -> normal Claude`, `sclaude -> sandboxed Claude`, `codex -> normal Codex`, `scodex -> sandboxed Codex`, etc.


## Security model

This sandbox is primarily designed to prevent an AI coding agent from accidentally reading or modifying files outside the current project. The agent can:

* read and modify the current project
* use selected system tools
* use explicitly exposed development tools
* access the network
* persist its own configuration in the fake home

The agent cannot normally:
* read the real ~/.ssh
* read the real ~/.aws
* access sibling projects
* modify /usr, /bin or system libraries
* access arbitrary files from the real home directory

However, this is **not a VM**. The sandbox shares the host Linux kernel. Network access is deliberately enabled with `--share-net`. Therefore, secrets that are present inside the current project are still visible to the agent and can potentially be transmitted over the network. The current project is also mounted read-write `--bind "$PROJECT" "$PROJECT"`, so the agent can modify or delete files inside that project. Use Git as an additional recovery boundary and commit important work before giving an agent broad autonomy.

Bubblewrap should be treated as a strong filesystem/process isolation layer for this use case, not as protection against arbitrary malicious code or kernel-level attacks.

## Useful diagnostics

Enter the sandbox manually:

```bash
sbash
```

Check the current directory:

```bash
pwd
```

Check the fake home:

```bash
ls -la ~
```

Verify that SSH keys are hidden:

```bash
ls ~/.ssh
```

Verify that another project is hidden:

```bash
ls ~/dev/other-project
```

Check available tools:

```bash
command -v git
command -v python3
command -v claude
command -v codex
command -v rtk
```

Check a uv virtual environment:

```bash
readlink -f .venv/bin/python
.venv/bin/python --version
```

If an executable inside the project reports `No such file or directory` even though the file exists, check whether it is a symlink to a path outside the project:

```bash
ls -l path/to/executable
readlink -f path/to/executable
```

That external target may need to be selectively mounted into the sandbox.
