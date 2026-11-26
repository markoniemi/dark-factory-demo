# Running Claude Code and OpenCode with sbx

[Docker Sandboxes](https://docs.docker.com/ai/sandboxes/) (`sbx`) runs a coding
agent inside an isolated microVM. The agent works on your project directory and
runs without permission prompts, but it cannot reach the rest of your machine, and
its network access goes through a policy you control.

You **do not install Claude Code or OpenCode on your host**. Each sandbox image
already contains its agent. You install `sbx`, store a credential, and run the
agent.

Commands below were taken from the Docker documentation in November 2026. Check the
linked pages if a command has changed.

## 1. Install sbx

Requires Windows 11 on a 64-bit Intel or AMD processor. Docker Desktop is not
required.

1. Turn on Windows Hypervisor Platform. In an **elevated** PowerShell prompt:

   ```powershell
   Enable-WindowsOptionalFeature -Online -FeatureName HypervisorPlatform -All
   ```

   Restart Windows if it asks you to.

2. Install `sbx` for the current user:

   ```powershell
   winget install -h Docker.sbx
   ```

   To install it for all users instead, download `DockerSandboxesMachine.msi`
   from the [releases page](https://github.com/docker/sbx-releases/releases) and
   run `msiexec.exe /i DockerSandboxesMachine.msi /quiet` as an administrator.

Source: [Install the sbx CLI](https://docs.docker.com/ai/sandboxes/install/).

## 2. Sign in and choose a network policy

```powershell
sbx login
```

This opens a browser to sign in to your Docker account.

The first time you run a sandbox from an interactive terminal, `sbx` asks for a
network policy. Choose **Balanced**: common development services are allowed, and
any other host waits for your approval. In a script, set it up beforehand:

```powershell
sbx policy init balanced
```

Optional: let agents use your GitHub credentials for `git push` and `gh`:

```powershell
sbx secret set github --command 'gh auth token'
```

## 3. Claude Code

Pick **one** way to sign in:

* **Claude subscription**: nothing to set up
  beforehand. Run `/login` inside Claude Code the first time it starts.
* **Anthropic API key**: store it once. Every sandbox can then use it.

  ```powershell
  sbx secret set anthropic
  ```

  To pipe the key in instead of typing it:
  `$env:ANTHROPIC_API_KEY | sbx secret set anthropic`.

Run it on your project directory:

```powershell
sbx run claude C:\path\to\your\project
```

The path is the directory the agent works on. You can leave it out of `sbx run`,
and the current directory is used, but the examples here always pass it.

Name the sandbox so you can come back to it later:

```powershell
sbx run --name my-project claude C:\path\to\your\project
```

Inside the sandbox, Claude Code starts as
`claude --dangerously-skip-permissions`. The microVM is the safety boundary, so
there are no permission prompts. Your host's `~/.claude` configuration is **not**
copied in. Only project-level configuration in the working directory (for example
`CLAUDE.md` and `.claude/`) is available.

Source: [Claude Code in sbx](https://docs.docker.com/ai/sandboxes/agents/claude-code/).

## 4. OpenCode

Store a key for each model provider you want to use. Supported providers include
`anthropic`, `openai`, `google`, `xai`, `groq` and `openrouter`:

```powershell
sbx secret set anthropic
sbx secret set openai
```

OpenCode finds the stored credentials and offers those providers in its TUI. A
GitHub credential with Copilot access (see step 2) also enables the Copilot
provider.

To use **OpenCode Zen** instead, store the key as a custom secret and allow the
host:

```powershell
$env:OPENCODE_API_KEY | sbx secret set-custom `
    --host opencode.ai `
    --env OPENCODE_API_KEY `
    --value-stdin
sbx policy allow network opencode.ai:443
```

Run it on your project directory:

```powershell
sbx run opencode C:\path\to\your\project
```

OpenCode starts with no extra flags. Anything after `--` is passed to it, for
example to resume a session: `sbx run --name my-project -- -s <session-id>`. Your
host's OpenCode configuration is not copied in; only project-level configuration
is available.

Source: [OpenCode in sbx](https://docs.docker.com/ai/sandboxes/agents/opencode/).

## 5. Save your setup in settings files

Instead of repeating flags and secrets every time, you can describe a sandbox in an
`sbxenv.yaml` file and start it with `sbx env run`. Use two files:

| File | Holds | Location |
| :- | :- | :- |
| Personal settings | Your git identity and secrets, the same for every project | `C:\Users\<you>\.sbxenv.yaml` |
| Project settings | Agent, workspace and resources for one project | `sbxenv.yaml` in the project's root folder, committed with the project |

### Personal settings

Create `C:\Users\<you>\.sbxenv.yaml`. `sbx env` loads it as your user defaults on
top of the project file. It can hold anything except `name`.

```yaml
schemaVersion: "1"

# Commit as yourself. Without these, commits made in the sandbox are
# authored by "agent <agent@...docker.internal>".
env:
  GIT_AUTHOR_NAME: Your Name
  GIT_AUTHOR_EMAIL: you@example.com
  GIT_COMMITTER_NAME: Your Name
  GIT_COMMITTER_EMAIL: you@example.com

# Fetch secrets when the sandbox starts, instead of writing them in the file.
secrets:
  github:
    command: gh auth token
  anthropic:
    ref: op://Private/Anthropic/api-key   # 1Password reference
```

A secret can come from a `command` whose output is the value, or from a `ref` (a
1Password `op://` reference or an AWS Secrets Manager ARN). A literal `value:`
also works, but anyone who can read the file sees the key. Leave out the
`anthropic` entry if you sign in to Claude Code with `/login`.

### Project settings

Create `sbxenv.yaml` in the project's root folder and commit it, so everyone
working on the project gets the same sandbox. This project's own
[`sbxenv.yaml`](../sbxenv.yaml) is an example. A fuller one:

```yaml
schemaVersion: "1"
name: my-project
agent: claude              # or: opencode
workspace: .               # the folder this file is in

sandboxOptions:
  cpus: 4
  memory: 8g

ports:
  - sandbox: 3000
    host: 3000
```

To use OpenCode, set `agent: opencode`. To work on a Git clone instead of the
mounted folder, write the workspace as `workspace: {path: ., clone: true}`.

Run it from the project folder, or pass the folder's path from anywhere:

```powershell
cd C:\path\to\your\project
sbx env plan                         # show what will be created
sbx env run --skip-host-commands     # create the sandbox if needed, and attach

sbx env run --skip-host-commands C:\path\to\your\project
```

Other commands: `sbx env create` creates the sandbox without attaching, and
`sbx env rm` removes it.

**Keep host commands out of the project file.** The project folder is mounted
into the sandbox, so the agent can edit this file. An `sbxenv.yaml` can define
`lifecycle` commands, which run **on your Windows machine**, so an edited file
could make your machine run a command the agent wrote. Docker's own advice is to
keep the file outside mounted folders. Keeping it in the project is safe enough
if you:

* leave `lifecycle` out of the file,
* start with `--skip-host-commands`, so lifecycle commands never run anyway,
* check `git diff sbxenv.yaml` and the `sbx env plan` output before you start.
  An edit could also add mounts, ports or environment variables.

Not confirmed by the docs: whether the personal file also applies when you pass a
path to `sbx env run`, and whether it affects a plain `sbx run` at all. It
belongs to `sbx env`, so probably not.

Source: [Environment files](https://docs.docker.com/ai/sandboxes/configuration/environment-files/).

## Good to know

* **Edits reach your project immediately.** By default the project directory is
  mounted into the sandbox. To isolate the agent on its own Git clone instead, use
  `sbx run --clone claude C:\path\to\your\project`. Then fetch its commits on the host with
  `git fetch sandbox-<name>`.
* **Create a sandbox now and attach later:**
  `sbx create --name my-project opencode C:\path\to\your\project`, then `sbx run --name my-project`.
  Attaching by name needs no path. **`sbx create` needs one:** without a path it
  creates a sandbox with no project mounted.
* **A request was blocked:** review the pending approval with
  `sbx policy approval ls`, or allow a host up front with
  `sbx policy allow network <host>`.
* **Reach a service inside the sandbox:**
  `sbx port publish <sandbox-name> 8080:8080/tcp`. The service must listen on
  `0.0.0.0`.
* The old `docker sandbox` commands are deprecated. Use `sbx`.
