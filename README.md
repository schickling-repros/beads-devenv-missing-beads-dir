# beads devenv module missing BEADS_DIR export

## Problem

The beads devenv module sets `BEADS_PREFIX` and `BEADS_REPO` but not `BEADS_DIR`.

The `bd` CLI can't discover the database when the `.beads/` directory is not an ancestor of the working directory.

## How it currently works

The devenv module provides a `bd()` shell wrapper that runs `bd` from inside the beads repo directory:

```bash
bd() {
  if [ ! -d "$BEADS_REPO/.beads" ]; then
    echo "[beads] Beads repo not found at ${beadsRepoPath}. Run: dt megarepo:sync" >&2
    return 1
  fi
  (cd "$BEADS_REPO" && command bd --no-daemon --no-db "$@")
}
```

This wrapper works because it changes directory into `$BEADS_REPO` before running `bd`.

## The Problem

However, if you try to use `bd` without the wrapper (or if the wrapper doesn't work for some reason), the CLI fails:

```bash
$ env | grep BEADS
BEADS_PREFIX=oep
BEADS_REPO=/home/user/project/repos/overeng-beads-public
# Note: no BEADS_DIR

$ bd list
Error: no beads database found

$ BEADS_DIR=$DEVENV_ROOT/repos/overeng-beads-public/.beads bd list
○ oep-j3x [● P2] [epic] - OTEL observability stack...
# Works!
```

## Why this matters

1. **Consistency**: Other tools can discover the beads database via `BEADS_DIR` environment variable
2. **Reliability**: The wrapper can fail in certain shell environments or when using `command bd` directly
3. **Integration**: External tools and scripts that want to integrate with beads need `BEADS_DIR`
4. **Debugging**: When troubleshooting, it's helpful to run `bd` directly without the wrapper

## Reproduction

In a devenv project that imports the beads devenv module:

```nix
# devenv.nix
{
  imports = [
    (inputs.overeng-beads-public.devenvModules.beads {
      beadsPrefix = "oep";
      beadsRepoName = "overeng-beads-public";
    })
  ];
}
```

The beads database lives at `$DEVENV_ROOT/repos/overeng-beads-public/.beads/`

Run `bd list` from the project root:

```bash
$ bd list
Error: no beads database found
```

## Expected

`bd list` should work because `BEADS_DIR` is exported by the devenv module.

## Workaround

Manually export `BEADS_DIR`:

```bash
export BEADS_DIR=$DEVENV_ROOT/repos/overeng-beads-public/.beads
bd list  # works
```

## Evidence from real project

```bash
$ env | grep BEADS
BEADS_PREFIX=oep
BEADS_REPO=/home/schickling/code/worktrees/effect-utils/schickling--2026-02-05-dt-install-overhead/repos/overeng-beads-public
# Note: no BEADS_DIR

$ cd /home/schickling/code/worktrees/effect-utils/schickling--2026-02-05-dt-install-overhead
$ bd list  # Using the wrapper
○ oep-j3x [● P2] [epic] - OTEL observability stack...
# Works via the wrapper

$ command bd list  # Bypass the wrapper
Error: no beads database found
# Fails without the wrapper
```

## Proposed Fix

The devenv module should add to its env vars in `enterShell`:

```nix
enterShell = ''
  export BEADS_PREFIX="${beadsPrefix}"
  export BEADS_REPO="$DEVENV_ROOT/${beadsRepoPath}"
  export BEADS_DIR="$DEVENV_ROOT/${beadsRepoPath}/.beads"

  # ... rest of the shell setup
'';
```

This would make `bd` work reliably without requiring the wrapper function.

## Current Module Source

From `nix/devenv-module.nix` in overeng-beads-public:

```nix
{ beadsPrefix, beadsRepoName, beadsRepoPath ? "repos/${beadsRepoName}" }: { pkgs, config, ... }:
{
  enterShell = ''
    export BEADS_PREFIX="${beadsPrefix}"
    export BEADS_REPO="$DEVENV_ROOT/${beadsRepoPath}"

    # Wrapper: run bd in --no-db mode from the beads repo directory.
    # This makes `bd list`, `bd show`, etc. work from anywhere in the devenv.
    bd() {
      if [ ! -d "$BEADS_REPO/.beads" ]; then
        echo "[beads] Beads repo not found at ${beadsRepoPath}. Run: dt megarepo:sync" >&2
        return 1
      fi
      (cd "$BEADS_REPO" && command bd --no-daemon --no-db "$@")
    }
  '';

  # ... tasks and hooks
}
```

## Links

- Issue: https://github.com/overengineeringstudio/overeng-beads-public/issues/2
- Beads devenv module: https://github.com/overengineeringstudio/overeng-beads-public/blob/main/nix/devenv-module.nix
