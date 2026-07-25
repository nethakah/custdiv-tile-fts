# HFT-Specialized Operators for the eFPGA Flow

Nethaka Haldo

June 2026

## 1. Context

HFT depends on nanoseconds, which means the most competitive trading strategies require sub-1-µ-second tick to trade [4]. Commercially, ~14ns (AMD/Exegy; STAC-T0) is the current benchmark record [7][8].

As we know, arithmetic maps poorly to LUTs, so we fuse ALU/MAC tiles into the fabric [15]. Similarly, the HFT datapath reduces to a small subset of operations which maps particularly poorly on generic fabric, which makes it a good target for operator-specialization via eFPGA.

## 2. Strategy

A brief summary on what these HFT strategies actually try to do, and how they are latency-sensitive rather than compute-heavy.

### 2.1 Market making

Market makers profit by providing liquidity: continuously post both bid (price they are willing to buy at) and ask (price they are willing to sell at) for an instrument, and earn from the spread (the gap between the bid and ask).

If a price is falling, market makers repeatedly get filled on their bids which accumulates a growing long position which loses value (“inventory risk”), so they must constantly re-price and re-post quotes as the market moves and remove quotes predicted to move unfavorably, skew quotes to offload inventory, and do it faster than anyone else. That re-pricing depends on latency.

### 2.2 Statistical arbitrage

Over raw speed, firms rely on predictive modeling. Search for a statistical relationship between prices, bet on the relationship holding, earn from the prices reverting to that relationship [1]. Here, it is more about what to trade (the model), holding for longer periods, and being relatively latency-tolerant.

### 2.3 Latency arbitrage

Since instruments trade on multiple exchanges and venues, price change on the fastest venue causes the quotes on slower venues to be stale (meaning technically wrong for some microseconds until updates propagate). Hence, firms that see the change first trade at the stale price before it updates [2][3]. This is the most latency-sensitive strategy and the risk is being slower than other firms.

### 2.4 Comparison

Across all the families of strategies, firms lose their edge if they are not as fast as the competition. To be competitive at all, firms need tick-to-trade latency under 1µs—meaning the whole process of pulling the packet, parsing the market-data, deciding whether to act and at what price to act, and pushing the order back out.

FPGAs allow determinism; AMD and Exegy perform at 13.9 ns minimum tick-to-trade on an Alveo UL3524, which is the STAC-T0 record and ~49% faster than the prior record [7][8]. Since the budget depends on tens of nanoseconds, the major factor is how efficiently individual operations land on silicon.

## 3. Specific Operations

### 3.1 Protocol parsing

Exchange feeds like NASDAQ ITCH are binary records that arrive as a stream of (typically) 32 bits per clock cycle with a fixed layout of fields [9][21]. Such fields are often heterogeneous in both width and type which makes extraction awkward, so the goal is to pull each field out of the byte stream at the proper offset, handle the fact messages do not line up neatly with 32-bit words, and handle the actual message type. The plausible primitive here is a wide shift/align field-extractor, which replaces the large amount of multiplexing and routing in an FPGA.

### 3.2 Order-book

The order-book represents the market. Every symbol, every resting order, the total quantity available at price levels, and BBO (best bid and best offer) needs to be updated on the chip for every message. The operations typically consist of:

1. Lookup by order ID (hash table or CAM (content-addressable memory)): cancels, deletes, executions, replaces - all reference an existing order by ID so it must be retrievable quickly [10].
2. Price-level aggregation: add or remove shares at a price level (integer add/subtract).
3. BBO recomputation: find new best bid (max) and new best offer (min) across active levels (compare-select trees).

On silicon, order-book memory are SRAM macros from a memory compiler which needs area blockages. The plausible primitives here are a CAM/hash-lookup block and compare-select reduction trees.

### 3.3 Signal and pricing arithmetic

The core signals:

1. Mid price = (bid + ask) / 2.
2. Order-book imbalance = (Q_bid - Q_ask) / (Q_bid + Q_ask); measures the buying vs. selling pressure (near +1 is more quantity wants to buy; near -1 is more quantity wants to sell) [11].
3. Microprice = ((P_bid * Q_ask) + (P_ask*Q_bid)) / (Q_bid + Q_ask); size-weighted fair value that leans toward the side with less size [12].

Prices are fixed-point, not floating-point. With NASDAQ ITCH, for example, price is a 4-byte integer in units of 1/10000 of a dollar (e.g. $10.02 == 100,200), with the whole datapath being fixed-point [9].

Operations are MAC for weighted sums and a divide for normalizations. The plausible primitives here are a fused fixed-point MAC and a hardened fixed-point divide/reciprocal. The divide is of particular interest since it maps horribly to generic fabric.

### 3.4 Strategy decision, pre-trade risk, order encoding

