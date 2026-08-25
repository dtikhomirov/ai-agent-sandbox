# =========================================================
# STAGE 1: Base image for the Agent runtime
# =========================================================
FROM ubuntu:24.04 AS agent-runtime

# Fail a RUN step if any command in a pipeline fails (e.g., curl | bash)
SHELL ["/bin/bash", "-o", "pipefail", "-c"]

# Install required packages for extensions
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    curl \
    git \
    python3 \
    python3-pip \
    python3-venv \
    && rm -rf /var/lib/apt/lists/*

RUN groupadd --gid 10001 agent \
    && useradd --uid 10001 --gid agent --create-home --shell /bin/bash agent \
    && mkdir /workspace \
    && chown agent:agent /workspace

ENV HOME=/home/agent
ENV PATH="/home/agent/.local/bin:${PATH}"

USER agent
WORKDIR /workspace

# =========================================================
# STAGE 2: Build the GitHub Copilot CLI image
# =========================================================
FROM agent-runtime AS ghc-cli

RUN curl -fsSL https://gh.io/copilot-install | bash

ENTRYPOINT ["copilot"]

# =========================================================
# STAGE 3: Build the Claude CLI image
# =========================================================
FROM agent-runtime AS claude-cli

RUN curl -fsSL https://claude.ai/install.sh | bash

ENTRYPOINT ["claude"]

# =========================================================
# STAGE 4: Build the Antigravity CLI image
# =========================================================
FROM agent-runtime AS agy-cli

RUN curl -fsSL https://antigravity.google/cli/install.sh | bash

ENTRYPOINT ["agy"]