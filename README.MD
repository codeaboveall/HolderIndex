# Holder Index (Solana Edition)

![Holder Index Logo](holderindexsolana.png)

## Overview

Holder Index is a continuous on-chain ranking and reward distribution engine built natively for Solana. It maintains a live, deterministically computed leaderboard of wallet addresses based on configurable behavioral metrics, with rewards flowing continuously to top-ranked participants.

This is not a snapshot-based airdrop system. This is not a governance voting mechanism. This is a persistent, real-time indexing layer that tracks wallet-level activity across the Solana blockchain and redistributes value based on quantifiable on-chain behavior.

### What This Is

Holder Index is an experiment in continuous incentive alignment. Traditional reward systems operate on discrete snapshots: measure state at block N, calculate distributions, execute payouts, end epoch. Holder Index inverts this model by treating the leaderboard itself as a living data structure that responds to every relevant state transition on-chain.

Wallets are ranked continuously. Rankings drive reward eligibility. Eligibility is recalculated with every indexer cycle. Rewards flow proportionally to rank positioning over time.

The system is designed to answer: **What happens when you make wallet ranking a primitive, rather than a derived metric?**

### Why This Exists

Solana's architecture—high throughput, deterministic slot progression, sub-second finality, and affordable state reads—makes continuous indexing economically viable. On other chains, the cost of maintaining a live ranking system would be prohibitive. On Solana, it becomes a tractable infrastructure problem.

Holder Index exists to explore:

- Continuous incentive mechanisms vs epoch-based systems
- Wallet-level behavioral scoring vs token-centric metrics
- Real-time leaderboard dynamics and their game-theoretic implications
- Deterministic reward distribution without governance intervention

### Problem Space

Most on-chain reward systems suffer from temporal discretization. Users optimize behavior around snapshot timing. Activity spikes before epochs. Mercenary capital rotates in and out. Long-term alignment is difficult to enforce.

Holder Index attempts to solve this by making the reward function continuous. There is no optimal time to enter or exit. Your rank is a function of sustained behavior over time. Gaming the system requires sustained resource commitment, not timing exploitation.

This is particularly relevant for:

- Protocol-owned liquidity incentives
- Holder retention mechanisms
- Anti-dumping safeguards
- Long-term ecosystem alignment
- Merit-based distribution without manual curation

![System Architecture Blueprint](system_architecture.png)

---

## System Architecture

Holder Index is composed of five primary subsystems:

```
┌─────────────────────────────────────────────────────────────────┐
│                         SOLANA BLOCKCHAIN                        │
│                                                                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐          │
│  │   Wallets    │  │  SPL Tokens  │  │ Transactions │          │
│  └──────────────┘  └──────────────┘  └──────────────┘          │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            │ RPC + WebSocket
                            │
                ┌───────────▼──────────────┐
                │   INDEXER LAYER          │
                │                          │
                │  - Account discovery     │
                │  - Balance tracking      │
                │  - Transaction parsing   │
                │  - State synchronization │
                └───────────┬──────────────┘
                            │
                            │ Normalized data
                            │
                ┌───────────▼──────────────┐
                │   SCORING ENGINE         │
                │                          │
                │  - Balance weighting     │
                │  - Time decay functions  │
                │  - Activity modifiers    │
                │  - Deterministic compute │
                └───────────┬──────────────┘
                            │
                            │ Score vectors
                            │
                ┌───────────▼──────────────┐
                │   LEADERBOARD ENGINE     │
                │                          │
                │  - Rank computation      │
                │  - Sort optimization     │
                │  - Collision resolution  │
                │  - Historical tracking   │
                └───────────┬──────────────┘
                            │
                            │ Ranked wallets
                            │
                ┌───────────▼──────────────┐
                │   REWARD DISTRIBUTION    │
                │                          │
                │  - Eligibility checking  │
                │  - Proportional splitting│
                │  - Transaction batching  │
                │  - Emission scheduling   │
                └───────────┬──────────────┘
                            │
                            │ Payout instructions
                            │
                ┌───────────▼──────────────┐
                │   FRONTEND LAYER         │
                │                          │
                │  - REST API              │
                │  - WebSocket streams     │
                │  - Cached leaderboard    │
                │  - Historical data       │
                └──────────────────────────┘
```

### Indexer Layer

The indexer is responsible for maintaining synchronized state between the canonical Solana blockchain and the Holder Index database. It operates in two modes:

**Historical Mode**: On initialization, the indexer performs a full historical scan from a genesis slot to current. It identifies all relevant wallet addresses, reconstructs balance timelines, and hydrates the scoring engine with initial state.

**Live Mode**: Post-synchronization, the indexer subscribes to Solana's WebSocket API for real-time account updates. It listens for:

- Account balance changes (SOL)
- Token account mutations (SPL)
- Relevant program invocations
- Slot confirmations for finality

The indexer maintains a local state machine that mirrors on-chain state with minimal latency. It does not trust RPC nodes implicitly—all data is cross-validated against multiple endpoints where possible.

**Key Operations:**

- Account discovery via `getProgramAccounts` with filtering
- WebSocket subscriptions to `accountSubscribe` and `logsSubscribe`
- Slot tracking via `slotSubscribe` for temporal ordering
- Commitment-level handling (processed vs confirmed vs finalized)

**Data Structures:**

```rust
struct IndexedWallet {
    address: Pubkey,
    sol_balance: u64,
    token_balances: HashMap<Pubkey, u64>,
    first_seen_slot: u64,
    last_updated_slot: u64,
    transaction_count: u64,
}

struct IndexerState {
    wallets: HashMap<Pubkey, IndexedWallet>,
    current_slot: u64,
    last_sync_timestamp: i64,
    rpc_endpoints: Vec<String>,
}
```

### Scoring Engine

The scoring engine transforms raw wallet state into scalar ranking values. Scores are deterministic—given identical input state and scoring parameters, the engine will always produce identical outputs.

Scoring happens in discrete cycles. Each cycle:

1. Reads current wallet state from indexer
2. Applies configured scoring functions
3. Outputs a score vector for each wallet
4. Persists scores with cycle metadata

Scores are **not** cumulative across cycles. Each cycle represents an independent evaluation of current state.

**Core Scoring Model:**

```
Score(wallet, t) = BaseScore(wallet, t) × TimeWeight(wallet, t) × ActivityModifier(wallet, t)
```

Where:

- `BaseScore` = f(balance, holdings, token diversity)
- `TimeWeight` = g(holding_duration, entry_slot, current_slot)
- `ActivityModifier` = h(transaction_frequency, interaction_patterns)

**Determinism Requirements:**

- All scoring functions must be pure (no side effects)
- No external data dependencies (no API calls, no oracles)
- Floating point operations must be avoided (use fixed-point math)
- Slot-based timing only (no wall-clock timestamps)

![Scoring Algorithm Blueprint](scoring_blueprint.png)

### Leaderboard Engine

