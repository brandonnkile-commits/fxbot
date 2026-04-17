import MetaTrader5 as mt5
import pandas as pd
import numpy as np
import time
from datetime import datetime

# =========================
# SETTINGS
# =========================
SYMBOL = "EURUSD"
LOT = 0.1
TIMEFRAME = mt5.TIMEFRAME_M5
RISK_PIPS = 20
REWARD_PIPS = 40
MAGIC = 777

# =========================
# CONNECT TO MT5
# =========================
if not mt5.initialize():
    print("MT5 failed to initialize")
    quit()

print("Connected to MT5")

# =========================
# GET DATA
# =========================
def get_data():
    rates = mt5.copy_rates_from_pos(SYMBOL, TIMEFRAME, 0, 300)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    return df

# =========================
# INDICATORS (SCALPING)
# =========================
def apply_indicators(df):
    df['ema20'] = df['close'].ewm(span=20).mean()
    df['ema50'] = df['close'].ewm(span=50).mean()
    df['momentum'] = df['close'] - df['close'].shift(5)
    return df

# =========================
# ICT + CRT STRATEGY LOGIC
# =========================
def analyze_market(df):
    last = df.iloc[-1]
    prev = df.iloc[-2]

    # Trend (ICT concept)
    bullish = last['ema20'] > last['ema50']
    bearish = last['ema20'] < last['ema50']

    # Liquidity sweep (ICT idea)
    sweep_high = last['high'] > prev['high'] and last['close'] < prev['high']
    sweep_low = last['low'] < prev['low'] and last['close'] > prev['low']

    # CRT reversal logic
    bullish_reversal = sweep_low and last['close'] > last['open']
    bearish_reversal = sweep_high and last['close'] < last['open']

    # Scalping momentum
    momentum_up = last['momentum'] > 0
    momentum_down = last['momentum'] < 0

    # FINAL SIGNALS
    if bullish and bullish_reversal and momentum_up:
        return "BUY"

    if bearish and bearish_reversal and momentum_down:
        return "SELL"

    return None

# =========================
# CHECK OPEN POSITIONS
# =========================
def has_open_trade():
    positions = mt5.positions_get(symbol=SYMBOL)
    return positions is not None and len(positions) > 0

# =========================
# EXECUTE TRADE
# =========================
def place_trade(signal):
    tick = mt5.symbol_info_tick(SYMBOL)

    if signal == "BUY":
        price = tick.ask
        sl = price - (RISK_PIPS * 0.0001)
        tp = price + (REWARD_PIPS * 0.0001)
        order_type = mt5.ORDER_TYPE_BUY

    else:
        price = tick.bid
        sl = price + (RISK_PIPS * 0.0001)
        tp = price - (REWARD_PIPS * 0.0001)
        order_type = mt5.ORDER_TYPE_SELL

    request = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": SYMBOL,
        "volume": LOT,
        "type": order_type,
        "price": price,
        "sl": sl,
        "tp": tp,
        "deviation": 20,
        "magic": MAGIC,
        "comment": "ICT-CRT Scalping Bot",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
    }

    result = mt5.order_send(request)
    print(f"{datetime.now()} → {signal} placed:", result)

# =========================
# MAIN LOOP
# =========================
while True:
    try:
        df = get_data()
        df = apply_indicators(df)

        signal = analyze_market(df)

        if signal and not has_open_trade():
            print(f"{datetime.now()} Signal: {signal}")
            place_trade(signal)

        time.sleep(60)

    except Exception as e:
        print("Error:", e)
        time.sleep(10)
