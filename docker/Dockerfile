# run_metrics_and_sample.py, containerized.
#
# Build context MUST be the DataLabs root (not this docker/ directory) so it
# can COPY datalabs_paths.py and skills/repo-metrics/scripts/ directly:
#
#   docker build -f skills/repo-metrics/docker/Dockerfile -t repo-metrics-sample .
#
# See skills/repo-metrics/docker/README.md for full run instructions,
# volume/token setup, and — importantly — the fallback behavior for every
# tool this image installs.

FROM python:3.12-slim

# git            required, no fallback.
# curl           only used at build time, to fetch the scc binary below.
# nodejs, npm    for jscpd (duplication detection). If this install step is
#                ever unavailable at runtime, the scripts detect that and
#                fall back to a weaker built-in detector automatically —
#                see README.md. Not a hard requirement, but installed here
#                so the strong path is the default.
# docker.io      the `docker` CLI only (not a nested daemon) — lets
#                --build-check docker work IF the host's docker socket is
#                bind-mounted in at run time (see README.md). Without that
#                mount, repo_metrics.py detects the daemon is unreachable
#                and downgrades to --build-check detect automatically; this
#                package's CLI presence alone does not change that.
RUN apt-get update && apt-get install -y --no-install-recommends \
        git curl ca-certificates nodejs npm docker.io \
    && rm -rf /var/lib/apt/lists/*

RUN npm install -g jscpd@5

# scc — the one hard, non-fallback-able requirement. TARGETARCH is set
# automatically by BuildKit; mapped to scc's own release-asset naming.
ARG TARGETARCH
ARG SCC_VERSION=4.0.0
RUN set -eux; \
    case "${TARGETARCH}" in \
        amd64) SCC_ARCH=x86_64 ;; \
        arm64) SCC_ARCH=arm64 ;; \
        *) echo "unsupported architecture: ${TARGETARCH}" >&2; exit 1 ;; \
    esac; \
    curl -fsSL "https://github.com/boyter/scc/releases/download/v${SCC_VERSION}/scc_Linux_${SCC_ARCH}.tar.gz" \
        -o /tmp/scc.tar.gz; \
    tar -xzf /tmp/scc.tar.gz -C /usr/local/bin scc; \
    chmod +x /usr/local/bin/scc; \
    rm /tmp/scc.tar.gz; \
    scc --version

WORKDIR /app
COPY datalabs_paths.py /app/datalabs_paths.py
COPY skills/repo-metrics/scripts/ /app/skills/repo-metrics/scripts/

# Everything the scripts write lands under here — bind-mount it (see
# README.md) or it disappears when the container exits.
RUN mkdir -p /app/outputs /app/clones

WORKDIR /app/skills/repo-metrics/scripts
ENTRYPOINT ["python3", "run_metrics_and_sample.py"]
CMD ["--help"]