The leaderboard engine consumes score vectors and produces a ranked ordering of wallets. It operates on a fixed update cadence (configurable, default: every 10 slots).

**Ranking Algorithm:**

```
1. Collect all scores from latest scoring cycle
2. Sort wallets by score (descending)
3. Assign ranks 1..N
4. Handle ties via deterministic tiebreaker (e.g., lexicographic address ordering)
5. Persist ranked list with cycle ID and slot height
6. Calculate rank deltas from previous cycle
```

**Optimization Considerations:**

For large wallet sets (100k+ wallets), full sorting on every cycle becomes expensive. The engine implements:

- Partial sorting for top-K results (most queries only need top 100)
- Incremental updates for wallets whose scores haven't changed
- Binary heap for efficient rank tracking
- Bloom filters for quick eligibility checks

**Collision Handling:**

When two wallets have identical scores, rank assignment must be deterministic. The tiebreaker is:

```
if score_a == score_b:
    rank = compare(address_a.bytes, address_b.bytes)
```

This ensures that leaderboard ordering is fully reproducible across different implementations.

### Reward Distribution Engine

The distribution engine allocates protocol-sourced rewards to top-ranked wallets. It operates independently from scoring and ranking—the leaderboard is the source of truth.

**Distribution Model:**

Rewards can be configured as:

**Proportional**: Rewards are split based on relative score

```
reward_share(wallet) = score(wallet) / sum(scores[top_N])
```

**Tiered**: Fixed payouts per rank band

```
Ranks 1-10:    100 tokens each
Ranks 11-50:   50 tokens each
Ranks 51-100:  10 tokens each
```

**Exponential Decay**: Rewards decay exponentially with rank

```
reward(rank) = base_reward × decay_factor^(rank-1)
```

**Emission Scheduling:**

Distributions occur on a fixed interval (default: every 100 slots). Each distribution:

1. Reads current leaderboard state
2. Calculates reward allocation per configured model
3. Generates payout instructions
4. Batches transactions for efficiency
5. Submits to Solana network
6. Tracks distribution success/failure

**Safeguards:**

- Maximum payout per wallet per cycle (prevents runaway emissions)
- Minimum holding duration before eligibility (anti-gaming)
- Cooldown periods after large balance changes (anti-wash trading)
- Emergency pause mechanism

![Data Flow Diagram](data_flow.png)

### Frontend Consumption Layer

The frontend layer exposes leaderboard and reward data via REST API and WebSocket streams.

**Endpoints:**

```
GET  /leaderboard              - Current top 100 wallets
GET  /leaderboard?limit=1000   - Top N wallets
GET  /wallet/{address}         - Specific wallet rank and score
GET  /wallet/{address}/history - Historical rank progression
GET  /rewards/{address}        - Reward claim history
WS   /stream/leaderboard       - Real-time leaderboard updates
WS   /stream/wallet/{address}  - Real-time wallet updates
```

**Caching Strategy:**

- Leaderboard responses cached for 5 seconds (balances freshness vs load)
- Individual wallet queries bypass cache (users expect real-time)
- Historical data cached indefinitely (immutable)

**Pagination:**

Large leaderboards are paginated:

```
GET /leaderboard?offset=100&limit=50
```

Returns ranks 101-150.

---

## Wallet Indexing Model

The indexer must discover, track, and maintain state for all relevant wallet addresses on Solana. This is a non-trivial problem given Solana's scale.

### Discovery Mechanisms

**Method 1: Program Account Scanning**

For SPL token holders, use `getProgramAccounts` filtered by token mint:

```javascript
const accounts = await connection.getProgramAccounts(
  TOKEN_PROGRAM_ID,
  {
    filters: [
      { dataSize: 165 },
      {
        memcmp: {
          offset: 0,
          bytes: targetMint.toBase58(),
        }
      }
    ]
  }
);
```

This returns all token accounts for a given mint. Extract owner addresses to build initial wallet list.

**Method 2: Transaction Parsing**

Monitor recent transactions and extract signers:

```javascript
const signatures = await connection.getSignaturesForAddress(
  programId,
  { limit: 1000 }
);

for (const sig of signatures) {
  const tx = await connection.getTransaction(sig.signature);
  const wallets = tx.transaction.message.accountKeys.slice(0, tx.transaction.message.header.numRequiredSignatures);
  // Index wallets
}
```

**Method 3: Explicit Registration**

Allow wallets to opt-in via a registration program. This reduces indexer scope but requires user action.

### Inclusion Logic

A wallet is included in the index if:

- It holds a minimum balance threshold (configurable, e.g., 0.1 SOL or 100 tokens)
- It has been active within the last N slots (configurable, e.g., 432,000 slots = ~48 hours)
- It is not blacklisted (anti-spam, anti-exploit)

### Exclusion Logic

A wallet is excluded if:

- Balance falls below minimum threshold for M consecutive cycles
- Flagged by anti-manipulation heuristics
- Explicitly blacklisted (program authority action)

### Base58 Address Handling

Solana addresses are 32-byte Ed25519 public keys encoded as base58 strings. Internal storage uses raw byte arrays for efficiency:

```rust
struct WalletAddress([u8; 32]);

impl WalletAddress {
    fn from_base58(s: &str) -> Result<Self, Error> {
        let bytes = bs58::decode(s).into_vec()?;
        if bytes.len() != 32 {
            return Err(Error::InvalidAddress);
        }
        let mut arr = [0u8; 32];
        arr.copy_from_slice(&bytes);
        Ok(WalletAddress(arr))
    }

    fn to_base58(&self) -> String {
        bs58::encode(&self.0).into_string()
    }
}
```

### New Wallet vs Existing Wallet

When a wallet first appears in indexer output:

1. Assign `first_seen_slot` = current slot
2. Initialize balance history with current state
3. Set score = 0 (ineligible until minimum holding duration met)
4. Add to index with `status = pending`

After `MINIMUM_HOLDING_DURATION` slots, wallet becomes eligible for scoring.

When an existing wallet is updated:

1. Append new state to balance history
2. Calculate time-weighted metrics
3. Trigger score recalculation
4. Update leaderboard if rank changes

---

## Scoring Mechanics

Scoring is the most critical and most configurable component. The goal is to produce a scalar value that represents "desirability" of a wallet from the protocol's perspective.

### Balance-Based Scoring

The simplest scoring function is raw balance:

```
Score = Balance(SOL) + Σ(Balance(Token_i) × Weight_i)
```

Where `Weight_i` is a per-token multiplier (e.g., stablecoins weighted lower than native tokens).

**Limitations:**

- Favors whales disproportionately
- No incentive for long-term holding
- Vulnerable to sybil attacks via balance splitting

### Time-Weighted Holding

To reward duration, incorporate holding time into scoring:

```
Score = Balance × HoldingDuration
```

Where `HoldingDuration` is measured in slots since balance last changed significantly.

**Decay Function:**

Use exponential decay to prevent ancient balances from dominating:

```
TimeWeight = 1 - e^(-λ × HoldingDuration)
```

