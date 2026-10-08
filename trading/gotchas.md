You have built an incredibly strong foundation. By linking Auction Market Theory (AMT) with Microstructure Momentum, and factoring in Regime Decay and Time Exits, you are already thinking far ahead of most retail traders.
However, before you freeze these concepts into code to risk your first $100, there is one massive, final school of thought that you are missing.
In institutional quantitative finance, this is known as Market Microstructure Friction and Micro-Structure Invariance. It dictates exactly how your strategy will interact with the real-world plumbing of the exchange.
------------------------------
You have focused heavily on when to buy (signals) and when to leave (decay). You must now account for how the act of your algorithm entering a trade structurally shifts the market itself. This school of thought, pioneered heavily by researchers like Albert Kyle and Anna Obizhaeva, introduces three final concepts:
## 1. The Square-Root Law of Market Impact

* The School of Thought: You cannot assume that if a stock is trading at $100.00, your buy order will execute at $100.00. The very act of buying chews through liquidity, pushing the price away from you.
* The Quantitative Reality: Quantitative research proves that immediate transaction costs (slippage) scale roughly with the square root of the size of your trade relative to the daily volume.
* Your Strategy Application: Since you are starting with a small $100 capital base, your physical market impact is virtually zero. However, your broker's routing latency will mimic this effect. You must program your backtesting engine to assume that your entries will always suffer a penalty of at least 0.5 to 1 full spread tick.

## 2. The Micro-Structural Lead-Lag Invalidation

* The School of Thought: In an auction, not all participants see information at the same time. High-Frequency Trading (HFT) firms pay millions to locate their servers inside the exchange data centers (co-location).
* The Quantitative Reality: If you execute your "Velocity Breakout" using a standard home internet connection, you are experiencing Information Latency. By the time your local script processes the tick, calculates the velocity, and transmits a market order to the broker, the HFT firms have already detected the anomaly, swept the liquidity vacuum, and left you with a sub-optimal entry price at the very peak of the spike.
* Your Strategy Application: To defeat this, your strategy design should look for Absorption Fades or Re-Auction Mean Reversions rather than trying to chase ultra-fast, raw momentum breakouts. Fades give your retail infrastructure time to process the information safely.

------------------------------
To bridge your theoretical ideas into a real, tradable script, your strategy development pipeline must pass through three distinct quantitative filters:

[ Algorithmic Idea ] ──> [ Filter 1: Transaction Cost Drag ] ──> [ Filter 2: Regime Gate ] ──> [ Filter 3: Execution Logic ]


* Filter 1: The Transaction Cost Friction Test
* The Rule: If your strategy exits based on strict time stops (e.g., exiting after 5 minutes), it will execute a massive number of trades. You must verify that your expected statistical win size is significantly larger than the combined cost of the bid-ask spread and broker commissions. If your strategy earns an average of 4 ticks per trade, but the spread cost is 3 ticks, your model is a charity for the broker.
* Filter 2: The Binary Regime Gate
* The Rule: Your strategy script must have an absolute conditional clause at the very top of the execution loop: if (Hurst_Exponent < 0.5) { Strategy_Enabled = false }. You never let the execution model run blindly without its analytical master switch active.
* Filter 3: The Order Type Protocol
* The Rule: For a small capital account, you must decide between Market Orders (guaranteed execution, poor price due to slippage) vs. Limit Orders (guaranteed price, risk of the trade leaving without you). For momentum strategies, use market orders with a strict price slippage tolerance limit coded into the API call.

------------------------------
You are now ready to construct your first concrete trading engine. To map out the code, you need to choose your architectural starting point:

   1. The Mean-Reversion Absorption Engine: Built to exploit balanced regimes. It identifies the Value Area boundaries, monitors when volume per tick expands while price velocity drops to zero (Absorption), and trades back toward the Point of Control.
   2. The Velocity Time-Stop Momentum Engine: Built to exploit imbalanced regimes. It identifies range breakouts, tracks the kinetic energy of Cumulative Volume Delta, and uses a dynamic time-based exit to ruthlessly cut trades the microsecond momentum velocity drops.

------------------------------
To write the actual code framework, let me know:

* Which of these two structural engine options aligns best with your trading intuition?
* Do you want to build this baseline engine inside Python (for complete raw data control) or Pine Script (for rapid visual paper-trading prototyping)?


