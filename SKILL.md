---
name: notobraddo-auditor
description: Multi-phase security auditor for Solidity smart contracts
invocation: /notobraddo-audit
---

## Prerequisites

Before running this audit skill, run static analysis tools on the project:

### Option 1: Slither (Python)
Install: `pip install slither-analyzer`
Run: `slither . --json .audit/slither-output.json`

### Option 2: Aderyn (Rust)
Install: `cargo install aderyn`
Run: `aderyn . --output .audit/aderyn-report.md`

If you have run either tool, this skill will automatically read the output and cross-reference findings.

## Tool Integration

When starting the audit, check for static analysis results:

1. Look for `.audit/slither-output.json` in the project root
   - If found: read and extract detector findings (name, severity, file, line)
   - Use these as additional signals during detection
2. Look for `.audit/aderyn-report.md` in the project root
   - If found: read and extract issues
   - Use these as additional signals during detection

### How to Use Tool Findings
Static analysis findings serve as HINTS, not final results:
- If a tool finding overlaps with an AI finding → BOOST confidence
- If a tool finds something the AI missed → Investigate further
- If the AI finds something tools missed → Keep it (tools miss semantic issues)
- Tool-only findings without AI corroboration → Include but mark as "Tool-detected, needs review"

If no tool output files are found, proceed with AI-only analysis and note in the report: "Static analysis tools not run — AI-only analysis"

# Multi-Phase Security Auditor

## When Invoked

### Phase 1: Reconnaissance

1. **File Inventory**: List all `.sol` files, line counts, and purposes (contracts, libraries, interfaces)
   - For each contract, also record: inheritance chain, state variables, and function visibility
2. **Architecture Map**: Create a system diagram (mermaid or text) showing:
   - Contract relationships and inheritance
   - External dependencies (oracles, tokens, other protocols)
   - Trust boundaries and access control roles
3. **Entry Point Catalog**: List all external and public functions, noting:
   - State changes made
   - Value transfers (ETH, tokens)
   - Access controls (owner-only, role-based, public)
4. **Value Flow Analysis**: Trace how value moves through the system:
   - Deposits, withdrawals, transfers
   - Fee collection and distribution
   - Token minting and burning
5. **Inspect Dependencies**: For each external call (token, oracle, etc.), note:
   - Assumed behavior (e.g., "token transfer succeeds")
   - What could break (e.g., "oracle price manipulation")
   - Whether dependency is trusted
6. **Produce a Recon Summary**: Before proceeding, summarize:
   - Top 3 highest-risk areas
   - Complex interactions that need deep analysis
   - Any assumptions about the protocol

### Phase 2: Detection

Starting with the highest-risk files from Phase 1, analyze each contract for:

#### A. Reentrancy
Check all external calls before state changes. Flag:
- ETH transfers before state updates
- Token transfers integrated with logic

#### B. Access Control
Verify:
- All privileged functions have access controls
- No missing modifiers (onlyOwner, etc.)
- Role assignments are correct

#### C. Oracle Manipulation
Check price feed usage:
- Are prices from a reliable source?
- Flash loan manipulation possible?

### Phase 3: Verification

For each candidate finding, validate carefully:

1. **Prove Impact**: Is the impact real?
   - Can funds be stolen or locked?
   - Can governance be manipulated?
   - Can the protocol be paused (DoS)?
2. **Verify Preconditions**: What is required (attacker role, collateral, timing)?
   - Why is it possible?
   - Can an attacker exploit it?
   - Are there assumptions?
3. **Check Edge Cases**:
   - Does it still work at the boundary?
4. **Eliminate False Positives**:
   - Any safeguards that prevent the attack?
   - Is the protocol already protected?
   - Rules that make this scenario unlikely?
5. **Test Exploitability**:
   - Can an attacker exploit this?
   - Are barriers too high?
   - Would it be worth an attacker's effort?
6. **Assign Severity**:
   - Critical: Can steal/lock funds or break system
   - High: Can manipulate governance or cause major loss
   - Medium: Can cause minor loss or DoS
   - Low: Small impact, theoretical concern

**Kill Conditions** — If ANY of these is true, DISCARD the finding:
- The attack requires privileged access the attacker doesn't have
- The vulnerable code is unreachable
- A safeguard already prevents exploitation
- The impact is negligible
- The code is not deployed or not in scope

### Phase 4: Reporting

## Report Format

For each confirmed finding, include:
[SEVERITY] Title of the finding