Where `λ` controls decay rate. Higher `λ` = faster decay = more emphasis on recent holdings.

**Implementation:**

```rust
fn calculate_time_weight(holding_duration_slots: u64, lambda: f64) -> f64 {
    let duration_normalized = holding_duration_slots as f64 / SLOTS_PER_DAY as f64;
    1.0 - (-lambda * duration_normalized).exp()
}
```

### Activity Modifiers

Raw balance × time can be gamed by creating a wallet, funding it, and forgetting it. To reward active participation:

```
ActivityScore = f(transaction_count, interaction_diversity, recency)
```

Where:

- `transaction_count` = total transactions signed by wallet
- `interaction_diversity` = number of unique programs interacted with
- `recency` = slots since last transaction

**Example Modifier:**

```
ActivityMultiplier = 1 + (0.1 × log(1 + transaction_count)) × RecencyBonus
```

Where:

```
RecencyBonus = {
    1.0  if last_tx_slot > current_slot - 1000
    0.8  if last_tx_slot > current_slot - 10000
    0.5  otherwise
}
```

This rewards active wallets without making activity a hard requirement.

### Configurable Scoring Weights

The full scoring function is:

```
Score = (W_balance × BalanceScore) + (W_time × TimeScore) + (W_activity × ActivityScore)
```

Where `W_balance`, `W_time`, `W_activity` are configurable weights that sum to 1.0.

**Example Configurations:**

**Pure Holdings:**
```json
{
  "W_balance": 1.0,
  "W_time": 0.0,
  "W_activity": 0.0
}
```

**Balanced:**
```json
{
  "W_balance": 0.5,
  "W_time": 0.3,
  "W_activity": 0.2
}
```

**Activity-Heavy:**
```json
{
  "W_balance": 0.3,
  "W_time": 0.2,
  "W_activity": 0.5
}
```

### Deterministic Recalculation Cycles

Scores must be recalculated regularly. The cycle length determines how responsive the system is to state changes.

**Short Cycles (10-50 slots):**
- Pros: Near real-time responsiveness
- Cons: High computational load, frequent leaderboard churn

**Long Cycles (1000+ slots):**
- Pros: Stable leaderboard, lower compute
- Cons: Delayed reaction to state changes

**Recommended:** 100-500 slot cycles (12-60 seconds on Solana).

**Cycle Workflow:**

```
1. Wait for cycle_trigger_slot
2. Read all indexed wallet states at slot S
3. For each wallet:
     score = calculate_score(wallet, config, S)
4. Persist scores with cycle_id and slot
5. Trigger leaderboard update
6. Schedule next cycle at S + cycle_length
```

### Pseudo-Mathematical Formalization

Let `W` be the set of all indexed wallets. For wallet `w ∈ W` at slot `s`, define:

```
B(w, s) = SOL balance + Σ(Token_i balance × α_i)
H(w, s) = s - slot(last significant balance change)
A(w, s) = transaction count since first seen
D(w, s) = number of unique program interactions

TimeWeight(w, s) = 1 - e^(-λ × H(w, s) / SLOTS_PER_DAY)

ActivityWeight(w, s) = 1 + β × log(1 + A(w, s)) × RecencyFactor(w, s)

Score(w, s) = [W_b × B(w, s) + W_h × H(w, s) × B(w, s)] × ActivityWeight(w, s)
```

Where:
- `α_i` = per-token weight multipliers
- `λ` = time decay constant
- `β` = activity boost multiplier
- `W_b`, `W_h` = balance and holding weight coefficients

This formulation is deterministic, reproducible, and tunable via parameters.

![Scoring Weight Visualization](scoring_weights.png)

---

## Leaderboard Engine

The leaderboard is an ordered list of wallets derived from scores. It must be efficiently computable, queryable, and historically traceable.

### Sorting Logic

Standard approach:

```python
def update_leaderboard(scores, previous_leaderboard):
    # Sort by score descending, tiebreak by address
    ranked = sorted(
        scores.items(),
        key=lambda x: (-x[1], x[0])  # Score desc, address asc
    )
    
    leaderboard = []
    for rank, (address, score) in enumerate(ranked, start=1):
        prev_rank = previous_leaderboard.get(address, None)
        delta = prev_rank - rank if prev_rank else None
        
        leaderboard.append({
            'rank': rank,
            'address': address,
            'score': score,
            'previous_rank': prev_rank,
            'rank_change': delta
        })
    
    return leaderboard
```

### Update Cadence

Leaderboard updates are decoupled from scoring cycles. A typical setup:

- Scoring cycle: every 100 slots
- Leaderboard update: every 10 slots

This means the leaderboard uses slightly stale scores but updates more frequently for frontend responsiveness.

**Alternative:** Update leaderboard only when scores change. This saves computation but adds complexity (need to track score deltas).

### Handling Rank Collisions

When two wallets have identical scores, rank assignment must be deterministic and stable. Use lexicographic ordering of base58 addresses:

```rust
fn compare_addresses(a: &str, b: &str) -> Ordering {
    a.cmp(b)
}

fn rank_wallets(scores: Vec<(String, u64)>) -> Vec<RankedWallet> {
    let mut sorted = scores;
    sorted.sort_by(|a, b| {
        match b.1.cmp(&a.1) {  // Score descending
            Ordering::Equal => a.0.cmp(&b.0),  // Address ascending
            other => other,
        }
    });
    
    sorted.into_iter()
        .enumerate()
        .map(|(i, (addr, score))| RankedWallet {
            rank: i + 1,
            address: addr,
            score,
        })
        .collect()
}
```

### On-Chain vs Off-Chain Computation

**Off-Chain Leaderboard (Current Approach):**

- Indexer + scorer + leaderboard engine run on centralized infrastructure
- Fast, flexible, cheap
- Not trustless (requires trust in operator)
- Suitable for most use cases

**On-Chain Leaderboard (Theoretical):**

- Entire leaderboard stored in Solana accounts
- Updates via program instructions
- Fully trustless and verifiable
- Extremely expensive (rent costs, compute limits)
- Not practical for large leaderboards (1000+ wallets)

**Hybrid Approach:**

- Leaderboard computed off-chain
- Top-N wallets (e.g., top 100) committed to on-chain account
- Merkle root of full leaderboard stored on-chain
- Allows for trustless verification of inclusions without full on-chain storage

```rust
// On-chain storage structure
pub struct LeaderboardState {
    pub current_slot: u64,
    pub merkle_root: [u8; 32],
    pub top_wallets: [Pubkey; 100],
    pub top_scores: [u64; 100],
    pub authority: Pubkey,
}
```

### Historical Tracking

Maintain a timeseries database of leaderboard snapshots:

```sql
CREATE TABLE leaderboard_history (
    cycle_id INTEGER,
    slot INTEGER,
    rank INTEGER,
    address TEXT,
    score INTEGER,
    PRIMARY KEY (cycle_id, rank)
);

CREATE INDEX idx_address_history ON leaderboard_history(address, cycle_id);
```

