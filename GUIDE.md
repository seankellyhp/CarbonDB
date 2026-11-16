# CarbonDB guide

The details behind the [README](README.md).

## Commands

```bash
carbon start  [label]          # start tracking (default label: "default"); returns at once
carbon end    [run]            # stop, then print the report
carbon report [run]            # report for a past or running measurement
carbon compare A B             # B against A, with changes
carbon compare --baseline I T  # T with idle run I subtracted
carbon list                    # history: every run, newest first, with totals
carbon export [run] [-o FILE]  # a run's samples as CodeCarbon's emissions.csv
carbon delete <run>            # delete one stopped run
```

`carbon_start`, `carbon_end` and `carbon_report` also work, as aliases.
Every `carbon start` is a new run with its own id; starting a label again keeps
the earlier runs. Wherever a command takes a run, give a label (its newest run)
or `@<id>` from `carbon list`. Without one, `end` and `report` use the run
started from the same agent session, so two sessions never stop each other's
runs; in a plain terminal they use the most recent run and say so.
`carbon list --session`, `--label L` and `-n N` narrow the history.

Tracking stops automatically when the agent session that started it ends, so
a forgotten measurement never keeps running.


## Reducing the footprint

The measurement is of the whole computer, so measure an idle baseline first,
then the task, and subtract:

```bash
carbon start idle      # touch nothing for ~60 s
carbon end   idle
carbon start task-a    # let the agent do the task
carbon end   task-a
carbon compare --baseline idle task-a
```

Then change one thing (a smaller model, clearing context between tasks,
narrower file reads), run the same task as `task-b`, and
`carbon compare task-a task-b`. The skill suggests what to try based on the
report; `carbon compare` decides whether it helped. One pair of runs can be
noise: repeat before concluding.

---

## Measuring CPU power

The first lines of `carbon start` show whether the CPU is **measured** or
**estimated**:

```
CPU: RAPL (measured)
CPU: cpu_load - ESTIMATE, not measured
```

This is CodeCarbon's own choice. The skill shows what CodeCarbon decided and
does not change it.

