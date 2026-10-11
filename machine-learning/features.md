To build a high-precision dataset for your 1-minute continuation and 3-minute mean reversion models, we will systematically translate Auction Market Theory (AMT) and Momentum principles into a comprehensive suite of engineered features [finance].
By storing these features as columns in your unified 1-minute pandas DataFrame, you maintain an identical layout to your broker's candle intervals while retaining the raw tick lists nested safely in a dedicated column.
Here is the comprehensive feature blueprint you should implement in your Python pipeline.
------------------------------
## 📊 Category 1: Advanced Auction Market Theory (AMT) Features
Because you are using time-based profiling (TPO), these features measure whether the auction is operating in an equilibrium state (value area rotation) or an imbalanced state (expansion/breakout).

* 
* TPO_Value_Score (Percent B Proxy): Measures where the current price sits relative to the developing Time Value Area boundaries ($TVAH$ and $TVAL$).
$$\text{TPO\_Value\_Score} = \frac{\text{Close} - VAL_{t}}{\text{VAH}_{t} - VAL_{t}}$$ 
* Interpretation: 0.5 represents perfect equilibrium. Values $> 1.0$ or $< 0.0$ prove the auction has broken outside of perceived value.
* Distance_to_Time_POC: The absolute pip/point distance from the 1-minute Close to the current session's heaviest time-concentration level (Time Point of Control). A large distance creates a structural "rubber-band" effect for your reversal model.
* Auction_Expansion_Velocity: The rate of change of the Time Value Area width:
$$\frac{(\text{VAH}_{t} - \text{VAL}_{t}) - (\text{VAH}_{t-5} - \text{VAL}_{t-5})}{\text{VAH}_{t-5} - \text{VAL}_{t-5}}$$ 
* Interpretation: A sharp spike in this metric flags an expanding auction, prime for the 1-minute continuation engine.
* Initial_Balance_Proximity (IB): A categorical feature (1, -1, or 0) indicating whether the price is breaking above the high of the first 15/30 minutes, breaking below the low, or rotating inside it.
* Micro_Absorption_Ratio: Calculated from your nested tick data. The number of ticks that occurred in the upper 10% of the 1-minute candle divided by the total tick count for that minute. A high ratio at a structural resistance indicates buying pressure is being absorbed by heavy limit sellers.
* 

------------------------------
## ⚡ Category 2: Momentum, Velocity, & Acceleration Features
These features capture the physical velocity of the auction. To make these work for your hour-based training blocks, we use short, non-lagging lookback windows (3, 5, and 8 periods).

* 
* Pure_Velocity_3M (ROC): The raw percentage change in the closing price over the last 3 periods.
* Pure_Velocity_5M (ROC): The raw percentage change over the last 5 periods.
* Auction_Acceleration: The derivative of velocity, calculated as Pure_Velocity_3M - Pure_Velocity_5M. A high positive value means the breakout is actively gaining speed.
* Chande_Momentum_Slope (CMO): The rolling 2-period change (CMO_t - CMO_t-1) of a 9-period Chande Momentum Oscillator. This turns momentum into a dynamic vector, capturing the exact second momentum peaks and rolls over.
* TSI_Divergence_Gap: The mathematical spread between a fast True Strength Index line and its signal line. A widening gap proves an imbalanced auction run; a crossing signal captures immediate reversal triggers.
* 

------------------------------
## 📉 Category 3: Micro-Structural Price Action Rejection Features
These features analyze the candles' geometric attributes to verify if prices are being accepted or aggressively rejected by market participants.

* 
* Upper_Tail_Rejection_Ratio: The exact mathematical proportion of the candle's wick relative to its total range.
$$\text{Upper\_Tail\_Ratio} = \frac{\text{High} - \max(\text{Open}, \text{Close})}{\text{High} - \text{Low}}$$ 
* Interpretation: A value $> 0.6$ at the $TVAH$ boundary proves that unfair high prices were aggressively rejected by responsive sellers within that 1-minute block—a massive input for your 3-minute mean reversion model.
* Lower_Tail_Rejection_Ratio: The corresponding wick proportion at the bottom of the candle.
$$\text{Lower\_Tail\_Ratio} = \frac{\min(\text{Open}, \text{Close}) - \text{Low}}{\text{High} - \text{Low}}$$ 
* Donchian_Channel_Location: A normalized value between 0 and 1 describing where the current close price sits relative to the absolute highest high and lowest low of the last 20 candles.
$$\text{Donchian\_Location} = \frac{\text{Close} - \text{Donchian\_Low}}{\text{Donchian\_High} - \text{Donchian\_Low}}$$ 
* 