This allows queries like:

- "What was wallet X's rank at slot S?"
- "Show me the top 10 wallets for the last 100 cycles"
- "Which wallets have been in top 10 for >50 consecutive cycles?"

---

## Continuous Rewards Model

The reward distribution mechanism ties directly to leaderboard position. Rewards flow continuously to eligible wallets based on their rank over time.

### How Rewards Flow

**Source of Rewards:**

Rewards come from protocol-controlled treasury accounts. These can be funded via:

- Protocol fee collection
- Token emissions
- External capital allocations
- Yield from protocol-owned liquidity

**Emission Rate:**

Define a per-slot emission rate:

```
EmissionPerSlot = TotalEmissionBudget / SlotsInPeriod
```

For example:

- Total budget: 1,000,000 tokens
- Period: 30 days = 30 × 24 × 60 × 60 × 2 slots ≈ 5,184,000 slots
- Emission per slot: 1,000,000 / 5,184,000 ≈ 0.193 tokens/slot

Every distribution cycle (e.g., every 100 slots), accumulate emissions:

```
EmissionForCycle = EmissionPerSlot × CycleLength
```

### Distribution Intervals

**Fixed Interval Model:**

Distribute every N slots (e.g., N=100):

```
Cycle 1:  Slots 0-99    → Distribute 19.3 tokens
Cycle 2:  Slots 100-199 → Distribute 19.3 tokens
...
```

**Dynamic Interval Model:**

Distribute when accumulated emissions exceed threshold:

```
if accumulated_emissions >= MIN_DISTRIBUTION_AMOUNT:
    trigger_distribution()
    accumulated_emissions = 0
```

This reduces transaction frequency for low-emission protocols.

### Proportional vs Tiered Payouts

**Proportional Distribution:**

Split emissions based on score ratio:

```python
def proportional_distribution(leaderboard, total_emission):
    top_n = leaderboard[:100]  # Top 100 only
    total_score = sum(w['score'] for w in top_n)
    
    distributions = []
    for wallet in top_n:
        share = (wallet['score'] / total_score) * total_emission
        distributions.append({
            'address': wallet['address'],
            'amount': share
        })
    
    return distributions
```

**Tiered Distribution:**

Fixed payouts per rank band:

```python
def tiered_distribution(leaderboard):
    distributions = []
    
    for wallet in leaderboard:
        if wallet['rank'] <= 10:
            amount = 100
        elif wallet['rank'] <= 50:
            amount = 50
        elif wallet['rank'] <= 100:
            amount = 10
        else:
            amount = 0
        
        if amount > 0:
            distributions.append({
                'address': wallet['address'],
                'amount': amount
            })
    
    return distributions
```

**Hybrid:**

Combine both approaches:

- Base payout per rank tier
- Bonus proportional to score within tier

### Emissions Logic

**Linear Emissions:**

Constant emission rate throughout period.

```
E(t) = E_0
```

**Decay Emissions:**

Emission rate decreases over time to incentivize early participation:

```
E(t) = E_0 × e^(-δt)
```

Where `δ` is decay constant.

**Stepped Emissions:**

Emission rate changes at predefined milestones:

```
E(t) = {
    100 tokens/cycle  if t < 1000 cycles
    50 tokens/cycle   if t < 2000 cycles
    25 tokens/cycle   otherwise
}
```

### Safeguards Against Gaming

**Maximum Payout Per Wallet:**

Prevent single wallet from extracting disproportionate value:

```python
MAX_PAYOUT_PER_WALLET_PER_CYCLE = 1000

for distribution in distributions:
    distribution['amount'] = min(
        distribution['amount'],
        MAX_PAYOUT_PER_WALLET_PER_CYCLE
    )
```

**Minimum Holding Duration:**

Wallet must be indexed for minimum duration before eligibility:

```python
MINIMUM_HOLDING_DURATION = 1000  # slots

def is_eligible(wallet, current_slot):
    holding_duration = current_slot - wallet['first_seen_slot']
    return holding_duration >= MINIMUM_HOLDING_DURATION
```

**Cooldown Windows:**

After large balance increase, wallet enters cooldown:

```python
COOLDOWN_DURATION = 500  # slots
LARGE_INCREASE_THRESHOLD = 10000  # tokens

def check_cooldown(wallet, current_slot):
    if wallet['last_large_increase_slot']:
        cooldown_remaining = (wallet['last_large_increase_slot'] + COOLDOWN_DURATION) - current_slot
        if cooldown_remaining > 0:
            return False, cooldown_remaining
    return True, 0
```

**Anti-Sybil Wallet Clustering:**

Detect wallets with correlated behavior (see next section).

![Reward Distribution Flow](reward_flow.png)

---

## Anti-Manipulation Design

Gaming the ranking system is the primary threat. Robust anti-manipulation mechanisms are essential.

### Sybil Resistance Strategies

**Challenge:** Attacker creates many wallets with small balances to flood leaderboard.

**Mitigation 1: Minimum Balance Threshold**

Require wallets to hold minimum balance to be eligible:

```
MINIMUM_BALANCE = 1.0 SOL or 1000 tokens
```

This creates economic cost for sybil attacks.

**Mitigation 2: Transaction Cost Analysis**

Track transaction fees paid by wallet. Wallets with abnormally low fee expenditure relative to holdings may be dormant sybils:

```python
def calculate_sybil_score(wallet):
    balance = wallet['balance']
    fees_paid = wallet['total_fees_paid']
    
    expected_fees = balance * 0.0001  # Heuristic: 0.01% of balance
    
    if fees_paid < expected_fees:
        sybil_score = 1.0 - (fees_paid / expected_fees)
    else:
        sybil_score = 0.0
    
    return sybil_score
```

Wallets with high sybil score get downweighted in scoring.

**Mitigation 3: Identity Staking**

Require one-time staking of tokens to participate. Staked amount is locked but earns yield. This creates opportunity cost for sybil creation.

### Wallet Clustering Heuristics

**Challenge:** Attacker controls multiple wallets but tries to hide relationship.

**Detection Method 1: Common Funding Sources**

Build a transaction graph and identify wallets funded from same source:

```
Wallet A ← Funded by ← Exchange Wallet X
Wallet B ← Funded by ← Exchange Wallet X
Wallet C ← Funded by ← Exchange Wallet X
```

All three wallets likely controlled by same entity.

**Detection Method 2: Temporal Correlation**

Analyze balance change timing:

```python
def calculate_temporal_correlation(wallet_a, wallet_b):
    # Get balance change timestamps
    changes_a = wallet_a['balance_change_slots']
    changes_b = wallet_b['balance_change_slots']
    
    # Calculate correlation coefficient
    correlation = numpy.corrcoef(changes_a, changes_b)[0, 1]
    
    return correlation
```

High correlation suggests coordinated behavior.

**Detection Method 3: Round-Robin Transfers**

Detect circular transfer patterns:

```
Wallet A → Wallet B → Wallet C → Wallet A
```

