# SC-Auditor

Automated smart contract security analysis pipeline. Feeds Solidity source through static analysis, pattern detection, and LLM-assisted review, then spits out a markdown audit report with Foundry PoC scaffolds.

Built this because manual auditing is slow and I kept writing the same Slither wrapper scripts. Now it's one command.

## How It Works

```
                    ┌─────────────┐
                    │  Solidity   │
                    │   Source    │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Slither   │
                    │  Analysis   │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
      ┌───────▼──────┐ ┌──▼───────┐ ┌──▼──────────┐
      │ DeFi Pattern │ │   LLM    │ │  Standard   │
      │  Detection   │ │ Analysis │ │  Detectors  │
      └───────┬──────┘ └──┬───────┘ └──┬──────────┘
              │            │            │
              └────────────┼────────────┘
                           │
                    ┌──────▼──────┐
                    │   Report    │
                    │  Generator  │
                    └──────┬──────┘
                           │
                ┌──────────┼──────────┐
                │          │          │
         ┌──────▼───┐ ┌───▼────┐ ┌──▼────────┐
         │ Markdown │ │ Foundry│ │  Severity  │
         │  Report  │ │  PoCs  │ │  Summary   │
         └──────────┘ └────────┘ └───────────┘
```

## Features

- **Slither integration** — wraps Slither's detectors and extends them with custom checks
- **DeFi pattern detection** — catches reentrancy, oracle manipulation, flash loan attacks, MEV exposure, access control issues, proxy pitfalls, and math errors
- **LLM-assisted analysis** — feeds flagged code paths to an LLM for deeper reasoning about exploitability
- **Foundry PoC generation** — auto-generates proof-of-concept test scaffolds for confirmed findings
- **Markdown reports** — clean, structured output you can drop straight into a client deliverable

## Installation

```bash
git clone https://github.com/spectarsworld/sc-auditor.git
cd sc-auditor
pip install -e .
```

Requirements:
- Python 3.10+
- [Slither](https://github.com/crytic/slither) (`pip install slither-analyzer`)
- [Foundry](https://book.getfoundry.sh/) (for PoC generation)
- An OpenAI-compatible API key (for LLM analysis, optional)

## Usage

```bash
# Basic scan
sc-auditor scan ./contracts/Vault.sol

# Full pipeline with PoC generation
sc-auditor scan ./contracts/ --poc --output report.md

# Target specific patterns
sc-auditor scan ./contracts/ --patterns reentrancy,oracle,flash-loan

# Skip LLM analysis (offline mode)
sc-auditor scan ./contracts/ --no-llm
```

## Configuration

Create `sc-auditor.toml` in your project root:

```toml
[slither]
solc_version = "0.8.20"
exclude_detectors = ["naming-convention"]

[patterns]
enabled = ["reentrancy", "oracle", "flash-loan", "mev", "access-control", "proxy", "math"]

[llm]
model = "gpt-4"
api_key_env = "OPENAI_API_KEY"   # reads from env var
max_tokens = 4096

[report]
format = "markdown"
include_poc = true
severity_threshold = "low"       # minimum severity to include
```

## Project Structure

```
sc-auditor/
├── src/
│   ├── analyzers/
│   │   ├── slither_runner.py    # Slither wrapper & result parser
│   │   ├── pattern_detector.py  # DeFi-specific pattern matching
│   │   └── llm_reviewer.py     # LLM-assisted code review
│   ├── detectors/
│   │   ├── reentrancy.py
│   │   ├── oracle.py
│   │   ├── flash_loan.py
│   │   ├── mev.py
│   │   ├── access_control.py
│   │   ├── proxy.py
│   │   └── math.py
│   ├── generators/
│   │   ├── report.py            # Markdown report builder
│   │   └── poc.py               # Foundry PoC scaffolding
│   ├── cli.py
│   └── config.py
├── sample-reports/
├── tests/
├── sc-auditor.toml
└── README.md
```

## Sample Output

Check [`sample-reports/`](./sample-reports/) for example audit reports generated by the tool.

## Contributing

PRs welcome. If you've got a detector idea or pattern that's missing, open an issue first so we can talk about scope.

1. Fork it
2. Create your branch (`git checkout -b detector/new-pattern`)
3. Write tests
4. Submit a PR

## License

MIT

---

[@spectarsworld](https://github.com/spectarsworld)
