# For AI agents

## Project overview

`metatrader-docker` runs MetaTrader 5, MetaEditor, Expert Advisors (EAs), and Strategy Tester inside a hardened Linux Docker container. The project targets `linux/amd64` exclusively. The container uses:

- Debian 13 (trixie).
- LinuxServer.io Selkies desktop base image.
- WineHQ Staging for Windows application compatibility.
- Nginx for HTTPS access and request validation.
- s6-overlay for process supervision.
- Optional Windows Python 3.14 and MetaTrader5 Python integration.
- Docker Compose for deployment.

**Primary objective:** maintain a secure, reliable, reproducible, and lightweight MetaTrader 5 container without compromising desktop functionality or persistent trading data.

**Priority order:** security, then data integrity, reliability, compatibility, maintainability, and performance.

## Repository map

- `Dockerfile`: the production image.
- `compose.yaml`: the canonical deployment configuration.
- `root/`: files copied into the image.
  - `defaults/`: desktop autostart, menu, and the nginx site template.
  - `etc/s6-overlay/s6-rc.d/`: s6-overlay init and service units (such as `init-mt-access`, `init-mt-cert`, `init-nginx`, `svc-mt-log`, `mt-stop`).
  - `opt/mt/`: runtime scripts (`run-terminal.sh`, `stop-terminal.sh`, `healthcheck.sh`, `verify-signature.sh`, `notice.py`), certificates (`certs/`), and Windows Python requirements (`python-requirements.txt`).
- `tests/`
  - `access.sh`: access and security integration tests.
  - `python.sh`: Python integration tests.
  - `compose.dev.yaml`: overlay for a development container (project `mt-dev`, port 3201).