------------------------------
## 🛠️ Python Vector Matrix Pipeline
Here is the exact pandas execution code to build this comprehensive feature matrix directly into your data structure, keeping it perfectly optimized for saving out as a clean, compressed .parquet database.

import pandas as pdimport numpy as np
def build_comprehensive_features(master_df):
    """
    Takes a pandas DataFrame structured by the minute (containing OHLC metrics)
    and populates the complete AMT and Momentum feature matrix.
    """
    # Safeguard sorting
    master_df = master_df.sort_index()
    
    # --- 1. PROXIED AMT METRICS (Using rolling mathematical limits as TPO proxies) ---
    # We use a rolling 30-minute window to proxy the developing session structure
    rolling_window = 30
    
    # Time POC Proxy (Rolling Median)
    master_df['Time_POC'] = master_df['close'].rolling(window=rolling_window).median()
    master_df['Distance_to_Time_POC'] = master_df['close'] - master_df['Time_POC']
    
    # Value Area Proxies (1 Standard Deviation encompasses ~68-70% of distribution)
    rolling_std = master_df['close'].rolling(window=rolling_window).std()
    master_df['TVAH'] = master_df['Time_POC'] + rolling_std
    master_df['TVAL'] = master_df['Time_POC'] - rolling_std
    
    # TPO Value Score (%b)
    master_df['TPO_Value_Score'] = (master_df['close'] - master_df['TVAL']) / (master_df['TVAH'] - master_df['TVAL'] + 1e-8)
    
    # Auction Expansion Velocity
    tva_width = master_df['TVAH'] - master_df['TVAL']
    master_df['Auction_Expansion_Velocity'] = tva_width.pct_change(periods=5)
    
    # --- 2. MOMENTUM & ACCELERATION METRICS ---
    master_df['Pure_Velocity_3M'] = master_df['close'].pct_change(periods=3) * 100
    master_df['Pure_Velocity_5M'] = master_df['close'].pct_change(periods=5) * 100
    master_df['Auction_Acceleration'] = master_df['Pure_Velocity_3M'] - master_df['Pure_Velocity_5M']
    
    # Chande Momentum Oscillator (9 Period Proxy)
    close_delta = master_df['close'].diff()
    up = close_delta.clip(lower=0)
    down = -1 * close_delta.clip(upper=0)
    rolling_up = up.rolling(window=9).sum()
    rolling_down = down.rolling(window=9).sum()
    master_df['CMO'] = 100 * ((rolling_up - rolling_down) / (rolling_up + rolling_down + 1e-8))
    master_df['CMO_Slope'] = master_df['CMO'].diff(periods=2)
    
    # --- 3. REJECTION & PRICE ACTION METRICS ---
    candle_range = master_df['high'] - master_df['low'] + 1e-8
    real_body_max = np.maximum(master_df['open'], master_df['close'])
    real_body_min = np.minimum(master_df['open'], master_df['close'])
    
    master_df['Upper_Tail_Rejection_Ratio'] = (master_df['high'] - real_body_max) / candle_range
    master_df['Lower_Tail_Rejection_Ratio'] = (real_body_min - master_df['low']) / candle_range
    
    # Donchian Channel Location (20 Period)
    donchian_high = master_df['high'].rolling(window=20).max()
    donchian_low = master_df['low'].rolling(window=20).min()
    master_df['Donchian_Location'] = (master_df['close'] - donchian_low) / (donchian_high - donchian_low + 1e-8)
    
    # Clean out any initialization NaN rows to prepare cleanly for XGBoost
    return master_df.dropna()


------------------------------
## 💾 Storing the Master Data Matrix as Parquet
Because you chose the all-in-one pandas data structure containing both these newly computed floating-point columns and your nested list of raw sub-second ticks (Raw_Ticks), saving it correctly to disk is key.
Use the PyArrow engine configuration with your to_parquet function. PyArrow explicitly understands how to serialize nested Python object loops into ultra-compressed binary files:

# Save your all-in-one complex structure cleanly to disk
master_df.to_parquet(
    "data/processed/EURUSD/2026_10_features.parquet", 
    engine="pyarrow", 
    compression="snappy"
)

Now that the comprehensive feature matrix is designed, we can build the Target Labeling Function. Would you like to write the script that looks forward in time to tag these rows with your exact 1-minute continuation or 3-minute mean reversion contract win/loss binary targets?