This is a classic wash trading pattern.

**Action on Detected Clusters:**

- Option 1: Treat cluster as single entity (sum balances, assign single score)
- Option 2: Downweight all wallets in cluster by factor (e.g., 0.5x)
- Option 3: Exclude cluster from leaderboard entirely

### Cooldown Windows

After significant balance change, wallet enters cooldown where score is frozen or reduced:

```python
COOLDOWN_DURATION = 1000  # slots
SIGNIFICANT_CHANGE_THRESHOLD = 0.25  # 25% balance change

def apply_cooldown(wallet, previous_balance, current_balance, current_slot):
    change_ratio = abs(current_balance - previous_balance) / previous_balance
    
    if change_ratio >= SIGNIFICANT_CHANGE_THRESHOLD:
        wallet['cooldown_until_slot'] = current_slot + COOLDOWN_DURATION
        wallet['cooldown_active'] = True
        return True
    
    return False

def get_cooldown_multiplier(wallet, current_slot):
    if wallet.get('cooldown_active') and current_slot < wallet['cooldown_until_slot']:
        return 0.5  # Reduced scoring during cooldown
    return 1.0
```

### Penalty Scoring

Apply multiplicative penalties for suspicious behavior:

```python
def calculate_penalty_multiplier(wallet):
    multiplier = 1.0
    
    # Sybil penalty
    if wallet['sybil_score'] > 0.7:
        multiplier *= 0.3
    
    # Wash trading penalty
    if wallet['wash_trading_detected']:
        multiplier *= 0.1
    
    # Cooldown penalty
    if wallet['cooldown_active']:
        multiplier *= 0.5
    
    # Cluster penalty
    if wallet['in_cluster']:
        multiplier *= 0.6
    
    return multiplier

# Apply in scoring
final_score = base_score * calculate_penalty_multiplier(wallet)
```

### Rate Limiting

Prevent rapid score manipulation by limiting how fast scores can change:

```python
MAX_SCORE_INCREASE_PER_CYCLE = 1000

def apply_rate_limit(wallet, new_score, previous_score):
    score_delta = new_score - previous_score
    
    if score_delta > MAX_SCORE_INCREASE_PER_CYCLE:
        return previous_score + MAX_SCORE_INCREASE_PER_CYCLE
    
    return new_score
```

This prevents sudden jumps from large balance transfers.

---

## Solana-Specific Considerations

Building on Solana introduces unique architectural constraints and opportunities.

### RPC Load Management

**Challenge:** Polling Solana RPC endpoints at high frequency can hit rate limits or overwhelm nodes.

**Solution 1: Multi-Endpoint Rotation**

Distribute requests across multiple RPC providers:

```javascript
const RPC_ENDPOINTS = [
    'https://api.mainnet-beta.solana.com',
    'https://solana-api.projectserum.com',
    'https://rpc.ankr.com/solana',
];

let currentEndpointIndex = 0;

function getConnection() {
    const endpoint = RPC_ENDPOINTS[currentEndpointIndex];
    currentEndpointIndex = (currentEndpointIndex + 1) % RPC_ENDPOINTS.length;
    return new Connection(endpoint);
}
```

**Solution 2: Request Batching**

Batch multiple account queries into single `getMultipleAccounts` call:

```javascript
const BATCH_SIZE = 100;

async function batchFetchAccounts(addresses) {
    const batches = [];
    for (let i = 0; i < addresses.length; i += BATCH_SIZE) {
        batches.push(addresses.slice(i, i + BATCH_SIZE));
    }
    
    const results = [];
    for (const batch of batches) {
        const accounts = await connection.getMultipleAccounts(batch);
        results.push(...accounts);
    }
    
    return results;
}
```

**Solution 3: Caching**

Cache account data with short TTL:

```javascript
const accountCache = new Map();
const CACHE_TTL = 5000; // 5 seconds

async function getCachedAccount(address) {
    const cached = accountCache.get(address);
    if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
        return cached.data;
    }
    
    const account = await connection.getAccountInfo(address);
    accountCache.set(address, {
        data: account,
        timestamp: Date.now()
    });
    
    return account;
}
```

### WebSocket Subscriptions

**Best Practice:** Use WebSockets for real-time updates, fallback to polling for historical sync.

```javascript
// Subscribe to account changes
const subscriptionId = connection.onAccountChange(
    accountAddress,
    (accountInfo, context) => {
        console.log('Account updated at slot:', context.slot);
        indexer.handleAccountUpdate(accountAddress, accountInfo, context.slot);
    },
    'confirmed'  // Commitment level
);

// Subscribe to slot updates for temporal tracking
const slotSubscriptionId = connection.onSlotChange((slotInfo) => {
    indexer.updateCurrentSlot(slotInfo.slot);
});
```

**WebSocket Stability:**

WebSocket connections can drop. Implement reconnection logic:

```javascript
let wsConnection;
let reconnectAttempts = 0;
const MAX_RECONNECT_ATTEMPTS = 10;

function connectWebSocket() {
    wsConnection = new Connection(RPC_ENDPOINT, 'confirmed');
    
    wsConnection.onAccountChange(/* ... */);
    
    // Detect disconnection
    const heartbeat = setInterval(async () => {
        try {
            await wsConnection.getSlot();
        } catch (error) {
            console.error('WebSocket disconnected, reconnecting...');
            clearInterval(heartbeat);
            if (reconnectAttempts < MAX_RECONNECT_ATTEMPTS) {
                reconnectAttempts++;
                setTimeout(connectWebSocket, 1000 * reconnectAttempts);
            }
        }
    }, 30000);  // Check every 30 seconds
}
```

### Slot Timing

Solana targets 400ms slot time but actual time varies. Never rely on wall-clock time for ordering—use slot numbers exclusively.

```javascript
// BAD: Using timestamps
const now = Date.now();
if (wallet.lastUpdated < now - 60000) {
    // This is fragile
}

// GOOD: Using slots
const currentSlot = await connection.getSlot();
if (wallet.lastUpdatedSlot < currentSlot - 150) {  // 150 slots ≈ 60 seconds
    // This is deterministic
}
```

### Finality Assumptions

Solana has three commitment levels:

- **Processed**: Transaction included in a block, not yet finalized
- **Confirmed**: Supermajority of cluster has voted on block
- **Finalized**: Block is confirmed by 31 blocks (~12.8 seconds)

**Recommendation:** Use **confirmed** for indexing (faster updates) and **finalized** for distribution eligibility (prevents reorg issues).

```javascript
// For balance tracking: confirmed
const balance = await connection.getBalance(address, 'confirmed');

// For reward distribution: finalized
const eligibilitySlot = await connection.getSlot('finalized');
```

### Compute Budget Awareness

If implementing on-chain components, be aware of compute limits:

- Max 1.4M compute units per transaction (can be increased with `setComputeUnitLimit`)
- Complex calculations should be done off-chain

For leaderboard verification on-chain:

```rust
pub fn verify_inclusion(
    merkle_proof: Vec<[u8; 32]>,
    leaf: [u8; 32],
    root: [u8; 32]
) -> bool {
    let mut computed_hash = leaf;
    for sibling in merkle_proof {
        computed_hash = if computed_hash < sibling {
            hash(&[&computed_hash, &sibling].concat())
        } else {
            hash(&[&sibling, &computed_hash].concat())
        };
    }
    computed_hash == root
}
```

This allows wallets to prove their inclusion in leaderboard without requiring full leaderboard storage on-chain.

![Solana Architecture Integration](solana_integration.png)

---

## Example Data Flow

Let's trace the complete lifecycle of a wallet through the system.

### Scenario: New Wallet Enters Index

**T=0 (Slot 150,000,000)**

User creates new wallet:
```
Address: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU
```

User deposits 10 SOL:

```
Transaction: 3Jk8F...
From: Exchange Wallet
To: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU
Amount: 10 SOL
Slot: 150,000,000
```

**T=1 (Slot 150,000,001)**

Indexer detects new account via WebSocket:

```javascript
onAccountChange(7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU) {
    balance: 10 SOL,
    slot: 150,000,001
}
```

Indexer adds wallet to database:

```javascript
{
    address: '7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU',
    sol_balance: 10_000_000_000,  // lamports
    token_balances: {},
    first_seen_slot: 150_000_001,
    last_updated_slot: 150_000_001,
    status: 'pending',
    cooldown_until_slot: 150_001_001  // 1000 slot cooldown
}
```

**T=100 (Slot 150,000,100)**

First scoring cycle runs. Wallet still in cooldown, score = 0:

```javascript
{
    address: '7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU',
    score: 0,
    reason: 'cooldown_active'
}
```

**T=1000 (Slot 150,001,000)**

Cooldown expires. Next scoring cycle:

```javascript
balance_score = 10 SOL = 10
holding_duration = 150_001_000 - 150_000_001 = 999 slots
time_weight = 1 - exp(-0.01 × 999 / 216) = 0.044
activity_score = 1 + 0.1 × log(1 + 1) × 1.0 = 1.03

score = 10 × 0.044 × 1.03 = 0.45
```

Wallet enters leaderboard at rank 8,542.

**T=10,000 (Slot 150,010,000)**

User holds steadily. Score increases due to time weighting:

```javascript
holding_duration = 150_010_000 - 150_000_001 = 9,999 slots
time_weight = 1 - exp(-0.01 × 9999 / 216) = 0.36

score = 10 × 0.36 × 1.03 = 3.71
```

Wallet climbs to rank 3,214.

**T=50,000 (Slot 150,050,000)**

Wallet enters top 100 (rank 87). Now eligible for rewards.

**T=50,100 (Slot 150,050,100)**

Distribution cycle runs:

```javascript
top_100_wallets = get_leaderboard(limit=100)
total_emission_this_cycle = 19.3 tokens

wallet = top_100_wallets[86]  // Rank 87
score = 8.24
total_top_100_score = 12,847

wallet_share = (8.24 / 12847) × 19.3 = 0.0124 tokens
```

Distribution transaction sent:

```
Transaction: 2Mn9A...
Program: Token Program
Instruction: Transfer
From: Protocol Treasury
To: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU
Amount: 0.0124 tokens
Slot: 150,050,100
```

Wallet receives first reward.

### Scenario: Wallet Climbs Leaderboard

**T=100,000 (Slot 150,100,000)**

User adds more capital:

```
Transaction: 5Pk2D...
From: User's Other Wallet
To: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU
Amount: 40 SOL
Slot: 150,100,000
```

New balance: 50 SOL

Indexer triggers cooldown:

```javascript
balance_change = (50 - 10) / 10 = 4.0 = 400%
cooldown_until_slot = 150,100,000 + 1000 = 150,101,000
```

**T=100,100 (Slot 150,100,100)**

Scoring cycle runs during cooldown:

```javascript
base_score = 50 × time_weight × activity = 34.2
cooldown_multiplier = 0.5
final_score = 34.2 × 0.5 = 17.1
```

Rank drops temporarily to 142 (below top 100, no rewards).

**T=101,000 (Slot 150,101,000)**

Cooldown expires:

```javascript
score = 34.2  // Full score restored
```

Wallet jumps to rank 23.

**T=101,100 (Slot 150,101,100)**

Distribution cycle:

```javascript
wallet_share = (34.2 / 12847) × 19.3 = 0.0514 tokens
```

User receives 4x larger payout due to higher rank.

### Scenario: Wallet Exits Leaderboard

**T=200,000 (Slot 150,200,000)**

User withdraws most balance:

```
Transaction: 8Qn1F...
From: 7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU
To: Exchange Wallet
Amount: 49 SOL
Slot: 150,200,000
```

New balance: 1 SOL

**T=200,100 (Slot 150,200,100)**

Scoring cycle:

```javascript
balance_score = 1 SOL
score = 1 × time_weight × activity = 0.89
```

Wallet drops to rank 12,483 (out of top 100, no longer eligible).

**T=300,000 (Slot 150,300,000)**

Balance falls below minimum threshold. Wallet marked for removal:

```javascript
if (wallet.balance < MINIMUM_BALANCE && 
    current_slot - wallet.last_balance_change_slot > GRACE_PERIOD) {
    wallet.status = 'inactive';
    remove_from_index(wallet);
}
```

Wallet removed from index.

![Data Flow Lifecycle](lifecycle_diagram.png)

---

## Example Pseudocode

### Wallet Scoring Loop