**Linux.** Most Intel and AMD CPUs have energy counters (RAPL), but Linux only
lets root read them, as a fix for a security issue
([CVE-2020-8694](https://nvd.nist.gov/vuln/detail/CVE-2020-8694)). Without them
CodeCarbon estimates the CPU's power from its load. CodeCarbon's own tests put
that estimate at roughly 60% too low to 190% too high. When this happens,
`carbon start` prints CodeCarbon's
[recommended fix](https://docs.codecarbon.io/latest/how-to/enable-rapl/). You
have two options:

- Until the next reboot (lets every user on the machine read the counters):

  ```bash
  sudo chmod a+r /sys/class/powercap/*/energy_uj
  ```

- Permanently, for your user only (CodeCarbon's recommended setup):

  ```bash
  sudo groupadd codecarbon
  sudo usermod -a -G codecarbon $USER
  printf '%s\n' 'SUBSYSTEM=="powercap", ACTION=="add", TEST=="energy_uj", RUN+="/bin/chgrp codecarbon /sys%p/energy_uj", RUN+="/bin/chmod 0440 /sys%p/energy_uj"' | sudo tee /etc/udev/rules.d/99-codecarbon-rapl.rules
  sudo udevadm control --reload-rules
  sudo udevadm trigger --subsystem-match=powercap --action=add
  ```

  Then log out and back in, so your new group membership takes effect.

Both options need your password, so the skill never runs them for you. In a
virtual machine or container there are usually no counters at all, and the
CPU figure stays an estimate.

**macOS.** CodeCarbon estimates CPU power from the CPU's load, and does not
use `sudo`. According to CodeCarbon's documentation, there is no known way to
read Apple Silicon energy without admin rights, so there is nothing to enable.
On a Mac, CPU figures are always estimates and the GPU is not tracked
separately.

**Windows.** CodeCarbon measures CPU energy through the Windows Energy Meter
Interface on Windows 11 running on real hardware (not a virtual machine), or
through Intel Power Gadget if it is installed. Otherwise it estimates from CPU
load, and `carbon start` says so.

**GPU.** An NVIDIA or AMD GPU is measured. An idle GPU still draws power,
though, and that power counts towards the total.

---


### Measured or estimated, per OS

| | Linux | macOS | Windows |
|---|---|---|---|
| CPU | measured (RAPL) if readable, else estimate | estimate | measured (Windows EMI, Power Gadget) if available, else estimate |
| GPU (NVIDIA, AMD) | measured | not tracked separately | measured |
| RAM | estimate | estimate | estimate |
| Tokens (Claude Code, pi) | exact | exact | exact |
| Provider inference | estimate | estimate | estimate |

---

## Settings

Set these before `carbon start`:

| Variable | Default | |
|---|---|---|
| `CARBON_COUNTRY` | detected | Country whose electricity grid sets this computer's CO2 per kWh, as an ISO-3 code (`NLD`, `DEU`, `GBR`, `USA`...). Set it to skip the location lookup. A two-letter code (`NL`) is converted; a code CodeCarbon has no grid data for is refused. |
| `CARBON_REGION` | detected | Region within the country. CodeCarbon uses it for the USA, Canada, China and the Nordic countries (e.g. `california`). |
| `CARBON_MEASURE_SECS` | `10` | Seconds between power readings, and between saved rows. |
| `CARBON_RUN_ROOT` | user data folder | Where measurements are stored. |
| `CARBON_ECOLOGITS_ZONE` | provider's own | Grid zone for the provider-side estimate, if you know where it was served. |
| `CARBON_SESSION_TRANSCRIPT` | detected | Path to the session log to count tokens from, if detection fails. |
| `CARBON_PARENT_PID` | detected | Tie the run to this process instead of the detected agent. |

---

## Which electricity grid?

Emissions are energy times the CO2 per kWh of your electricity grid, so the
skill needs to know where you are. Like CodeCarbon, it looks up your public IP
address at [get.geojs.io](https://www.geojs.io), or
[ipinfo.io](https://ipinfo.io) if that fails, once each time tracking starts.
`carbon start` shows the result:

```
Grid: NLD / north holland - Netherlands, north holland - detected from your IP address via get.geojs.io
```

If the lookup fails, CodeCarbon on its own would quietly assume Canada
(Quebec), whose grid is almost carbon-free. The skill uses the last location
it detected on your computer instead, or the Netherlands if it has never
detected one, and says so. On a VPN, the detected country is the VPN's. In
either case, set your country yourself:

```bash
CARBON_COUNTRY=DEU carbon start          # bash, zsh
env CARBON_COUNTRY=DEU carbon start      # fish
$env:CARBON_COUNTRY="DEU"; carbon start  # PowerShell
```

## What leaves your computer

Only the location lookup above, which sends your IP address to get.geojs.io
(or ipinfo.io), as CodeCarbon itself does. Setting `CARBON_COUNTRY` turns it
off. Measuring is offline, and nothing is sent to codecarbon.io. Token counts come from the session log that
your agent already keeps on your disk (`~/.claude/projects/` for Claude Code). The skill uses
only the usage numbers and timestamps in it, not what was said.

---


---

## Where things are

| What | Where |
|---|---|
| the program | uv's tool folder (`uv tool dir`), with `carbon` on your PATH |
| the skill | `~/.claude/skills/carbon/` (Claude Code, respects `CLAUDE_CONFIG_DIR`), `~/.agents/skills/carbon/` (Codex) |
| your measurements | `~/.local/share/carbondb/runs/` (Linux), `~/Library/Application Support/carbondb/runs/` (macOS), `%LOCALAPPDATA%\carbondb\carbondb\runs\` (Windows) |

Every run is kept in one SQLite database, `carbon.db`, in that folder;
`carbon start` prints its path as `Data:`. It holds the samples CodeCarbon
measured (one cumulative row per interval, so the last is the run's total),
the tokens of each API call, and the EcoLogits estimate for each, saved when
the run ends, so a report still works after your agent deletes old session
logs. The tables are described in the skill's `reference/platforms.md`.

CodeCarbon's raw output is one export away:
`carbon export default > emissions.csv` writes the newest `default` run with
CodeCarbon's own `emissions.csv` columns, so notebooks that read CodeCarbon
files keep working. `carbon export --format sql -o backup.sql` writes the
whole database as SQL text. Each run also has a small folder named after its
id with CodeCarbon's log, `monitor.log`. Set `CARBON_RUN_ROOT` to store
everything somewhere else (not a synced folder such as OneDrive). Run folders
from older versions are imported on first use and moved to `imported/`.

Measurements live outside the skill folder, so reinstalling or uninstalling
the skill keeps them.

## Upgrading from the bash version

`uv tool install` refuses to overwrite the old `~/.local/bin/carbon_start`,
`carbon_end` and `carbon_report` wrappers. Delete those three files first (or
pass `--force` to `uv tool install`). Then `carbon install-skill` replaces the
old skill folder and moves its past runs to the new measurements folder. If
the old install predates the installer's marker file, `install-skill` refuses
to touch it; `carbon install-skill --force` moves it to
`~/.claude/carbondb-backup-<time>/` (your old runs stay there, in `runs/`).

An old run is left where it was, in `~/.claude/skills/carbon/runs/`, if a run
with the same label already exists in the new folder. `install-skill --remove`
then removes the skill but keeps that `runs/` folder, and says so.

---

## Troubleshooting

| Symptom | Fix |
|---|---|
| `carbon: command not found` | Run `uv tool update-shell`, then open a new terminal. Or run it from a clone with `uv run carbon`. |
| `/carbon` does not appear in your agent | Restart the agent. Skills are loaded when a session starts. Run `carbon doctor`. |
| `Monitor failed to start` | The lines printed after it come from CodeCarbon's log; `carbon doctor` repeats the test. |
| `Tokens: session NOT identified` | Energy is still measured. To count tokens, set `CARBON_SESSION_TRANSCRIPT` to the session's `.jsonl` file before `carbon start`. |
| A run says `stopped by: monitor died` | The monitor was killed (sleep, reboot, task manager). Figures are those of the last saved row, at most one interval old. |
| Anything else | Look in the run's `monitor.log`, in the folder named after its id next to `carbon.db` (`carbon start` prints the database path and the id), then run `carbon doctor`. |

---