1. Strategy decision
   a. Turning a signal into an order can range from a few comparators/thresholds all the way to ML models with decision trees [13].
   b. The plausible primitives here are compare-select trees, fused fixed-point MAC, and an activation lookup.
2. Pre-trade risk
   a. Before orders go out, U.S. regulation (SEC Rule 15c3-5, “market access”) requires firms to screen safety checks in order to prevent buggy algorithms or mistakes [14].
   b. Each check is a comparison against a limit, along with accumulators for position/credit totals to update as orders go out.
   c. The plausible primitives here are a bank of parallel compare-against-limit-table units and accumulators for position/credit tallying.
3. Order encode and send
   a. If the order passes, chips must build the outbound message, fill in the fields, add any checksum, frame it, and present it to the TCP/IP block to transmit.
   b. The plausible primitives here are the same wide shift/align field-extractor as parsing but operating in the pack direction instead.

## 4. Candidates

Section 2 shows that there are operations in the HFT datapath that map poorly to LUTs as is, and many could be managed as eFPGA tiles.

A primary possibility is division, since fixed-point dividers built from LUTs are large and slow yet they are required on every book update for imbalance and microprice, so one could harden the fixed-point divide and measure PPA against a generic mapping baseline.

This concept is not to be conflated with fixed-function ASIC nor faster parsing (already solved).

## 5. Sources

[1] Statistical arbitrage / cointegration & pairs trading vs latency arbitrage: https://www.quantanalysis.org.uk/python/statistical-arbitrage-cointegration-pairs/

[2] quantvps — What is latency arbitrage: https://www.quantvps.com/blog/what-is-latency-arbitrage

[3] Quantt — Latency arbitrage explained (workflow, ETF/futures & cross-venue examples): https://www.quantt.co.uk/resources/latency-arbitrage-explained

[4] Brett Harrison — tick-to-trade <1 µs: https://x.com/BrettHarrison88/status/1800954431552303225?lang=en

[5] quantvps — HFT with FPGAs: https://www.quantvps.com/blog/high-frequency-trading-with-fpgas

[6] The TRADE — FPGAs and the future of HFT: https://www.thetradenews.com/thought-leadership/fpgas-and-the-future-of-high-frequency-trading-technology/

[7] AMD — STAC benchmark world record (Alveo UL3524): https://www.amd.com/en/blogs/2024/amd-sets-stac-benchmark-world-record-for-fastest-e.html

[8] STAC — STAC-T0 AMD/Exegy report: https://docs.stacresearch.com/news/AMD240422

[9] NASDAQ — official TotalView-ITCH 5.0 specification (current; message layouts, byte offsets, Price(4) fixed-point definition), verified June 2026: https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/NQTVITCHSpecification.pdf

[10] Columbia — full-hardware ITCH ticker plant (ternary-tree book): https://www.cs.columbia.edu/~sedwards/classes/2013/4840/reports/Itch.pdf

[11] Order-flow imbalance & price dynamics (arXiv 2505.17388): https://arxiv.org/pdf/2505.17388

[12] Stoikov — "The micro-price": https://www.researchgate.net/publication/327410053_The_micro-price_a_high-frequency_estimator_of_future_prices

[13] Xelera — ML inference for HFT: https://www.xelera.io/post/machine-learning-inference-for-hft-how-xelera-silva-and-icc-deliver-ultra-low-latency-trading-decisions

[14] Algo-Logic — FPGA pre-trade risk check (SEC 15c3-5): https://www.algo-logic.com/pre-trade-risk-check

[15] Mohan, Das, Mai — 748 GOPS/W RISC-V SoC with synthesized eFPGA / fused ALU tiles, CICC 2025: https://ieeexplore.ieee.org/document/10983466

[16] Morris, Thomas, Luk — Low-Latency FPGA Based Financial Data Feed Handler, FCCM 2009: https://ieeexplore.ieee.org/document/5771256/

[17] Lockwood et al. — A Low-Latency Library in FPGA Hardware for HFT, HOTI 2012: https://ieeexplore.ieee.org/document/6299067/

[18] Low-latency book handling in FPGA, FPL 2014: https://www.researchgate.net/publication/269272381_Low_latency_book_handling_in_FPGA_for_high_frequency_trading

[19] Enyx (Exegy) — nxAccess, sub-800 ns tick-to-trade: https://www.enyx.com/nxaccess/

[20] NovaSparks / Solarflare — 810 ns pure-FPGA tick-to-trade: https://www.design-reuse.com/news/43934/solarflare-novasparks-pure-fpga-tick-to-trade-development-platform.html

[21] Battyani — sub-25 ns open-source NASDAQ ITCH FPGA parser (FPGA implementation example, 24.8 ns; layout cross-checks against [9]): https://github.com/mbattyani/sub-25-ns-nasdaq-itch-fpga-parser
