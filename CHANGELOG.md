# Changelog

All notable user-visible changes to the Actual Budget TOS application package.

## 26.11.0-1 — 2026-11-09

### Added
- First release. Actual Budget (`actualbudget/actual-server:26.11.0`, Docker Hub official image, unmodified)
  packaged as a TOS 7 Docker application, published on port 8033 and running
  as a non-root user (1000:1000).
- Persistent data is kept under `/Volume1/DockerAppData/actualbudgetdocker/`.
