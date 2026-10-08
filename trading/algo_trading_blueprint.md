# Quantitative Trading Architecture Blueprint
## Structural Architecture for Intraday AMT & Momentum Trading

This architecture document acts as the definitive design blueprint for a retail quantitative execution engine. It integrates **Auction Market Theory (AMT)** for baseline structural profiling, **Microstructure Momentum** for execution triggers, and an automated **Temporal Decay Logic** to govern programmatic trade horizons (ranging from 5-minute stagnation cuts to an absolute 8-hour safety firewall).

---

## 1. System Topology & Global Data Pipelines

To achieve absolute control over execution metrics and avoid platform vendor lock-in, the system follows a modular pipeline decoupled across four explicit software boundaries.

```
[ Raw Market Data Ingestion Feed (WebSockets) ]
                       │
                       ▼
         [ State Monitor Subroutine ] <─── Calculates Global Regime Gauges (Hurst, VAW)
                       │
                       ▼
       [ Algorithmic Signal Engine ]  <─── Evaluates Inefficiencies (OBI, ML-OFI, LVN)
                       │
                       ▼
        [ Automated Risk Management Gate ] <── Enforces Safety Limits, Position Sizing, Time-Stops
                       │
                       ▼
       [ Execution API Router (JSON API) ] ───> Live Broker Order Routing
```

### Module 1.1: Raw Data Ingestion Engine
*   **Operational Protocol:** Secure WebSocket connection (`wss://`) established directly with primary venue data relays.
*   **Ingestion Granularity:** Ingests raw aggregate transaction packets (Level 1 Tick updates including transaction price, trade size, side, inside bid, inside ask, bid depth, and ask depth).
*   **Memory Matrix Configuration:** Streaming events are piped asynchronously into a thread-safe, rolling structure. To prevent system out-of-memory crashes, arrays enforce a static length queue pattern (`N = 10,000`). When new ticks append, index 0 is programmatically dropped via FIFO shifts.

---

## 2. Quantitative Regime Detection Master Switch

Before any tactical entry rules can fire, the system computes macro market states. This prevents structural mismatch errors (such as deploying high-speed breakout scripts into institutional absorption nodes).

### Algorithm 2.1: Statistical Dispersion Model (Rolling Hurst Exponent)
The script calculates the scaling property of the range of cumulative deviations of a time-series to dynamically isolate directional persistence from rotational mean reversion.

```python
FUNCTION Calculate_Rolling_Hurst_Exponent(Price_Queue, Window_Size):
    # Establish lookback window matrices
    Set Log_Returns = Array_Of_Size(Window_Size - 1)
    
    FOR i FROM 1 TO Window_Size - 1:
        Log_Returns[i-1] = ln(Price_Queue[i] / Price_Queue[i-1])
    
    # Divide the series into sub-periods and calculate rescaled range
    Set Mean_Return = Average(Log_Returns)
    Set S_Dev = Standard_Deviation(Log_Returns)
    
    Set Cumulative_Deviations = Cumulative_Sum(Log_Returns - Mean_Return)
    Set R_Range = Max(Cumulative_Deviations) - Min(Cumulative_Deviations)
    
    # Calculate Hurst Approximation
    Set Hurst_Exponent = ln(R_Range / S_Dev) / ln(Window_Size)
    
    RETURN Hurst_Exponent
END FUNCTION
```

### Algorithm 2.2: Profile Compression Switch (Value Area Width Z-Score)
*   **Calculation:** Track the price boundaries containing exactly 70% of the session's cumulative traded volume to construct the Value Area Width (VAW).
*   **Mathematical Switching Gate Logic:**
    ```python
    Set Rolling_VAW_ZScore = (Current_VAW - Moving_Average(VAW, 20)) / StdDev(VAW, 20)
    
    IF Rolling_VAW_ZScore < 0.0 AND Hurst_Exponent < 0.45:
        Set GLOBAL_MARKET_REGIME = "ROTATIONAL_BALANCE"
        # Activate System 3.1: Micro-Imbalance Absorption Reversals
        # Deactivate all Breakout Engines
        
    ELSE IF Rolling_VAW_ZScore > 1.5 AND Hurst_Exponent > 0.55:
        Set GLOBAL_MARKET_REGIME = "DIRECTIONAL_IMBALANCE"
        # Activate System 3.2: High-Velocity LVN Vacuum Continuations
        # Deactivate all Mean Reversion Engines
    ELSE:
        Set GLOBAL_MARKET_REGIME = "RANDOM_NOISE"
        # Restrict system entry to preserve capital engine
    ENDIF
    ```

