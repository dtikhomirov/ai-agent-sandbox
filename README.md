# AI Agent Box

## Why Sandbox Isolation Matters

Running AI agents in isolation is critical for security and stability. Agents have broad capabilities — they can execute code, modify files, and make API calls — which makes them a significant security risk if run directly on your host system:

- **Containment**: A compromised agent or malicious prompt injection cannot affect your system, credentials, or other projects
- **Access Control**: Sandboxing limits an agent's access to only the files and resources it explicitly needs
- **Reliability**: Container isolation prevents resource exhaustion or runaway processes from affecting other work
- **Auditability**: Each agent session runs in a separate container, making it easy to track what was executed and when

## Project Goals

AI Agent Box provides a **lightweight, secure setup** for running AI agent CLIs safely from your terminal. Rather than installing agents globally or running them with full system access, this project uses Docker containers with minimal overhead:

- **Secure by default**: Agents run as unprivileged users in isolated containers
- **Lightweight**: Multi-stage builds keep images lean; minimal dependencies ensure fast startup
- **Convenient**: Mount only what you need; persist settings and history across sessions; run parallel sessions without conflicts
- **Terminal-native**: Simple shell functions wrap Docker invocations, making agent CLIs feel like native commands

We provide a transparent pipeline for building Docker images. Each image uses `/home/agent/workspace` as its working directory and sets the CLI as its entrypoint, so the container runs the agent directly.

Zsh shell functions abstract away Docker lifecycle management, giving you a classical terminal experience without the container orchestration overhead.

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

Mount only the directory the agent needs. 

### Example 1: Copilot CLI with local OpenAI-compatible endpoint

Add the following function to your `~/.zshrc`:

```sh
copilot-custom() {
    local project="${PWD##*/}"
    local copilot_home="${HOME}/.docker_volumes/.copilot"
    local container="copilot-${project}-$(date +%Y%m%d-%H%M%S)-$$"
    mkdir -p "$copilot_home"

    cleanup() {
        docker rm -f "$container" >/dev/null 2>&1 || true
        trap - INT TERM HUP
    }
    trap cleanup INT TERM HUP

    # Format: "model_name:max_prompt_tokens:max_output_tokens"
    local models=(
        "qwen3.8-max:800000:128000"
        "qwen3.8-flash:800000:128000"
        "deepseek-v4-pro-0813:800000:128000"
        "glm-5.3:800000:128000"
        "glm-5.3-flash:800000:128000"
        "nemotron-3-super-120b-a12b:800000:128000"
        "nemotron-3.5-lightning-30b-a3b:800000:32000"
        "qwen3.6-35b-a3b-mlx:220000:32000"
        "gemma-4-26b-a4b-it-mlx:220000:32000"
    )

    echo -e "\n\033[1;34m=== Select Copilot Model ===\033[0m"
    local i=1
    for item in "${models[@]}"; do
        echo "$i) ${item%%:*}"
        ((i++))
    done

    local choice
    while true; do
        echo -en "\n\033[1;36mSelect model number: \033[0m"
        # Portable single-key read: -k for Zsh, -n for Bash
        if [ -n "$ZSH_VERSION" ]; then
            read -k 1 choice
        else
            read -r -n 1 choice
        fi
        echo ""

        if [[ "$choice" =~ ^[1-9]$ ]] && (( choice <= ${#models[@]} )); then
            break
        fi
        echo -e "\033[1;31mInvalid choice. Press a number from 1 to ${#models[@]}.\033[0m"
    done

    # Portable array access (Zsh is 1-indexed, Bash is 0-indexed)
    local chosen_entry="${models[$([ -n "$ZSH_VERSION" ] && echo "$choice" || echo "$((choice - 1))")]}"

    # Extract model and tokens (Prompt context first, Output completion second)
    IFS=":" read -r chosen_model max_prompt_tokens max_output_tokens <<< "$chosen_entry"

    echo "Running $chosen_model (Prompt context: $max_prompt_tokens, Output limit: $max_output_tokens)"

    docker run --rm -it --init \
        --name "$container" \
        --user "$(id -u):$(id -g)" \
        -v "$(pwd)":/home/agent/workspace \
        -v "${copilot_home}:/home/agent/.copilot" \
        -w /home/agent/workspace \
        --add-host=host.docker.internal:host-gateway \
        -e COPILOT_PROVIDER_TYPE="openai" \
        -e COPILOT_MODEL="$chosen_model" \
        -e COPILOT_PROVIDER_BASE_URL="http://host.docker.internal:4000/v1" \
        -e COPILOT_PROVIDER_API_KEY="your-api-key" \
        -e COPILOT_OFFLINE=true \
        -e COPILOT_PROVIDER_MAX_OUTPUT_TOKENS="$max_output_tokens" \
        -e COPILOT_PROVIDER_MAX_PROMPT_TOKENS="$max_prompt_tokens" \
        ghc:latest \
        "$@"

    local ret=$?
    cleanup
    return $ret
}
```

