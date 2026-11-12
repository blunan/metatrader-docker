# For AI Agents

## Project Overview

`metatrader-docker` runs MetaTrader 5, MetaEditor, Expert Advisors (EAs), and Strategy Tester inside a hardened Linux Docker container.

The container uses:

- Debian 13 (Trixie).
- LinuxServer.io Selkies desktop base image.
- WineHQ Staging for Windows application compatibility.
- Nginx for HTTPS access and request validation.
- s6-overlay for process supervision.
- Optional Windows Python 3.14 and MetaTrader5 Python integration.
- Docker Compose for deployment.

The project targets `linux/amd64` exclusively.

**Primary objective:** Maintain a secure, reliable, reproducible, and lightweight MetaTrader 5 container without compromising desktop functionality or persistent trading data.

## Repository Structure

```text
.
├── .github/
│   └── workflows/
│       ├── image.yml
│       ├── docs.yml
│       └── release.yml
├── docs/
│   ├── index.md
│   ├── quick-start.md
│   ├── access.md
│   ├── settings.md
│   ├── using.md
│   ├── operations.md
│   ├── troubleshooting.md
│   ├── verify.md
│   └── building.md
├── root/
│   └── opt/mt/
│       ├── scripts and runtime components
│       ├── certificates
│       └── Python requirements
├── tests/
│   ├── access.sh
│   ├── python.sh
│   └── compose.dev.yaml
├── Dockerfile
├── compose.yaml
├── mkdocs.yml
├── renovate.json
├── release-please-config.json
├── .release-please-manifest.json
├── CHANGELOG.md
├── .dockerignore
├── .trivyignore.yaml
├── README.md
├── LICENSE
```

Always inspect the actual repository before assuming a file exists. Preserve existing directory conventions.

## General Development Principles

1. Prefer minimal, focused changes over unnecessary refactoring.
2. Preserve existing architecture and integration contracts.
3. Avoid introducing dependencies unless they provide clear value.
4. Prefer native Bash, Python standard-library functionality, and existing system utilities.
5. Keep the Docker image reproducible and minimize unnecessary image layers.
6. Never introduce proprietary MetaTrader or Microsoft binaries into the base image.
7. Keep documentation consistent with actual implementation, concise, and easy to read.
8. Do not silently change existing environment variable defaults or behavior.
9. Avoid unrelated modifications.
10. Never claim tests passed unless they were actually executed.

## Security Requirements

Security is a fundamental architectural requirement, not an optional enhancement.

### Desktop Access

- HTTPS must remain mandatory.
- Password authentication must remain mandatory.
- Passwords must be validated before services become available.
- Prefer Docker secrets through `FILE__PASSWORD`.
- Never log passwords, account credentials, API keys, or private keys.
- Do not expose desktop services through unrestricted public interfaces.
- Preserve WebSocket origin validation.
- Preserve security response headers and framing restrictions.

### Network Security

- Default deployment must bind published ports to `127.0.0.1`.
- Do not expose internal services unnecessarily.
- Never expose Wine, Python IPC, or MetaTrader management interfaces to the host network.
- Do not introduce unrestricted CORS or wildcard WebSocket origins.
- Preserve trusted-proxy validation.
- Maintain network isolation between desktop streaming and internal services.

### Container Security

Preserve the existing security posture:

- `no-new-privileges:true`.
- Drop all Linux capabilities by default.
- Add only explicitly required capabilities.
- Never use privileged containers.
- Never mount the Docker socket.
- Never add `SYS_ADMIN`.
- Do not restore terminal or sudo access.
- Keep Selkies command execution disabled.
- Do not enable microphone, webcam, or unnecessary desktop functionality.

Never add `init: true` to Compose. The image relies on s6-overlay as PID 1.

### Installer Verification

All downloaded proprietary installers must undergo Authenticode signature verification before execution.

This includes:

- MetaTrader 5.
- Microsoft Edge WebView2.
- Optional Windows Python.

Requirements:

1. Verify the actual cryptographic signature.
2. Validate the expected signer identity.
3. Reject invalid, missing, or unexpected signatures.
4. Do not trust filenames, download URLs, or certificate display names alone.
5. Preserve certificate-chain and revocation validation.
6. Never bypass verification to make an installation succeed.

### Security Regression Testing

Any change affecting authentication, TLS, origins, networking, permissions, installer verification, or container capabilities must include appropriate regression tests.

Update `tests/access.sh` when adding or modifying security behavior.

## Docker Development

### Dockerfile

The root `Dockerfile` defines the production image.

When modifying it:

- Preserve the pinned base-image digest.
- Preserve Debian 13 compatibility.
- Keep WineHQ Staging version controlled.
- Preserve WineHQ signing-key fingerprint verification.
- Avoid unnecessary build dependencies.
- Use `--no-install-recommends` for Debian packages.
- Clean package caches and temporary installation files.
- Preserve existing Wine debug-symbol stripping behavior.
- Preserve certificate fingerprint validation.
- Maintain OCI image metadata.
- Keep the health check functional.

Use explicit build arguments for configurable versions.

Do not replace pinned dependencies with floating versions without a clear justification.

### Docker Compose

`compose.yaml` is the canonical deployment configuration.

Preserve:

- Persistent `/config` storage.
- Optional `/mql5` mount.
- Docker secrets for authentication.
- Loopback-only port publishing.
- Resource limits.
- Restricted Linux capabilities.
- Graceful shutdown configuration.

One container represents one MetaTrader trading account.

Do not introduce changes that cause account data or Wine configuration to be lost during container recreation.

### Persistent Storage

The `/config` directory contains sensitive and persistent application state, including:

- Wine prefix.
- MetaTrader installation.
- Broker credentials.
- Trading history and price data.
- Charts and terminal configuration.
- TLS certificates.
- Optional Windows Python installation.

Never delete, overwrite, or recreate persistent state without explicit migration logic and appropriate safeguards.

Maintain compatibility with existing installations.

## Runtime Development

Runtime components are located under `root/opt/mt/` and related s6-overlay service directories.

Before changing runtime behavior:

1. Identify the relevant service and execution order.
2. Understand its user and permission context.
3. Inspect its dependencies.
4. Consider first-boot and subsequent-start behavior.
5. Verify interactions with Wine and MetaTrader.
6. Consider graceful shutdown and recovery behavior.

### Process Management

- Preserve s6-overlay supervision.
- Avoid unmanaged background processes.
- Handle signals and shutdown correctly.
- Ensure child processes do not survive unexpectedly.
- Avoid duplicate MetaTrader instances.
- Preserve automatic recovery of unexpected terminal exits.

### Wine

Wine is responsible for executing Windows applications.

When modifying Wine configuration:

- Preserve `WINEPREFIX=/config/.wine`.
- Maintain compatibility with WineHQ Staging.
- Avoid unnecessary prefix recreation.
- Preserve existing DLL overrides.
- Consider Wine process ownership and permissions.
- Test changes against actual MetaTrader behavior whenever possible.

### Python Integration

Windows Python is optional and controlled through `MT_PYTHON=true`.

When modifying Python integration:

- Preserve the existing Windows Python installation workflow.
- Keep Python package versions pinned and hash-verified.
- Preserve MetaTrader5 and NumPy compatibility.
- Avoid exposing additional network listeners.
- Verify MetaTrader IPC connectivity.
- Keep Python installation optional.

Do not confuse host/container Python with Windows Python running under Wine.

## Shell Scripting Standards

All shell scripts must:

- Use an appropriate shebang.
- Prefer Bash when Bash-specific functionality is needed.
- Use `set -euo pipefail` where appropriate.
- Quote variable expansions.
- Validate external inputs.
- Handle missing files and failed commands explicitly.
- Avoid unsafe `eval`.
- Avoid exposing secrets in command arguments or logs.
- Clean temporary files and resources using traps.
- Return meaningful exit codes.

Use ShellCheck-compatible syntax.

When modifying scripts, preserve their executable permissions.

Run ShellCheck on all tracked shell scripts before submitting changes.

## Python Standards

When modifying Python code:

