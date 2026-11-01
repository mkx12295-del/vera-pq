# VERA-PQ

**Verifiable Edge Robust Aggregation with Post-Quantum, Hash-Only Attestation**
A smart contract with staking and slashing that audits an edge gateway's federated-learning aggregation.

> **Confidential.** Keep this repository **private** until VIT's IPR&TT Cell has decided on a patent filing.
> All rights reserved. This is a course project prototype, not production software.

Author: Manojkumar Ashokkumar, B.Tech CSE (Blockchain), VIT Vellore.

---

## What it does

In federated learning on IoT, devices send model updates to an **edge gateway** that combines them. A robust
rule such as the coordinate-wise **trimmed mean** protects the model from poisoned devices, but nothing forces
the gateway to use it. A lazy or compromised gateway can use a plain average, and a few poisoned devices then
wreck the model unnoticed.

VERA-PQ adds a blockchain referee:

1. Each device commits to its update with a **Merkle tree** and signs one digest with **ML-DSA-44** (FIPS 204, post-quantum).
2. The gateway computes the trimmed mean and commits two Merkle roots (client set, aggregate) to a smart contract,
   locking an **ETH bond**.
3. The contract derives a random challenge from the commit block's hash and picks `k` coordinates.
4. The gateway opens those coordinates with Merkle proofs. The contract recomputes the trimmed mean on them
   using **Keccak-256 only** (no zero-knowledge prover, no trusted setup, no pairings).
5. **Pass:** bond released, reward paid (half gateway, half devices). **Fail:** bond slashed.
   **No answer in 256 blocks:** anyone calls `expire()`, bond slashed, caller earns a 10% bounty.

Detection probability per round is `1 - (1 - f)^k` for a gateway that is wrong on a fraction `f` of coordinates.

## Repository layout

```
README.md              this file
demo_code/
  contracts/VeraPQLedger.sol   Solidity contract (audit, stake, reward, slash, expire)
  verapq/                      protocol, Merkle trees, ML-DSA wrapper, simulated and EVM ledgers, FL
  run_demo.py                  end-to-end demo
  run_all.sh                   demo-day scenarios
  experiments.py, bench.py     experiments and benchmarks
  test_verapq.py               10 unit tests
  results/                     measured outputs
paper/
  VERA-PQ_IEEE_paper.docx/.pdf IEEE-style paper
  build_paper.js, make_figures.py, figs/
```

## Quick start (Windows PowerShell)

Requires Python 3.10 to 3.13. Node.js 18 or later is needed only for the EVM option.

```powershell
cd demo_code
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install numpy scikit-learn pandas matplotlib dilithium-py pycryptodome
python test_verapq.py          # expect 10 PASS lines
python run_demo.py --rounds 5
```

Linux and macOS: `source .venv/bin/activate` instead of the Activate script.

## Demo scenarios

```powershell
python run_demo.py --rounds 5                                   # honest gateway: accepted, rewarded
python run_demo.py --rounds 5 --gateway lazy --no-audit         # lazy gateway, no audit: model wrecked, cheater paid
python run_demo.py --rounds 5 --gateway lazy                    # lazy gateway, audited: rejected, bond slashed
python run_demo.py --rounds 5 --gateway tamper --tamper-frac 0.1  # sparse tampering: mostly rejected
python run_demo.py --rounds 3 --gateway lazy --withhold         # gateway stays silent: expired, keeper bounty
```

Useful flags: `--clients --malicious --trim --k --bond --reward --deposit --pool --ledger {sim,evm}`.

### Run the contract on an in-process EVM (optional, real gas)

```powershell
python -m pip install web3 "eth-tester[py-evm]"
npm install solc@0.8.26
$env:VERAPQ_SOLCJS = "$PWD\node_modules\.bin\solcjs.cmd"
python run_demo.py --ledger evm --rounds 3
```

### Faster post-quantum signatures (optional)

The default back end is pure-Python ML-DSA (`dilithium-py`), which is slow but needs no build step. Installing
`liboqs` and `liboqs-python` is about 500 times faster and gives identical protocol results (Linux, WSL or macOS):

```bash
sudo apt install -y cmake ninja-build build-essential git       # macOS: brew install cmake ninja
git clone --depth 1 --branch 0.15.0 https://github.com/open-quantum-safe/liboqs
cmake -S liboqs -B build -GNinja -DBUILD_SHARED_LIBS=ON -DOQS_BUILD_ONLY_LIB=ON \
  -DOQS_MINIMAL_BUILD="SIG_ml_dsa_44;SIG_ml_dsa_65;SIG_falcon_512;SIG_sphincs_sha2_128f_simple;KEM_ml_kem_768" \
  -DCMAKE_INSTALL_PREFIX=$HOME/_oqs
ninja -C build install
export OQS_INSTALL_PATH=$HOME/_oqs
pip install liboqs-python
```

Choose with `VERAPQ_PQ_BACKEND=pure|liboqs`.

## Measured results (8 clients, trim 2, synthetic data)

| Result | Value |
|---|---|
| Honest rounds accepted | 210 of 210 |
| Lazy-gateway rounds rejected | 210 of 210 |
| Accuracy, plain-mean gateway, 2 poisoned clients | about 0.18 (trimmed mean: about 0.91) |
| Audit gas, n=8, k=16, d=32 | about 1.39M (about 6x a Groth16 verification) |
| Commit gas / deployment gas | about 149k / 1.65M |
| Gateway time to build the audit | at most 3 ms |
| 100 rounds of 10% tampering | 84 rounds slashed (80.4 predicted) |

## What is real and what is simulated

| Part | Status |
|---|---|
| Solidity contract | Real code, executed as bytecode on an in-process EVM (py-evm) with metered gas |
| ML-DSA signatures, Merkle trees, trimmed mean, audit checks | Real computation |
| Default `--ledger sim` | A Python mirror of the contract (balances matched the EVM) |
| Blockchain network | **In-process only.** Not deployed to a testnet or mainnet |
| IoT clients | Simulated processes, not physical devices |
| Data | Synthetic; Edge-IIoTset is future work |

## Limitations

- Detection is probabilistic: one round catches tampering with probability `1 - (1 - f)^k`, not with certainty.
- Gas is higher than a zk-SNARK verification and grows with `n x k`; sampled values are public, so there is no update privacy.
- Not fully post-quantum on Ethereum: transaction signatures are still ECDSA, and the block-hash challenge can be
  biased by a block proposer (RANDAO, a VRF or commit-reveal would fix this).
- Only coordinate-wise order statistics (trimmed mean, median) are covered, with `n > 2t`.
- Bond and reward values are illustrative; no game-theoretic simulation.

## Tests

```powershell
python test_verapq.py
```

Covers honest acceptance, full tampering, forged signature, substituted value, values not matching the signed
root, fabricated client, reward payout, slashing, expiry with keeper bounty, and commit without enough stake.

## Disclosure

This work is under review for possible patent protection. Do not publish, post or redistribute the code, paper or
results until that decision is made.