### Example 2: Copilot CLI with Github Auth (PAT token) 

Add the following function to your `~/.zshrc`:

```sh
copilot() {
    local project="${PWD##*/}"
    local copilot_home="${HOME}/.docker_volumes/.copilot"
    local container="copilot-${project}-$(date +%Y%m%d-%H%M%S)-$$"

    mkdir -p "$copilot_home"

    cleanup() {
        docker rm -f "$container" >/dev/null 2>&1 || true
    }
    trap cleanup EXIT HUP INT TERM

   docker run --rm -it --init \
        --name "$container" \
        --user "$(id -u):$(id -g)" \
        -v "$(pwd)":/home/agent/workspace \
        -v "${copilot_home}:/home/agent/.copilot" \
        -w /home/agent/workspace \
        --add-host=host.docker.internal:host-gateway \
        -e COPILOT_GITHUB_TOKEN="${COPILOT_GITHUB_TOKEN}" \
        ghc:latest \
        "$@"
    local ret=$?
    cleanup
    return $ret
}
```

Then reload your shell: `source ~/.zshrc`. The CLI command is `copilot-qwen38max`.

### Copy/paste in macOS Terminal.app

Copilot's TUI captures the mouse for its own text selection, which stops Terminal.app's normal click-drag copy from working. Hold **Fn** while dragging to select text (bypasses the app's mouse capture), then `Cmd+C`/`Cmd+V` as usual. (iTerm2 and other terminals with OSC 52 support aren't affected.)

## Persistence

The container is ephemeral (`--rm`); the bind mount `${HOME}/.docker_volumes/.copilot` persists `~/.copilot` (settings, auth, session history) on your Mac — visible in Finder and included in Time Machine.

The `mkdir -p` creates the directory on first use. Both examples run the container as your host UID/GID, so the mounted directory is writable by your user. Because every container mounts the same directory, all sessions are visible and resumable (`/resume`) from any of them, and you authenticate only once.

If you use a named volume instead, check that it is writable by the UID/GID passed with `--user`; named volumes may also need permission setup. Prefer isolated histories per project? Use `-v "copilot-${project}:/home/agent/.copilot"` instead.

## License

This project is licensed under the [MIT License](LICENSE).

## Running parallel sessions

Each `docker run` is a fully isolated container, so parallel sessions work out of the box. Containers get readable names like `ghc-myrepo-153012` (project + start time); list active sessions with `docker ps --filter name=ghc-`. Keep in mind:

- **Don't share one working directory across sessions** — two agents editing the same files will conflict. Use `git worktree add ../feature-x` to give each session its own checkout.
- **Shared session store** — all sessions live in one SQLite database inside the shared `~/.docker_volumes/.copilot`. SQLite handles concurrent writers via locking, but if you want strict isolation (or to avoid lock contention), switch to a per-project volume as described under Persistence.
- **Resources** — all containers share Docker Desktop's CPU/RAM allocation; raise the limits in settings if you run many sessions.
