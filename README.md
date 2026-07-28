# Veiled CKB

**Prove what you're allowed to do without revealing who you are — verified at consensus on Nervos CKB.**

An OAuth-shaped SDK for authentication that proves claims about a user (wallet ownership, balance range, token holdings) without revealing the wallet, the amount, or the holdings — and without a trusted authorization server.

> **Status: specification. No implementation yet.** The circuits and SDK this ports from live at [digitaldrreamer/veiled](https://github.com/digitaldrreamer/veiled) (Solana). Nothing here is audited.

---

## Why CKB

The Solana implementation verifies proofs client-side and stores a signed result on-chain, because generic ZK verification does not fit inside a Solana transaction's compute budget. That is a real architectural ceiling, and it is the weakest part of the design: whoever verifies is whoever you trust.

CKB does not have that ceiling. Mainnet `max_block_cycles` is 3,500,000,000 and there is **no per-transaction or per-script cycle limit** — a script may consume as much as it needs so long as the block fits ([docs](https://docs.nervos.org/docs/script/vm-cycle-limits)). Measured verifiers on CKB-VM sit far inside that budget:

| Verifier | Cycles | Share of a block |
|---|---|---|
| SP1 / Plonk, optimized with `ckb-alt-bn128` | 63M | ~1.8% |
| Groth16 / BN256 (`ckb-zkp`) | 195.5M | ~5.6% |

Sources: [Optimized SP1 verifier for ckb-vm](https://talk.nervos.org/t/optimized-sp1-verifier-for-ckb-vm/10144) (April 2026), [sec-bit/ckb-zkp](https://github.com/sec-bit/ckb-zkp) — the latter inactive since 2020 and explicitly not for production. Neither covers Noir/UltraHonk; porting a Honk verifier is unbuilt work, but the budget accommodates it.

**What that buys is the whole thesis.** If a type script verifies the proof at consensus, a relying party's integration is: read the cell, check the type script code hash, done. OAuth's developer ergonomics with no authorization server to trust — no JWKS to fetch, no company that sees every login, nothing to compromise. Consensus is the authorization server.

That is the difference between this and every hosted wallet-auth product, all of which are necessarily trusted brokers.

---

## What's specified

- **[SPEC-IDENTITY-ANCHOR.md](./SPEC-IDENTITY-ANCHOR.md)** — how a user's secret becomes rotatable and recoverable without a custodian. Covers the key hierarchy, the anchor cell, the announce/veto/finalize rotation lifecycle, proof binding, threat model, capacity and cycle costs, and honest limits.

Recovery is specified first on purpose. A prove-without-revealing system where losing your secret loses every account you ever made is not shippable, and the failure is a product failure rather than a cryptographic one. It needs an answer before circuits get written.

---

## How recovery works

The secret is a credential, not the identity.

```
R           recovery root        (wallet key, or guardian-reconstructable seed)
 ├── s_link = KDF(R, "link")     stable, never rotated, never on-chain
 └── s_auth                      rotatable, commitment published on-chain

sub = H(s_link, domain)          per-site identifier — survives rotation
```

An **Identity Anchor** cell holds a commitment to `s_auth` and to a recovery policy. Rotating that commitment is authorized by [Structural Authorization](https://github.com/digitaldrreamer/ckb-structural-authorization): a condition cell carrying a ZK proof that satisfies the committed policy, verified at consensus. No admin key, no recovery service, no custodian who learns who you are.

A timelock and veto window sit between announce and finalize, because Structural Authorization lets *anyone* construct the transaction — which is the point for a treasury and a liability for identity.

**The chain never holds the secret.** Cell data is public; anything a script can read, everyone can read. The chain holds a commitment and arbitrates which one is current.

---

## Build order

| Level | Mechanism | Gains | Costs |
|---|---|---|---|
| **0** | No anchor. `s_link` derived from the wallet. | Recovery via wallet. No cells, no capacity. | No revocation. |
| **1** | Anchor holds `H(s_auth)`; rotation proven by wallet control. | Revocation. | ~175 CKB deposit per user, returned on close. |
| **2** | Policy admits k-of-n guardians. | Survives wallet loss. | Guardian UX, larger circuits. |

Level 0 is a complete product with no on-chain state. Ship it first; the anchor is additive.

---

## Related work

| Repo | Relationship |
|---|---|
| [veiled](https://github.com/digitaldrreamer/veiled) | Origin. Noir circuits and SDK surface port over; the Anchor program and Solana adapters do not. |
| [ckb-structural-authorization](https://github.com/digitaldrreamer/ckb-structural-authorization) | Supplies the authorization pattern the rotation lifecycle is built on. |
| [ckb-transaction-firewall](https://github.com/digitaldrreamer/ckb-transaction-firewall) | Its treasury pattern fits the revocation registry. It does **not** fit per-user anchors — see the spec's cost section. |
| [ckb-agent-control-hub](https://github.com/digitaldrreamer/ckb-agent-control-hub) | Same problem one layer up: authorization for agents rather than humans. Its open question on runtime key access is the question this spec answers. |

---

## Known gaps

- No implementation, no tests, no audit.
- No Noir/UltraHonk verifier exists for CKB-VM yet.
- Unlinkability across sites requires an accumulator whose maintenance is unspecified. The simple alternative leaks exactly what the product promises to protect.
- The problem statement is asserted, not evidenced. Data on whether users actually avoid wallet-based login for privacy reasons is not yet gathered.

## License

MIT.
