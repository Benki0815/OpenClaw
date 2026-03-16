# ============================================================
# OpenClaw Sandbox – Custom Image mit vorinstallierten Paketen
# ============================================================
# All packages are pre-installed so the sandbox can run with
# network: "none" (zero network access at runtime).
# ============================================================

FROM debian:bookworm-slim

# Prevent interactive prompts during package installation
ENV DEBIAN_FRONTEND=noninteractive

# Install system packages (Python, common CLI tools)
RUN apt-get update && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    python3-venv \
    curl \
    jq \
    git \
    ca-certificates \
    locales \
    && rm -rf /var/lib/apt/lists/*

# Set locale to avoid encoding issues
RUN sed -i '/en_US.UTF-8/s/^# //g' /etc/locale.gen && locale-gen
ENV LANG=en_US.UTF-8

# Install Python packages (data analysis, plotting, HTTP)
RUN pip3 install --no-cache-dir --break-system-packages \
    pandas \
    matplotlib \
    requests \
    numpy \
    openpyxl \
    python-dateutil \
    tabulate

# Create non-root sandbox user (UID 1000 matches OC's node user)
RUN groupadd -g 1000 sandbox && \
    useradd -u 1000 -g sandbox -m -s /bin/bash sandbox

# Default working directory
WORKDIR /workspace

# Run as non-root by default
USER sandbox

# Keep container alive for exec-based usage
CMD ["sleep", "infinity"]
