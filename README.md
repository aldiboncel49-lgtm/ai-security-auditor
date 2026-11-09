# AI Security Auditor — a Claude Code skill

A multi-phase Solidity security auditor that runs reconnaissance, multi-mindset detection, exploit-context verification and confidence-scored verification, cross-referencing static analysis tools, and outputs JSON findings plus a Markdown report.

## Install

```bash
git clone https://github.com/aldiboncel49-lgtm/ai-security-auditor
```

Then copy `.claude/skills/security-scan/` into your project (or copy `SKILL.md` as `.claude/skills/security-scan/SKILL.md`).

## Run

In Claude Code, with the target contracts in scope:

```
/notobraddo-audit
```

## What it detects

Reentrancy, access-control failures, oracle manipulation, flash-loan vectors, integer/rounding errors, token and share/accounting bugs, spec/invariant violations and edge-case failures.

## Architecture choices

- **Pipeline:** Multi-Phase Pipeline (recon → detection → verification → reporting)
- **Detection:** Multi-Mindset Detection (Attacker, Accountant, Spec Auditor, Edge Case Hunter)
- **Verification:** Confidence Scoring (0–100, four factors, 70+ HIGH) with kill-gate filtering
- **Tool integration:** Static Analysis (Slither, Aderyn) — findings are hints, not verdicts
- **Output:** JSON + Markdown (`.audit/findings.json` + console summary)

## License

MIT — see [LICENSE](LICENSE).
