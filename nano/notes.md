# Nano (previously RaiBlocks) Whitepaper Notes

## TL;DR
* Mostly proof of Stake (some PoW for rate limiting)
* Zero fees
* DAG, or "block-lattice"
* Lightweight (can run on low power hardware, no power competition)
* Each account is fully asynchonous

## Performance
* 4.2 million transaction as of paper publication ~ 1.7GB block-lattice
* Average transaction measured in seconds (~0.16s good GPU, ~3s i7, *~10s wasm*)
* SSDs can handle ~10k transactions/second incoming

## Motivation

Bitcoin downsides:
* Scalability, esp block size & block cost
* Long transaction latency
* Consumes more power than many countries
  * https://www.weforum.org/agenda/2017/12/bitcoin-consume-more-power-than-world-2020/
  * We're all pro-environment, right? RIGHT?

## Mechanics
* One block per transation
* Each account maintains its own blockchain (account-chain)
  * Only updatable by its owner
* Genesis Account
  * Fixed starting funds
  * Cannot change this amount in the system
  * Can only shuffle existing Nano around
* Transactions are all
  * Async
  * Small enough to fit in a single UDP packet
  * Are either a `send` or a `receive` (since each user controls their own account)
    * ie: a transfer requires both
    * `receive`s are associative (because addition)
* Ledger pruning :ok_hand:
* Other nodes may keep a full transaction history of peers, or only the balances; it's up to them

* Agreements/transactions are processes in the range of seconds-or-below
* Two transaction types in network: settled and unsettled (ie: waiting for validation)

## Lifecycle

`SEND --broadcast--> PENDING --receive--> AWAITING CONFIRMATION --verify--> COMPLETE`

### `open`
* Every account starts with an `open` transaction from an existing account
  * Contains/broadcasts the public key/address

### `send`

```
send {
  type: send,
  previous: BLOCK_HASH_THIS_ORIGINATES_FROM,
  destination: RECEIVER_ADDRESS,
  balance: HOW_MUCH_BEING_SENT,
  work: ANTI_SPAM_NONCE,
  signature: TRANSACTION_SIGNATURE
}
```

* Immutable once confirmed (obvs)
* Pending funds (not yet received) are already considered spent

### `receive`

```
receive {
  type: receive,
  previous: PREVIOUS_BLOCK_HASH_FOR_RECEIVING_ACCOUNT,
  source: SEND_TRASNACTION_HASH,
  work: ANTI_SPAM_NONCE,
  signature: TRANSACTION_SIGNATURE
}
```

### Representatives
The PoS vote for an account may be delegated to a representative so their node doesn't
have to be online 24/7. This may be specified as part of an `open`,
but may be reassigned at any time.

### Forks
Occurs when two chains claim the same predecessor block

#### `change`

```
change {
  type: change,
  previous: PARENT_BLOCK,
  representative: REPRESENTATIVE_ACCOUNT,
  work: ANTI_SPAM_NONCE,
  signature: TRANSACTION_SIGNATURE
}
```

When a confict is found:
1. It's broadcast to the network
2. Network votes on which history is more accepted
  * Votes are weighted by how many accounts an account is using delegated votes from
3. 4 vpting rounds up to 1 minute
4. Winning block is confirmed
5. If someone tried to add to the loosing block, we have another vote

### Why the `work` field?
* Only an anti-spam tool
* PoW problem is fast (seconds) and low power
* Next block's PoW can start as soon as previous one has been broadcast
* Uses the `Blake2b` hash

## Valid Transactions
1. No duplicates
2. Signed by owner
3. Parent block is the account's head (otherwise this is a fork, see voting scenario)
4. The account must have an open block
5. Passes the PoW check
6. Resulting balance is >= 0

Similar for `receive` and `change` transactions.

## Attack Vectors

### Block Gap Synchonization
* Looks like a incorrectly broadcast block (parent is possibly a fork)
* Either ignore, or resync with other node(s) via TCP
* Causes extra network load, DDoS, &c
* To avoid DDoS, wait for `change` transactions
* Ignore if you don't get enough after X time.

### Transaction Flooding
* Nano has no transaction fees
* Malicious user creates X > 1 accounts, and floods the network with traffic
* PoW limits rate of such attacks
* Non-full nodes can prune their local copies, most users okay
* Full nodes _do_ need to record all transactions, though

### Sybil Attack
* Extra nodes don't give you any advanatge
* Votes are weighted by account balance
* No advanatge to malicious user

### Penny Spend
* Many tiny transactions to many users
* Cheap, but lots of network traffic
* PoW rate limits each account
* Partial nodes don't have to keep all history
* Space-efficient (1GB ~ 8 million penny-spend transactions)

> If nodes wanted to prune more aggressively, they can
> calculate a distribution based on access frequency and delegate
> infrequently used accounts to slower storage

### Precomputed PoW Attack
* Basically Transaction Flooding
* You can start on a transaction as soon as one PoW problem is complete
* Create a bunch of valid penny-spends, but don't broadcast... then broadcast all at once
* They're looking into it
* Attack would have to be relatively short lived (can only precompute so much)

### >50% Attack
* Attacker has >50% of all Nano voting weight
* Usual PoS incentives: lowering value of Nano, cost is porportional to market cap
* Can ignore known bad nodes
* Block cementing: make certain blocks that have high consensus unforkable (ie: always in history)
* Require a certain number of voting users to be online (no huge concentration of power via representatives)

```
+---------|----------|----------|--------|-------+
| Offline | Unsynced | Attacker | Active | Stake |
+---------|----------|----------|--------|-------+
               ^
               |
     Being attacked, messed up history
```

### Bootstrap Poisoning
* Then a new user bootstrapping from an old copy is suseptible to fake history
* Only new nodes suseptible