- `docs/`, `mkdocs.yml`: documentation site (Material for MkDocs).
- `.github/workflows/`: `image.yml` (image CI), `docs.yml` (documentation), `release.yml` (release-please).
- `renovate.json`: dependency update policy.
- `release-please-config.json`, `.release-please-manifest.json`, `CHANGELOG.md`: release automation; see [Versions and releases](#versions-and-releases).
- `.dockerignore`, `.trivyignore.yaml`: build context exclusions and documented vulnerability exceptions.

Always inspect the actual repository before assuming a file exists. Preserve existing directory conventions.

## Development environment

This repository builds an amd64 Docker image, so Docker is the toolchain and there is no dev container. Use the first option that works; agents follow the same order.

1. **Throwaway containers (recommended):** build the image, run the tests, ShellCheck and the docs preview with `docker` commands that leave nothing behind. The checkout is mounted read-only.
2. **Local toolchain:** ShellCheck and Python 3.14 for the docs, only without Docker. The image build and tests always need Docker on an amd64 host.

### Throwaway containers

Build the image:

```sh
docker build -t metatrader-docker:dev .
```

Access and security tests run container instances on an isolated internal network without internet access, and validate authentication, TLS, origin restrictions, security headers, network exposure, configuration refusals, and installer signature verification. `--host` adds the host-networking tests:

```sh
tests/access.sh
tests/access.sh --host
```

Python integration tests need the development container with `MT_PYTHON=true`, logged into a MetaTrader account. They validate Python-to-MetaTrader IPC connectivity and check that unexpected network ports are not exposed:

```sh
docker compose -f compose.yaml -f tests/compose.dev.yaml build
MT_PYTHON=true docker compose -f compose.yaml -f tests/compose.dev.yaml up -d
tests/python.sh
docker compose -f compose.yaml -f tests/compose.dev.yaml down   # data stays in ./dev/
```

ShellCheck on all tracked shell scripts:

```sh
git ls-files | while read -r f; do head -n 1 "$f" | grep -q -E '^#!.*[/ ](ba)?sh$' && echo "$f"; done |
  xargs docker run --rm -v "$PWD:/mnt:ro" -w /mnt koalaman/shellcheck:v0.11.0
```

Documentation build, on the Python version CI uses:

```sh
docker run --rm -v "$PWD:/work:ro" -w /work python:3.14-slim-trixie \
  sh -c 'pip install -q --root-user-action=ignore --require-hashes -r docs/requirements.txt && mkdocs build --strict --site-dir /tmp/site'
```

For a live preview, add `-it -p 127.0.0.1:8000:8000` and run `mkdocs serve -a 0.0.0.0:8000` instead (see `docs/building.md`).

### Local toolchain

- **ShellCheck:** pipe the same file list to `xargs shellcheck`.
- **Documentation,** with Python 3.14:

```sh
python3 -m venv .venv
.venv/bin/pip install --require-hashes -r docs/requirements.txt
.venv/bin/mkdocs build --strict
.venv/bin/mkdocs serve
```

## Commands and quality gates

Testing is mandatory for behavior-changing modifications. Run what the change touches:

- **Image components** (`Dockerfile`, `root/`): build the image and run `tests/access.sh`.
- **Shell scripts:** run ShellCheck on all tracked shell scripts.
- **Security behavior:** add or update regression tests (see [Security regression tests](#security-regression-tests)).
- **Python integration:** run `tests/python.sh`.
- **MetaTrader execution, Wine, desktop initialization, or Python integration:** perform additional runtime validation whenever practical. Do not treat a successful Docker build as proof that MetaTrader works correctly.
- **Documentation:** run `mkdocs build --strict`.

Never claim tests passed, or runtime validation succeeded, unless they were actually executed. Report any tests that could not be executed.

## Principles

1. Prefer minimal, focused changes; avoid unrelated modifications and unrequested architectural rewrites.
2. Preserve existing architecture and integration contracts.
3. Avoid introducing dependencies unless they provide clear value; prefer native Bash, Python standard-library functionality, and existing system utilities.
4. Keep the Docker image reproducible and minimize unnecessary image layers.
5. Never introduce proprietary MetaTrader or Microsoft binaries into the base image.
6. Do not silently change existing environment variable defaults or behavior.
7. Avoid broad permission changes, and never disable failing tests.

## Security requirements

### Desktop access

- HTTPS and password authentication must remain mandatory.
- Passwords must be validated before services become available.
- Prefer Docker secrets through `FILE__PASSWORD`.
- Never log passwords, account credentials, API keys, or private keys.
- Do not expose desktop services through unrestricted public interfaces.
- Preserve WebSocket origin validation.
- Preserve security response headers and framing restrictions.

### Network security

- Default deployment must bind published ports to `127.0.0.1`.
- Do not expose internal services unnecessarily.
- Never expose Wine, Python IPC, or MetaTrader management interfaces to the host network.
- Do not introduce unrestricted CORS or wildcard WebSocket origins.
- Preserve trusted-proxy validation.
- Maintain network isolation between desktop streaming and internal services.

### Container security

Preserve the existing security posture:

- `no-new-privileges:true`.
- Drop all Linux capabilities by default; add only explicitly required capabilities.
- Never use privileged containers, mount the Docker socket, or add `SYS_ADMIN`.
- Do not restore terminal or sudo access.
- Keep Selkies command execution disabled.
- Do not enable microphone, webcam, or unnecessary desktop functionality.
- Never add `init: true` to Compose: the image relies on s6-overlay as PID 1.

### Installer verification

All downloaded proprietary installers (MetaTrader 5, Microsoft Edge WebView2, optional Windows Python) must pass Authenticode signature verification before execution:

1. Verify the actual cryptographic signature.
2. Validate the expected signer identity.
3. Reject invalid, missing, or unexpected signatures.
4. Do not trust filenames, download URLs, or certificate display names alone.
5. Preserve certificate-chain and revocation validation.
6. Never bypass verification to make an installation succeed.

### Security regression tests

Any change affecting authentication, TLS, origins, networking, permissions, installer verification, or container capabilities must include appropriate regression tests. Update `tests/access.sh` when adding or modifying security behavior.

## Docker development

### Dockerfile

The root `Dockerfile` defines the production image. When modifying it:

- Preserve the pinned base-image digest and Debian 13 compatibility.
- Keep the WineHQ Staging version controlled, and preserve WineHQ signing-key fingerprint verification.
- Use explicit build arguments for configurable versions.
- Avoid unnecessary build dependencies; use `--no-install-recommends` for Debian packages, and clean package caches and temporary installation files.
- Preserve the existing Wine debug-symbol stripping behavior and certificate fingerprint validation.
- Maintain OCI image metadata, and keep the health check functional.
- Do not replace pinned dependencies with floating versions without a clear justification.

### Docker Compose

`compose.yaml` is the canonical deployment configuration. One container represents one MetaTrader trading account. Preserve:

- Persistent `/config` storage and the optional `/mql5` mount.
- Docker secrets for authentication.
- Loopback-only port publishing.
- Resource limits and restricted Linux capabilities.
- Graceful shutdown configuration.

Do not introduce changes that cause account data or Wine configuration to be lost during container recreation.

### Persistent storage

`/config` holds sensitive, persistent application state: the Wine prefix, the MetaTrader installation, broker credentials, trading history and price data, charts and terminal configuration, TLS certificates, and the optional Windows Python installation.

Never delete, overwrite, or recreate persistent state without explicit migration logic and appropriate safeguards. Maintain compatibility with existing installations.

## Runtime development

Runtime components live under `root/opt/mt/` and the related s6-overlay service directories. Before changing runtime behavior:

1. Identify the relevant service and execution order.
2. Understand its user and permission context.
3. Inspect its dependencies.
4. Consider first-boot and subsequent-start behavior.
5. Verify interactions with Wine and MetaTrader.
6. Consider graceful shutdown and recovery behavior.

### Process management

- Preserve s6-overlay supervision; avoid unmanaged background processes.
- Handle signals and shutdown correctly, so child processes don't survive unexpectedly.
- Avoid duplicate MetaTrader instances.
- Preserve automatic recovery of unexpected terminal exits.

### Wine

- Preserve `WINEPREFIX=/config/.wine` and existing DLL overrides.
- Maintain compatibility with WineHQ Staging.
- Avoid unnecessary prefix recreation.
- Consider Wine process ownership and permissions.
- Test changes against actual MetaTrader behavior whenever possible.

### Python integration

Windows Python is optional and controlled through `MT_PYTHON=true`. When modifying it:

- Preserve the existing Windows Python installation workflow, and keep Python installation optional.
- Keep Python package versions pinned and hash-verified.
- Preserve MetaTrader5 and NumPy compatibility.
- Avoid exposing additional network listeners.
- Verify MetaTrader IPC connectivity.

Do not confuse host/container Python with Windows Python running under Wine.

## Shell scripts

All shell scripts must:

- Use an appropriate shebang; prefer Bash when Bash-specific functionality is needed.
- Use `set -euo pipefail` where appropriate.
- Quote variable expansions and validate external inputs.
- Handle missing files and failed commands explicitly.
- Avoid unsafe `eval`, and avoid exposing secrets in command arguments or logs.
- Clean temporary files and resources using traps.
- Return meaningful exit codes.
- Use ShellCheck-compatible syntax, and keep their executable permissions when modified.

Run ShellCheck on all tracked shell scripts before submitting changes.

## Python

When modifying system Python components:

- Prefer Python 3.14-compatible syntax, and the standard library where practical.
- Follow existing formatting and naming conventions.
- Handle exceptions explicitly.
- Avoid unnecessary dependencies.
- Never log sensitive information.
- Use isolated execution where appropriate.
- Maintain compatibility with the existing GTK and runtime environment.

Do not assume host Python and Wine Python share the same packages or runtime environment.

## CI

`.github/workflows/image.yml` performs:

1. ShellCheck.
2. Docker image build.
3. Access and security integration tests.
4. Trivy vulnerability scanning.
5. Secret scanning.
6. Image publication to GHCR.
7. Cosign keyless image signing.

The documentation workflow independently validates documentation. Releases are described in [Versions and releases](#versions-and-releases).

Preserve:

- Pinned GitHub Actions references.
- Minimal workflow permissions.
- Image provenance.
- Software Bill of Materials generation.
- Cryptographic signing.
- Vulnerability scanning.

Never weaken CI checks to make a failing build pass. Security exceptions must be justified and documented in `.trivyignore.yaml`.

## Dependencies

Renovate manages dependency updates. When updating dependencies:

- Preserve version pinning and the SHA-256 hashes of Python packages.
- Verify WineHQ package compatibility, and compatibility with Debian 13 and Wine.
- Inspect upstream base-image changes.
- Review GitHub Actions updates.
- Test changes involving Windows Python or MetaTrader5 manually when necessary.

Do not automatically accept dependency updates that affect security-critical runtime behavior.

## Agent workflow

Before editing:

1. Inspect the relevant source files and understand the current implementation.
2. Identify security and compatibility implications.
3. Locate existing tests.
4. Determine whether documentation needs updating.

Before finishing:

1. Review the complete diff: it holds only intentional changes, and no credentials or sensitive artifacts.
2. Run the checks in [Commands and quality gates](#commands-and-quality-gates) that the change touches, and report any that could not be executed.
3. Confirm the requested functionality works as intended, existing functionality remains compatible, and security invariants are preserved.
4. Update documentation when necessary, following [Writing style](#writing-style).

## Documentation

### Writing style

Applies to README.md, AGENTS.md, docs and code comments.

- **Concise and direct:** no filler, conversational prose or over-explaining.
- **Scannable:** bullets, short tables and short paragraphs; essential commands and facts first.
- **Why, not what:** explain non-obvious choices, constraints and security rules; never narrate self-evident code.
- **Correct first:** never drop a constraint, error condition, edge case or exact command for brevity.
- **Voice:** plain second person, steps as commands, present tense; contractions are fine. No filler words (please, simply, just, easy, easily) and no wordplay.
- **Headings and bold labels:** sentence case; capitalize only the first word, proper nouns and acronyms. A README's title and each `##` heading start with an emoji (or the project's icon in the title); deeper headings don't.
- **Markdown:** `-` bullets, `**Term:** text` for definitions, no horizontal rules, GitHub alerts (`> [!NOTE]`, `> [!TIP]`, `> [!IMPORTANT]`, `> [!WARNING]`) for callouts outside `docs/`.
- **Code fences:** always name the language, one name each: `sh`, `js`, `ts`, `yaml`, `python`, `json`, and `text` for output, trees and diagrams. Never `bash`, `shell`, `javascript`, `typescript`, `yml`, `txt` or `py`.
- **Characters:** ASCII trees (`|--`, `` `-- ``, `|`) and flows (`->`); no box-drawing or arrow characters. Menu paths read **Settings** > **Devices & services**; name icons in words ("the three-dots menu").
- **English** only; no phase labels, scratch notes, obsolete comments or commented-out code.

In this repository:

- **Code comments:** explain the technical why: non-obvious constraints, workarounds, security invariants, protocol behaviors. Never rephrase variable or function names.
- **Preserve test contracts:** never alter logged strings or test outputs that regression tests assert against when editing messages or comments.

### README

`README.md` is the landing page on GitHub and anywhere else it's shown (HACS, GHCR). It says what the project does and gets a user started; the docs site covers the rest.

- **Outline:** icon and name; badges; a summary paragraph; the docs pointer (`> 📖 **Full documentation at [host](url):** what's there.`); `✨ Features`; getting started; project-specific sections, each ending with a link to its docs page; `📖 Documentation`, a table with one row per docs page in nav order; `🤝 Contributing`; `📄 License`.
- **Features:** `- <emoji> **Name** — what it does.`
- **Badges:** one per line, in this order: CI status of each required check, latest release, docs site, license, then compatibility and stack. shields.io, lowercase labels (acronyms excepted), the product's logo where shields has one, ranges or majors rather than exact versions.
- **Links:** absolute URLs for docs pages, repository files and images, so the README works wherever it's shown.
- **Sync:** when a README section changes, update the docs page it summarizes in the same change.

Keep the README quick start synchronized with `compose.yaml`.

### Docs site

- Material for MkDocs, from `docs/` and `mkdocs.yml`, published at <https://metatrader-docker.blunan.com/>. It must build with `mkdocs build --strict`; the `docs` workflow validates it.
- `docs/stylesheets/blunan-brand.css` is the vendored brand theme; never edit it by hand.
- The build tool is locked with hashes in `docs/requirements.txt`, generated from `docs/requirements.in` with `pip-compile --generate-hashes`, on the Python version that `.github/workflows/docs.yml` installs.
- Link to repository files outside `docs/` with absolute GitHub URLs. Give headings that the README links to an explicit `{ #id }`.
- Never document insecure configurations as recommended defaults.
- When modifying behavior, update the relevant page:
  - `settings.md`: environment variables and configuration.
  - `access.md`: network access and reverse proxies.
  - `using.md`: MetaTrader functionality and Python.
  - `operations.md`: lifecycle and updates.
  - `troubleshooting.md`: common failures.
  - `verify.md`: security and installer verification.
  - `building.md`: development and testing procedures.

### Docs formatting

Follow these rules on every edit under `docs/`, including files embedded with `--8<--`. They keep pages consistent and inside fonts' character set.

- **Characters:** Latin letters, digits, common punctuation, typographic quotes and dashes only. No box drawing (`├──`), arrows (`→`, `►`) or symbols (`⋮`, `✔`): the brand fonts lack them, so they render in a fallback font and break code-block alignment.
- **Diagrams:** ASCII in a `text` fence. Trees use `|--`, `` `-- `` and `|`; flows use `->`.
- **Menu paths:** write **Settings** > **Devices & services**. Name icons in words ("the three-dots menu").
- **Program output:** keep it ASCII, or paraphrase it.
- **Code fences:** always name the language, using one name per language: `sh`, `js`, `yaml`, `python`, `json`, and `text` for output and diagrams. Never `bash`, `shell`, `javascript`, `yml`, `txt` or `py`.
- **Callouts:** only `note`, `abstract`, `info`, `tip`, `success`, `question`, `warning`, `failure`, `danger`, `bug`, `example` and `quote`.
- **Emphasis:** bold or italic, never both at once.
- **Headings:** sentence case. Capitalize only the first word, proper nouns and acronyms. Proper nouns here: MetaTrader, MetaEditor, MetaQuotes, MQL5, Strategy Tester, Expert Advisor, Wine, WineHQ, Selkies, Docker, Docker Compose, Debian, Python, Edge WebView2, Tailscale Serve, Caddy, nginx, Traefik, cloudflared, Cosign, Trivy, Renovate, release-please, GitHub.
- **Page outline:** each page has one `#` title that equals its nav label (the home page uses the project name), a summary paragraph before the first `##`, and heading levels that never skip.
- **Voice:** plain second person ("you"), steps as commands, present tense; contractions are fine. No filler words (please, simply, just, easy, easily) and no wordplay.

## Git and commits

- Always use the Conventional Commits specification.
- Never commit secrets, credentials, environment files, personal configuration or build artifacts.
- Keep pull requests focused and document intentional compatibility changes.
- Keep changes logically grouped, and preserve existing repository conventions.
- Never commit certificates or local account data, nor generated documentation output unless explicitly required.

### Versions and releases

`main` takes squash merges only, so a PR's title becomes its commit on `main`, and release-please reads only that title to pick the next version. A new version means the image's contents or behavior changed:

- `feat:`: new or changed behavior. Minor version.
- `fix:`: a bug fix, or an update to what's in the image (base image, Wine, packages). Patch version.
- `feat!:` / `fix!:`, or a `BREAKING CHANGE:` footer: major version.
- `docs:`, `ci:`, `chore:`, `refactor:`, `test:`, `build:`, `style:`: no version. Use these when the image's contents and behavior stay the same, even if it gets rebuilt.

Renovate titles updates found in `Dockerfile` and `root/` as `fix(deps):`, and all other updates as `chore(deps):` (see `renovate.json`).

The `release` workflow (`.github/workflows/release.yml`) runs release-please on every push to `main`. It keeps a release PR open (`chore(main): release X.Y.Z`) with the next version and its `CHANGELOG.md` entry; merging that PR, by hand when the version should go out, tags `vX.Y.Z` and publishes the GitHub release. A release builds no new image. The workflow uses a GitHub App token (`RELEASE_APP_CLIENT_ID` variable, `RELEASE_APP_PRIVATE_KEY` secret), not `GITHUB_TOKEN`, because a PR opened with `GITHUB_TOKEN` runs no workflows and the required checks would never report.

- Don't edit `CHANGELOG.md` or `.release-please-manifest.json` by hand; release-please updates them in its release PR.
- To correct a merged PR's type before its release, add the corrected message to that PR's description between a `BEGIN_COMMIT_OVERRIDE` line and an `END_COMMIT_OVERRIDE` line; release-please reads it instead of the commit. To force a version, end that message with a `Release-As: X.Y.Z` footer.
- Don't start a paragraph in a commit body with `feat:`, `fix:` or another type: release-please reads it as another commit.

## Security guardrails

- Never commit secrets, credentials, tokens, or private keys.
- Always validate untrusted input at appropriate boundaries.
- Always use parameterized queries and safe APIs for external input.
- Avoid logging sensitive information.
- Always follow least-privilege principles.
- Never disable security controls to bypass errors.