---

## 3. Core Strategy Implementations

### System 3.1: Micro-Imbalance Absorption Reversal (Rotational Balance Setup)
*   **Operational Objective:** Capture rapid re-auctions back to the Point of Control (POC) when aggressive retail breakout traders are absorbed by institutional limit order barriers at the Value Area High (VAH) or Value Area Low (VAL).
*   **Microstructure Fingerprint:** High transaction volume per tick concurrent with near-zero price velocity (price stalling out under massive volume).

```python
FUNCTION Evaluate_Absorption_Fade_Strategy(Top_Of_Book_Stream, Global_Regime):
    IF Global_Regime IS NOT "ROTATIONAL_BALANCE":
        RETURN NO_SIGNAL
    ENDIF

    Set Current_Price = Top_Of_Book_Stream.Last_Traded_Price
    Set Volume_Per_Tick = Top_Of_Book_Stream.Last_Tick_Volume
    Set Price_Velocity = Absolute(Current_Price - Top_Of_Book_Stream.Price_5_Ticks_Ago)

    # Detect institutional absorption boundaries
    IF Current_Price >= Market_Profile.Value_Area_High:
        IF Volume_Per_Tick > (Moving_Average(Volume_Per_Tick, 100) * 4) AND Price_Velocity < Tick_Size:
            # Trigger Signal: Passive institutional distribution wall detected
            Set Trading_Signal.Direction = "SHORT"
            Set Trading_Signal.Target_Price = Market_Profile.Point_Of_Control
            Set Trading_Signal.Stop_Loss = Current_Price + (ATR_14 * 0.5)
            Set Trading_Signal.Time_To_Live_Bars = 24 # 2-Hour expectation on 5m framework
            
            RETURN Trading_Signal
        ENDIF
    ENDIF
    
    RETURN NO_SIGNAL
END FUNCTION
```

### System 3.2: Cont-Stoikov Order Flow Imbalance Breakout (Directional Setup)
*   **Operational Objective:** Exploit structural queue clearing events where passive market ask layers evaporate at the Value Area High, paving the way for a rapid momentum lift.

```python
FUNCTION Calculate_Cont_Stoikov_OFI(Level_1_Data, Previous_Level_1_Data):
    Set Delta_Bid_Size = 0
    Set Delta_Ask_Size = 0
    
    # Evaluate Bid Queue Shift
    IF Level_1_Data.Bid_Price > Previous_Level_1_Data.Bid_Price:
        Delta_Bid_Size = Level_1_Data.Bid_Size
    ELSE IF Level_1_Data.Bid_Price == Previous_Level_1_Data.Bid_Price:
        Delta_Bid_Size = Level_1_Data.Bid_Size - Previous_Level_1_Data.Bid_Size
    ELSE:
        Delta_Bid_Size = 0
    ENDIF

    # Evaluate Ask Queue Shift
    IF Level_1_Data.Ask_Price < Previous_Level_1_Data.Ask_Price:
        Delta_Ask_Size = Level_1_Data.Ask_Size
    ELSE IF Level_1_Data.Ask_Price == Previous_Level_1_Data.Ask_Price:
        Delta_Ask_Size = Level_1_Data.Ask_Size - Previous_Level_1_Data.Ask_Size
    ELSE:
        Delta_Ask_Size = 0
    ENDIF

    Set OFI_Value = Delta_Bid_Size - Delta_Ask_Size
    RETURN OFI_Value
END FUNCTION
```

---

## 4. Automated Risk Management & Execution Gate

Every position passed to the broker routing matrix must be screened through a rigid, automated transaction gate that protects small capital accounts from liquidation traps.

