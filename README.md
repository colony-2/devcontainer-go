# devcontainer-go

A Go 1.24 library for interpreting VS Code `devcontainer.json` files and turning them into runnable `docker run` configurations or fully managed container lifecycles. It is designed for agents and automation that need to reason about devcontainers without shelling out to the VS Code CLI.

## Quick Start

```go
import (
	"context"
	"path/filepath"

	"github.com/colony-2/devcontainer-go/pkg/api"
	"github.com/colony-2/devcontainer-go/pkg/devcontainer"
)

func buildCommand(repo string) ([]string, error) {
	dc, err := devcontainer.LoadDevContainer(filepath.Join(repo, ".devcontainer", "devcontainer.json"))
	if err != nil {
		return nil, err
	}

	cfg, err := devcontainer.BuildDockerRunCommand(dc, repo)
	if err != nil {
		return nil, err
	}
	if err := cfg.Validate(); err != nil {
		return nil, err
	}
	return cfg.ToDockerRunArgs(), nil
}

func manageContainer(ctx context.Context, repo string) (string, error) {
	mgr, err := devcontainer.NewManager()
	if err != nil {
		return "", err
	}
	defer mgr.Close()

	_ = mgr.ConfigureMounts([]api.Mount{
		{Type: "bind", Source: "/tmp/cache", Target: "/cache"},
	})

	id, err := mgr.Create(ctx, repo)
	if err != nil {
		return "", err
	}
	defer mgr.Remove(ctx, id)

	if err := mgr.Start(ctx, id); err != nil {
		return "", err
	}
	_, err = mgr.Exec(ctx, id, []string{"go", "test", "./..."})
	return id, err
}
```

## What's Supported
- Parsing `.devcontainer/devcontainer.json` (image definitions, features, mounts, ports, lifecycle commands, `runArgs`, `${localEnv:VAR}` expansion).
- Building validated `DockerRunConfig` structs and CLI arguments with deduplicated ports, normalized mounts, and automatic workspace bindings.
- Docker lifecycle management through `devcontainer.Manager` (create/start/stop/remove/exec) plus optional interactive terminal attachment.
- Custom mount injection via `Manager.ConfigureMounts`, including conflict-aware merges with existing object-style mounts.
- Dry-run and validation utilities (`ValidateDockerCommand`, `ExtractDockerImage`, `DryRunDockerCommand`) for gating agent actions before invoking Docker.

## What's Not Yet Supported
- Docker Compose-based devcontainers: the schema is parsed but only single-container `docker run` flows are generated today.
- `pkg/api.NewManager` returns a stub; consumers should instantiate `pkg/devcontainer.Manager` directly.
- WebSocket terminal streaming, registry auth plumbing, and remote Docker contexts are placeholders.
- Non-Docker engines (Podman/Containerd) and Windows container hosts have no adapters yet.

## Test Coverage
- `pkg/devcontainer` has extensive unit suites covering config parsing, mount merging, lifecycle script generation, variable expansion, and Docker CLI validation (`*_test.go` files such as `mount_test.go`, `merge_test.go`, `validation_test.go`).
- Docker client behavior is exercised via `docker_test.go` (mocked) and `docker_real_test.go`/`integration_test.go`, which hit a real daemon when available to verify mounts, port bindings, and image pulls.
- Terminal flows (`terminal_test.go`, `terminal_integration_test.go`) assert PTY resizing and cleanup logic.
- The API surface in `pkg/api` is currently stubbed and therefore untested beyond compile-time checks.

Run `go test ./...` for fast feedback; set `GOLOG=debug` or `-run Integration` to scope heavier suites when Docker access is available.