```python
def scoring_cycle(indexer_state, scoring_config, current_slot):
    """
    Execute one complete scoring cycle.
    """
    cycle_id = generate_cycle_id()
    scores = {}
    
    # Iterate all active wallets
    for address, wallet in indexer_state.wallets.items():
        # Skip if below minimum balance
        if wallet.sol_balance < scoring_config.minimum_balance:
            continue
        
        # Skip if in cooldown
        if wallet.cooldown_until_slot and current_slot < wallet.cooldown_until_slot:
            scores[address] = {
                'score': 0,
                'reason': 'cooldown'
            }
            continue
        
        # Calculate base balance score
        balance_score = wallet.sol_balance / LAMPORTS_PER_SOL
        
        # Add weighted token balances
        for token_mint, token_balance in wallet.token_balances.items():
            token_weight = scoring_config.token_weights.get(token_mint, 0.1)
            balance_score += (token_balance * token_weight)
        
        # Calculate holding duration
        holding_duration = current_slot - wallet.first_seen_slot
        
        # Apply time weighting
        time_weight = calculate_time_weight(
            holding_duration,
            scoring_config.time_decay_lambda
        )
        
        # Calculate activity modifier
        activity_modifier = calculate_activity_modifier(
            wallet.transaction_count,
            wallet.unique_programs_count,
            wallet.last_transaction_slot,
            current_slot
        )
        
        # Compute final score
        score = (
            scoring_config.weight_balance * balance_score +
            scoring_config.weight_time * (balance_score * time_weight)
        ) * activity_modifier
        
        # Apply penalties
        penalty_multiplier = calculate_penalty_multiplier(wallet)
        score *= penalty_multiplier
        
        # Store score
        scores[address] = {
            'score': score,
            'components': {
                'balance': balance_score,
                'time_weight': time_weight,
                'activity': activity_modifier,
                'penalty': penalty_multiplier
            }
        }
    
    # Persist scores
    persist_scores(cycle_id, current_slot, scores)
    
    return scores


def calculate_time_weight(holding_duration_slots, lambda_param):
    """
    Exponential decay time weighting.
    """
    slots_per_day = 216000  # ~24 hours
    duration_days = holding_duration_slots / slots_per_day
    return 1.0 - math.exp(-lambda_param * duration_days)


def calculate_activity_modifier(tx_count, program_count, last_tx_slot, current_slot):
    """
    Activity-based score multiplier.
    """
    base_modifier = 1.0 + (0.1 * math.log(1 + tx_count))
    
    # Recency bonus
    slots_since_last_tx = current_slot - last_tx_slot
    if slots_since_last_tx < 1000:
        recency_bonus = 1.0
    elif slots_since_last_tx < 10000:
        recency_bonus = 0.8
    else:
        recency_bonus = 0.5
    
    # Diversity bonus
    diversity_bonus = min(1.0 + (program_count * 0.05), 1.5)
    
    return base_modifier * recency_bonus * diversity_bonus


def calculate_penalty_multiplier(wallet):
    """
    Apply cumulative penalties for suspicious behavior.
    """
    multiplier = 1.0
    
    if wallet.sybil_score > 0.7:
        multiplier *= 0.3
    
    if wallet.wash_trading_detected:
        multiplier *= 0.1
    
    if wallet.in_cluster:
        multiplier *= 0.6
    
    return multiplier
```

### Leaderboard Update Loop

```python
def update_leaderboard(scores, previous_leaderboard, current_slot):
    """
    Generate new leaderboard from scores.
    """
    # Convert scores dict to sortable list
    wallet_scores = [
        (address, data['score'])
        for address, data in scores.items()
        if data['score'] > 0
    ]
    
    # Sort by score descending, address ascending (tiebreaker)
    wallet_scores.sort(key=lambda x: (-x[1], x[0]))
    
    # Build leaderboard with rank metadata
    leaderboard = []
    for rank, (address, score) in enumerate(wallet_scores, start=1):
        # Get previous rank for delta calculation
        prev_entry = previous_leaderboard.get(address)
        prev_rank = prev_entry['rank'] if prev_entry else None
        rank_change = prev_rank - rank if prev_rank else None
        
        leaderboard.append({
            'rank': rank,
            'address': address,
            'score': score,
            'previous_rank': prev_rank,
            'rank_change': rank_change,
            'percentile': (len(wallet_scores) - rank) / len(wallet_scores)
        })
    
    # Persist leaderboard
    persist_leaderboard(current_slot, leaderboard)
    
    # Update cache
    update_leaderboard_cache(leaderboard)
    
    # Emit WebSocket updates for top 100
    emit_leaderboard_update(leaderboard[:100])
    
    return leaderboard


def get_leaderboard_page(offset=0, limit=100):
    """
    Paginated leaderboard query.
    """
    cache_key = f"leaderboard:{offset}:{limit}"
    
    # Check cache
    cached = redis.get(cache_key)
    if cached:
        return json.loads(cached)
    
    # Query database
    leaderboard = db.query(
        """
        SELECT rank, address, score, previous_rank, rank_change
        FROM current_leaderboard
        ORDER BY rank ASC
        LIMIT ? OFFSET ?
        """,
        limit, offset
    )
    
    # Cache for 5 seconds
    redis.setex(cache_key, 5, json.dumps(leaderboard))
    
    return leaderboard


def get_wallet_rank(address):
    """
    Get specific wallet's current rank.
    """
    result = db.query(
        """
        SELECT rank, score, previous_rank, rank_change, percentile
        FROM current_leaderboard
        WHERE address = ?
        """,
        address
    )
    
    return result[0] if result else None
```

### Reward Distribution Loop

```python
def distribution_cycle(leaderboard, emission_config, current_slot):
    """
    Calculate and execute reward distributions.
    """
    # Calculate emission for this cycle
    slots_since_last_distribution = current_slot - emission_config.last_distribution_slot
    emission_amount = emission_config.emission_per_slot * slots_since_last_distribution
    
    # Get eligible wallets (top N)
    eligible_wallets = leaderboard[:emission_config.top_n_eligible]
    
    # Calculate distributions based on model
    if emission_config.distribution_model == 'proportional':
        distributions = proportional_distribution(eligible_wallets, emission_amount)
    elif emission_config.distribution_model == 'tiered':
        distributions = tiered_distribution(eligible_wallets, emission_amount)
    else:
        raise ValueError(f"Unknown distribution model: {emission_config.distribution_model}")
    
    # Apply per-wallet caps
    for dist in distributions:
        dist['amount'] = min(
            dist['amount'],
            emission_config.max_payout_per_wallet_per_cycle
        )
    
    # Filter out amounts below minimum
    distributions = [
        d for d in distributions
        if d['amount'] >= emission_config.minimum_payout_amount
    ]
    
    # Execute distributions
    transaction_results = execute_distributions(distributions, current_slot)
    
    # Log distribution event
    log_distribution(
        slot=current_slot,
        total_amount=sum(d['amount'] for d in distributions),
        recipient_count=len(distributions),
        transactions=transaction_results
    )
    
    # Update emission config
    emission_config.last_distribution_slot = current_slot
    
    return distributions


def proportional_distribution(wallets, total_emission):
    """
    Proportional reward split based on score.
    """
    total_score = sum(w['score'] for w in wallets)
    
    distributions = []
    for wallet in wallets:
        share = (wallet['score'] / total_score) * total_emission
        distributions.append({
            'address': wallet['address'],
            'amount': share,
            'rank': wallet['rank'],
            'score': wallet['score']
        })
    
    return distributions


def tiered_distribution(wallets, total_emission):
    """
    Fixed payouts per rank tier.
    """
    tiers = [
        {'ranks': range(1, 11), 'amount': 100},
        {'ranks': range(11, 51), 'amount': 50},
        {'ranks': range(51, 101), 'amount': 10},
    ]
    
    distributions = []
    total_allocated = 0
    
    for wallet in wallets:
        rank = wallet['rank']
        
        # Find matching tier
        amount = 0
        for tier in tiers:
            if rank in tier['ranks']:
                amount = tier['amount']
                break
        
        if amount > 0:
            distributions.append({
                'address': wallet['address'],
                'amount': amount,
                'rank': rank,
                'tier': next(i for i, t in enumerate(tiers) if rank in t['ranks'])
            })
            total_allocated += amount
    
    # Scale if over budget
    if total_allocated > total_emission:
        scale_factor = total_emission / total_allocated
        for dist in distributions:
            dist['amount'] *= scale_factor
    
    return distributions


def execute_distributions(distributions, slot):
    """
    Send reward tokens to wallets.
    """
    # Batch distributions into transactions
    batch_size = 10  # Max recipients per transaction
    batches = [
        distributions[i:i + batch_size]
        for i in range(0, len(distributions), batch_size)
    ]
    
    results = []
    
    for batch in batches:
        # Build transaction
        transaction = Transaction()
        
        for dist in batch:
            # Add transfer instruction
            instruction = create_transfer_instruction(
                source=TREASURY_ACCOUNT,
                destination=dist['address'],
                amount=dist['amount'],
                token_mint=REWARD_TOKEN_MINT
            )
            transaction.add(instruction)
        
        # Sign and send
        try:
            signature = send_transaction(transaction)
            results.append({
                'batch': batch,
                'signature': signature,
                'status': 'success',
                'slot': slot
            })
        except Exception as e:
            results.append({
                'batch': batch,
                'error': str(e),
                'status': 'failed',
                'slot': slot
            })
    
    return results
```