- Prefer Python 3.14-compatible syntax for system Python components.
- Use the standard library where practical.
- Follow existing formatting and naming conventions.
- Handle exceptions explicitly.
- Avoid unnecessary dependencies.
- Never log sensitive information.
- Use isolated execution where appropriate.
- Maintain compatibility with the existing GTK and runtime environment.

Do not assume host Python and Wine Python share the same packages or runtime environment.

## Testing

Testing is mandatory for behavior-changing modifications.

### Build the Docker Image

```bash
docker build -t metatrader-docker:dev .
```

### Access and Security Tests

```bash
tests/access.sh
```

For additional host-networking tests:

```bash
tests/access.sh --host
```

These tests validate authentication, TLS, origin restrictions, security headers, network exposure, configuration refusals, and installer signature verification.

### Python Integration Tests

Start a development container with `MT_PYTHON=true`, log into a MetaTrader account, and run:

```bash
tests/python.sh
```

This validates Python-to-MetaTrader IPC connectivity and checks that unexpected network ports are not exposed.

### ShellCheck

Run ShellCheck against all tracked shell scripts:

```bash
git ls-files | while read -r f; do
  head -n 1 "$f" | grep -q -E '^#!.*[/ ](ba)?sh$' && echo "$f"
done | xargs shellcheck
```

### Documentation

Install documentation dependencies:

```bash
python3 -m venv .venv

.venv/bin/pip install \
  --require-hashes \
  -r docs/requirements.txt
```

Build documentation:

```bash
.venv/bin/mkdocs build --strict
```

Preview documentation:

```bash
.venv/bin/mkdocs serve
```

### Integration Testing

For changes involving actual MetaTrader execution, Wine, desktop initialization, or Python integration, perform additional runtime validation whenever practical.

Do not treat a successful Docker build as proof that MetaTrader works correctly.

## Documentation Requirements

Documentation is maintained using Material for MkDocs.

When modifying behavior, update the relevant documentation under `docs/`.

Pay particular attention to:

- `settings.md`: environment variables and configuration.
- `access.md`: network access and reverse proxies.
- `using.md`: MetaTrader functionality and Python.
- `operations.md`: lifecycle and updates.
- `troubleshooting.md`: common failures.
- `verify.md`: security and installer verification.
- `building.md`: development and testing procedures.

Keep README quick-start instructions synchronized with `compose.yaml`.

Never document insecure configurations as recommended defaults.

## Documentation and Commenting Standards

Maintain concise, high-signal documentation and code comments across the repository.

### Principles

- **Clear and easy to read:** Use straightforward language, active voice, and direct explanations.
- **Avoid wordiness and over-explaining:** Get directly to the point. Do not pad explanations with conversational filler, narrative detours, or redundant restatements.
- **Never lose meaning or correctness:** Brevity must never compromise technical accuracy, security nuances, edge cases, or exact commands.
- **Code comments:** Explain the *what* and technical *why* (non-obvious constraints, workarounds, security invariants, protocol behaviors). Never narrate self-explanatory code or rephrase variable/function names.
- **User-facing docs (`docs/`, `README.md`):** Prioritize scannability, clear headings, exact commands, structured bullet points, and concise troubleshooting guidance.
- **Preserve test contracts:** Never alter logged strings or test outputs that regression tests assert against when editing messages or comments.

## CI/CD

The primary image workflow is:

`.github/workflows/image.yml`

It performs:

1. ShellCheck.
2. Docker image build.
3. Access and security integration tests.
4. Trivy vulnerability scanning.
5. Secret scanning.
6. Image publication to GHCR.
7. Cosign keyless image signing.

The documentation workflow independently validates documentation.

The release workflow, `.github/workflows/release.yml`, runs release-please on every push to `main`. It keeps a release PR open with the next version and its `CHANGELOG.md` entry; merging that PR tags `vX.Y.Z` and publishes the GitHub release. A release builds no new image. The workflow uses a GitHub App token (`RELEASE_APP_CLIENT_ID` variable, `RELEASE_APP_PRIVATE_KEY` secret), not `GITHUB_TOKEN`, because a PR opened with `GITHUB_TOKEN` runs no workflows and the required checks would never report.

