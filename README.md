# AI Agent Box

## Why Sandbox Isolation Matters

Running AI agents in isolation is critical for security and stability. Agents have broad capabilities — they can execute code, modify files, and make API calls — which makes them a significant security risk if run directly on your host system:

- **Containment**: A compromised agent or malicious prompt injection cannot affect your system, credentials, or other projects
- **Access Control**: Sandboxing limits an agent's access to only the files and resources it explicitly needs
- **Reliability**: Container isolation prevents resource exhaustion or runaway processes from affecting other work
- **Auditability**: Each agent session runs in a separate container, making it easy to track what was executed and when

## Project Goals

AI Agent Box provides a **lightweight, highly secure setup** for running AI agent CLIs safely from your terminal. Rather than installing agents globally or running them with full system access, this project uses Docker containers with minimal overhead:

- **Secure by default**: Agents run as unprivileged users in isolated containers
- **Lightweight**: Multi-stage builds keep images lean; minimal dependencies ensure fast startup
- **Convenient**: Mount only what you need; persist settings and history across sessions; run parallel sessions without conflicts
- **Terminal-native**: Simple shell functions wrap Docker invocations, making agent CLIs feel like native commands

We provide a transparent pipeline for building Docker images. Each image uses `/workspace` as its working directory and sets the CLI as its entrypoint, so the container runs the agent directly. Zsh shell functions abstract away Docker lifecycle management, giving you a classical terminal experience without the container orchestration overhead.

## Build

Run from this directory:

```sh
docker build --target ghc-cli -t ghc:latest -f docker/Dockerfile .
docker build --target claude-cli -t claude:latest -f docker/Dockerfile .
docker build --target agy-cli -t agy:latest -f docker/Dockerfile .
```

## Verify

```sh
docker run --rm -it ghc:latest
docker run --rm -it claude:latest
docker run --rm -it agy:latest
```

## How to use

Mount only the directory the agent needs. `--network none` prevents network access; omit it when the selected CLI must authenticate or call a remote service.

### Example: Copilot CLI against a local OpenAI-compatible endpoint

Add the following function to your `~/.zshrc`:

```sh
copilot-qwen38max() {
    local project="${PWD##*/}"
    local copilot_home="${HOME}/.docker_volumes/.copilot"
    mkdir -p "$copilot_home"

    docker run --rm -it --init \
    --name "copilot-${project}-$(date +%H%M%S)" \
    -v "$(pwd)":/workspace \
    -v "${copilot_home}:/home/agent/.copilot" \
    -w /workspace \
    --add-host=host.docker.internal:host-gateway \
    -e COPILOT_PROVIDER_TYPE="openai" \
    -e COPILOT_MODEL="qwen3.8-max" \
    -e COPILOT_PROVIDER_BASE_URL="http://host.docker.internal:4000/v1" \
    -e COPILOT_PROVIDER_API_KEY="your-api-key" \
    -e COPILOT_OFFLINE=true \
    -e COPILOT_PROVIDER_MAX_OUTPUT_TOKENS=262144 \
    -e COPILOT_PROVIDER_MAX_PROMPT_TOKENS=32768 \
    ghc:latest \
        "$@"
}
```

Then reload your shell: `source ~/.zshrc`. The CLI command is `copilot-qwen38max`.

## Persistence

The container is ephemeral (`--rm`); the bind mount `${HOME}/.docker_volumes/.copilot` persists `~/.copilot` (settings, auth, session history) on your Mac — visible in Finder and included in Time Machine.

The `mkdir -p` creates the directory on first use; if Docker creates it instead, ownership may be wrong. Because every container mounts the same directory, all sessions are visible and resumable (`/resume`) from any of them, and you authenticate only once.

If the in-container `agent` user (uid 10001) cannot write to it, fall back to a named volume (`-v "copilot-home:/home/agent/.copilot"`), which has no ownership issues. Prefer isolated histories per project? Use `-v "copilot-${project}:/home/agent/.copilot"` instead.

## Running parallel sessions

Each `docker run` is a fully isolated container, so parallel sessions work out of the box. Containers get readable names like `ghc-myrepo-153012` (project + start time); list active sessions with `docker ps --filter name=ghc-`. Keep in mind:

- **Don't share one working directory across sessions** — two agents editing the same files will conflict. Use `git worktree add ../feature-x` to give each session its own checkout.
- **Shared session store** — all sessions live in one SQLite database inside the shared `~/.docker_volumes/.copilot`. SQLite handles concurrent writers via locking, but if you want strict isolation (or to avoid lock contention), switch to a per-project volume as described under Persistence.
- **Resources** — all containers share Docker Desktop's CPU/RAM allocation; raise the limits in settings if you run many sessions.
