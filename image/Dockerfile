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
  apt-get install -y --no-install-recommends nodejs && \
  echo "**** npm: bundled with NodeSource's nodejs (which the base image uses); Debian's nodejs needs the separate package ****" && \
  (command -v npm >/dev/null || apt-get install -y --no-install-recommends npm) && \
  echo "**** install playwright mcp ****" && \
  export OPENSSL_CONF=/dev/null && \
  mkdir -p /opt/mcp && cd /opt/mcp && \
  npm init -y >/dev/null && \
  npm install --omit=dev @playwright/mcp@latest && \
  echo "**** let non-root processes (abc) read the OpenSSL config ****" && \
  chmod a+rx /etc/ssl && chmod a+r /etc/ssl/openssl.cnf && \
  echo "**** cleanup ****" && \
  npm cache clean --force && \
  apt-get autoclean && \
  rm -rf /var/lib/apt/lists/* /var/tmp/* /tmp/* /root/.npm

# Don't `COPY root/ /` directly: Docker's classic builder (what Synology uses)
# stamps the checkout's directory modes onto / and /etc. On a Synology share
# those are ACL-style modes (no "other" access), which locks the non-root abc
# user out of /etc — breaking name resolution ("localhost") and OpenSSL.
# Stage the files, then copy them in without their modes.
COPY root/ /tmp/overlay/
RUN \
  cp -r --no-preserve=mode,ownership /tmp/overlay/. / && \
  rm -rf /tmp/overlay && \
  chmod 755 / /etc /etc/s6-overlay /etc/s6-overlay/s6-rc.d && \
  chmod -R a+rX /etc/s6-overlay/s6-rc.d/svc-mcp /etc/s6-overlay/s6-rc.d/user /etc/chromium/policies && \
  chmod 755 /etc/s6-overlay/s6-rc.d/svc-mcp/run

EXPOSE 3000 8931
