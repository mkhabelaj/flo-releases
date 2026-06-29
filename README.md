# flo

**Status:** beta (`0.1.0b1`) — usable, API may change before 1.0.

A Textual TUI for guided decision trees. Pick an option, advance to the next step, walk back through history.

This repo is the **download channel** for flo — prebuilt, standalone binaries. No Python required.

## Download

Grab a binary from the [latest release](https://github.com/mkhabelaj/flo-releases/releases/latest):

| Platform | File |
|----------|------|
| Linux (x86_64) | `flo-linux-x86_64` |
| macOS (Apple Silicon) | `flo-macos-arm64` |
| Windows (x86_64) | `flo-windows-x86_64.exe` |

> The Windows build is best-effort and may occasionally lag behind a release.

## Run it

The binaries are self-contained — no Python install needed. Examples below assume you renamed the download to `flo`.

**Linux / macOS:**

```sh
chmod +x flo-linux-x86_64
./flo-linux-x86_64 start
```

On macOS the binary is unsigned, so Gatekeeper blocks it on first launch. Clear the quarantine flag once:

```sh
xattr -d com.apple.quarantine ./flo-macos-arm64
```

(or right-click the file → **Open** → confirm).

**Windows:**

```sh
flo-windows-x86_64.exe start
```

The Linux binary is built against glibc 2.35 (Ubuntu 22.04), so it runs on Debian 12, Fedora, Arch, and most current distros. glibc is forward-compatible only.

## Commands

flo is a small command-line tool. The TUI does not start unless you ask for it:

- `flo start` — launch the interactive TUI.
- `flo config --view` — open the config builder in your browser (visual workflow editor).
- `flo --version` — print the version.

## Controls

### Navigation

- `↑` / `↓` or `j` / `k` — move between options
- `Enter` or double-click — select option / advance to next step
- `Backspace` / `Esc` — return to previous step

### Running a command

Every command goes through a two-step flow:

1. `Enter` — arms the preview and reveals execution-mode keys
2. Pick how to run:
   - `i` — interactive (suspend TUI, run in terminal)
   - `c` — capture (run with stdout/stderr shown in-app)
   - `d` — dry-run (show resolved command, don't execute)
   - `y` — copy command to clipboard
   - `e` — edit the command before running interactively

### Screens

- `Ctrl+W` — Workflow (main decision-tree screen)
- `Ctrl+R` — History (recent commands, most recent first)
- `Ctrl+Y` — Rankings (most-used commands by frequency)
- `Ctrl+S` — Settings (theme, history limit, clear history)

### Other

- `Ctrl+F` — toggle inline filter on the current screen (Workflow, History, Rankings)
- `Ctrl+P` — command palette (search commands and switch themes)

## Settings

The Settings screen (`Ctrl+S`) has three sections:

- **Theme** — displays the active theme name. Use `Ctrl+P` to change it.
- **History limit** — caps how many distinct commands appear in History and Rankings. Enter a number and press `Enter` to save; leave blank for unlimited (default).
- **Clear history** — deletes all command history. Two-step: press the button once to arm it, press again to confirm. `Escape` cancels.

## Configuring your own workflows

flo works out of the box — a default set of workflows ships inside the binary.

To customise, flo reads `~/.config/flo/workflows.yaml` (or `$XDG_CONFIG_HOME/flo/workflows.yaml` if `XDG_CONFIG_HOME` is set). If that file doesn't exist, the bundled defaults are used.

Two ways to edit:

1. **Visual editor (easiest):** run `flo config --view` to open the browser-based config builder.
2. **By hand:** create `~/.config/flo/workflows.yaml` and write it using the schema below.

```sh
mkdir -p ~/.config/flo
$EDITOR ~/.config/flo/workflows.yaml
```

Schema (each node is a mapping with a `type` field):

- **step** — a question screen
  - `description` — small dim help line shown above the options
  - `options` — list of option nodes (omit if dynamic/input)
  - `key` — capture the picked/typed value under this name for later `{key}` substitution
  - `from_command` — shell command whose stdout lines become options at display time. Supports `{key}` substitution from previously-captured values, e.g. `"tmux list-panes -t {window} -F '#{pane_index}'"`. Each line may contain a tab: the part **before** the tab is the rich label the user sees; the part **after** is the bare value captured under `key` and used in downstream `{key}` substitutions. Lines without a tab keep the simple behavior (label == value). Example with rich labels: `"tmux list-panes -t {window} -F '#{pane_index}: #{pane_current_command}  #{pane_current_path}\t#{pane_index}'"`.
  - `input: true` — free-text input instead of options
  - `default` — pre-fill the input field (input steps only)
  - `placeholder` — placeholder text shown when the input is empty (input steps only)
  - `validate` — regex the submitted value must fully match; mismatch shows an inline red error and blocks submit (input steps only)
  - `next` — for `from_command` / `input` steps, the step or option that follows
- **option** — a leaf choice
  - `name` *(required)* — the label shown
  - `run` — shell command to execute. `{key}` substitution from captured values uses identifier-only braces, so docker template syntax (`{{.Names}}`) and tmux format strings (`#{window_name}`) pass through untouched.
  - `next` — the next step shown after selecting this option
  - `cwd: "/path"` — working directory for the run command (supports `{key}` substitution)
  - `env: {KEY: "value"}` — extra env vars merged over the current environment (values support `{key}`)
  - `when: "{key} == value"` — conditional visibility for options. The option only renders when the expression matches the current session values. Supports `==` and `!=` against captured keys.

Non-zero exit codes from a `run` command surface as a red toast `exit N: <command>` after the TUI resumes.

Quote any string containing `#`, `{`, `}`, `:`, or `'` in YAML (e.g. `"tmux list-windows -F '#{window_name}'"`) — these are otherwise special.

Errors in the config (bad YAML, missing required fields, conflicting flags) are printed before the TUI launches; flo exits with code `2`.