- **Description**: Concise summary of the issue.
- **Location**: `file.sol:line` of the bug.
- **Exploit Path**: Step-by-step attack scenario.
- **Impact**: What an attacker can achieve (loss in USD or ETH).
- **Recommendation**: How to fix the issue.

## Reconnaissance Phase

Before analyzing for vulnerabilities, perform these steps:

### 1. File Inventory
List all Solidity (.sol) files in scope. For each file, report:
- File path
- Line count
- Whether it is a contract, interface, library, or abstract contract
- Number of external/public functions
- Number of state variables

### 2. Risk Scoring
For each contract (not interfaces or libraries), calculate a rough risk score:
- +5 for each external call (address.call, token.transfer, etc.)
- +4 for each state-modifying external/public function
- +4 for each payable function
- +3 for each use of assembly or unchecked blocks
- +1 for each 20 lines of code

Sort files by risk score descending. The top files get the deepest analysis.

### 3. Architecture Overview
Identify:
- Contract inheritance chains
- Which contracts call which other contracts
- Trust boundaries (who can call what, access control patterns)
- External protocol integrations (oracles, DEXes, lending pools)

### 4. Entry Point Catalog
List all external/public functions with:
- Function signature
- Whether it modifies state
- Whether it handles value (ETH or tokens)
- Access control (onlyOwner, role-based, or unrestricted)

Output this as a "Recon Summary" section BEFORE proceeding to detection.

## Exploit Context

When analyzing for vulnerabilities, reference these real-world examples to calibrate your severity judgments. For each finding, ask: "Has this exact pattern caused a real exploit?"

### Reentrancy Examples
- DAO (2016): $60M. Recursive calls drained funds before state updates.
- Curve (2023): $62M. Vyper reentrancy protection bypass in same-function reentrancy.
- Fei Rari (2022): $80M. Reentrancy through exit() function in Fuse pools.

Pattern to detect: External calls before state changes. Checks-Effects-Interactions violations.

### Oracle Manipulation Examples
- Mango Markets (2022): $114M. Attacker pumped spot price of MNGO token, borrowed against inflated collateral.
- Inverse Finance (2022): $15.6M. Used flash loan to manipulate Uniswap TWAP in single block.
- Bonq (2023): $120M. Manipulated TellorFlex oracle with single staking transaction.

Pattern to detect: Any use of spot price (balanceOf / totalSupply, or single DEX read) for valuing collateral or positions.

### Access Control Examples
- Ronin Bridge (2022): $624M. Compromised 5 of 9 multisig validators. Threshold too low.
- Wormhole (2022): $326M. Uninitialized proxy allowed attacker to call guardian set upgrade.
- Wintermute (2022): $162M. Vanity address generator vulnerability gave attacker private key access.

Pattern to detect: Missing access modifiers on functions that change critical state. Unprotected initializer functions.

### Flash Loan Examples
- bZx (2020): $8M. Flash loan + oracle manipulation + leveraged trade in single transaction.
- Pancake Bunny (2021): $45M. Flash loan to manipulate LP token price, then deposit at inflated value.
- Euler Finance (2023): $197M. Flash loan + donate function to create bad debt.

Pattern to detect: Functions where large temporary capital could create profit. Price calculations that can be temporarily manipulated.

When reporting findings, reference the most similar real exploit to justify the severity and demonstrate impact.

## Detection Phase: Multi-Mindset Analysis

After completing reconnaissance, analyze each contract using these four mindsets.

### Mindset 1: The Attacker
Think like an attacker with access to unlimited capital and no morals.
- What is the maximum damage I can cause?
- How can I extract value from this contract?
- What happens if I call functions in an unexpected order?
- What if I combine multiple calls in a single transaction?
- Can I manipulate any price, balance, or state?
Tag findings as [ATTACKER].

### Mindset 2: The Accountant
Think like an auditor checking every calculation.
- Are all calculations correct?
- Do fees/shares/amounts add up?
- What happens with rounding (up vs down)?
- Are there precision loss issues?
- Do token balances match accounting?
Tag findings as [ACCOUNTANT].

### Mindset 3: The Spec Auditor
Think like someone verifying code matches documentation.
- Does this function do what its name implies?
- Are documented invariants actually enforced in code?
- Does error handling match the expected behavior?
- Are events emitted for all state changes?
- Does the access control match the documented roles?
- Are edge cases in the specification handled?
Tag findings as [SPEC].

