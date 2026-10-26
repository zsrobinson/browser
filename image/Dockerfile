# syntax=docker/dockerfile:1
#
# linuxserver's Chromium (browser + Selkies web viewer, Debian trixie)
# plus Playwright MCP running as a second supervised service in the same
# container, talking to Chromium over localhost.

# Same image is published to GHCR and Docker Hub; pick with BASE_IMAGE in .env
# (lscr.io is just a redirector to GHCR).
ARG BASE_IMAGE=ghcr.io/linuxserver/chromium:latest
FROM ${BASE_IMAGE}

ENV PLAYWRIGHT_SKIP_BROWSER_DOWNLOAD=1

RUN \
  echo "**** install node ****" && \
  apt-get update && \
  apt-get install -y --no-install-recommends nodejs npm && \
  echo "**** install playwright mcp ****" && \
  mkdir -p /opt/mcp && cd /opt/mcp && \
  npm init -y >/dev/null && \
  npm install --omit=dev @playwright/mcp@latest && \
  echo "**** cleanup ****" && \
  npm cache clean --force && \
  apt-get autoclean && \
  rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/* /root/.npm

COPY root/ /
RUN chmod +x /etc/s6-overlay/s6-rc.d/svc-mcp/run

EXPOSE 3000 8931
