import MetaTrader5 as mt5
import pandas as pd
from datetime import datetime, timedelta
import requests
from telegram import Bot
import asyncio



TELEGRAM_TOKEN = "7577745027:AAFK_Wj0BfAoS0CQocMvOGA5FE6IAyM3lFM"
CHAT_ID = 5573261363
message = "Hello Châu Ngọc Trường!"

url = f"https://api.telegram.org/bot{TELEGRAM_TOKEN}/sendMessage?chat_id={CHAT_ID}&text={message}"
response = requests.get(url)
print(response.json())

mt5_path = "C:\\Program Files\\MetaTrader 5 -2\\terminal64.exe"

if not mt5.initialize(mt5_path):
    print(f"Error initializing MT5: {mt5.last_error()}")
    quit()
    

account = 189788099
password = "Truong31072003@"
server = "Exness-MT5Trial14"

if mt5.login(account, password, server):
    print(f"Successfully logged into account {account}")
else:
    print(f"Login failed: {mt5.last_error()}")

symbol = "XAUUSD"
lot = 0.01

bot = Bot(token=TELEGRAM_TOKEN)

    
async def send_telegram_message(message):
    """Send a message via Telegram."""
    try:
        await bot.send_message(chat_id=CHAT_ID, text=message)
    except Exception as e:
        print(f"Unable to send message via Telegram: {e}")
        

async def log_message(message):
    """Log and send message via Telegram."""
    timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
    full_message = f"{timestamp} - {message}"
    print(full_message)
    await send_telegram_message(full_message)


def is_trading_time():
    """Check trading hours (Always True for 24/24 operation)."""
    return True


def get_latest_candles(symbol, timeframe, n=1):
    """Retrieve the latest n candles."""
    rates = mt5.copy_rates_from_pos(symbol, timeframe, 0, n)
    if rates is None or len(rates) < n:
        print("Not enough candle data available.")
        return None

    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    df.set_index('time', inplace=True) 
    return df 




def calculate_ema(data, period):
    if data is None or len(data) < period:
        print(f"Not enough data to calculate EMA {period}.")
        return None
    return data['close'].ewm(span=period, adjust=False).mean()



def detect_smc():
    """Detect Break of Structure (BOS) and Order Block."""
    candles = get_latest_candles(symbol, mt5.TIMEFRAME_M5, 10)
    if candles is None:
        print("No candle data.")
        return None

    highs = candles['high']
    lows = candles['low']
    
    bos = None
    ob = None
    if highs.iloc[-2] > highs.iloc[-3]:
        print("Bullish BOS detected.")
        bos = 'buy'
    elif lows.iloc[-2] < lows.iloc[-3]:
        print("Bearish BOS detected.")
        bos = 'sell'
    else:
        print("No BOS detected.")

    if candles.iloc[-2]['close'] < candles.iloc[-2]['open']:
        print("Sell Order Block detected.")
        ob = 'sell'
    elif candles.iloc[-2]['close'] > candles.iloc[-2]['open']:
        print("Buy Order Block detected.")
        ob = 'buy'
    else:
        print("No Order Block detected.")

    return {
        "BOS": bos,
        "Order Block": ob
    }


def detect_trend():
    """Determine trend based on EMA 34 and EMA 89."""
    candles = get_latest_candles(symbol, mt5.TIMEFRAME_M1, 100)
    if candles is None:
        print("No candle data available.")
        return None

    candles['EMA_34'] = calculate_ema(candles, 34)
    candles['EMA_89'] = calculate_ema(candles, 89)

    if candles['EMA_34'].iloc[-1] > candles['EMA_89'].iloc[-1]:
        print("Trend: Uptrend.")
        return 'uptrend'
    elif candles['EMA_34'].iloc[-1] < candles['EMA_89'].iloc[-1]:
        print("Trend: Downtrend.")
        return 'downtrend'
    else:
        print("No clear trend detected.")
        return None

def wait_for_m5_close():
    now = datetime.now()
    next_close = (now + timedelta(minutes=5 - now.minute % 5)).replace(second=0, microsecond=0)
    wait_time = (next_close - now).total_seconds() + 1
    print(f"Waiting {wait_time} seconds until the M5 candle closes.")
    return wait_time

def get_open_orders_count():
    """Retrieve the count of open orders."""
    positions = mt5.positions_get()
    orders = mt5.history_orders_get(datetime.now() - timedelta(hours=6), datetime.now())

    if positions is None:
        print("No positions found.")
        return 0

    total_orders = len(positions) 
    if orders:
        total_orders += sum(1 for order in orders if order.symbol == symbol and order.type in 
                            (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL) and order.reason in 
                            (mt5.ORDER_REASON_SL, mt5.ORDER_REASON_TP))

    print(f"Total orders (open + hit SL/TP): {total_orders}")
    return total_orders



def get_open_position_price(order_type):
    """Get the open price of the current buy or sell order, if available."""
    positions = mt5.positions_get(symbol=symbol)
    
    if positions is None or len(positions) == 0:
        print(f"No open {order_type} orders.")
        return None  

    for pos in positions:
        if (order_type == 'buy' and pos.type == mt5.ORDER_TYPE_BUY) or \
           (order_type == 'sell' and pos.type == mt5.ORDER_TYPE_SELL):
            print(f"Open price of the current {order_type} order: {pos.price_open}")
            return pos.price_open

    print(f"No current {order_type} orders.")
    return None

