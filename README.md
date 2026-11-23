# DEX SilverScript contracts

A constant-product AMM (KAS / KCC20) written in [SilverScript](https://github.com/kaspanet/silverscript) for Kaspa covenants.

Uploaded by **La Gauffre**.

> ## ⚠️ AI-generated first draft — not audited, not for production
>
> **The contracts and this document were produced by an AI assistant.** They are a **first draft**.
>
> - They have **not been audited**. They have only been exercised by a small VM test suite (32 tests in the development workspace, not published here).
> - They **must be reviewed several times**, by several people, before they can be considered final. Both the logic and the code need human review, especially the KCC20 interactions, the 128-bit arithmetic, the genesis / `init` flow and every "who may spend what" rule.
> - **Do not deploy them or use them with real funds** until that review is complete and the open issues listed in §10 are resolved.
>
> This draft has already had one real bug, found by the tests: the original `init` used `OpCovOutputCount` on genesis outputs and could never succeed. Expect more.

## Files

| File | Role |
|---|---|
| `DexAmm.sil` | Core of a pool: central state, owns the token vault and the LP vault. Entrypoints `init`, `swap`, `liquidity` |
| `DexKasPool.sil` | KAS reserve of the pool (its UTXO value) |
| `DexDeposit.sil` | Liquidity deposit module: LP issued from the LP vault |
| `DexWithdraw.sil` | Liquidity withdrawal module: LP returned to the LP vault |

**Note on paths.** Paths such as `DEX/test/…`, `DEX/ARCHITECTURE.md`, `DEX/kcc20.sil` or `DEX/contrats/…` below refer to the development workspace. Only the four contracts and this README are published in this repository. The token standard (`kcc20.sil`, `mint.sil`) is not included.

---

# Execution logic

## 1. Scope

A constant-product AMM for one **KAS / KCC20** pair per pool. It supports:

| Feature | Where |
|---|---|
| Swap KAS ⇄ token | `DexAmm.swap` |
| Add liquidity (LP tokens issued) | `DexAmm.liquidity` + `DexDeposit.deposit` |
| Remove liquidity (LP tokens returned) | `DexAmm.liquidity` + `DexWithdraw.withdraw` |

The order book is out of scope. It is covered separately by the escrow contracts in `KaspaRocket-Silverscript-Contract/`.

**Token standard**: only `DEX/kcc20.sil` (KCC20 reference, 2 delegates / 3 outputs) and `DEX/mint.sil` (`PublicMint`).

**Source files**: `DEX/contrats/DexAmm.sil`, `DexKasPool.sil`, `DexDeposit.sil`, `DexWithdraw.sil`.

---

## 2. Facts about the token standard that the design relies on

| Fact (from `DEX/kcc20.sil` / `DEX/mint.sil`) | Consequence |
|---|---|
| `transfer` enforces `output_total == input_total` and `amount >= 0`. There is no mint and no burn in the token. | Burning LP is impossible, and minting is only possible through `PublicMint`. |
| `PublicMint.mint` and `PublicMint.split` have **no caller restriction**. | A live LP `PublicMint` would let anyone mint LP. The whole LP supply is therefore minted **once**, at `init`, into a vault (**option A**). |
| A UTXO with `owner_scheme = 0x04` (covenant id) is released whenever `OpCovInputCount(owner) > 0`. | Both vaults are released whenever `DexAmm` is an input, so **every** `DexAmm` path must constrain or forbid them. |
| Anyone can create a KCC20 UTXO *displaying* `owner = X`. | A UTXO "owned by the AMM" proves nothing. Vaults are identified by an **exhaustive scan** plus a **balance pinned in the AMM state**. |
| Borrow path: a third party may top up a borrow-enabled UTXO as leader. | Vaults must have `borrow_scheme = BORROW_DISABLED`. This is checked on every use. |
| The first token input is the `transfer` leader. Its `next_states` define **all** outputs of that token. All `extension_commitment` values must match. | Transaction builders must order inputs and outputs accordingly. Vault continuations copy the vault's `extension_commitment`. |
| Token "types" differ only by `MAX_DELEGATES` / `MAX_TOKEN_OUTPUTS`. That changes the template hash and suffix length, never the state layout. | One set of DEX contracts serves every type. Only `token_prefix_len`, `token_suffix_len` and `token_hash` (stored in AMM state) vary. |
| `PublicMint` shares the token's covenant id. | A `PublicMint` input of a pooled token makes the vault scan fail (template mismatch), so it must never be part of a DEX transaction. |

---

## 3. Components of a pool

| # | Component | Kind | Holds | Spending paths |
|---|---|---|---|---|
| 1 | `DexAmm` | Singleton covenant | Central state, owns both vaults | `init`, `swap`, `liquidity` |
| 2 | `DexKasPool` | Singleton covenant | **Its value is the KAS reserve** | `spend` |
| 3 | `DexDeposit` | Singleton covenant | Deposit logic | `deposit` |
| 4 | `DexWithdraw` | Singleton covenant | Withdrawal logic | `withdraw` |
| 5 | Token vault | KCC20 UTXO (traded token) | Token reserve. `owner = DexAmm`, `0x04`, borrow off | KCC20 `transfer` / `transfer_delegator` |
| 6 | LP vault | KCC20 UTXO (LP token) | Unissued LP. `owner = DexAmm`, `0x04`, borrow off | KCC20 `transfer` / `transfer_delegator` |

All covenant entrypoints are declared `#[covenant(binding = auth, from = 1, to = 1, groups = single)]`. This means:
- each continues as **exactly one** output with its own template;
- no second UTXO with the same covenant id can be continued in the same transaction;
- the compiler-generated wrapper validates that continuation's state.

Each pool has its **own** LP token (own covenant id). LP tokens of different pools are not interchangeable.

### 3.1 State

**`DexAmm`** (13 fields, order matters; copied verbatim as `AmmState` in `DexDeposit` / `DexWithdraw`):

| Field | Meaning | Mutable? |
|---|---|---|
| `initialized` | `false` until `init` | `false → true` once |
| `token_id`, `lp_token_id` | Covenant ids of the traded token and the LP token | Never |
| `kas_pool_id`, `deposit_id`, `withdraw_id` | Module covenant ids | Set by `init`, then frozen |
| `modules_hash` | `blake3(kas_pool_hash ‖ deposit_hash ‖ withdraw_hash)`, the expected module templates | Never |
| `token_prefix_len`, `token_suffix_len`, `token_hash` | KCC20 template of the traded token | Never |
| `reserve_token` | Exact balance of the token vault | Swap, deposit, withdraw |
| `lp_vault` | Exact balance of the LP vault | Deposit, withdraw |
| `lp_supply` | LP issued: circulating + `MIN_LIQUIDITY` | Deposit, withdraw |

**Compile-time parameters of `DexAmm`**, identical for every pool: LP KCC20 template (`lp_prefix_len`, `lp_suffix_len`, `lp_hash`), LP PublicMint template (`pm_*`), `fee_bps`.

**`DexKasPool`, `DexDeposit`, `DexWithdraw`**: a single state field, `amm_id`, frozen from birth.

**Compile-time parameters of `DexDeposit` / `DexWithdraw`**: the `DexAmm` template (`amm_prefix_len`, `amm_suffix_len`, `amm_hash`) and the LP KCC20 template.

### 3.2 Why there is no circular dependency

A KIP-20 covenant id is `hash(authorizing outpoint, the group's outputs incl. scriptPubKey)`, and the scriptPubKey contains the state. Two covenants therefore cannot embed each other's ids at creation. The draft breaks the cycle as follows:

1. `DexAmm` is created first, with zero links.
2. The modules are created **during `DexAmm.init`**, with `amm_id` already in their state. Their ids only depend on their own outputs, so they are computable.
3. `init` checks each module output (template and `amm_id`), then records the module ids.
4. **Template hashes** are not circular either. Module hashes live in `DexAmm`'s **state** (`modules_hash`). `DexAmm`'s hash lives in the modules' **code**. `DexAmm`'s code does not depend on them.

At every liquidity operation, the link is verified **in both directions**: the AMM requires its designated module, and the module requires that the AMM designates it.

---

## 4. Trust model: who verifies what

```
Token vault ──owner──▶ DexAmm ◀──requires── DexKasPool
LP vault    ──owner──▶   │
                         ├── swap       : verifies everything itself; LP vault, Deposit, Withdraw forbidden
                         └── liquidity  : requires Deposit XOR Withdraw
                                               │
                                    (they verify EVERYTHING: reserves,
                                     both vaults, AMM output state)
```

**Rule:** every covenant that releases value must either verify the economics itself, or require the presence of a covenant that does.

A covenant cannot see which entrypoint another covenant selected, only that it is present. That is why `DexDeposit` and `DexWithdraw` have exactly **one** path: their presence is proof that their logic runs.

| Covenant / path | Requires present | Forbids | Verifies itself |
|---|---|---|---|
| `DexKasPool.spend` | `DexAmm` (exactly 1 input) | — | Single continuation, same state |
| `DexAmm.swap` | `DexKasPool`, token vault | `DexDeposit`, `DexWithdraw`, **any** LP input | Config and links frozen, token vault identity and continuation, KAS reserve, swap formula |
| `DexAmm.liquidity` | Exactly one of `DexDeposit` / `DexWithdraw` | — | Config and links frozen, own dust |
| `DexDeposit.deposit` | `DexAmm` designating it, `DexKasPool`, both vaults | `DexWithdraw` | AMM output state, both vaults, KAS reserve, deposit formula |
| `DexWithdraw.withdraw` | `DexAmm` designating it, `DexKasPool`, both vaults | `DexDeposit` | AMM output state, both vaults, KAS reserve, withdrawal formula |
| `DexAmm.init` | LP `PublicMint` input | `initialized == true` | Module geneses, recorded ids, full LP mint into the LP vault |

---

## 5. Shared building blocks

### 5.1 Vault identification: `scanOwned` + `checkVault`

Used by `swap` (token vault) and by both modules (both vaults).

1. `OpCovInputCount(token) <= MAX_KCC20_SCAN` (10), so the scan is never truncated.
2. Read **every** input of that covenant with the KCC20 template. Count those with `owner == DexAmm` and `owner_scheme == 0x04`.
3. Require **exactly one**. Zero means the vault is missing; two or more means an injected look-alike.
4. Require `amount == expected` (the balance pinned in AMM state) and `borrow_scheme == BORROW_DISABLED`.
5. Build the expected continuation: same owner and scheme, borrow off, same `borrow_guard` and `extension_commitment`, new `amount`, `kcc20_template = token template hash`.
6. `validateOutputStateWithInputTemplate(out_idx, …)`: the output scriptPubKey is the KCC20 template, reused from the vault input's own sigscript, carrying that state.
7. `OpOutputCovenantId(out_idx) == token`: the output really belongs to the token covenant, not to a look-alike script without a binding.
8. The output value is at least the vault's input value (its dust cannot be skimmed).

A forged vault is only accepted if it holds **exactly** the pinned balance. The attacker's tokens then become the pool's reserve, so there is nothing to gain.

### 5.2 128-bit arithmetic: `mul128` + `requireMulLe`

Script integers are int64, and reserve × reserve overflows above ~3·10⁹ per side. Every economic check is therefore written as `a·b ≤ c·d`, computed exactly in 128 bits:

- split each operand (`0 ≤ x < 2⁶²`) into two 31-bit limbs;
- build `hi·2⁶² + lo` from three partial products (bounds are documented in the code; no intermediate exceeds 2⁶³);
- compare `(hi, lo)` lexicographically.

**There is no division in the economic checks.** The transaction *proposes* the new balances and the contract *verifies* them. Rounding cannot work against the pool, because any surplus is simply a donation.

### 5.3 Fees

`feeOn(x) = ceil(x · fee_bps / 10000)`, computed without overflow. It is only applied to what **enters** the pool. Fees stay in the reserves, so LPs earn them. There is no protocol fee in this draft.

---

## 6. Execution flows

Notation: `K` = `DexKasPool` value, `T` = `reserve_token`, `L` = `lp_supply`, `V` = `lp_vault`. A prime (′) means "after".

### 6.1 Pool genesis (2 transactions)

**Tx 1 — create `DexAmm`.** The front end creates the UTXO with `initialized = false`, zero links, `modules_hash`, the token template, `lp_vault = lp_total`, `reserve_token = 0`, `lp_supply = 0`.

Separately, the LP token must exist with a `PublicMint` holding `remaining = mint_amount = lp_total`.

**Tx 2 — `DexAmm.init`.**

| Inputs | Outputs |
|---|---|
| 0 `DexAmm` (uninitialized), `init` | 0 `DexAmm′` (initialized, links set) |
| 1 LP `PublicMint`, `mint` | 1 `DexKasPool` genesis (authorized by input 2) |
| 2, 3, 4 funding inputs | 2 `DexDeposit` genesis (authorized by input 3) |
| | 3 `DexWithdraw` genesis (authorized by input 4) |
| | 4 `PublicMint′` (`remaining = 0`), authorized by input 1 |
| | 5 LP vault (`lp_total`, owner = AMM), authorized by input 1 |

`DexAmm.init` checks, in order:
1. `!prev.initialized`, `next.initialized`, config unchanged, own dust kept.
2. Sanity: `token_id != lp_token_id`, `0 ≤ fee_bps < 10000`, `lp_vault > 0`.
3. Accounting: `reserve_token′ = 0`, `lp_vault′ = lp_vault`, `lp_supply′ = 0`.
4. `blake3(the three hashes passed as arguments) == modules_hash`.
5. For each module output: `validateOutputStateWithTemplate(out, { amm_id: this AMM }, prefix, suffix, hash)`. That is the right template **and** a pointer to this AMM.
6. The three module ids (`OpOutputCovenantId`) are pairwise distinct and equal the ids written into `DexAmm′`.
7. LP mint:
   - `OpInputCovenantId(pm_in) == lp_token_id`;
   - the `PublicMint` state (read with its template) has `kcc20_template == lp_hash` and `remaining == lp_total`;
   - its continuation (auth-output 0) has `remaining = 0`, which excludes `split`, since a split keeps at least half;
   - its recipient (auth-output 1) is exactly the LP vault: `lp_total`, owner = this AMM, `0x04`, borrow off, `borrow_guard = 0`, the PublicMint's `extension_commitment`, bound to `lp_token_id`.

`PublicMint.mint` runs too and enforces its own rules (`amount ≤ mint_amount`, `amount ≤ remaining`, recipient template).

KIP-20 genesis ids are recomputed by the VM. A wrong id rejects the whole transaction.

After `init`, the **token vault** (amount 0, owner = AMM, borrow off, token's `extension_commitment`) must be created by any token transfer before the first deposit. A zero-balance look-alike is worthless.

### 6.2 Swap

| Inputs | Outputs |
|---|---|
| 0 `DexAmm`, `swap(newStates, token_vault_out)` | 0 `DexAmm′` |
| 1 `DexKasPool`, `spend` | 1 `DexKasPool′` (K′) |
| 2 token vault (KCC20 leader, witness `0x00`) | 2 token vault′ (T′) |
| (sell) trader's token UTXOs (signed delegates) | trader's tokens (buy) / token change |
| KAS wallet | KAS change / KAS proceeds (sell) |

Scripts executed:
- **`DexKasPool.spend`**: AMM present, single continuation.
- **KCC20 leader (vault)**: conservation. Released because the AMM is present.
- **KCC20 delegates**: owner signatures.
- **`DexAmm.swap`**:
  1. config and links frozen, own dust kept;
  2. `lp_vault` and `lp_supply` unchanged, `lp_supply > 0` (pool seeded);
  3. `OpCovInputCount(deposit_id) == 0`, `OpCovInputCount(withdraw_id) == 0`;
  4. `OpCovInputCount(lp_token_id) == 0`: no LP input at all, so the LP vault cannot be smuggled out;
  5. `checkVault(token, …, reserve_token → reserve_token′, token_vault_out)` and `reserve_token′ > 0`;
  6. exactly one `DexKasPool` input and output; `K = input value`, `K′ = output value > 0`;
  7. `K_adj = K′ − fee(K′ − K)` if KAS came in, otherwise `K′`. Same for `T_adj`. Both must be > 0;
  8. `K · T ≤ K_adj · T_adj` (128-bit).

### 6.3 Deposit (LP issued from the LP vault)

| Inputs | Outputs |
|---|---|
| 0 `DexAmm`, `liquidity(newStates)` | 0 `DexAmm′` |
| 1 `DexKasPool`, `spend` | 1 `DexKasPool′` (K + dk) |
| 2 `DexDeposit`, `deposit(newStates, T′, V′, L′, token_vault_out, lp_vault_out)` | 2 `DexDeposit′` |
| 3 token vault (token leader) | 3 token vault′ (T + dt) |
| 4 depositor's tokens (signed delegate) | 4 LP vault′ (V − dl) |
| 5 LP vault (LP leader) | 5 depositor's LP (dl) |
| 6 KAS wallet | (token / KAS change) |

Scripts executed:
- **`DexAmm.liquidity`**: config and links frozen, own dust, `count(deposit_id) + count(withdraw_id) == 1`.
- **`DexKasPool.spend`**, the **KCC20 leaders** and the **delegates**, as above.
- **`DexDeposit.deposit`**:
  1. own state unchanged, own dust kept;
  2. exactly one AMM input and output. Read the AMM input state with the AMM template. Require `initialized`, `deposit_id == my covenant id` and `count(withdraw_id) == 0`;
  3. rebuild the full AMM output state from the input state plus the three new accounting values, and validate the AMM output against it. This freezes config and links a second time;
  4. `checkVault` on the token vault (T → T′) and on the LP vault (V → V′);
  5. exactly one `DexKasPool` input and output: `K`, `K′`;
  6. `dk = K′ − K > 0`, `dt = T′ − T > 0`, `dl = V − V′ > 0`;
  7. first deposit (`L == 0`): `(dl + MIN_LIQUIDITY)² ≤ dk · dt` and `L′ = dl + MIN_LIQUIDITY`;
  8. later deposits: `dl · K ≤ dk · L`, `dl · T ≤ dt · L` and `L′ = L + dl`. This is `dl ≤ min(dk·L/K, dt·L/T)`; any surplus is a gift to existing LPs.

`MIN_LIQUIDITY` (1000) is counted in `lp_supply` but stays in the vault, so it can never be withdrawn. This is the share-price inflation defence.

### 6.4 Withdrawal (LP returned to the LP vault)

| Inputs | Outputs |
|---|---|
| 0 `DexAmm`, `liquidity` | 0 `DexAmm′` |
| 1 `DexKasPool`, `spend` | 1 `DexKasPool′` (K − dk) |
| 2 `DexWithdraw`, `withdraw(…)` | 2 `DexWithdraw′` |
| 3 token vault (token leader) | 3 token vault′ (T − dt) |
| 4 LP vault (LP leader) | 4 withdrawer's tokens (dt) |
| 5 withdrawer's LP (signed delegates) | 5 LP vault′ (V + dl) |
| 6 wallet (network fee) | 6 withdrawer's KAS (dk), optional LP change |

**`DexWithdraw.withdraw`** runs steps 1–5 as in the deposit (with `withdraw_id`, and `DexDeposit` forbidden), then:
- `L > 0`, `dk = K − K′ ≥ 0`, `dt = T − T′ ≥ 0`, `dl = V′ − V > 0`, `K′ > 0`;
- `L′ = L − dl` and `L′ ≥ MIN_LIQUIDITY`;
- `dk · L ≤ dl · K` and `dt · L ≤ dl · T`.

---

## 7. Transaction-builder rules

1. **Token leaders.** For each token, the first input of that token is the KCC20 leader, and its `next_states` list **every** output of that token, in output order. A vault can be the leader (witness = single byte `0x00`).
2. **Token limits.** With the 2/3 type, each operation fits in ≤ 3 inputs and ≤ 3 outputs per token (see `DEX/ARCHITECTURE.md` §7).
3. **Output indices** of the vault continuations (`token_vault_out`, `lp_vault_out`) are entrypoint arguments. They are not trusted: each one is fully validated (template, state, covenant binding).
4. **No `PublicMint`** of a pooled token in a DEX transaction (§2).
5. **Slippage.** Each transaction spends one specific `DexAmm` UTXO. If anything else touches the pool first, the transaction becomes invalid.
6. **Genesis ids** of the modules must be computed exactly as KIP-20 does (outpoint of the authorizing input, output index, value, scriptPubKey).

---

## 8. Security invariants and where they are enforced

| Invariant | Enforced by |
|---|---|
| The LP supply is fixed forever after `init` | `init`: PublicMint emptied (`remaining = 0`), whole supply in the vault |
| LP leave the vault only against a proportional deposit | `DexDeposit` formula + `checkVault(LP)`. `swap` forbids LP inputs. `liquidity` requires a module |
| Reserves leave only against returned LP | `DexWithdraw` formula + `checkVault`. `DexKasPool` requires the AMM |
| `x·y` never decreases on a swap | `DexAmm.swap`, 128-bit check |
| `MIN_LIQUIDITY` can never be withdrawn | `DexWithdraw`: `L′ ≥ MIN_LIQUIDITY` |
| Vault look-alikes are useless | Exhaustive scan + `count == 1` + pinned balance + covenant binding of the continuation |
| Vaults cannot be topped up by third parties | `borrow_scheme == BORROW_DISABLED` checked on input, forced on output |
| Config and links never change | `checkConfig` / `checkLinks` in `DexAmm`, plus the AMM state rebuild in the modules |
| Modules only serve their AMM, and the AMM only delegates to its modules | Two-way check: `liquidity` (designated ids), modules (`amm_id` + `deposit_id`/`withdraw_id`) |
| No double initialization | `init` requires `!initialized` |

---

## 9. Size and limits

| Contract | Bytecode | State |
|---|---|---|
| `DexAmm` | ~12.2 KB | 278 bytes |
| `DexKasPool` | 219 bytes | 33 bytes |
| `DexDeposit` | ~15.2 KB | 33 bytes |
| `DexWithdraw` | ~14.9 KB | 33 bytes |

These fit post-Toccata script limits. A first version of `DexAmm` exceeded the **244-item consensus stack limit**, which drove three choices:
- the state was reduced to 13 fields (LP templates and `fee_bps` moved to code; module hashes folded into `modules_hash`);
- `swap` forbids any LP input instead of scanning for the LP vault;
- the modules receive only the three new accounting values and rebuild the AMM output state themselves.

---

## 10. Known limitations and open issues (to resolve during review)

1. **Not audited.** AI-generated first draft.
2. **Code duplication.** `AmmState` and the helpers (`mul128`, `requireMulLe`, `scanOwned`, `checkVault`) are copied in three files and must be kept identical by hand.
3. **LP genesis trust.** If `TokenSeed` (not in the provided files) can create several `PublicMint`s for one covenant id, `init` cannot prove that the total LP supply equals `lp_total`. This needs on-chain proof or an off-chain history check.
4. **Front-end trust.** Whitelisting of token templates, correct `modules_hash`, correct genesis construction, and display of only fully consistent pools.
5. **Singleton contention.** One operation per pool per `DexAmm` UTXO, so in practice about one per block.
6. **No protocol fee**, and fixed `fee_bps` / `MIN_LIQUIDITY` / `lp_total` values still to be decided.
7. **No composition with the escrows** (order book) in one transaction. Their 4-output block layout is not used here.
8. **Untested areas**:
   - arithmetic at the 2⁶² bounds;
   - long sequences of swaps (monotonic `k`);
   - several pools in one transaction;
   - withdrawals that empty a reserve;
   - other KCC20 types (e.g. 9/10);
   - a `PublicMint` present in a DEX transaction.
9. **Genesis-output opcodes.** `OpCovOutputCount` does not see genesis outputs (this was the bug fixed in `init`). Any future check on freshly created covenants must account for that.

---

## 11. Test status

`DEX/test/` contains 32 A/B VM tests. In each pair, an honest transaction is accepted by every input, and a variant with one wrong parameter is rejected by the intended contract alone. They cover mint and burn, swap, vault attacks, AMM ↔ module links and `init`. All pass. See `DEX/test/TESTS.md`.

```bash
cargo test -p dex-vm-tests --release -- --test-threads=1
```

Passing tests do not prove security. They only show that the scenarios listed behave as intended.
