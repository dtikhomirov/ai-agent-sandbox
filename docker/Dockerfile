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

# Convention for arbitrary UID containers:
# 1. Base unprivileged user (10001) with dedicated workspace under $HOME
# 2. Allow any arbitrary runtime host UID to write temporary state to $HOME
ARG USER_ID=10001
ARG GROUP_ID=10001
ENV HOME=/home/agent

RUN groupadd --gid ${GROUP_ID} agent \
    && useradd --uid ${USER_ID} --gid ${GROUP_ID} --create-home --shell /bin/bash agent \
    && mkdir -p /home/agent/workspace \
    && chmod 777 /home/agent /home/agent/workspace

WORKDIR /home/agent/workspace

# =========================================================
# STAGE 2: Build the GitHub Copilot CLI image
# =========================================================
FROM agent-runtime AS ghc-cli
# Install system-wide to /usr/local/bin so ANY runtime UID can execute it
RUN curl -fsSL https://gh.io/copilot-install | bash
USER agent
ENTRYPOINT ["copilot"]

# =========================================================
# STAGE 3: Build the Claude CLI image
# =========================================================
FROM agent-runtime AS claude-cli
RUN curl -fsSL https://claude.ai/install.sh | bash
USER agent
ENTRYPOINT ["claude"]

# =========================================================
# STAGE 4: Build the Antigravity CLI image
# =========================================================
FROM agent-runtime AS agy-cli
RUN curl -fsSL https://antigravity.google/cli/install.sh | bash
USER agent
ENTRYPOINT ["agy"]