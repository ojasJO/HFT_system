# High-Frequency Trading (HFT) Simulation & Execution Engine

A low-latency, multi-threaded Proof of Concept (PoC) built in Python for backtesting and simulating high-frequency trading (HFT) strategies on historical market data.

This project is a complete quantitative pipeline: it captures live websocket data, replays it with high chronological fidelity, and runs concurrent arbitrage strategies via a decoupled, message-passing architecture.

## Performance Metrics (PoC Benchmarks)

Since this is an HFT simulation, minimizing End-to-End (E2E) latency is the primary goal. 
* **Tick-to-Signal E2E Latency:** [~2.50 ms] average (Server dispatch to Client strategy execution).
* **Message Throughput:** Capable of processing [10,000+] ticks per second without queue degradation.
* **Network Overhead (ZMQ):** Sub-millisecond serialization and transport via local PUB/SUB topology.

## Architecture & Separation of Concerns

The system operates as a four-stage distributed pipeline. Each component runs as an isolated process to prevent I/O blocking and ensure high throughput, communicating exclusively via a ZeroMQ (ZMQ) backbone.

| Component | Responsibility | Tech / Pattern |
| :--- | :--- | :--- |
| **Data Recorder** | Connects to crypto exchange APIs to capture live Trade and Level 1 Quote data. Saves to local CSV. | WebSockets, asyncio |
| **Server** | The market simulator. Reads data files, sorts by origin timestamp, and broadcasts events as if happening live. | ZMQ PUB, Chronological Replay |
| **Relay Node** | Message forwarder. Subscribes to the server and re-publishes to clients, preventing server overload. | ZMQ SUB/PUB Forwarder |
| **Client Engine** | Ingests the data feed, evaluates multi-factor strategy logic, tracks inventory, and updates the GUI. | Threading, Queue, Tkinter |

## Core Systems

### The Data Flow

    Historical CSVs 
          ↓
    [ Market Server ] --(ZMQ PUB)--> [ Relay Node ] --(ZMQ PUB)--> [ Client Thread: Ingestion ]
                                                                          ↓ (Thread-Safe Queue)
                                                                   [ Client Thread: Strategy Engine ]
                                                                          ↓ (Execution & Stats)
                                                                   [ Main Thread: Tkinter UI ]

### Trading Strategies
The engine concurrently evaluates multiple independent strategies:
1.  **Latency Arbitrage:** Exploits micro-structural delays between trade prints and limit order book quotes. Tracks price dislocations in O(1) time.
2.  **Statistical Arbitrage (Pairs Trading):** Monitors cointegrated assets (e.g., BTC/ETH). Uses rolling standard deviations to calculate real-time Z-scores, buying/selling the spread upon mean reversion triggers.

### State Management & Tracking
Instead of heavy database lookups, the execution client relies on high-performance Python data structures:
* `collections.defaultdict`: For O(1) state tracking of real-time bids/asks across symbols.
* `collections.deque`: For O(1) append/pop operations on rolling historical data (PnL charts, Latency tracking).
* `queue.Queue`: For thread-safe, lock-free data passing between the ZMQ ingestion thread and the strategy execution engine.

## Technology Stack

* **Core:** Python 3.10+
* **Networking:** ZeroMQ (pyzmq) (High-throughput IPC)
* **Data Analysis:** Pandas, NumPy (Vectorized rolling calculations)
* **Visualization:** Matplotlib, Tkinter (Real-time threaded UI updates)