### Module 4.1: Position Sizing & Fractional Capital Control
*   **The 1% Micro-Account Rule:** Total capital exposure per trade must be restricted to exactly 1% of total account value to tolerate long statistical drawdown series.
*   **Automated Slippage Penalty Adjustments:** The execution routine assumes a fixed 1-tick spread slip penalty inside the entry calculations to verify profitability under realistic execution friction.

```python
FUNCTION Calculate_Precise_Position_Size(Account_Equity, Entry_Price, Stop_Price, Asset_Type):
    Set Allowed_Risk_Capital = Account_Equity * 0.01  # Hard 1% Capital Boundary
    Set Price_Risk_Delta = Absolute(Entry_Price - Stop_Price)
    
    # Factor in execution spread friction
    Set Corrected_Risk_Delta = Price_Risk_Delta + Market_Microstructure.Inside_Spread
    
    Set Raw_Position_Size = Allowed_Risk_Capital / Corrected_Risk_Delta
    
    IF Asset_Type IS "EQUITY_FRACTIONAL" OR Asset_Type IS "CRYPTO":
        RETURN Raw_Position_Size # Utilize micro fractional routing execution
    ELSE:
        RETURN Floor(Raw_Position_Size) # Enforce lot rounding constraints
    ENDIF
END FUNCTION
```

### Module 4.2: Temporal Decay & Dynamic Time-Stops
Exits are evaluated non-linearly. Time is integrated into risk management as a distinct parameter of statistical edge preservation.

```python
FUNCTION Evaluate_Temporal_Decay_Exits(Open_Trade_Object, Current_Data_Stream):
    Set Ticks_Elapsed = Current_Data_Stream.Current_Tick_Index - Open_Trade_Object.Entry_Tick_Index
    Set Time_Elapsed_Minutes = Current_Data_Stream.Current_Timestamp - Open_Trade_Object.Entry_Timestamp
    
    Set Price_Delta = Current_Data_Stream.Last_Traded_Price - Open_Trade_Object.Entry_Price
    Set Current_ATR = Current_Data_Stream.ATR_14
    
    # Boundary 1: The 15-Minute Stagnation Cut (Dead Money Protection)
    IF Time_Elapsed_Minutes >= 15 AND Absolute(Price_Delta) < (Current_ATR * 0.15):
        RETURN Action_Close_Position(Reason="Stagnation: Zero Momentum Velocity Detected")
    ENDIF
    
    # Boundary 2: Volatility-Velocity Decoupling Check (Adverse Selection Avoidance)
    IF Open_Trade_Object.Direction == "LONG" AND Price_Delta > 0:
        IF Current_Data_Stream.Rolling_ROC < 0.1 AND Current_Data_Stream.Rolling_ATR_Velocity > 1.5:
            # Volatility is expanding but price progression has stalled out (Absorption Flag)
            RETURN Action_Close_Position(Reason="Decoupling: High Entropy Distribution Detected")
        ENDIF
    ENDIF

    # Boundary 3: Absolute Safety Time Horizon Firewall
    IF Time_Elapsed_Minutes >= 480: # Hard 8-Hour Threshold Enforced
        RETURN Action_Close_Position(Reason="Firewall: Maximum Intraday Holding Horizon Reached")
    ENDIF

    RETURN Action_Hold_Position
END FUNCTION
```

---

## 5. System Execution Verification Logs

To verify infrastructure consistency during daily system post-mortems, the tracking database writes execution data matching the standard template below:

```json
{
  "Timestamp": "2026-10-08T14:41:00Z",
  "Strategy_ID": "MOM_OFI_BREAKOUT_V1",
  "Execution_State": "ORDER_FIRED",
  "Metrics": {
    "Calculated_Hurst": 0.68,
    "VAW_ZScore": 1.85,
    "Target_Horizon_Minutes": 45,
    "Calculated_Slippage_Ticks": 1.2
  },
  "Risk_Parameters": {
    "Capital_Allocated_USD": 1.00,
    "Fractional_Size": 0.0145,
    "Hard_Stop_Price": 99.15,
    "Time_Stop_Horizon_Timestamp": "2026-10-08T22:41:00Z"
  }
}
```