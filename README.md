# flo

**Status:** `0.3.0` — config schema stable; breaking changes only with a 0.x minor bump.

A Textual TUI for guided decision trees. Pick an option, advance to the next step, walk back through history.

This repo is the **download channel** for flo — prebuilt, standalone binaries. No Python required.

<img width="900" height="828" alt="output2" src="https://github.com/user-attachments/assets/6fe8b459-b193-43ff-8f3c-50d009acfc67" />

## Download

Grab a binary from the [latest release](https://github.com/mkhabelaj/flo-releases/releases/latest):

| Platform | File |
|----------|------|
| Linux (x86_64) | `flo-linux-x86_64` |
| macOS (Apple Silicon) | `flo-macos-arm64` |

## Run it

The binaries are self-contained — no Python install needed. Examples below assume you renamed the download to `flo`.

**Linux / macOS:**

```sh
chmod +x flo-linux-x86_64
./flo-linux-x86_64 start
```
or if you are using `mise` add this to your `config.toml`
```toml
"github:mkhabelaj/flo-releases" = { version = "latest", bin = "flo" }
```

On macOS the binary is unsigned, so Gatekeeper blocks it on first launch. Clear the quarantine flag once:

```sh
xattr -d com.apple.quarantine ./flo-macos-arm64
```

(or right-click the file → **Open** → confirm).

The Linux binary is built against glibc 2.35 (Ubuntu 22.04), so it runs on Debian 12, Fedora, Arch, and most current distros. glibc is forward-compatible only.

## Commands

flo is a small command-line tool. The TUI does not start unless you ask for it:

- `flo start` — launch the interactive TUI.
- `flo run` — list every runnable leaf as `path<TAB>command`.
- `flo run <path>` — run a leaf without the TUI, e.g. `flo run git/status`. Supply input/picker values with `--set key=value` (repeatable); `--dry-run` (`-n`) prints the resolved command instead. Exits with the command's exit code.
- `flo config --view` — open the config builder in your browser (visual workflow editor).
- `flo config --check` — validate the config and exit (`2` on error); also prints lint warnings for unknown `{placeholders}`, dead leaves, and unmatched `when:` keys.
- `flo --version` — print the version.

## Controls

### Navigation

- `↑` / `↓` or `j` / `k` — move between options
- `Enter` or double-click — select option / advance to next step
- `Backspace` / `Esc` — return to previous step
- `Ctrl+G` — jump back to the root step (clears captured values)
- `Ctrl+L` — reload the config from disk (workflow screen)
- `Ctrl+U` — jump: search every leaf command and go straight there (workflow screen)

### Running a command

Every command goes through a two-step flow:

1. `Enter` — arms the preview and reveals execution-mode keys
2. Pick how to run:
   - `i` — interactive (suspend TUI, run in terminal, return to flo)
   - `x` — run & close (record to history, exit flo, run in the terminal — for k9s/ssh-style takeovers)
   - `c` — capture (run with stdout/stderr shown in-app)
   - `d` — dry-run (show resolved command, don't execute)
   - `y` — copy command to clipboard
   - `e` — edit the command inline, then pick any of the modes above
   - `E` — edit the command in `$EDITOR` (`$VISUAL` preferred), then pick any of the modes above

After a command finishes (or a dry-run/copy), flo returns to the step you picked it from — not the root — so nearby commands are one keypress away. Press `g` any time to go back to the root.

While a captured command (or a `from_command` picker) is running, `Esc` cancels it — the shell process is killed and you return to the previous step. Captured commands read stdin from `/dev/null`: anything that prompts for input (auth helpers etc.) fails fast instead of hanging — run those interactively (`i`) instead.

In the captured-output panel: `y` copies the output to the clipboard, `a` appends the run (command, exit code, output) to the scratch pad. In History/Rankings, `y` copies the highlighted command.

### Screens

- `Ctrl+W` — Workflow (main decision-tree screen)
- `Ctrl+R` — History (recent commands, most recent first)
- `Ctrl+Y` — Rankings (most-used commands by frequency)
- `Ctrl+S` — Settings (theme, history limit, clear history)

### Other

- `Ctrl+T` (or `:` on the workflow screen) — ad-hoc run: type or paste any command, preview it, then run in any mode; runs are recorded to history
- `Ctrl+E` — open a persistent scratch pad in `$EDITOR` (notes live in flo's data dir as `scratch.md`)
- `Ctrl+F` — toggle inline filter on the current screen (Workflow, History, Rankings)
- `Ctrl+P` — command palette (search commands and switch themes)

## Custom keybindings

Remap flo's screen and navigation keys in `~/.config/flo/keys.toml` (respects `XDG_CONFIG_HOME`). Each entry is a binding id set to either a key string, or a table with `key` and/or `show` (footer visibility):

```toml
reload = "f5"                             # remap
jump = { key = "ctrl+k", show = false }   # remap and hide from the footer
notes = { show = false }                  # keep default key, hide from footer
```

| id | default | action |
|----|---------|--------|
| `settings` | `ctrl+s` | Settings screen |
| `history` | `ctrl+r` | History screen |
| `rankings` | `ctrl+y` | Rankings screen |
| `workflow` | `ctrl+w` | Workflow screen |
| `filter` | `ctrl+f` | toggle inline filter |
| `notes` | `ctrl+e` | scratch pad in `$EDITOR` |
| `adhoc` | `ctrl+t` | ad-hoc run box |
| `palette` | `ctrl+p` | Textual command palette (key remap only, no `show`) |
| `back` | `backspace` | previous step / cancel (all screens) |
| `cancel` | `escape` | previous step / cancel (all screens) |
| `back_alt` | `h` | vim-style back (hidden) |
| `reload` | `ctrl+l` | reload config |
| `jump` | `ctrl+u` | command palette |
| `root` | `ctrl+g` | back to root step |
| `adhoc_alt` | `:` | ad-hoc run, vim-style (hidden) |

Avoid `ctrl+j`, `ctrl+m`, `ctrl+i`, `ctrl+h`, and `ctrl+[` — in the terminal protocol these are the same bytes as Enter/Tab/Backspace/Escape, so they get interpreted as those keys instead of your binding. App-level bindings take priority over focused text fields, so a remap works on every screen. Unknown ids or invalid values show a warning toast at startup and are ignored.

## Settings

The Settings screen (`Ctrl+S`) has three sections:

- **Theme** — displays the active theme name. Use `Ctrl+P` to change it.
- **History limit** — caps how many distinct commands appear in History and Rankings. Enter a number and press `Enter` to save; leave blank for unlimited (default).
- **Clear history** — deletes all command history. Two-step: press the button once to arm it, press again to confirm. `Escape` cancels.

## Configuring your own workflows

flo works out of the box — a default set of workflows ships inside the binary.

Configs load in this order:

1. `.flo.yaml` in the current directory or any parent — per-project workflows. When both a project and a user config exist, the project's root options are shown first, followed by the user's.
2. `~/.config/flo/workflows.yaml` (or `$XDG_CONFIG_HOME/flo/workflows.yaml` if set).
3. The bundled defaults.

Two ways to edit:

1. **Visual editor (easiest):** run `flo config --view` to open the browser-based config builder. **Save to flo** writes straight to `~/.config/flo/workflows.yaml` (backing up the previous file to `workflows.yaml.bak`); **Download YAML** exports a copy instead.
2. **By hand:** create `~/.config/flo/workflows.yaml` (or a project `.flo.yaml`) using the schema below.

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
  - `run` — shell command to execute, or a **list of commands** run as a sequence (joined with `&&`, so it stops at the first failure). `{key}` substitution from captured values uses identifier-only braces, so docker template syntax (`{{.Names}}`, `{{end}}`) and tmux format strings (`#{window_name}`) pass through untouched. Substituted values are shell-quoted automatically (a value with spaces stays one argument); use `{key|raw}` to opt out, e.g. to inject multiple flags.
  - `next` — the next step shown after selecting this option
  - `cwd: "/path"` — working directory for the run command (supports `{key}` substitution)
  - `env: {KEY: "value"}` — extra env vars merged over the current environment (values support `{key}`)
  - `when: "{key} == value"` — conditional visibility for options. The option only renders when the expression matches the current session values. Supports `==` and `!=` against captured keys, and multiple comparisons combined with `and` / `or` (standard precedence — `and` binds tighter), e.g. `when: "{env} == dev or {env} == staging"`.

Non-zero exit codes from a `run` command surface as a red toast `exit N: <command>` after the TUI resumes.

Quote any string containing `#`, `{`, `}`, `:`, or `'` in YAML (e.g. `"tmux list-windows -F '#{window_name}'"`) — these are otherwise special.

Errors in the config (bad YAML, missing required fields, conflicting flags) are printed before the TUI launches; flo exits with code `2`.
