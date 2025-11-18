# Repository Guidelines

## Project Structure & Module Organization
This module targets Go 1.24.1 (`go.mod`) and keeps all exported code in `pkg`. `pkg/api` exposes the narrow public surface, while `pkg/devcontainer` holds the Docker and lifecycle logic plus nearly all fixtures. Tests live next to the code (`*_test.go`) so new packages should follow the same colocated pattern to inherit helper utilities already available there.

## Build, Test, and Development Commands
- `go test ./...` — runs the full suite, including e2e and integration cases under `pkg/devcontainer`.
- `go test ./pkg/devcontainer -run Integration` — scopes to heavier integration paths when iterating on Docker behaviors.
- `go vet ./...` — performs static analysis; run before opening a PR.
- `gofmt -w pkg` — enforce formatting whenever introducing new files. Pair with `golangci-lint run` if you have it installed locally for quicker hygiene checks.

## Coding Style & Naming Conventions
Adhere to standard Go formatting (`gofmt`) and keep imports organized via `goimports`. Use mixedCaps for exported identifiers (`DevcontainerManager`), and keep unexported helpers lowerCamel. Prefer constructor-style helpers named `NewX`. Configuration structs that map to JSON or YAML should stay in `pkg/api` to avoid leaking internal wiring. Keep files concise (~400 lines) and split by responsibility (e.g., Docker adapters vs. terminal helpers).

## Testing Guidelines
Add unit tests beside the implementation and mirror the existing `foo_test.go` naming. Integration tests that touch Docker, sockets, or terminals belong in `pkg/devcontainer/*_integration_test.go`. Favor the table-driven patterns already present in `validation_test.go`. Aim to cover new failure modes and update golden data when serialization changes. Run `GOLOG=debug go test ./pkg/devcontainer -run Terminal` if you need richer traces while debugging interactive flows.

## Commit & Pull Request Guidelines
The repository is greenfield, so adopt Conventional Commits (`feat: add docker manager mounts`, `fix: handle terminal detach`) to keep history searchable. Each PR should describe the user-facing change, call out risky dependencies (Docker daemon, tty), and link any tracking issue. Include reproduction steps or `go test` excerpts for regressions, and attach terminal screenshots if the change affects interactive UX.

## Environment & Tooling Tips
Docker must be running locally because `pkg/devcontainer/docker.go` talks to the daemon using the default socket. Set `DOCKER_HOST` when targeting remote engines, and document any non-default environment variables inside your PR. Use `wsl.exe` or a Linux host when validating pseudo-TTY code paths; macOS/Linux behave differently for PTYs, so capture both behaviors in tests when possible.