def should_execute_order(order_type, current_price, open_price):
    """Check if a buy or sell order should be executed."""
    if open_price is None:
        print("No previous open order to compare.")
        return True 
    
    if order_type == 'buy':
        if current_price >= open_price:
            print(f"Current price ({current_price}) is greater than or equal to the open price ({open_price}), not executing buy order.")
            return False
        else:
            print(f"Current price ({current_price}) is less than the open price ({open_price}), executing buy order.")
            return True 

    elif order_type == 'sell':
        if current_price <= open_price:
            print(f"Current price ({current_price}) is less than or equal to the open price ({open_price}), not executing sell order.")
            return False 
        else:
            print(f"Current price ({current_price}) is greater than the open price ({open_price}), executing sell order.")
            return True 

    print("Invalid order type.")
    return False  


def execute_trade(order_type, lot):
    """Place a buy or sell order based on signals."""
    price = mt5.symbol_info_tick(symbol).ask if order_type == 'buy' else mt5.symbol_info_tick(symbol).bid
    point = mt5.symbol_info(symbol).point
    sl = price - 5000 * point if order_type == 'buy' else price + 5000 * point
    tp = price + 5000 * point if order_type == 'buy' else price - 5000 * point

    if abs(price - sl) < 100 * point or abs(price - tp) < 100 * point:
        error_msg = "Error: SL/TP distance is insufficient."
        print(error_msg)
        asyncio.run(send_telegram_message(error_msg))
        return

    order = {
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": symbol,
        "volume": lot,
        "type": mt5.ORDER_TYPE_BUY if order_type == 'buy' else mt5.ORDER_TYPE_SELL,
        "price": price,
        "sl": sl,
        "tp": tp,
        "deviation": 1000,
        "magic": 1,
        "comment": "SMC Trade",
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_FOK,
    }

    print(f"Sending order: {order_type}, Lot: {lot}, SL: {sl}, TP: {tp}, Price: {price}")

    result = mt5.order_send(order)

    if result.retcode == mt5.TRADE_RETCODE_DONE:
        print(f"{order_type.capitalize()} order successful: {result}")
    else:
        error_msg = f"Order failed: {result.comment}, Error Code: {result.retcode}"
        print(error_msg)
        asyncio.create_task(send_telegram_message(error_msg))

    return result


async def run_bot():
    """Run the bot to detect SMC signals and execute trades if signals are present."""
    if not mt5.symbol_select(symbol, True):
        await log_message(f"Unable to select symbol {symbol}.")
        mt5.shutdown()
        return None
    
    while is_trading_time():
        open_orders_count = get_open_orders_count() 
        if open_orders_count >= 10:
         await log_message("The transaction has exceeded the allowable limit.")
        

        
        
        wait_time = wait_for_m5_close()
        await asyncio.sleep(wait_time)
        trend = detect_trend()
        smc_signals = detect_smc()
         
        if smc_signals is None:
            await log_message(f"Number of open orders: {open_orders_count}")
            await log_message("No SMC signal detected.")
            await log_message("Determining trend.")
            continue

        try:
            current_price = mt5.symbol_info_tick(symbol).ask 
            buy_open_price = get_open_position_price('buy')  
            sell_open_price = get_open_position_price('sell') 

            if smc_signals['BOS'] == 'buy' and smc_signals['Order Block'] == 'buy' and trend == 'uptrend':
                if buy_open_price is None or current_price < buy_open_price:
                    await log_message("Executing buy order...")
                    execute_trade('buy', lot)
                    await log_message("Buy signal from SMC and uptrend, executed buy order.")
                else:
                    await log_message(f"Current price {current_price} >= Buy open price {buy_open_price}, not executing buy order.")

            elif smc_signals['BOS'] == 'sell' and smc_signals['Order Block'] == 'sell' and trend == 'downtrend':
                if sell_open_price is None or current_price > sell_open_price:
                    await log_message("Executing sell order...")
                    execute_trade('sell', lot)
                    await log_message("Sell signal from SMC and downtrend, executed sell order.")
                else:
                    await log_message(f"Current price {current_price} <= Sell open price {sell_open_price}, not executing sell order.")

        except Exception as e:
            await log_message(f"Error executing trade: {e}")
        message = "\n===== SMC Signals =====\n"
        message += f"BOS: {smc_signals['BOS']}\n"
        message += f"Order Block: {smc_signals['Order Block']}\n"
        message += f"EMA Trend: {trend}\n"
        message += f"Current Price: {current_price}\n"
        message += f"Trade Volume: {lot}\n"
        message += f"Number of open orders: {open_orders_count}\n"
        await log_message(message)

if __name__ == "__main__":
    asyncio.run(run_bot())
    

- 👋 Hi, I’m @chaungoctruong
- 👀 I’m interested in ...
- 🌱 I’m currently learning ...
- 💞️ I’m looking to collaborate on ...
- 📫 How to reach me ...
- 😄 Pronouns: ...
- ⚡ Fun fact: ...

<!---
chaungoctruong/chaungoctruong is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