### Mindset 4: The Edge Case Hunter
Think like someone testing boundary conditions.
- What happens with amount = 0?
- What happens at uint256 max value?
- What happens with empty arrays or zero-length inputs?
- What happens if an external call reverts?
- What happens on first use (empty state)?
- What happens on last use (final withdrawal, last NFT)?
- What happens if called twice in the same transaction?
Tag findings as [EDGE].

### Merging (Important: How This Works in Practice)

Claude processes each mindset one at a time in a single response. It tags findings as it goes.
1. **Tagging**: Each finding is tagged with its source mindset [ATTACKER], [ACCOUNTANT], etc.
2. **After all 4 mindsets complete**: review all tagged findings
3. **Group by root cause**: If [ATTACKER] and [EDGE] both flagged the withdraw() function for reentrancy, group them as one finding.
4. **Note consensus**: "Found by: ATTACKER, EDGE" (2 mindsets = higher confidence)
5. **Resolve severity conflicts**: If mindsets disagree on severity, use the HIGHER severity.

## Verification Phase: Confidence Scoring

After detection, assign a confidence score to each finding based on multiple factors.

### Scoring Factors (each 0-25 points, total 0-100):

#### Factor 1: Evidence Strength (0-25)
- 25: Complete exploit scenario with exact function calls, parameters, and profit calculation
- 20: Exploit scenario with some assumptions
- 15: Clear vulnerability pattern but no complete exploit
- 10: Suspicious code pattern, theoretical risk
- 5: Minor concern, stylistic issue
- 0: Generic observation, no security relevance

#### Factor 2: Code Context (0-25)
- 25: Vulnerability in a function that handles significant value (>$10K typical flow)
- 20: Vulnerability in access-controlled function with bypass
- 15: Vulnerability in frequently called function
- 10: Vulnerability in edge-case path
- 5: Vulnerability in rarely triggered code
- 0: Vulnerability in unreachable or dead code

#### Factor 3: Detection Consensus (0-25)
If you used multi-mindset detection:
- 25: Found by 3+ mindsets independently
- 20: Found by 2 mindsets
- 15: Found by 1 mindset with strong evidence
- 10: Found by 1 mindset with moderate evidence
- 5: Found by 1 mindset with weak evidence
If single-pass: score based on pattern match strength instead.

#### Factor 4: Historical Precedent (0-25)
- 25: Identical pattern caused known exploit (cite specific incident)
- 20: Very similar pattern caused known exploit
- 15: Same vulnerability class has caused exploits in other protocols
- 10: Class well-documented but no close match
- 5: Theoretical vulnerability, no known precedent
- 0: No historical evidence

### Confidence Classification
- 70-100 HIGH → Report as-is with full severity
- 50-69 MEDIUM → Report with severity, note "verify manually"
- 30-49 LOW → Report only if HIGH/CRITICAL severity
- 0-29 INSUFFICIENT → Do not report (log for analysis)

### How to Read the Score
A clear reentrancy with a complete exploit scenario in a value-handling function should score 70-100.
For each finding, show: [SEVERITY] Title (Confidence: XX/100). Break down the 4 scoring factors so the reader understands WHY the confidence score is what it is.

## Output Format

Generate TWO outputs:

### 1. JSON Report
Write to `.audit/findings.json`:

{
  "projectName": "[project name from directory]",
  "timestamp": "[ISO 8601 date]",
  "auditor": "[your skill name]",
  "summary": {
    "totalFindings": [count],
    "critical": [count],
    "high": [count],
    "medium": [count],
    "low": [count],
    "filesAnalyzed": [count],
    "linesOfCode": [count]
  },
  "findings": [
    {
      "id": "[SKILL-001]",
      "title": "[finding title]",
      "severity": "critical|high|medium|low",
      "file": "[path/to/file.sol]",
      "line": [line number],
      "description": "[full description with exploit scenario]",
      "impact": "[quantified impact]",
      "remediation": "[specific fix]",
      "category": "[reentrancy|access-control|integer|oracle|token|other]"
    }
  ]
}

### 2. Markdown Summary
After writing the JSON, print a human-readable summary to the console:

# Audit Complete

**Findings:** X Critical, X High, X Medium, X Low
**Files:** X analyzed
**Output:** .audit/findings.json

Then list each finding briefly:
- [CRITICAL] Title (file.sol:line)
- [HIGH] Title (file.sol:line)
...
