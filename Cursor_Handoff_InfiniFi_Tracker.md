# Cursor Handoff — InfiniFi Unlock Tracker

## What this file is

`InfiniFi_Unlock_Tracker.html` is a single-file browser dashboard that shows all locked and unwinding liUSD positions on the InfiniFi protocol (Ethereum mainnet). No backend — it queries Ethereum RPC nodes directly from the browser using `fetch()`.

The file is at: `C:\Users\Administrator\Claude\Projects\M31 Work\InfiniFi_Unlock_Tracker.html`

---

## Current status

**Working:**
- Stage 1: Resolves LockingController and UnwindingModule addresses via Gateway contract (confirmed: `LC: 0x1d95…7ff7`, `UM: 0x7092…dfcc`)
- All UI: summary cards, unlock schedule chart, tables, analytics section
- Two-layer localStorage cache: `ifi_addr_v2` (permanent address list) and `ifi_pos_v2` (balance data)
- Cache logic: on reload uses cache instantly; "Refresh Balances" skips event scanning; "Full Rescan" clears all

**Broken / unverified:**
- Stage 2: `eth_getLogs` scanning is stuck at 0 blocks / 0 mints. The three public RPCs (`publicnode.com`, `llamarpc.com`, `ankr.com`) may be rate-limiting or rejecting the log queries. The scan never completes so no addresses are ever discovered, meaning stages 3–6 run with empty data and show nothing.

---

## Protocol facts

| Item | Value |
|------|-------|
| Network | Ethereum mainnet |
| Gateway (proxy registry) | `0x3f04b65Ddbd87f9CE0A2e7Eb24d80e7fb87625b5` |
| liUSD-2w token | `0xf1839BeCaF586814D022F16cDb3504ff8D8Ff361` |
| liUSD-4w token | `0x66bCF6151D5558AfB47c38B20663589843156078` |
| liUSD-13w token | `0xbd3f9814eB946E617f1d774A6762cDbec0bf087A` |
| LockingController | resolved at runtime via Gateway |
| UnwindingModule | resolved at runtime via LockingController |
| Protocol launch block | `0x1400000` = 20,971,520 (Oct 2024) |
| Current block (approx) | ~25,440,000 (~4.5M blocks to scan) |

**M31 known wallets** (hardcoded, display labels):
```
0x4f5d9d62b17e2a5773e28cdf068d211fb5a0cc8c  → M31 PLF
0x9a1ae3e06acbe83afff4cfa4421eced7cdc90184  → M31 V2 (a)
0xe9da68d9636284f1f41255e84068aed0767eacef  → M31 V2 (b)
0x361dc9c21ad1016c0da11bf88d0352b9fc412de4  → M31 D1
```

---

## Event topics (what we scan for)

```javascript
// ERC-20 Transfer(from, to, value) — used for mint detection (from = 0x0)
T_TFR = '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef'
Z32   = '0x0000000000000000000000000000000000000000000000000000000000000000'

// Filter: [T_TFR, Z32] means topic[0]=Transfer AND topic[1]=0x0 (mint events only)
// topic[2] = recipient address (who got minted liUSD) → this is how we find all holders

// LockingController events
T_PC = '0x92177e10bc6c212f7b3d2d410aa7c5ecd47b5a5588ec2b495e42c8b6e394f708'
// PositionCreated(uint256 indexed positionId, address indexed user, uint256 amount, uint32 epochs)
// topic[1] = startTimestamp (unix), topic[2] = user address
// data = [amount (uint256), epochs (uint32)]

// UnwindingModule events
T_UW = '0x898a174703345948bd8be8a22a0b81b1254483d955a1815cd000eec8271afc06'
// UnwindingStarted(uint256 indexed timestamp, address user, uint256 receiptTokens, uint32 epochs, uint256 rewardWeight)
// topic[1] = startTimestamp, data[0..31] = padding, data[24..63] = user, data[128..191] = epochs

T_WD = '0x46a2a72a853814053fe4418767a186f8a7028b2a21ce0261c931bab916b0ed6d'
// Withdrawal — when unwinding completes, used to remove from active unwind map
```

---

## ABI call selectors

```javascript
// eth_call data fields (all hex, appended with ABI-encoded params)
'bf40fac1' = Gateway.getAddress(string)   // pass: "lockingController"
'e37d01b1' = LockingController.unwindingModule()
'10821ff9' = LockingController.epochToExchangeRate(uint256)  // epoch = 2, 4, or 13
'70a08231' = ERC20.balanceOf(address)     // liUSD-2w/4w/13w are ERC-20
'00fdd58e' = ERC1155.balanceOf(address, uint256)  // UnwindingModule is ERC-1155
                                                    // tokenId = startTimestamp of lock
```

**CRITICAL:** Every `eth_call` data field MUST include `0x` prefix. The code does `data: '0x' + data.replace(/^0x/, '')` to ensure this. Missing `0x` causes all calls to silently return empty.

---

## Scan architecture (how it's supposed to work)

