Yes, you absolutely can, and it is completely normal to do so. In data science, you can easily store both granular tick details and high-level candle indicators inside a single, unified pandas DataFrame using a design pattern called Nested Data Structuring or Columnar Compounding.
Since you are processing data candle-by-candle, you can make every single row of your main DataFrame represent a 1-minute candle, while one of the columns in that exact same row holds a Python list or nested array containing all the individual ticks that occurred inside that minute.
------------------------------
## The All-in-One DataFrame Structure
This layout mimics the exact structural reality of the broker charts while preserving 100% of your sub-second data:

| Timestamp (Index) | Open | High | Low | Close | AMT Feature (Distance to POC) | Momentum Feature (ROC_3) | Raw Ticks Stream |
|---|---|---|---|---|---|---|---|
| 14:00:00 | 1.0850 | 1.0856 | 1.0849 | 1.0855 | -0.0002 | +0.12% | [[14:00:01, 1.0850], [14:00:15, 1.0852], ...] |
| 14:01:00 | 1.0855 | 1.0859 | 1.0853 | 1.0854 | +0.0004 | +0.08% | [[14:01:04, 1.0855], [14:01:32, 1.0858], ...] |

------------------------------
## How to Code This All-in-One Structure in Python
Here is how you can write a clean loop to take an incoming stream of ticks, construct the 1-minute candle parameters, calculate your indicators, and attach the underlying raw tick records into a single unified row [finance]:

import pandas as pdimport numpy as np
def build_candle_with_ticks(tick_stream_block):
    """
    Processes a block of ticks belonging to a single 1-minute timeframe 
    and returns a single row containing OHLC, Indicators, and Raw Ticks.
    """
    # 1. Convert the current block of ticks into a temporary DataFrame
    df_temp = pd.DataFrame(tick_stream_block)
    df_temp['Timestamp'] = pd.to_datetime(df_temp['Timestamp'])
    
    # 2. Extract standard OHLC metrics
    candle_time = df_temp['Timestamp'].iloc[0].floor('1Min')
    open_p = df_temp['Ask'].iloc[0]
    high_p = df_temp['Ask'].max()
    low_p = df_temp['Ask'].min()
    close_p = df_temp['Ask'].iloc[-1]
    
    # 3. Package the raw ticks into a clean list of tuples to store inside the row
    # This prevents you from losing your sub-second microstructural data
    raw_ticks_list = list(zip(df_temp['Timestamp'].dt.strftime('%H:%M:%S.%f'), df_temp['Ask']))
    
    # 4. Construct the unified single-row dictionary
    candle_row = {
        "Timestamp": candle_time,
        "Open": open_p,
        "High": high_p,
        "Low": low_p,
        "Close": close_p,
        "Raw_Ticks": raw_ticks_list # Nested column!
    }
    
    return candle_row
# --- Example Usage / Simulation ---# Imagine these arrive from your broker over the course of 1 minuteticks_minute_1 = [
    {"Timestamp": "2026-10-11 14:00:02.120", "Ask": 1.0850},
    {"Timestamp": "2026-10-11 14:00:15.450", "Ask": 1.0854},
    {"Timestamp": "2026-10-11 14:00:59.880", "Ask": 1.0852},
]
# Run the pipeline for that minuteprocessed_row = build_candle_with_ticks(ticks_minute_1)
# Convert to your master pandas DataFramemaster_df = pd.DataFrame([processed_row]).set_index("Timestamp")
# 5. Now you can easily calculate indicators directly across the rows
master_df['ROC_1'] = master_df['Close'].pct_change() * 100# Simple example proxy indicator for distance to baseline
master_df['Distance_to_FairValue'] = master_df['Close'] - master_df['Open']

print(master_df[['Open', 'Close', 'Distance_to_FairValue', 'Raw_Ticks']])

------------------------------
## Pro-Tips for Keeping Your Python Script Fast

   1. Drop Raw_Ticks Before Training: Machine learning frameworks like XGBoost do not understand nested lists. When you are ready to feed your data to model.fit(), simply filter them out using:
   
   X_train = master_df.drop(columns=['Raw_Ticks'])
   
   This keeps your training script incredibly fast while your master dataset keeps the records perfectly safe on your hard drive.
   2. Saving to Disk: Standard CSV files struggle to store nested lists properly (they convert them into annoying strings). Because you have a nested structure, save this DataFrame using Parquet format (master_df.to_parquet('filename.parquet')). It naturally supports nested arrays and loads perfectly back into Python without changing your data types.

Would you like to write the logic that takes this unified row and automatically checks if your 1-minute continuation or 3-minute reversal target conditions were hit in the future?
Let me know if we should build that labeling function next!

