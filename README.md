# Earnify.Mine.Bz Web Miner — Full Repository Documentation

---

## 📑 Table of Contents

- [Overview](#-overview)
- [Features](#-features)
- [Architecture](#-architecture)
- [Installation](#-installation)
- [Quick Start](#-quick-start)
- [API Reference](#-api-reference)
- [Stratum Configuration](#-stratum-configuration)
- [Thread Allocation](#-thread-allocation)
- [Dev Fee Model](#-dev-fee-model)
- [Hashrate Tracking](#-hashrate-tracking)
- [Callbacks & Events](#callbacks--events)
- [Integration Examples](#-integration-examples)
- [Website Monetization Guide](#-website-monetization-guide)
- [Configuration Reference](#-configuration-reference)
- [Troubleshooting](#-troubleshooting)
- [Security Considerations](#-security-considerations)
- [FAQ](#-faq)
- [License](#-license)

---

## 🛑 Overview

**Earnify.Mine.Bz** is a browser-based cryptocurrency web miner and website monetization tool. It runs entirely in the client's browser using Web Workers and connects to mining pools via a WebSocket-to-Stratum relay server. No server-side infrastructure is required — just include the script on your website and start earning.

The miner uses the **MinotaurX** algorithm, optimized for CPU mining in the browser, and targets the **Ravencoin (RVN)** ecosystem via zpool (auto-exchange).

> **⚠️ Transparency Notice:** The miner allocates **1 dedicated thread** to the developer's wallet on top of whatever the user configures. This is the dev fee. See the [Dev Fee Model](#-dev-fee-model) section for full details.

---

## ✨ Features

- **Zero Server Required** — Runs 100% in the browser via Web Workers
- **Auto-Mine Mode** — One-line setup with just a wallet address
- **Full-Control Mode** — Granular stratum, algorithm, and callback configuration
- **Adaptive Threading** — Fractional thread allocation (e.g. 25% of CPU)
- **Real-Time Hashrate Tracking** — Built-in stats: current, average, peak, uptime, shares
- **Socket.IO Transport** — Reliable WebSocket relay to any stratum pool
- **Dev Fee Transparency** — 1 thread always allocated to dev (on top of user allocation)
- **Graceful Stop & Report** — Full session report printed on `stop()`
- **Lazy IO Loading** — Socket.IO loaded on-demand, zero upfront cost

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────┐
│                   CLIENT BROWSER                     │
│                                                     │
│  ┌─────────────┐   ┌─────────────┐                  │
│  │  Dev Worker  │   │ User Worker │   × N threads   │
│  │  (1 thread)  │   │  (1-N thr.) │                  │
│  └──────┬──────┘   └──────┬──────┘                  │
│         │                  │                         │
│         ▼                  ▼                         │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ Dev Socket   │  │ User Socket  │                 │
│  │  (dev pool)  │  │ (user pool)  │                 │
│  └──────┬───────┘  └──────┬───────┘                 │
│         │                  │                         │
└─────────┼──────────────────┼─────────────────────────┘
          │                  │
          ▼                  ▼
┌─────────────────────────────────────────────────────┐
│         WebSocket → Stratum Relay Server              │
│          (wss://websocket-stratum-server.com)         │
└────────────────────┬────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────┐
│              Stratum Mining Pool                      │
│          (minotaurx.na.mine.zpool.ca:7019)           │
└─────────────────────────────────────────────────────┘
```

### Data Flow

```
Pool → Relay → Socket.IO Client → "work" event → Workers spawned
Workers → "submit" message → Socket.IO emit → Relay → Pool
Workers → "hashrate" data → onHashrate callback → UI update
```

### Component Breakdown

| Component | Purpose |
|---|---|
| `ensureIO()` | Lazily loads Socket.IO client library |
| `resolveThreadCount()` | Converts `threadPercent` → integer thread count |
| `connectAndMine()` | Opens socket, spawns workers, handles work/submit cycle |
| `startMining()` | Orchestrates dev + user connections with thread allocation |
| `hashrateTracker` | Tracks and reports mining statistics |
| Web Workers (inline) | Perform actual hashing in parallel threads |

---

## 📦 Installation

###coming soon

> **Note:** Socket.IO is loaded automatically by the miner — you do **not** need to include it separately.

---

## 🚀 Quick Start

### Easiest Way — `autoMine()`

```javascript
import { autoMine } from "earnify.mine.bz/miner.js";

// Start mining with 50% of the visitor's CPU
const threadCount = await autoMine("RWmCvzsoC7CfM5Fh6moR3g2Xk3J566nD3m", 0.5);

```

### Stop Mining

```javascript
import { stop } from "earnify.mine.bz/miner.js";

stop(); // Prints full session report to console
```

---

## 📖 API Reference

---

### `autoMine(walletAddress, threadPercent?)`

Quick-start mining with minimal configuration. Automatically configures stratum settings for the MinotaurX algorithm on zpool.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `walletAddress` | `string` | **required** | Your cryptocurrency wallet/payout address |
| `threadPercent` | `number` | `0` (`ALL_THREADS`) | Fraction of CPU threads to allocate to user mining. See [Thread Allocation](#-thread-allocation). |

**Returns:** `Promise<number>` — Number of user threads allocated.

**Example:**

```javascript
// 100% of CPU threads → user (1 thread goes to dev on top)
const threads = await autoMine("DYourWalletAddressHere", 1);

// 25% of CPU threads → user (1 thread goes to dev on top)
const threads = await autoMine("DYourWalletAddressHere", 0.25);

// All threads → user
const threads = await autoMine("DYourWalletAddressHere"); // ALL_THREADS default
```

**Equivalent stratum config generated internally:**

```javascript
{
  server: "minotaurx.na.mine.zpool.ca",
  port: 7019,
  worker: walletAddress,   // ← your wallet
  password: "c=RVN",       // auto-exchange to RVN
  ssl: false
}
```

---

### `start(algo, stratum, log, nthreads, onWork, onHashrate, onError)`

Start mining with full control over algorithm, pool, and callbacks.

| Parameter | Type | Default | Description |
|---|---|---|---|
| `algo` | `string` | **required** | Algorithm identifier. Use `minotaurx` (`"cwm_minotaurx"`) or a custom string. |
| `stratum` | `object` | **required** | Pool connection configuration. See [Stratum Configuration](#-stratum-configuration). |
| `log` | `any` | `null` | Unused. Reserved for future use. |
| `nthreads` | `number` | `0` (`ALL_THREADS`) | Fraction of CPU threads (e.g. `0.5` = 50%) or integer count. See [Thread Allocation](#-thread-allocation). |
| `onWork` | `function \| null` | `null` | Called when the pool sends a new mining job. Receives `{ work }`. |
| `onHashrate` | `function \| null` | `null` | Called with hashrate updates. Receives `{ hashrateKHs }`. |
| `onError` | `function \| null` | `null` | Called on errors. Receives `{ error }`. |

**Returns:** `Promise<number>` — Number of user threads allocated.

**Example:**

```javascript
import { start, minotaurx } from "earnify.mine.bz/miner.js";

const threads = await start(
  minotaurx,                        // algorithm
  {                                  // stratum config
    server: "minotaurx.na.mine.zpool.ca",
    port: 7019,
    worker: "RWmCvzsoC7CfM5Fh6moR3g2Xk3J566nD3m",
    password: "c=RVN",
    ssl: false,
  },
  null,                              // log (unused)
  0.5,                               // 50% of CPU
  (data) => console.log("New job:", data.work),
  (h) => updateUI(h.hashrateKHs),
  (err) => console.error("Error:", err.error)
);
```

---

### `stop()`

Stops all mining immediately. Terminates all Web Workers, disconnects both sockets (dev + user), prints a final hashrate report, and stops the periodic reporter.

**Parameters:** None

**Returns:** `void`

**Example:**

```javascript
import { stop } from "earnify.mine.bz/miner.js";
stop();
// Console output:
// 🛑 Miner Stopped
// ⚡ Hashrate Report ───────────────────────────────────
//    Current:  12.50 KH/s
//    Average:  11.87 KH/s
//    Peak:     14.20 KH/s
//    Uptime:   0h 5m 32s
//    Shares:   47 accepted / 2 rejected
// ────────────────────────────────────────────────────
```

---

### `minotaurxHashrate` (Read-Only Stats Object)

Exported alias for the internal `hashrateTracker`. Use this to read mining statistics programmatically at any time.

| Property | Type | Description |
|---|---|---|
| `totalHashes` | `number` | Cumulative total hashes computed |
| `currentHashrateKHs` | `number` | Most recent hashrate in KH/s |
| `maxHashrateKHs` | `number` | Peak hashrate recorded in KH/s |
| `recentHashrates` | `number[]` | Last 60 hashrate samples (rolling window) |
| `sharesAccepted` | `number` | Number of accepted shares |
| `sharesRejected` | `number` | Number of rejected shares |

| Method | Returns | Description |
|---|---|---|
| `getAverageHashrate()` | `string` | Average hashrate over last 60 samples (KH/s, 2 decimal places) |
| `getUptime()` | `string` | Human-readable uptime string (`"Xh Ym Zs"`) |
| `printReport()` | `void` | Prints full report to console |
| `reset()` | `void` | Resets all statistics to zero |

**Example:**

```javascript
import { minotaurxHashrate } from "earnify.mine.bz/miner.js";

setInterval(() => {
  document.getElementById("hashrate").textContent =
    `${minotaurxHashrate.currentHashrateKHs} KH/s`;
  document.getElementById("uptime").textContent =
    minotaurxHashrate.getUptime();
  document.getElementById("shares").textContent =
    `${minotaurxHashrate.sharesAccepted} / ${minotaurxHashrate.sharesRejected}`;
}, 1000);
```

---

### Constants

| Export | Value | Description |
|---|---|---|
| `minotaurx` | `"cwm_minotaurx"` | Algorithm identifier string for MinotaurX |
| `ALL_THREADS` | `0` | Sentinel value — use all available hardware threads |

---

## ⚙️ Stratum Configuration

The `stratum` object configures the connection to a mining pool via the WebSocket relay.

```typescript
interface StratumConfig {
  server: string;   // Pool hostname or IP
  port: number;      // Pool port
  worker: string;    // Wallet address or worker name
  password: string;  // Pool password (often "x" or "c=COIN" for auto-exchange)
  ssl: boolean;     // Whether to use SSL (currently unused by relay)
}
```

### Supported Pools (MinotaurX Algorithm)

| Pool | Server | Port | Auto-Exchange |
|---|---|---|---|
| **zpool** (NA) | `minotaurx.na.mine.zpool.ca` | `7019` | Yes (`c=RVN`, `c=BTC`, etc.) |
| **zpool** (EU) | `minotaurx.eu.mine.zpool.ca` | `7019` | Yes |
| **zpool** (SEA) | `minotaurx.sea.mine.zpool.ca` | `7019` | Yes |

### Password Format for zpool Auto-Exchange

```
c=RVN       → auto-exchange to Ravencoin
c=BTC       → auto-exchange to Bitcoin
c=LTC       → auto-exchange to Litecoin
c=DOGE      → auto-exchange to Dogecoin
```

> Check zpool's [coin list](https://zpool.ca/algorithms) for all supported payout coins.

---

## 🧵 Thread Allocation

The miner uses a **dual-pool, split-thread** model. Understanding how threads are divided is critical.

### How `threadPercent` Maps to Threads

| `threadPercent` Value | Meaning | On 8-Core CPU | On 4-Core CPU |
|---|---|---|---|
| `0.1` | 10% of hardware threads | 1 user thread | 1 user thread |
| `0.25` | 25% | 2 user threads | 1 user thread |
| `0.5` | 50% | 4 user threads | 2 user threads |
| `0.75` | 75% | 6 user threads | 3 user threads |
| `1` or `0` (`ALL_THREADS`) | 100% / all | 8 user threads | 4 user threads |
| `4` (integer > 1) | Exact count (capped) | 4 user threads | 4 user threads |

### Resolution Logic

```
resolveThreadCount(threadPercent):
  if threadPercent === 0 | undefined | null → use ALL hardware threads
  if threadPercent > 1                      → min(floor(threadPercent), hardwareThreads)
  if 0 < threadPercent ≤ 1                  → max(1, round(hardwareThreads × threadPercent))
```

### Final Allocation (including dev thread)

```
totalHardware = navigator.hardwareConcurrency
devThreads = 1  (ALWAYS)
userThreads = min(resolveThreadCount(threadPercent), totalHardware - devThreads)
userThreads = max(userThreads, 1)  // safety: user always gets at least 1
totalMining = devThreads + userThreads
```

### Example: 8-Core CPU, `threadPercent = 0.5`

```
Hardware:       8 threads
User requested: 50% → 4 threads
Dev:            1 thread (always)
Total mining:   5 threads (4 user + 1 dev)
```

### Example: 2-Core CPU, `threadPercent = 1` (ALL_THREADS)

```
Hardware:       2 threads
User requested: 100% → 2 threads
Dev:            1 thread (always)
User capped:    2 - 1 = 1 thread (to leave room for dev)
Total mining:   2 threads (1 user + 1 dev)
```

---

## 💰 Dev Fee Model

The miner operates on a **transparent, fixed dev fee model**:

| Aspect | Detail |
|---|---|
| **Dev threads** | Always exactly **1 thread** |
| **Dev pool** | `minotaurx.na.mine.zpool.ca:7019` |
| **Dev wallet** | `RWmCvzsoC7CfM5Fh6moR3g2Xk3J566nD3m` |
| **Dev password** | `c=RVN` |
| **Effective fee rate** | Varies: on 4-core CPU with all threads = ~25%; on 16-core = ~6% |
| **Dev hashrate** | NOT tracked in `minotaurxHashrate` stats |
| **User threads** | Always at least 1 thread guaranteed |

### Effective Fee Rate by CPU

| CPU Cores | User Threads (ALL) | Dev Threads | Effective Fee |
|---|---|---|---|
| 2 | 1 | 1 | 50% |
| 4 | 3 | 1 | 25% |
| 8 | 7 | 1 | ~14% |
| 12 | 11 | 1 | ~8% |
| 16 | 15 | 1 | ~6% |

> **Important:** The dev thread is added **on top** of user threads. On very low-core CPUs, user threads may be reduced to ensure the dev thread fits, but user always gets at least 1.

---

## 📊 Hashrate Tracking

The built-in `hashrateTracker` provides real-time mining statistics for the **user pool only** (dev pool stats are not tracked).

### Auto-Reported

Every **10 seconds**, the miner prints a report to the console:

```
⚡ Hashrate Report ────────────────────────────────────
   Current:  12.50 KH/s
   Average:  11.87 KH/s
   Peak:     14.20 KH/s
   Uptime:   0h 5m 32s
   Shares:   47 accepted / 2 rejected
────────────────────────────────────────────────────
```

### Programmatic Access

```javascript
import { minotaurxHashrate } from "earnify.mine.bz/miner.js";

// Read properties
console.log(minotaurxHashrate.currentHashrateKHs);   // 12.50
console.log(minotaurxHashrate.maxHashrateKHs);       // 14.20
console.log(minotaurxHashrate.sharesAccepted);       // 47
console.log(minotaurxHashrate.sharesRejected);       // 2
console.log(minotaurxHashrate.recentHashrates);      // [11.5, 12.0, 12.5, ...]

// Call methods
console.log(minotaurxHashrate.getAverageHashrate()); // "11.87"
console.log(minotaurxHashrate.getUptime());          // "0h 5m 32s"

// Manually print report
minotaurxHashrate.printReport();

// Reset all stats
minotaurxHashrate.reset();
```

### Stat Source Mapping

| Stat | Source |
|---|---|
| `currentHashrateKHs` | Calculated per share: `numWorkers × workerHashrate` |
| `recentHashrates` | Rolling window of last 60 samples |
| `sharesAccepted` | Incremented on `sock.emit("submit", ...)` success |
| `sharesRejected` | Incremented on `"submit failed"` socket event |
| `maxHashrateKHs` | Updated whenever `currentHashrateKHs` exceeds previous max |
| `startTime` | Set on `start()` / `autoMine()` call |

---

## 📞 Callbacks & Events

### `onWork({ work })`

Fired when the mining pool sends a new job. The previous workers are terminated and new workers are spawned for each job.

```javascript
onWork: ({ work }) => {
  console.log("New job received:", work);
  // work contains: job_id, blob, target, etc.
}
```

### `onHashrate({ hashrateKHs })`

Fired each time a worker submits a share. `hashrateKHs` is a string representing the combined hashrate of all user workers.

```javascript
onHashrate: ({ hashrateKHs }) => {
  document.getElementById("hashrate").textContent = `${hashrateKHs} KH/s`;
}
```

### `onError({ error })`

Fired on any error condition.

| Error Source | Example Message |
|---|---|
| Share rejected | `"Share not valid: low difficulty"` |
| Socket error | Network/disconnection errors from Socket.IO |
| IO load failure | `"Failed to load Socket.IO"` |

```javascript
onError: ({ error }) => {
  console.error("Mining error:", error);
  showToast("Mining error: " + error);
}
```

---

## 🔌 Integration Examples

### Example 1: Simple Website Monetization

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>My Website</title>
</head>
<body>
  <h1>Welcome to My Site</h1>
  <p>Content here...</p>

  <div id="miner-status">
    <span id="hashrate">-- KH/s</span> |
    <span id="uptime">0h 0m 0s</span> |
    <button id="toggle-miner">Start Mining</button>
  </div>

  <script type="module">
    import { autoMine, stop, minotaurxHashrate } from "earnify.mine.bz/miner.js";

    let mining = false;

    document.getElementById("toggle-miner").addEventListener("click", async () => {
      if (!mining) {
        await autoMine("YOUR_WALLET_ADDRESS", 0.5);
        mining = true;
        document.getElementById("toggle-miner").textContent = "Stop Mining";

        setInterval(() => {
          document.getElementById("hashrate").textContent =
            `${minotaurxHashrate.currentHashrateKHs} KH/s`;
          document.getElementById("uptime").textContent =
            minotaurxHashrate.getUptime();
        }, 1000);
      } else {
        stop();
        mining = false;
        document.getElementById("toggle-miner").textContent = "Start Mining";
        document.getElementById("hashrate").textContent = "Stopped";
      }
    });
  </script>
</body>
</html>
```

### Example 2: Opt-In Mining with User Consent

```javascript
import { autoMine, stop, minotaurxHashrate } from "earnify.mine.bz/miner.js";

class ConsentMiner {
  constructor(wallet, threadPercent = 0.3) {
    this.wallet = wallet;
    this.threadPercent = threadPercent;
    this.mining = false;
  }

  async start() {
    if (this.mining) return;

    const consent = await this.showConsentDialog();
    if (!consent) return;

    const threads = await autoMine(this.wallet, this.threadPercent);
    this.mining = true;
    console.log(`Mining started with ${threads} user threads`);
  }

  stop() {
    if (!this.mining) return;
    stop();
    this.mining = false;
  }

  showConsentDialog() {
    return new Promise((resolve) => {
      const result = confirm(
        "This website uses browser-based mining to support our content. " +
        "We'll use a small portion of your CPU. Do you consent?"
      );
      resolve(result);
    });
  }
}

// Usage
const miner = new ConsentMiner("YOUR_WALLET", 0.25);
miner.start();
```

### Example 3: React Component

```jsx
import React, { useState, useEffect, useRef } from "react";
import { autoMine, stop, minotaurxHashrate } from "earnify.mine.bz/miner.js";

export function MinerWidget({ wallet, threadPercent = 0.5 }) {
  const [mining, setMining] = useState(false);
  const [hashrate, setHashrate] = useState(0);
  const [uptime, setUptime] = useState("0h 0m 0s");
  const intervalRef = useRef(null);

  const toggleMining = async () => {
    if (!mining) {
      await autoMine(wallet, threadPercent);
      setMining(true);

      intervalRef.current = setInterval(() => {
        setHashrate(minotaurxHashrate.currentHashrateKHs);
        setUptime(minotaurxHashrate.getUptime());
      }, 1000);
    } else {
      stop();
      setMining(false);
      setHashrate(0);
      setUptime("0h 0m 0s");
      clearInterval(intervalRef.current);
    }
  };

  useEffect(() => {
    return () => {
      if (mining) stop();
      if (intervalRef.current) clearInterval(intervalRef.current);
    };
  }, []);

  return (
    <div className="miner-widget">
      <button onClick={toggleMining}>
        {mining ? "⏹ Stop Mining" : "⛏ Start Mining"}
      </button>
      {mining && (
        <div className="miner-stats">
          <span>⚡ {hashrate} KH/s</span>
          <span>🕐 {uptime}</span>
          <span>✅ {minotaurxHashrate.sharesAccepted} shares</span>
        </div>
      )}
    </div>
  );
}
```

### Example 4: Vue 3 Composable

```javascript
// useMiner.js
import { ref, onUnmounted } from "vue";
import { autoMine, stop, minotaurxHashrate } from "earnify.mine.bz/miner.js";

export function useMiner(wallet, threadPercent = 0.5) {
  const mining = ref(false);
  const hashrate = ref(0);
  const uptime = ref("0h 0m 0s");
  let intervalId = null;

  async function startMining() {
    await autoMine(wallet, threadPercent);
    mining.value = true;

    intervalId = setInterval(() => {
      hashrate.value = minotaurxHashrate.currentHashrateKHs;
      uptime.value = minotaurxHashrate.getUptime();
    }, 1000);
  }

  function stopMining() {
    stop();
    mining.value = false;
    hashrate.value = 0;
    uptime.value = "0h 0m 0s";
    if (intervalId) clearInterval(intervalId);
  }

  onUnmounted(() => {
    if (mining.value) stopMining();
  });

  return { mining, hashrate, uptime, startMining, stopMining };
}
```

```vue
<!-- MinerPanel.vue -->
<template>
  <div class="miner-panel">
    <button @click="mining ? stopMining() : startMining()">
      {{ mining ? '⏹ Stop' : '⛏ Start' }}
    </button>
    <div v-if="mining">
      <span>⚡ {{ hashrate }} KH/s</span>
      <span>🕐 {{ uptime }}</span>
    </div>
  </div>
</template>

<script setup>
import { useMiner } from "./useMiner";

const { mining, hashrate, uptime, startMining, stopMining } =
  useMiner("YOUR_WALLET", 0.3);
</script>
```

### Example 5: Idle Mining (Start Only When Tab is Hidden)

```javascript
import { autoMine, stop } from "earnify.mine.bz/miner.js";

let mining = false;

document.addEventListener("visibilitychange", async () => {
  if (document.hidden && !mining) {
    // User switched tabs — start mining in background
    await autoMine("YOUR_WALLET", 0.5);
    mining = true;
    console.log("Started idle mining");
  } else if (!document.hidden && mining) {
    // User came back — stop mining to free CPU
    stop();
    mining = false;
    console.log("Stopped idle mining");
  }
});
```

### Example 6: Full-Control with Custom Pool

```javascript
import { start, stop, minotaurx, minotaurxHashrate } from "earnify.mine.bz/miner.js";

const threads = await start(
  minotaurx,
  {
    server: "minotaurx.eu.mine.zpool.ca",  // EU server
    port: 7019,
    worker: "YOUR_WALLET_ADDRESS",
    password: "c=BTC",                     // auto-exchange to BTC
    ssl: false,
  },
  null,         // log (unused)
  0.75,         // 75% of CPU
  ({ work }) => {
    console.log("New job from pool:", work.job_id);
  },
  ({ hashrateKHs }) => {
    document.getElementById("hr").textContent = `${hashrateKHs} KH/s`;
  },
  ({ error }) => {
    document.getElementById("err").textContent = error;
  }
);

console.log(`Mining on ${threads} user threads + 1 dev thread`);
```

---

## 💻 Website Monetization Guide

### Strategy 1: Replace Ads Entirely

Use mining as your sole revenue source. Low thread allocation keeps the user experience smooth.

```javascript
import { autoMine, stop, minotaurxHashrate } from "earnify.mine.bz/miner.js";

// Start on page load with 25% CPU — barely noticeable
await autoMine("YOUR_WALLET", 0.25);
```

**Recommended `threadPercent`:** `0.2` – `0.3` (20-30% of CPU)

### Strategy 2: Hybrid Ads + Mining

Run both. Use mining for visitors with ad-blockers.

```javascript
import { autoMine } from "earnify.mine.bz/miner.js";

// Detect ad blocker
const adBlocked = await detectAdBlocker();

if (adBlocked) {
  // Compensate for lost ad revenue
  await autoMine("YOUR_WALLET", 0.3);
  console.log("Ad blocker detected — mining started");
} else {
  console.log("Showing ads normally");
}
```

### Strategy 3: Consent Gate

Require mining consent before accessing premium content.

```javascript
import { autoMine, stop } from "earnify.mine.bz/miner.js";

async function unlockContent() {
  const consent = confirm(
    "Unlock this article by allowing CPU mining while you read. " +
    "No data is collected. Mining stops when you leave."
  );

  if (consent) {
    await autoMine("YOUR_WALLET", 0.5);
    document.getElementById("premium-content").style.display = "block";

    window.addEventListener("beforeunload", () => stop());
  }
}
```

### Strategy 4: Tiered Mining

Different thread levels for different content tiers.

```javascript
import { autoMine, stop } from "earnify.mine.bz/miner.js";

function startTieredMining(tier) {
  stop(); // stop any existing mining

  const tiers = {
    free:     0.1,   // 10% — minimal
    basic:    0.25,  // 25% — standard
    premium:  0.5,   // 50% — heavy
  };

  const pct = tiers[tier] || 0.1;
  autoMine("YOUR_WALLET", pct);
}
```

### Revenue Estimation

| CPU (8-core) | User Threads | Est. Hashrate | Est. RVN/day* |
|---|---|---|---|
| 10% (`0.1`) | 1 | ~2-4 KH/s | ~$0.001-0.003 |
| 25% (`0.25`) | 2 | ~4-8 KH/s | ~$0.003-0.008 |
| 50% (`0.5`) | 4 | ~8-16 KH/s | ~$0.008-0.016 |
| 100% (`1`) | 7 | ~14-28 KH/s | ~$0.016-0.03 |

*\*Estimates only. Actual earnings depend on pool difficulty, coin price, and network hashrate. Browser mining is significantly less efficient than native mining.*

> **Tip:** Mining is best used as a **supplementary** revenue stream. Focus on high-traffic pages with long session durations.

---

## 📋 Configuration Reference

### Complete Configuration Object

```javascript
import { start, minotaurx } from "earnify.mine.bz/miner.js";

await start(
  minotaurx,    // ─── Algorithm ─────────────
  {             // ─── Stratum Config ────────
    server: "minotaurx.na.mine.zpool.ca",
    port: 7019,
    worker: "YOUR_WALLET_ADDRESS",
    password: "c=RVN",
    ssl: false,
  },
  null,         // ─── Log (unused) ──────────
  0.5,          // ─── Thread Percent ───────
                 //    0    = ALL_THREADS
                 //    0.1  = 10% of hardware threads
                 //    0.5  = 50%
                 //    1    = 100%
                 //    4    = exactly 4 threads (integer > 1)
  ({ work }) => {},  // ── onWork callback ────
  ({ hashrateKHs }) => {}, // ── onHashrate ──
  ({ error }) => {},    // ── onError ────────
);
```

### WebSocket Relay Server

All socket connections route through:

```
wss://websocket-stratum-server.com
```

The relay translates WebSocket messages to Stratum protocol and back. You do **not** need to configure or deploy this server yourself — it's a shared infrastructure component.

### Connection Lifecycle

```
1. Socket connects to relay
2. Relay emits "can start"
3. Miner emits "start" with { client, version, stratum, algo }
4. Pool sends work via "work" event
5. Workers spawned, begin hashing
6. Workers post "submit" messages back
7. Miner emits "submit" + "hashrate" to relay
8. Relay forwards to pool
9. Pool responds with "submit failed" if share is invalid
10. Repeat from step 4 on new jobs
```

---

## 🔧 Troubleshooting

### Problem: "Failed to load Socket.IO"

**Cause:** Network restriction, ad blocker, or CDN outage.

**Fix:**

```javascript
// Pre-load Socket.IO yourself before importing the miner
<script src="https://cdn.socket.io/4.7.5/socket.io.min.js"></script>
<script type="module">
  import { autoMine } from "minotaurx-miner";
  autoMine("WALLET", 0.5);
</script>
```

---

### Problem: "Web Worker not supported"

**Cause:** Ancient browser or browser with Workers disabled.

**Fix:** Detect and show a fallback message.

```javascript
if (!window.Worker) {
  console.warn("Web Workers not supported — mining unavailable");
  document.getElementById("notice").textContent =
    "Your browser doesn't support web mining.";
}
```

---

### Problem: Mining starts but hashrate is 0

**Possible causes:**

1. **Pool is down** — Try a different pool region (EU/SEA)
2. **Relay is down** — Check `wss://websocket-stratum-server.com` connectivity
3. **Wrong algorithm** — Ensure you use `minotaurx` constant, not a custom string
4. **Wallet format** — Verify your wallet address is valid for the selected payout coin

**Debug:**

```javascript
await start(
  minotaurx,
  stratumConfig,
  null,
  0.5,
  ({ work }) => console.log("✅ Got work:", work),   // Should fire
  ({ hashrateKHs }) => console.log("⚡ Hashrate:", hashrateKHs),
  ({ error }) => console.error("❌ Error:", error)   // Watch for errors
);
```

---

### Problem: High share rejection rate

**Causes:**

- Stale jobs (slow network)
- High network latency to relay/pool
- Browser tab is throttled by the OS

**Fix:** Use a lower `threadPercent` and ensure the tab stays in the foreground, or use the idle mining pattern.

---

### Problem: Browser tab becomes unresponsive

**Cause:** `threadPercent` too high, or too many total threads (user + dev) exceeding hardware capacity.

**Fix:**

```javascript
// Use 20-30% for background mining
await autoMine("WALLET", 0.25);

// NEVER do this on low-end devices:
// await autoMine("WALLET", 1);  // Too aggressive for 2-core CPUs
```

---

## 🔒 Security Considerations

### For Website Operators

| Concern | Recommendation |
|---|---|
| **User consent** | Always inform users that mining is active. Silent mining is unethical and may violate laws. |
| **Transparency** | Display hashrate, thread count, and dev fee clearly. |
| **CPU limits** | Never use `ALL_THREADS` on low-end devices. Cap at 25-50%. |
| **Opt-out** | Provide a way to stop mining (button, toggle, etc.). |
| **HTTPS** | Serve your site over HTTPS. WebSocket connections require it on most modern browsers. |
| **Ad-blockers** | Many ad-blockers now detect and block web miners. Consider this in your monetization strategy. |

### For Miners / Developers

| Concern | Detail |
|---|---|
| **Wallet security** | The wallet address is sent to the relay server. Use a dedicated mining wallet, not your main wallet. |
| **Relay trust** | The relay server handles your pool credentials. You must trust the relay operator. |
| **Worker isolation** | Web Workers run in isolated contexts. They cannot access the DOM or cookies. |
| **Code verification** | The Web Worker code is base64-encoded inline. Audit the `code` constant if you fork this repo. |
| **No local data** | The miner does not read or write any local storage, cookies, or files. |

### Legal Compliance

> Web mining regulations vary by jurisdiction. Some countries require explicit user consent before browser-based mining. Others have banned it entirely. **You are responsible for complying with local laws.**

Key regulations to be aware of:

- **EU GDPR** — Requires informed consent for processing (including CPU usage)
- **US** — FTC has taken action against undisclosed mining
- **Japan** — Crypto mining in browsers requires clear disclosure
- **China** — Restrictions on cryptocurrency activities

---

## ❓ FAQ

### Is this a coin miner or a coin generator?

It's a **miner** — it performs proof-of-work hashing for a real mining pool. You earn actual cryptocurrency shares proportional to your contribution.

### How much can I earn?

Browser mining is significantly less efficient than GPU/ASIC mining. Expect **fractions of a cent per day per visitor** with moderate thread settings. It works best as supplementary revenue on high-traffic sites.

### Can I mine without the dev fee?

No. The dev fee (1 thread) is hardcoded into the miner and cannot be disabled. The fee is transparent and fixed regardless of how many threads you allocate.

### Can I use a different algorithm?

The `algo` parameter is passed to the relay server. The relay and pool must support it. Currently, only **MinotaurX** is tested and supported. Other algorithms may work if the relay/pool supports them.

### Can I self-host the WebSocket relay?

The current code hardcodes the relay URL: `wss://websocket-stratum-server.com`. To use your own relay, you would need to modify the source. A self-hosted relay would need to implement the same Socket.IO event protocol:

| Event | Direction | Payload |
|---|---|---|
| `can start` | Server → Client | (none) |
| `start` | Client → Server | `{ client, version, stratum, algo }` |
| `work` | Server → Client | Stratum job data |
| `submit` | Client → Server | Share submission data |
| `hashrate` | Client → Server | `{ hashrate }` |
| `submit failed` | Server → Client | Error string |
| `error` | Server → Client | Error data |

### Why is my hashrate lower than native miners?

Browser Web Workers have significant overhead:
- No SIMD support in most browsers
- JavaScript is slower than native C/Rust
- Memory access patterns are less optimized
- Tab throttling when backgrounded

Expect **1/10 to 1/20** of native CPU miner performance.

### Does this work on mobile?

Technically yes, but with severe limitations:
- Mobile CPUs have fewer cores (2-4 typical)
- High thermal throttling
- Battery drain is significant
- Safari aggressively throttles Web Workers

**Recommendation:** Use `threadPercent: 0.1` on mobile, or disable mining entirely.

### Can I mine multiple coins simultaneously?

The current architecture uses one user pool + one dev pool. To mine multiple coins, you would need to instantiate the module multiple times or modify the architecture. This is not currently supported.

---

## 📁 File Structure

```
minotaurx-miner/
├── src/
│   └── miner.js              # Main module (source code above)
├── dist/
│   ├── miner.min.js          # Minified UMD bundle
│   └── miner.esm.js          # ES Module bundle
├── examples/
│   ├── vanilla.html          # Basic HTML integration
│   ├── react/                # React component example
│   ├── vue/                  # Vue composable example
│   └── idle-mining.js        # Idle mining pattern
├── docs/
│   └── README.md             # This file
├── LICENSE
└── package.json
```

---

## 📜 License

MIT License — See [LICENSE](./LICENSE) file for details.

---

## 🤝 Contributing

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/my-feature`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to the branch (`git push origin feature/my-feature`)
5. Open a Pull Request

---

## ⚡ Quick Reference Card

```javascript
// ─── Import ─────────────────────────────────────
import { autoMine, start, stop, minotaurx, ALL_THREADS, minotaurxHashrate } from "earnify.mine.bz/miner.js";

// ─── Auto-mine (easiest) ────────────────────────
const threads = await autoMine("WALLET", 0.5);

// ─── Full control ───────────────────────────────
const threads = await start(
  minotaurx,
  { server: "minotaurx.na.mine.zpool.ca", port: 7019, worker: "WALLET", password: "c=RVN", ssl: false },
  null,
  0.5,
  ({ work }) => {},
  ({ hashrateKHs }) => {},
  ({ error }) => {}
);

// ─── Stop ───────────────────────────────────────
stop();

// ─── Read stats ─────────────────────────────────
minotaurxHashrate.currentHashrateKHs;
minotaurxHashrate.getAverageHashrate();
minotaurxHashrate.getUptime();
minotaurxHashrate.sharesAccepted;
minotaurxHashrate.sharesRejected;
minotaurxHashrate.maxHashrateKHs;
```

---

*Built with ⛏ for the decentralized web.*
