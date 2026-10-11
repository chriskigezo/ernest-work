# Binary Options Automated Trading System Blueprint
## Complete Architectural Framework: AMT, Momentum, & Microstructure (1-Min & 3-Min Models)

This document outlines the complete architectural blueprint for a statistical, hour-based machine learning trading framework optimized for 1-minute continuation and 3-minute mean reversion strategies using **Auction Market Theory (AMT)** and **Momentum**.

---

## 1. Unified Nested Data Structure Configuration

The data model utilizes a single, time-aligned `pandas.DataFrame` where every row represents exactly one **1-minute candle** matching the broker's asset charts. Sub-second tick details are preserved inside a nested Python object list column to prevent data leakage and maintain granular historical depth.

```python
# Master DataFrame Column Specification
# ---------------------------------------------------------------------------------------------------
# Index:            Timestamp (DatetimeIndex, frequency='1Min')
# OHLC Core:        'open', 'high', 'low', 'close'
# Nested Records:   'Raw_Ticks' -> List of tuples: [(Timestamp_String, Price_Float), ...]
# Features (AMT):   'Distance_to_Time_POC', 'TPO_Value_Score', 'Auction_Expansion_Velocity', ...
# Features (Mom):   'Pure_Velocity_3M', 'Pure_Velocity_5M', 'Auction_Acceleration', 'CMO_Slope', ...
# Targets:          'Target_Continuation_1M', 'Target_Reversion_3M'
```

---

## 2. Definitive Feature Engineering & Data Pipeline

This engine processes live or archived tick streams block-by-block, aggregates them into candle records, derives the technical parameters, and appends the complete feature vector into a multi-dimensional matrix.

```python
FUNCTION build_production_feature_matrix(master_df):
    SORT master_df chronologically by Timestamp index
    
    FOR EACH row in master_df:
        // A. Extract Microstructural Features from Nested Ticks List
        Calculate candle_range = high - low + 0.00000001
        Calculate Intraday_Drift_Vector = (close - open) / candle_range
        
        Calculate standard_deviation of prices inside 'Raw_Ticks' list
        Calculate Tick_Volatility_Ratio = standard_deviation / candle_range
        
        Calculate price changes from consecutive elements in 'Raw_Ticks' list
        Count upticks (changes > 0) and downticks (changes < 0)
        Calculate Tick_Delta_Momentum = upticks / (upticks + downticks + 0.00000001)
        
    // B. Calculate Proxied Auction Market Theory (AMT) Features
    Set rolling_window = 30
    Calculate Time_POC = 30-period rolling median of close
    Calculate Distance_to_Time_POC = close - Time_POC
    
    Calculate rolling_std = 30-period rolling standard deviation of close
    Calculate TVAH = Time_POC + rolling_std
    Calculate TVAL = Time_POC - rolling_std
    Calculate TPO_Value_Score = (close - TVAL) / (TVAH - TVAL + 0.00000001)
    
    Calculate tva_width = TVAH - TVAL
    Calculate Auction_Expansion_Velocity = 5-period percentage change of tva_width
    
    // C. Calculate Momentum & Velocity Features
    Calculate Pure_Velocity_3M = 3-period rate of change of close * 100
    Calculate Pure_Velocity_5M = 5-period rate of change of close * 100
    Calculate Auction_Acceleration = Pure_Velocity_3M - Pure_Velocity_5M
    
    Calculate Chande Momentum Oscillator (CMO) over a rolling 9-period window
    Calculate CMO_Slope = 2-period difference of CMO
    
    // D. Calculate Candle Geometry & Multi-Timeframe Fractal Filters
    Calculate Upper_Tail_Rejection_Ratio = (high - max(open, close)) / candle_range
    Calculate Lower_Tail_Rejection_Ratio = (min(open, close) - low) / candle_range
    
    Calculate donchian_high_20 = 20-period rolling maximum of high
    Calculate donchian_low_20 = 20-period rolling minimum of low
    Calculate Donchian_Location = (close - donchian_low_20) / (donchian_high_20 - donchian_low_20 + 0.00000001)
    
    Calculate donchian_high_155 = 15-period rolling maximum of high
    Calculate donchian_low_155 = 15-period rolling minimum of low
    Calculate Distance_to_15M_POC = close - 15-period rolling median of close
    
    // E. Extrapolate Temporal Session Indicators
    Extract Hour = hour value from timestamp index
    Extract Minute = minute value from timestamp index
    IF Hour is between 13 and 17 (GMT London/NY overlap):
        Set Is_Session_Overlap = 1
    ELSE:
        Set Is_Session_Overlap = 0
        
    Calculate atr_3 = 3-period rolling average of candle_range
    Calculate atr_30 = 30-period rolling average of candle_range
    Calculate Relative_Volatility = atr_3 / (atr_30 + 0.00000001)
    
    REMOVE rows containing initialization NaN values
    RETURN completed feature-engineered master_df
END FUNCTION
```

---

## 3. High-Friction Forward Target Labeling Engine

Targets are calculated by looking forward into the dataset to match contract expiries precisely, embedding a strict execution buffer to simulate the broker's bid/ask spread and trade execution slippage.

