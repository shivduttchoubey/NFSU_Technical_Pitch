# QBOM-ZT: Iterative, Zero-Trust Post-Quantum Readiness Engagements

**An automated Quantum Bill of Materials (QBOM) framework that runs a full
post-quantum readiness engagement, not just a one-off scan.**


## 1. The problem this solves

Most QBOM/CBOM tools stop at "here is your cryptographic inventory." This
one runs the whole loop: an organisation commits to an assessment, the tool
scans and produces prioritised recommendations, infra applies the fixes, it
rescans — and that cycle repeats, with the organisation reviewing and
re-accepting the readiness percentage at every step, until a target
post-quantum readiness (default **80%**) is reached or the organisation
stops the loop. A zero-trust disclosure layer lets an external party (a
regulator, a parent company, a customer) verify claims about that readiness
without ever seeing the underlying findings

## 2. Architecture

```mermaid
flowchart TD
    subgraph Org["Organisation infrastructure (nothing below this line leaves it, unencrypted)"]
        A["1. Abstraction & Commitment\ngive_consent()"] --> B
        A["Pedersen commitments\n+ Merkle root only, never raw findings"] --> B
        B["2. Scan\nscanner.py\n(AST + regex + X.509, no execution)"] --> C
        C["3. Threat analysis\nrisk.py\n(HNDL x exposure x sensitivity)"] --> D
        D["4. Recommendations\nreport.py / dashboard.py\nCERT-In Table 8/9 + VEX"]
        D --> E{"5. Org reviews\nreadiness %"}
        E -- "target reached" --> Z(["TARGET_MET"])
        E -- "accept, continue" --> F["6. Infra applies fixes\nremediate.py"]
        E -- "decline" --> Y(["ENDED_BY_ORG"])
        F --> B
    end

    style Z fill:#16A34A,color:#fff
    style Y fill:#D97706,color:#fff
  ```

Every iteration of that loop is one `IterationRecord`; the whole run is one
`Engagement` (`qbom/workflow.py`), persisted to `engagement.json` after every
state change so a run can be inspected or resumed at any point.

### Module map

```
qbom/
  catalog.py     -- ~49-entry algorithm/protocol catalog, classified by quantum threat
  scanner.py     -- static inventory extraction: Python AST, regex/config, X.509/PEM
                     (now also recognises PQC call sites -- liboqs-python, BouncyCastle-PQC,
                     ML-DSA/ML-KEM certs -- so a *remediation* can be verified by rescanning,
                     not just claimed)
  risk.py        -- prioritisation engine (HNDL urgency x exposure x sensitivity)
  report.py      -- CycloneDX 1.6 CBOM export, markdown/PDF reports, the CERT-In
                     Table 8/9 minimum-elements mapping, and the multi-iteration
                     engagement PDF
  dashboard.py   -- self-contained HTML dashboard, now with an engagement/
                     iteration-timeline panel
  ec.py / zk.py / merkle.py / privacy.py
                 -- the zero-trust disclosure layer (Pedersen commitments,
                     Schnorr/range/threshold ZK proofs, sorted Merkle tree) --
                     unchanged from the original prototype
  remediate.py   -- NEW: the "infra applies the fix" step. Deliberately the
                     least generic module -- see its docstring
  workflow.py    -- NEW: the Engagement/iteration state machine wiring
                     everything above into the consent -> scan -> recommend
                     -> decide -> remediate -> rescan loop
  cli.py         -- `qbom scan` / `qbom verify` (original) plus
                     `qbom engage` / `qbom engage-status` (new)
tests/           -- 117 pytest tests (101 original + 16 new for
                     remediate.py/workflow.py + scanner PQC-detection cases)
samples/         -- the deliberately-vulnerable sample app + manifest.json and secure sample app and secure manifest.json

```

## 3. The engagement loop, in detail

