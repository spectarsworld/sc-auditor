# SC-Auditor

Automated smart contract security analysis pipeline. Runs static analysis, detects common DeFi attack patterns, generates LLM-assisted findings, and produces Foundry proof-of-concept exploits — all outputting a clean markdown audit report.

Built for security researchers who want to move faster without sacrificing depth.

## Features

- **Slither integration** — runs static analysis and parses detector output into structured findings
- **DeFi pattern detection** — custom detectors for:
  - Reentrancy (cross-function, read-only, cross-contract)
  - Oracle manipulation and price feed staleness
  - Flash loan attack surfaces
  - MEV/sandwich vulnerabilities
  - Access control misconfigurations
  - Upgrade proxy storage collisions
  - Unsafe math and precision loss
- **LLM-assisted analysis** — feeds contract context to an LLM for logic-level review that static tools miss
- **PoC generation** — automatically scaffolds Foundry test files to validate findings
- **Report generation** — outputs professional markdown reports with severity ratings, descriptions, and recommendations

## Architecture

```
┌─────────────────────────────────────────────────┐
│                   SC-Auditor                     │
├──────────┬──────────┬───────────┬───────────────┤
│  Slither │  DeFi    │   LLM     │   Foundry     │
│  Static  │  Pattern │  Analysis │   PoC Gen     │
│  Analysis│  Engine  │           │               │
├──────────┴──────────┴───────────┴───────────────┤
│              Finding Aggregator                  │
│         (dedup, severity scoring, ranking)       │
├─────────────────────────────────────────────────┤
│              Report Generator                    │
│           (markdown / JSON output)              │
└─────────────────────────────────────────────────┘
```

**Flow:**

1. Target contract(s) are ingested and compiled
2. Slither runs all built-in detectors
3. DeFi pattern engine scans for domain-specific vulnerabilities
4. LLM receives contract source + static analysis context for deeper review
5. Findings are aggregated, deduplicated, and severity-scored
6. Foundry PoC templates are generated for High/Critical findings
7. Final report is rendered to markdown

## Installation

```bash
git clone https://github.com/hybridnand/sc-auditor.git
cd sc-auditor
pip install -e .
```

**Requirements:**
- Python 3.10+
- [Slither](https://github.com/crytic/slither) (`pip install slither-analyzer`)
- [Foundry](https://book.getfoundry.sh/getting-started/installation) (for PoC generation)
- OpenAI API key (for LLM analysis, optional)

## Usage

```bash
# Audit a single contract
sc-auditor analyze ./contracts/Vault.sol

# Audit with PoC generation
sc-auditor analyze ./contracts/Vault.sol --poc

# Audit an entire project (auto-detects Hardhat/Foundry)
sc-auditor analyze ./project/ --framework foundry

# Skip LLM analysis (static + patterns only)
sc-auditor analyze ./contracts/Vault.sol --no-llm

# Output JSON instead of markdown
sc-auditor analyze ./contracts/Vault.sol --format json
```

### Configuration

Create `sc-auditor.toml` in your project root:

```toml
[analysis]
severity_threshold = "medium"    # minimum severity to report
max_llm_findings = 20            # cap on LLM-generated findings
generate_poc = true

[llm]
model = "gpt-4"
temperature = 0.1

[detectors]
enabled = ["reentrancy", "oracle", "flash_loan", "access_control", "mev", "proxy", "math"]
```

## Sample Output

```
SC-Auditor v0.3.0 — Smart Contract Security Analysis

[*] Target: VulnerableVault.sol
[*] Compiler: solc 0.8.19
[*] Running Slither analysis... 14 detectors triggered
[*] Running DeFi pattern detection... 3 patterns matched
[*] Running LLM analysis... 2 additional findings
[*] Aggregating findings... 5 unique (1 Critical, 2 High, 1 Medium, 1 Low)
[*] Generating PoCs for 3 findings...
[*] Report written to ./reports/VulnerableVault-Audit.md

Done. 5 findings across 1 contract.
```

See [sample-reports/VulnerableVault-Audit.md](./sample-reports/VulnerableVault-Audit.md) for a full example report.

## Project Structure

```
sc-auditor/
├── src/
│   ├── analyzers/
│   │   ├── slither_runner.py      # Slither integration
│   │   ├── defi_patterns.py       # Custom DeFi detectors
│   │   └── llm_analyzer.py        # LLM-assisted review
│   ├── poc/
│   │   ├── generator.py           # Foundry PoC scaffolding
│   │   └── templates/             # PoC templates per vuln type
│   ├── reporting/
│   │   ├── aggregator.py          # Finding dedup and scoring
│   │   ├── markdown.py            # Markdown report renderer
│   │   └── json_output.py         # JSON output
│   └── cli.py                     # CLI entry point
├── tests/
├── sample-reports/
└── sc-auditor.toml
```

## Contributing

Contributions welcome, especially:

- New DeFi pattern detectors
- PoC templates for additional vulnerability classes
- Improved finding deduplication logic
- Integration with other static analysis tools (Mythril, Aderyn)

Open an issue first to discuss what you're working on. PRs without context will probably sit there.

## License

MIT