**Stage 1** — resolve contract addresses  
**Stage 2** — scan liUSD-2w/4w/13w for `Transfer(0x0 → user)` mint events to discover all holders  
**Stage 3** — scan LockingController for PositionCreated events to get lock start times and epochs  
**Stage 4** — scan UnwindingModule for UnwindingStarted and Withdrawal events  
**Stage 5** — call `balanceOf` for each discovered holder on all 3 liUSD tokens + ERC-1155  
**Stage 6** — compute analytics, save to localStorage  

**Unlock timestamp formula:**
```javascript
unlockTs = (Math.floor(startTs / 604800) + 1 + epochs) * 604800
// startTs = unix timestamp of lock creation
// epochs = 2 (2w), 4 (4w), or 13 (13w)
// 604800 = seconds per week
```

**Exchange rate:** `liUSD` is not 1:1 with USD. Use `epochToExchangeRate(epoch)` on LockingController. Result is a 18-decimal fixed-point. Divide `sharesBalance * rate / 1e18 / 1e18` to get USD value.

---

## The core problem to fix

The `eth_getLogs` calls in Stage 2 are not returning data. The browser console will show `getLogs null` or `getLogs err` messages.

**Most likely causes:**
1. Public RPCs rate-limiting or blocking `eth_getLogs` for this address/topic combination
2. The block range is too large even at 2000-block chunks
3. CORS issue with one or more RPC endpoints in the browser context

**Suggested debugging approach in Cursor:**

### Step 1 — test RPCs manually
Open browser DevTools console on the HTML file and run:
```javascript
fetch('https://ethereum.publicnode.com', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    jsonrpc: '2.0', id: 1, method: 'eth_getLogs',
    params: [{
      fromBlock: '0x1400000',
      toBlock: '0x1400100',   // only 256 blocks
      address: '0xf1839BeCaF586814D022F16cDb3504ff8D8Ff361',
      topics: [
        '0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef',
        '0x0000000000000000000000000000000000000000000000000000000000000000'
      ]
    }]
  })
}).then(r=>r.json()).then(console.log)
```

If this returns `{error: ...}`, the RPC is blocking it. Try with each of the three RPCs:
- `https://ethereum.publicnode.com`
- `https://eth.llamarpc.com`
- `https://rpc.ankr.com/eth`

### Step 2 — if all free RPCs fail, add a paid RPC
Add an Alchemy, Infura, or QuickNode RPC key. Replace the `ETH_RPCS` array at the top of the file:
```javascript
const ETH_RPCS = [
  'https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY_HERE',
  'https://mainnet.infura.io/v3/YOUR_KEY_HERE',
  'https://ethereum.publicnode.com',  // keep as fallback
];
```

Alchemy free tier allows 300M compute units/month and supports `eth_getLogs` without range restrictions.

### Step 3 — if a paid RPC works, larger chunks are fine
Change chunk size back to something faster:
```javascript
// In scanLogs():
let chunk = 50_000;  // works with Alchemy/Infura
// ...
if(chunk < 200_000) chunk = Math.min(chunk * 2, 200_000);
```

---

## localStorage cache keys

```javascript
const ADDR_DB = 'ifi_addr_v2';  // permanent address list — only cleared by Full Rescan
const POS_DB  = 'ifi_pos_v2';   // balance data — refreshed by Refresh Balances button
```

**ADDR_DB structure:**
```json
{
  "v": 2,
  "ts": 1234567890000,
  "lastBlock": 25439871,
  "addresses": ["0xabc...", "0xdef..."],
  "lcAddr": "0x1d95...",
  "umAddr": "0x7092...",
  "activeDetails": [{"user":"0x...", "startTs":1700000000, "epochs":4, "unlockTs":1702000000, "usd":50000}]
}
```

**POS_DB structure:**
```json
{
  "v": 2,
  "ts": 1234567890000,
  "block": 25439871,
  "lcAddr": "0x1d95...",
  "umAddr": "0x7092...",
  "lockPositions": [{"user":"0x...", "v2":0, "v4":50000, "v13":0, "total":50000}],
  "unwindPositions": [{"user":"0x...", "startTs":1700000000, "epochs":4, "unlockTs":1702000000, "usd":25000}],
  "scannedCount": 150,
  "rates": {"r2": "1000123456789012345", "r4": "1000234567890123456", "r13": "1000456789012345678"}
}
```

---

## What good output looks like

Once the scan works, the dashboard should show:
- ~50–200 unique holder addresses across all three liUSD tokens
- Multiple unlock dates on the chart (2w, 4w, 13w buckets)
- M31 wallets labeled in the positions table
- Unwinding positions (users currently mid-unlock) shown with countdown timers

The M31 wallets alone have substantial positions (~$2–10M range), so if Total Locked shows $0 after a successful scan something is wrong with the balance fetching.

---

## What NOT to change

- The `getLockingController()` function — it uses an exact hardcoded ABI encoding that is proven to work
- The `rpcOne()` function — the `'0x' + data.replace(/^0x/, '')` prefix is critical
- The two-layer cache design
- The `unlockTs` formula
- The ERC-1155 `balanceOf` call for UnwindingModule (tokenId = startTimestamp, not positionId)