| Step | What happens | Where |
|---|---|---|
| **Commit** | Org names an accountable person and consents to be assessed, against a stated target %. Required before any scan runs. | `workflow.give_consent()` |
| **Scan** | Iteration 1 scans the org's own read-only source. Nothing leaves the machine. No execution of target code, no network access, no elevated privileges. | `scanner.py` |
| **Threat analysis** | Findings are scored by *quantum principles*, not a big weighted checklist: Harvest-Now-Decrypt-Later urgency, Shor (existential break) vs. Grover (margin loss) vs. classically-broken, exposure, blast radius. | `risk.py` |
| **Recommendations** | A prioritised migration plan, a VEX document (CERT-In Section 6/8.4.1.6 vocabulary -- Not Affected / Affected / Fixed / Under Investigation -- applied here to PQ migration status), and reports in markdown, PDF, CycloneDX CBOM, and the CERT-In Table 8/9 shape. | `report.py`, `workflow.build_vex()` |
| **Org decides** | Readiness % is shown. If it already meets the target, the engagement completes. Otherwise the org explicitly accepts (and the loop continues) or declines (and it stops) -- every iteration, not just the first. | `workflow.decide_continue()` |
| **Infra applies fixes** | The recommended, highest-priority-first fixes are applied to a **working copy** (never the original source). See "what remediation actually does" below. | `remediate.py` |
| **Rescan** | Back to Scan, now against the remediated working copy. Repeats until target reached, org declines, or a max-iteration safety cap is hit. | `workflow.run_full_loop()` |

### What remediation actually does (and doesn't fake)

`remediate.py`'s docstring says this directly: it is the **least generic**
module in the framework, on purpose. Classifying crypto usage generalises to
any codebase; *fixing* it doesn't -- "replace this RSA-1024 signing key" is a
different concrete edit in every codebase. What ships here is a working,
honest demonstration against the bundled sample app:

- Classically-broken hashes/ciphers and clearly-wrong config lines (MD5/SHA-1,
  DES/3DES/RC4, ECB mode, legacy TLS/SSH versions) are mechanically fixed.
- **RSA/ECC/EdDSA findings are never "fixed" with a bigger key** -- a bigger
  RSA key is still Shor-breakable. They're migrated to ML-KEM/ML-DSA at the
  call-site level (via the liboqs-python API shape, or BouncyCastle-PQC for
  Java), which the extended scanner can then genuinely verify on rescan.
- **X.509 certificates are re-issued, never hand-edited** -- you cannot `sed`
  a signature algorithm. If the installed `cryptography` version supports it
  (>= 47), certs are re-issued as real ML-DSA-65 (FIPS 204) certificates,
  verified end-to-end against `cryptography==50.0.1` while building this.
- A handful of items are **left unfixed on purpose and explained**
  (`remediate.RESIDUAL_NOTES`), because a tool that claims to fix everything
  is lying: there is no standardised post-quantum SSH host-key algorithm
  yet; a symmetric key's runtime size can't be confirmed by static analysis;
  raising `default_bits` is classical hygiene only and doesn't change
  Shor-vulnerability. These show up verbatim in the engagement report.
- One scanner enhancement worth calling out: TLS 1.3 is catalogued
  Shor-vulnerable because it *defaults* to classical key exchange -- but if
  a hybrid PQ group is configured in the same file, the scanner now
  correlates that and relabels the finding (`TLS1.3-HYBRID`, threat
  `HYBRID`) instead of leaving a stale, misleading SHOR finding next to a
  correctly-configured hybrid group.


## 4. Zero-trust disclosure (unchanged from the original prototype)

Every iteration can optionally publish an **Attestation**: Pedersen
commitments per algorithm, a sorted Merkle tree over salted findings, and
zero-knowledge proofs -- never the raw findings. The engagement loop uses
this for its most compelling claim: at the end of an iteration where an
algorithm's count dropped to zero, it publishes a `prove_zero` proof --
*"RSA-SUB2048 is now provably absent"* -- verifiable by anyone with the
public attestation, without that verifier ever seeing the underlying scan.
See the module docstrings in `privacy.py`, `zk.py`, and `merkle.py` for the
full design rationale (Pedersen homomorphic aggregation across
subsidiaries, Schnorr/range/threshold proofs, sorted-Merkle selective
disclosure) -- unchanged from the original prototype and still accurate.