---

## Configuration

All system parameters are externalized in a configuration file. This allows tuning without code changes.

### Example Configuration

```yaml
# Holder Index Configuration

# Indexer Settings
indexer:
  rpc_endpoints:
    - https://api.mainnet-beta.solana.com
    - https://solana-api.projectserum.com
  websocket_endpoint: wss://api.mainnet-beta.solana.com
  commitment_level: confirmed
  initial_sync_slot: 150000000
  target_token_mints:
    - EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v  # USDC
    - Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB  # USDT
  minimum_balance_threshold: 100000000  # 0.1 SOL in lamports
  max_wallets_tracked: 1000000

# Scoring Settings
scoring:
  cycle_length_slots: 100
  weight_balance: 0.5
  weight_time: 0.3
  weight_activity: 0.2
  time_decay_lambda: 0.01
  activity_log_base: 10
  minimum_holding_duration_slots: 1000
  token_weights:
    EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v: 0.1  # USDC
    Es9vMFrzaCERmJfrF4H2FYD4KCoNkY11McCe8BenwNYB: 0.1  # USDT

# Leaderboard Settings
leaderboard:
  update_frequency_slots: 10
  top_n_tracked: 10000
  enable_historical_tracking: true
  cache_ttl_seconds: 5

# Distribution Settings
distribution:
  model: proportional  # Options: proportional, tiered, hybrid
  emission_per_slot: 0.193
  distribution_frequency_slots: 100
  top_n_eligible: 100
  max_payout_per_wallet_per_cycle: 1000
  minimum_payout_amount: 0.001
  treasury_account: "9xQeWvG816bUx9EPjHmaT23yvVM2ZWbrrpZb9PusVFin"
  reward_token_mint: "EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v"

# Anti-Manipulation Settings
anti_manipulation:
  enable_sybil_detection: true
  enable_wash_trading_detection: true
  enable_clustering: true
  cooldown_duration_slots: 1000
  significant_balance_change_threshold: 0.25
  sybil_penalty_multiplier: 0.3
  cluster_penalty_multiplier: 0.6
  max_score_increase_per_cycle: 1000

# API Settings
api:
  port: 8080
  enable_cors: true
  rate_limit_per_minute: 100
  max_response_size_mb: 10
  enable_websocket_streaming: true
```

### Configuration Validation

```python
def validate_config(config):
    """
    Validate configuration file for correctness.
    """
    errors = []
    
    # Check weights sum to 1.0
    weight_sum = (
        config['scoring']['weight_balance'] +
        config['scoring']['weight_time'] +
        config['scoring']['weight_activity']
    )
    if abs(weight_sum - 1.0) > 0.001:
        errors.append(f"Scoring weights must sum to 1.0, got {weight_sum}")
    
    # Check emission rate is positive
    if config['distribution']['emission_per_slot'] <= 0:
        errors.append("Emission rate must be positive")
    
    # Check cycle lengths are reasonable
    if config['scoring']['cycle_length_slots'] < 10:
        errors.append("Scoring cycle too short (min 10 slots)")
    
    if config['distribution']['distribution_frequency_slots'] < 100:
        errors.append("Distribution frequency too high (min 100 slots)")
    
    # Check RPC endpoints are valid URLs
    for endpoint in config['indexer']['rpc_endpoints']:
        if not endpoint.startswith('http'):
            errors.append(f"Invalid RPC endpoint: {endpoint}")
    
    if errors:
        raise ValueError(f"Configuration validation failed:\n" + "\n".join(errors))
    
    return True
```

---

## Frontend Integration

The frontend consumes leaderboard data via REST API and WebSocket streams.

### REST API Endpoints

**GET /leaderboard**

Returns current leaderboard.

Query parameters:
- `limit` (default: 100): Number of results
- `offset` (default: 0): Pagination offset

Response:
```json
{
  "slot": 150123456,
  "timestamp": 1704412345,
  "total_wallets": 123456,
  "leaderboard": [
    {
      "rank": 1,
      "address": "7xKXtg2CW87d97TXJSDpbD5jBkheTqA83TZRuJosgAsU",
      "score": 1247.32,
      "previous_rank": 2,
      

Future Direction

   Holder Index is intentionally designed as a modular primitive rather than a single-purpose application. The current implementation focuses on SOL wallet ranking and continuous reward distribution, but the architecture is extensible by design.

Planned and potential extensions include:

Multiple parallel indices with different scoring models running simultaneously

Token-specific leaderboards scoped to individual SPL mints

Permissionless scoring plugins allowing external teams to define custom metrics

Program-level integrations where other protocols route incentives based on wallet rank

Hybrid on-chain verification using Merkle roots for trust-minimized validation

Cross-index composability where rank in one index influences eligibility in another

The long-term goal is not to optimize a single leaderboard, but to explore ranking itself as an on-chain coordination primitive.

Limitations and Tradeoffs

   Holder Index makes explicit tradeoffs in favor of scalability and flexibility.

The core indexing, scoring, and ranking logic is off-chain

Trust assumptions exist around the operator running the indexer

Full on-chain leaderboard computation is intentionally avoided due to cost and compute constraints

Anti-manipulation heuristics are probabilistic, not absolute guarantees

Wallet clustering and sybil detection rely on behavioral inference, not identity

These tradeoffs are deliberate. The system prioritizes experimentation velocity, real-time responsiveness, and economic viability on Solana.

Experimental Status

Holder Index is an experimental system.

All parameters, scoring functions, reward models, and eligibility rules are subject to change. No guarantees are made regarding fairness, continuity, or long-term operation. Participation implies acceptance of potential changes, interruptions, or termination of the system.

This repository represents an ongoing exploration of continuous ranking mechanisms on Solana.

Disclaimer

This project is provided as-is.

Nothing in this repository constitutes financial advice, investment solicitation, or a promise of rewards. Participation is voluntary and experimental. Use at your own risk.

End of document.