Preserve:

- Pinned GitHub Actions references.
- Minimal workflow permissions.
- Image provenance.
- Software Bill of Materials generation.
- Cryptographic signing.
- Vulnerability scanning.

Never weaken CI checks simply to make a failing build pass.

Security exceptions must be justified and documented in `.trivyignore.yaml`.

## Dependency Management

Renovate manages dependency updates.

When updating dependencies:

- Preserve version pinning.
- Preserve SHA-256 hashes for Python packages.
- Verify WineHQ package compatibility.
- Inspect upstream base-image changes.
- Review GitHub Actions updates.
- Consider compatibility with Debian 13 and Wine.
- Test changes involving Windows Python or MetaTrader5 manually when necessary.

Do not automatically accept dependency updates that affect security-critical runtime behavior.

## Change Management

Before implementing a modification:

1. Inspect relevant source files.
2. Understand the current implementation.
3. Identify security and compatibility implications.
4. Locate existing tests.
5. Determine whether documentation needs updating.

After implementing:

1. Review the complete diff.
2. Run relevant automated tests.
3. Run ShellCheck when shell scripts change.
4. Run Docker builds when image components change.
5. Validate documentation when documentation changes.
6. Report any tests that could not be executed.

### Avoid

- Unrequested architectural rewrites.
- Unnecessary dependencies.
- Broad permission changes.
- Unverified installer execution.
- Exposing additional network ports.
- Destructive operations on persistent storage.
- Removing security controls.
- Disabling failing tests.
- Unnecessary changes to pinned dependencies.
- Claims of successful runtime validation without actually running the container.

## Git Conventions

- Always use Conventional Commits spec.
- Always keep changes logically grouped.
- Never commit credentials, secrets, certificates, or local account data.
- Never commit generated documentation output or local build artifacts unless explicitly required.
- Always preserve existing repository conventions.
- Keep pull requests focused and document any intentional compatibility changes.

### Versions and releases

`main` takes squash merges only, so a PR's title becomes its commit on `main`, and release-please reads only that title to pick the next version. A new version means the image's contents or behavior changed:

- `feat:`: new or changed behavior. Minor version.
- `fix:`: a bug fix, or an update to what's in the image (base image, Wine, packages). Patch version.
- `feat!:` / `fix!:`, or a `BREAKING CHANGE:` footer: major version.
- `docs:`, `ci:`, `chore:`, `refactor:`, `test:`, `build:`, `style:`: no version. Use these when the image's contents and behavior stay the same, even if it gets rebuilt.

Renovate titles updates found in `Dockerfile` and `root/` as `fix(deps):`, and all other updates as `chore(deps):` (see `renovate.json`).

- Don't edit `CHANGELOG.md` or `.release-please-manifest.json` by hand; release-please updates them in its release PR (`chore(main): release X.Y.Z`), which is merged by hand when the version should go out.
- To correct a merged PR's type before its release, add the corrected message to that PR's description between a `BEGIN_COMMIT_OVERRIDE` line and an `END_COMMIT_OVERRIDE` line; release-please reads it instead of the commit. To force a version, end that message with a `Release-As: X.Y.Z` footer.
- Don't start a paragraph in a commit body with `feat:`, `fix:` or another type: release-please reads it as another commit.

## Definition of Done

A change is complete when:

- The requested functionality works as intended.
- Existing functionality remains compatible.
- Security invariants are preserved.
- Relevant tests pass.
- Shell scripts pass ShellCheck.
- Documentation is updated when necessary.
- Comments and documentation are concise, clear, easy to read, and free of unnecessary wordiness.
- No credentials or sensitive artifacts are introduced.
- The final diff contains only intentional changes.

**Priority order:** Security → Data Integrity → Reliability → Compatibility → Maintainability → Performance.

## Security and Guardrails

- Never commit secrets, credentials, tokens, or private keys.
- Always validate untrusted input at appropriate boundaries.
- Always use parameterized queries and safe APIs for external input.
- Avoid logging sensitive information.
- Always follow least-privilege principles.
- Never disable security controls to bypass errors.