## 5. CERT-In guideline mapping

| Guideline requirement | Where it's satisfied |
|---|---|
| Table 5 minimum SBOM elements | `report.to_cyclonedx_cbom` (unchanged) |
| Table 8/9 minimum QBOM/CBOM elements | `report.to_qbom_minimum_elements`, rendered as Section 7 of the engagement PDF |
| Section 6 / 8.4.1.6 VEX document | `workflow.build_vex`, rendered as Section 4 of the engagement PDF |
| Section 8.4.1.13 "established workflows... updated as new components... are introduced" | the iteration loop itself |
| Section 8.5 quantum-readiness / migration strategy | the prioritised migration plan + `remediate.py`'s algorithm-not-key-size migrations |
| SPDX/CycloneDX machine-readable format | `report.to_cyclonedx_cbom` (CycloneDX 1.6 + quantum fields) |

## 6. Quickstart

```bash
pip install -e . --break-system-packages
python -m pytest tests/ -q                      # 117 passed

# standalone scanner:

python -m qbom.cli scan samples/vulnerable-app --org "Acme Corp" --out out --manifest samples/manifest.json --attest  #vulnerable app

python -m qbom.cli scan samples/secure-app --org "Acme Corp (migrated)" --out out-secure --manifest samples/manifest-secure.json --attest --prove-zero MD5 --prove-zero SHA1 --prove-zero RC4 --prove-zero DES --prove-zero 3DES #secure app

python -m qbom.cli verify out/attestation.public.json --zero SSL-2/3 --below "RSA-SUB2048<10" #verify the results


# the engagement loop:
python -m qbom.cli engage samples/vulnerable-app --org "Acme Corp" --out out-engagement --manifest samples/manifest.json --accepted-by "Priya Sharma, CISO" --target 80 --max-iterations 8

python -m qbom.cli engage-status --out out-engagement
```

Outputs land under `<out>/iterations/iteration-N/` (per-round `qbom.json`,
CycloneDX CBOM, markdown + PDF report, `vex.json`, dashboard, optional
attestation) and `<out>/final/` (the consolidated engagement PDF + latest
dashboard + `engagement-summary.json`). `--interactive` prompts y/N at each
accept-and-continue decision instead of auto-accepting; omit it for a
non-interactive/CI/demo run.

directly in a browser, or `demo-run/final/qbom-engagement-report.pdf`.

## 7. Known limitations

- Static analysis only: no compiled binaries, third-party dependency
  internals, or runtime-negotiated TLS behaviour.
- `remediate.py` is intentionally specific to the bundled sample app's file
  layout -- a real deployment plugs its own remediation playbooks/CI
  automation into that exact module (see its docstring for the seam).
- Symmetric key sizes supplied at runtime (not a literal in source) can't be
  confirmed by static analysis (`AES-UNK` residual).
- OpenSSL cipher-string *negation* syntax (`!RC4`) isn't understood --
  a token is flagged whether negated or not, so remediation drops such
  tokens entirely rather than negating them.
- ZK range-proof performance (~1s per 24-bit threshold proof, pure-Python
  EC) suits periodic attestation, not a request-per-second service.
- `catalog.py` (~49 entries) is a starting vocabulary, not exhaustive.

## 8. Repository hygiene

`pyproject.toml` carries a `[tool.ruff]` config (line-length 120, matching
this codebase's existing convention; `samples/vulnerable-app` excluded since
it's a deliberately-vulnerable fixture, not source to be linted). This
change took the repo from 503 to 20 ruff findings; everything remaining is
either pre-existing in files this change didn't touch, or in the original
test suite's own style choices -- nothing introduced by this change is left
outstanding.
