# Mojo support for Zed

This extension combines features from [`BenWibking/zed_mojo`](https://github.com/BenWibking/zed_mojo) and [`vadim-su/zed_mojo`](https://github.com/vadim-su/zed_mojo), using the [`lsh/tree-sitter-mojo`](https://github.com/lsh/tree-sitter-mojo) grammar:

- Extension metadata in `extension.toml` and language config in `languages/mojo/config.toml`
- Tree-sitter syntax highlighting, bracket matching, auto-indentation, symbol outline, and Vim textobjects (`languages/mojo/*.scm`)
- Runnable gutter indicators and tasks for `main` (`mojo run`) and `test_*` functions (`mojo test`) in `languages/mojo/runnables.scm` and `languages/mojo/tasks.json`
- Mojo code snippets in `snippets/mojo.json`
- Mojo LSP integration through `mojo-lsp-server` with forwarding for `initialization_options` and workspace `settings`

The extension starts `mojo-lsp-server` through a small Node proxy. Mojo 1.0.0
advertises notebook synchronization, which Zed does not support; the proxy
removes that capability and forwards the rest of the protocol unchanged,
including semantic-token delta support.

## Requirements

The extension finds or installs `mojo-lsp-server` in this lookup order:

1. The path configured in Zed's `lsp.mojo-lsp-server.binary.path` setting
2. A workspace monorepo SDK under `.derived/bin/mojo-lsp-server`
3. The workspace's default Pixi environment under `.pixi/envs/default/bin/mojo-lsp-server`
4. A workspace virtual environment under `.venv/bin/mojo-lsp-server`
5. The worktree shell's `PATH`
6. A managed Mojo 1.0.0 installation downloaded from Modular's stable Conda
   channel into Zed's extension storage

The managed installation supports Apple Silicon, Linux x86-64, and Linux
ARM64. Native Windows is not supported by Mojo; use Zed in WSL instead. The
extension verifies the published SHA-256 hashes before extracting or launching
the downloaded packages. Downloads are subject to the Modular software license.

For Conda/Pixi installations, the extension derives `MODULAR_HOME` from the
selected server path so the LSP can find `share/max/modular.cfg` and the Mojo
standard library. An explicit `binary.env.MODULAR_HOME` setting overrides the
derived value.

By default, the server is started with `--log=error` (without `--skip-docstring-checks`). You can override the binary path, arguments (such as extra `-I` import search paths), environment variables, `initialization_options`, and `settings` in Zed's `settings.json`:

```json
{
  "lsp": {
    "mojo-lsp-server": {
      "binary": {
        "path": "/path/to/mojo-lsp-server",
        "arguments": ["--log=error", "-I", "/path/to/mojo/packages"],
        "env": {
          "MODULAR_HOME": "/path/to/prefix/share/max"
        }
      },
      "initialization_options": {},
      "settings": {}
    }
  }
}
```

Projects with a local default Pixi environment can be opened normally. To use a
named Pixi environment, launch Zed from that environment or configure its
`mojo-lsp-server` path explicitly:

```sh
pixi shell
zed .
```

## Snippets

The extension includes snippets in `snippets/mojo.json` for common Mojo forms such as `fn main`, `def main`, `struct`, `trait`, `alias`, `var`, loops, conditionals, and tests. They appear alongside normal completion items.

## Running Mojo files from Zed

Zed runs code through Tasks. `languages/mojo/runnables.scm` marks `def main` / `fn main` with the tag `mojo-main` and `test_*` functions with `mojo-test`, bound to default tasks in `languages/mojo/tasks.json` (`mojo run "$ZED_FILE"` and `mojo test "$ZED_FILE"`).

When `gutter.runnables` is enabled, inline runnable indicators appear next to `main` and `test_*` functions. You can override the default tasks per project in `.zed/tasks.json` (or globally via `zed: open tasks`) by defining tasks with `"tags": ["mojo-main"]` or `"tags": ["mojo-test"]`—for example, to run via `pixi run mojo`.

After installing or updating the dev extension, reload Zed with `zed: reload window` if runnables do not appear immediately.

## Formatting

`mojo-lsp-server` does not advertise `documentFormattingProvider`, and Zed extensions cannot register external CLI formatters directly. To enable format-on-save or `editor: format`, configure `mojo format` in your Zed user or project `.zed/settings.json`:

```json
{
  "languages": {
    "Mojo": {
      "formatter": {
        "external": {
          "command": "mojo",
          "arguments": ["format", "-q", "-"]
        }
      }
    }
  }
}
```

If you use `pixi` without activating a shell globally, use `"command": "pixi"` with `"arguments": ["run", "mojo", "format", "-q", "-"]` (or point `"command"` to `.venv/bin/mojo`).

## Local development

In Zed, run `zed: install dev extension` and select this repository. (On Windows with WSL, you can select the repo via `\\wsl.localhost\<Distro>\...`; if Cargo reports a file-locking error for incremental compilation over `\\wsl.localhost`, set `incremental = false` under `[build]` in your Windows `%USERPROFILE%\.cargo\config.toml`.)

After changing extension Rust code, reinstall or reload the dev extension (`zed: reload extensions`) so Zed rebuilds `extension.wasm`.

You can run the local validation checks before reloading:

```sh
cargo fmt --check
cargo test
cargo check --target wasm32-wasip2
bash scripts/check-highlight-order.sh
bash scripts/check-indents.sh
bash scripts/check-runnables.sh
bash scripts/check-snippets.sh
```

If Zed crashes or syntax highlighting disappears when opening a `.mojo` file, run `zed: open log` and check for Mojo Tree-sitter query errors (or delete any stale `grammars/` cache before reloading).

If LSP features do not start, check `zed: open log` for LSP startup errors and verify that Zed can find `mojo-lsp-server`:

```sh
which mojo-lsp-server
```

Repeated `mojo-lsp-server failed: server shut down` messages mean the server process exited and Zed is still draining stale requests; reload the window (`zed: reload window`) to restart it.

## License & Attribution

- Base extension, managed LSP installer, and Node proxy: [`BenWibking/zed_mojo`](https://github.com/BenWibking/zed_mojo) (BSD-3-Clause; see `LICENSE`).
- Snippets, runnables, tasks, config, check scripts, and LSP settings forwarding: [`vadim-su/zed_mojo`](https://github.com/vadim-su/zed_mojo) (MIT).
- Grammar and Tree-sitter queries (`highlights.scm`, `outline.scm`, `textobjects.scm`): [`lsh/tree-sitter-mojo`](https://github.com/lsh/tree-sitter-mojo) (MIT; `textobjects.scm` retains its upstream MPL-2.0 header).
