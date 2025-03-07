import ccxt
import time
import requests

# Khởi tạo kết nối với Binance và MEXC
binance = ccxt.binance()
mexc = ccxt.mexc()

# Cấu hình Telegram
TELEGRAM_BOT_TOKEN = "7302527509:AAGhHvr4C4jOb55t8GR3uiDHyrIs6qksgm4"
TELEGRAM_CHAT_ID = "7920057292"

# Ngưỡng lợi nhuận tối thiểu để thực hiện Arbitrage (%)
THRESHOLD = 0.5

# Khởi tạo API Binance & MEXC với quyền giao dịch
binance_api = ccxt.binance({
    'apiKey': 'YOUR_BINANCE_API_KEY',
    'secret': 'YOUR_BINANCE_SECRET_KEY'
})

mexc_api = ccxt.mexc({
    'apiKey': 'YOUR_MEXC_API_KEY',
    'secret': 'YOUR_MEXC_SECRET_KEY'
})

# Biến kiểm tra thời gian gửi tin nhắn Telegram
last_telegram_time = time.time()


def send_telegram_message(message):
    """Gửi thông báo qua Telegram"""
    url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {"chat_id": TELEGRAM_CHAT_ID, "text": message, "parse_mode": "Markdown"}
    requests.post(url, json=payload)


def get_prices():
    """Lấy giá BTC/USDT từ Binance và MEXC"""
    binance_price = binance.fetch_ticker('BTC/USDT')['last']
    mexc_price = mexc.fetch_ticker('BTC/USDT')['last']
    return binance_price, mexc_price


def check_arbitrage():
    """Kiểm tra cơ hội Arbitrage và gửi cảnh báo qua Telegram"""
    global last_telegram_time

    binance_price, mexc_price = get_prices()
    spread = ((mexc_price - binance_price) / binance_price) * 100

    print(f"Binance: {binance_price} | MEXC: {mexc_price} | Spread: {spread:.2f}%")

    current_time = time.time()

    if spread > THRESHOLD:
        message = (f"🚀 *Arbitrage Opportunity!*\n"
                   f"• *Buy Binance* ${binance_price:.2f}\n"
                   f"• *Sell MEXC* ${mexc_price:.2f}\n"
                   f"• *Spread:* {spread:.2f}%\n"
                   f"🔄 *Executing trade...*")
        send_telegram_message(message)
        place_orders(0.01)

    elif spread < -THRESHOLD:
        message = (f"🔥 *Arbitrage Opportunity!*\n"
                   f"• *Buy MEXC* ${mexc_price:.2f}\n"
                   f"• *Sell Binance* ${binance_price:.2f}\n"
                   f"• *Spread:* {spread:.2f}%\n"
                   f"🔄 *Executing trade...*")
        send_telegram_message(message)
        place_orders(0.01)

    # Gửi cập nhật Telegram mỗi 5 phút (300 giây), bất kể có Arbitrage hay không
    if current_time - last_telegram_time >= 100:
        message = (f"📊 *Market Update*\n"
                   f"• Binance: ${binance_price:.2f}\n"
                   f"• MEXC: ${mexc_price:.2f}\n"
                   f"• Spread: {spread:.2f}%\n"
                   f"🔎 *Checking for arbitrage opportunities...*")
        send_telegram_message(message)
        last_telegram_time = current_time


def place_orders(amount):
    """Đặt lệnh mua/bán tự động trên sàn Arbitrage"""
    binance_price, mexc_price = get_prices()

    if binance_price < mexc_price:
        print("🟢 Mua trên Binance, Bán trên MEXC!")
        binance_api.create_market_buy_order('BTC/USDT', amount)
        mexc_api.create_market_sell_order('BTC/USDT', amount)
        send_telegram_message("✅ *Trade Executed!* 🟢 Mua trên Binance, Bán trên MEXC!")

    elif binance_price > mexc_price:
        print("🔴 Mua trên MEXC, Bán trên Binance!")
        mexc_api.create_market_buy_order('BTC/USDT', amount)
        binance_api.create_market_sell_order('BTC/USDT', amount)
        send_telegram_message("✅ *Trade Executed!* 🔴 Mua trên MEXC, Bán trên Binance!")

    else:
        send_telegram_message("⚡ No profitable trade available.")


# Chạy bot kiểm tra giá và thực hiện Arbitrage nếu có cơ hội
while True:
    check_arbitrage()
    time.sleep(0.5)  # Kiểm tra giá mỗi 0.5 giây