```python
FUNCTION generate_forward_target_labels(master_df, slippage_pips):
    // 1-Minute Future Window (Continuation)
    FOR EACH row at index t:
        Find future_close_1m = close price at index (t + 1 minute)
        IF future_close_1m > (close price at t + slippage_pips):
            Set Target_Continuation_1M = 1 (Win / Up Case)
        ELSE:
            Set Target_Continuation_1M = 0 (Loss / Down Case)
            
    // 3-Minute Future Window (Mean Reversion)
    FOR EACH row at index t:
        Find future_close_3m = close price at index (t + 3 minutes)
        IF future_close_3m < (close price at t - slippage_pips):
            Set Target_Reversion_3M = 1 (Win / Down Case)
        ELSE:
            Set Target_Reversion_3M = 0 (Loss / Up Case)
            
    RETURN updated master_df
END FUNCTION
```

---

## 4. Hour-Based Walk-Forward Rolling Window Training Loop

To balance performance against operational overhead, models are retrained using a periodic walk-forward strategy. Every hour, the system uses the past 4 hours of history to fit shallow models capable of tracking local microstructural variations.

```python
FUNCTION execute_hourly_walk_forward_pipeline(master_df):
    Define features_list = [all computed AMT, Momentum, Rejection, and Temporal features]
    Set lookback_window = 240 // 4 hours in minutes
    Set step_window = 60      // 1 hour in minutes
    
    FOR current_pointer from lookback_window to length(master_df) steps of step_window:
        Set train_start = current_pointer - lookback_window
        Set train_end = current_pointer
        
        Set test_start = current_pointer
        Set test_end = current_pointer + step_window
        
        Extract X_train = master_df[train_start:train_end][features_list]
        Extract y_train_cont = master_df[train_start:train_end]['Target_Continuation_1M']
        Extract y_train_rev = master_df[train_start:train_end]['Target_Reversion_3M']
        
        Extract X_test = master_df[test_start:test_end][features_list]
        
        // Model A: Initialize shallow configurations to prevent overfitting
        Initialize continuation_model = XGBClassifier(
            n_estimators=50, max_depth=3, learning_rate=0.05, eval_metric='logloss'
        )
        Fit continuation_model using (X_train, y_train_cont)
        
        // Model B: Initialize Mean Reversion Engine
        Initialize reversion_model = XGBClassifier(
            n_estimators=50, max_depth=3, learning_rate=0.05, eval_metric='logloss'
        )
        Fit reversion_model using (X_train, y_train_rev)
        
        // Export artifacts for the upcoming production trading hour
        SERIALIZE continuation_model to disk as "models/continuation_hour_H.json"
        SERIALIZE reversion_model to disk as "models/reversion_hour_H.json"
    END FOR
END FUNCTION
```

---

## 5. Production Inference & The 80% Confidence Filter

During live trading, standard binary classifications are discarded. Instead, the model outputs raw prediction probabilities, triggering an order only when the statistical confidence matches or exceeds the 80% threshold needed to outrun broker payout math.

```python
FUNCTION evaluate_live_market_signal(live_1min_row, active_hour_model):
    Convert live_1min_row into formatted 2D feature vector matching training columns
    
    // Check global scale volatility filter
    IF live_1min_row['Relative_Volatility'] < 0.5:
        RETURN "🛑 CIRCUIT BREAKER: Market conditions lack transactional structure. Do not trade."
        
    // Generate prediction probabilities [Class 0 (Down/Put), Class 1 (Up/Call)]
    Calculate probabilities = active_hour_model.predict_proba(live_1min_row)
    Calculate confidence_down = probabilities[0] * 100
    Calculate confidence_up = probabilities[1] * 100
    
    // Apply Strict 80% Probability Filter
    IF confidence_up >= 80.0:
        RETURN "🔥 SIGNAL ENFORCED: Execute CALL entry. Confidence: " + confidence_up + "%"
    ELSE IF confidence_down >= 80.0:
        RETURN "🔥 SIGNAL ENFORCED: Execute PUT entry. Confidence: " + confidence_down + "%"
    ELSE:
        RETURN "🛑 STAND DOWN: Statistical matrix edge insufficient. Verification metrics below 80% threshold."
END FUNCTION
```

---

## 6. Continuous Feedback, Optimization, & Storage Rules

```python
FUNCTION execute_system_optimization_and_storage(master_df, model_registry):
    // 1. Serialization Constraints
    Save master_df containing nested tick data structures to disk using PyArrow engine:
    master_df.to_parquet("data/processed/EURUSD_master.parquet", engine="pyarrow", compression="snappy")
    
    // 2. Feedback Logging Architecture
    FOR EACH trade executed live:
        Log [Timestamp, Feature_Vector, Model_Confidence, Trade_Outcome (Win=1/Loss=0)]
        Append to file: "data/logs/production_performance.csv"
        
    // 3. Periodic Optimization & Feature Compounding Rules
    Every weekend, run an Unsupervised Engine (Hidden Markov Model or K-Means) on production logs:
        Cluster performance data by 'Hour_Of_Day' and 'Relative_Volatility'
        IDENTIFY "Toxic Regimes" where precision dropped below 56%
        UPDATE the system blacklist parameters to automatically switch models to inactive states during those hours
        
    IF systemic drift is detected across all hours:
        Increase training lookback window from 4 hours to 8 hours
        Introduce new microstructural combinations into 'feature_factory.py' rather than increasing tree depth
END FUNCTION
```
