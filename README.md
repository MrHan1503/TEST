# TEST
import ccxt
import pandas as pd
import requests
import time

# Thông tin Telegram Bot
TELEGRAM_TOKEN = "7693259463:AAGhl0ar6v6K5Y7EBS-7fLmSrlrQhfMOpMI"
CHAT_ID = "5392172124"

# Hàm gửi tin nhắn Telegram
def send_telegram_message(message):
    url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage"
    payload = {"chat_id": CHAT_ID, "text": message}
    requests.post(url, json=payload)

# Kết nối OKX
exchange = ccxt.okx()

# Hàm lấy dữ liệu nến
def get_candles(symbol, timeframe="1m", limit=50):
    ohlcv = exchange.fetch_ohlcv(symbol, timeframe, limit=limit)
    df = pd.DataFrame(ohlcv, columns=["timestamp", "open", "high", "low", "close", "volume"])
    return df

# Tính EMA
def calculate_ema(df, period):
    return df["close"].ewm(span=period, adjust=False).mean()

# Tính RSI
def calculate_rsi(df, period=14):
    delta = df["close"].diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    return 100 - (100 / (1 + rs))

# Kiểm tra tín hiệu mua/bán
def check_signal(df):
    df["ema_50"] = calculate_ema(df, 50)
    df["ema_200"] = calculate_ema(df, 200)
    df["rsi"] = calculate_rsi(df)

    last_row = df.iloc[-1]
    ema_50, ema_200, rsi = last_row["ema_50"], last_row["ema_200"], last_row["rsi"]

    if ema_50 > ema_200 and rsi < 30:
        return "🔵 BUY SIGNAL: Giá có thể tăng!"
    elif ema_50 < ema_200 and rsi > 70:
        return "🔴 SELL SIGNAL: Giá có thể giảm!"
    return None

# Chạy bot liên tục
while True:
    send_telegram_message("✅ Bot vẫn đang hoạt động...")
    try:
        data = get_candles("BTC/USDT", "1m", 100)
        signal = check_signal(data)

        if signal:
            send_telegram_message(signal)

        time.sleep(60)  # Kiểm tra mỗi phút

    except Exception as e:
        send_telegram_message(f"Lỗi bot: {e}")
        time.sleep(60)
