# CarbonDB

Measure the footprint of AI-assisted coding. Ask your agent to "start carbon
tracking", work, then "stop carbon tracking", and get a report of:

- **Energy and CO2e of this computer** while tracking ran, measured with [CodeCarbon](https://codecarbon.io)
- **Tokens** the agent session sent and received, as billed
- **Provider-side energy, CO2e and water** of generating those tokens, estimated with [EcoLogits](https://ecologits.ai)

Every run is kept in one local SQLite database, so runs can be compared.
Works with Claude Code and Codex. Tested on Linux; written for macOS and
Windows, where `carbon doctor` tells you whether it works.

## Install

You need [uv](https://docs.astral.sh/uv/getting-started/installation/).

```bash
uv tool install git+https://github.com/seankellyhp/CarbonDB
carbon install-skill
carbon doctor
```

Restart your agent afterwards. `carbon doctor` should print `ok` on every
line; a `PROBLEM` line says what to do.

Or paste this into your agent:

> Install CarbonDB by following https://github.com/seankellyhp/CarbonDB#instructions-for-the-agent

### Instructions for the agent

1. Check that `uv --version` works. If not, stop and ask the user to install uv.
2. Run the three commands above. Allow 5 minutes for the first.
3. Show the user the output of `carbon doctor` verbatim.
4. If a command refuses because something already exists, stop and ask the
   user; see [Upgrading from carbon-skill](#upgrading-from-carbon-skill).
   Never use `sudo` or `pip install`.

## Use

In your agent: "start carbon tracking", "how much energy so far?", "stop
carbon tracking and show me the report", "compare the last two runs". Or in a
terminal:

```bash
carbon start [label]     # start tracking
carbon end [run]         # stop and print the report
carbon report [run]      # report for a past or running measurement
carbon compare A B       # B against A
carbon list              # every run, newest first
carbon export [run]      # samples as CodeCarbon's emissions.csv
```

Set `CARBON_COUNTRY` (e.g. `NLD`) to choose your electricity grid instead of
having it looked up from your IP address. That lookup is the only thing that
leaves your computer.

## Update and uninstall

```bash
uv tool upgrade carbondb          # update
carbon install-skill --remove     # remove the skill (keeps your measurements)
uv tool uninstall carbondb        # remove the program
```

### Upgrading from carbon-skill

Until 0.3 this was `carbon-skill`, inside
[eco-measure-amsterdam](https://github.com/seankellyhp/eco-measure-amsterdam).
Run `uv tool uninstall carbon-skill` first, then install as above. Your
measurements are kept.

## More

[GUIDE.md](GUIDE.md) covers measured versus estimated CPU power,
settings, the electricity grid, where data is stored, reducing the footprint
and troubleshooting. To work on the code: clone, then `uv run pytest` and
`uv run carbon ...`.

MIT, see [LICENSE](LICENSE).
