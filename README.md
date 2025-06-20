# Adım 1: Bot İskeleti (Temel Yapı ve Coin Taraması)

import ccxt
import time
import logging
from datetime import datetime

# === LOG AYARI ===
logging.basicConfig(level=logging.INFO, format='%(asctime)s - %(levelname)s - %(message)s')

# === BYBIT API AYARLARI ===
BYBIT_API_KEY = 'YOUR_API_KEY'
BYBIT_API_SECRET = 'YOUR_API_SECRET'

exchange = ccxt.bybit({
    'apiKey': BYBIT_API_KEY,
    'secret': BYBIT_API_SECRET,
    'enableRateLimit': True,
    'options': {
        'defaultType': 'future'
    }
})

# === TAKİP EDİLECEK COİNLER ===
SYMBOLS = ['BTC/USDT', 'ETH/USDT', 'SOL/USDT']

# === SİNYAL KONTROLÜ ===
def get_signal(symbol):
    # Buraya indikatör analiz kodları entegre edilecek (TradingView sinyalleri veya Pine analizleri)
    logging.info(f"{symbol} için sinyal kontrol ediliyor...")
    
    # ÖRNEK: false sinyal (placeholder)
    signal = {
        'direction': None,  # 'long' veya 'short'
        'entry': None,
        'sl': None,
        'tp': None
    }
    
    return signal

# === ANA ÇALIŞMA FONKSİYONU ===
def main():
    while True:
        for symbol in SYMBOLS:
            try:
                signal = get_signal(symbol)
                if signal['direction']:
                    logging.info(f"Sinyal bulundu: {symbol} yön: {signal['direction']}")
                    # Buraya işlem açma fonksiyonu gelecek
                else:
                    logging.info(f"{symbol} için uygun sinyal yok.")
            except Exception as e:
                logging.error(f"{symbol} işleminde hata: {str(e)}")
        
        time.sleep(10)  # Her 10 saniyede bir tekrar et

if __name__ == '__main__':
    main()
# Adım 2: Sinyale göre otomatik emir gönderme fonksiyonu

def place_order(symbol, side, entry_price, tp_price, sl_price, qty, leverage):
    try:
        # Leverage ayarla
        exchange.set_leverage(leverage, symbol)

        # Pozisyon kapatmak için trigger edilen SL ve TP emirlerini ayarla
        params = {
            "reduce_only": True,
            "position_idx": 1  # Long: 1, Short: 2
        }

        # Emir yönüne göre Bybit order type ayarla
        order_side = "buy" if side == "long" else "sell"
        opposite_side = "sell" if side == "long" else "buy"

        # Ana emri market olarak gir (pozisyona giriş)
        exchange.create_market_order(symbol, order_side, qty)

        # TP
        exchange.create_order(
            symbol=symbol,
            type="take_profit_market",
            side=opposite_side,
            amount=qty,
            params={
                "trigger_price": tp_price,
                "trigger_direction": 1 if side == "long" else 2,
                **params
            }
        )

        # SL
        exchange.create_order(
            symbol=symbol,
            type="stop_market",
            side=opposite_side,
            amount=qty,
            params={
                "trigger_price": sl_price,
                "trigger_direction": 2 if side == "long" else 1,
                **params
            }
        )

        print(f"\n\U0001F7E2 {symbol} için {side.upper()} işlemi BAŞLATILDI. TP: {tp_price}, SL: {sl_price}\n")

    except Exception as e:
        print(f"\n\U0001F534 {symbol} için emir gönderilirken hata: {e}\n")

# Adım 3: Sinyal karar mekanizması - get_signal()
# Örnek: RSI + SuperTrend + Trend Signal + ADX kombinasyonu

def get_signal(df):
    # Önce en son barı alalım
    close = df['close'].iloc[-1]
    rsi = df['rsi'].iloc[-1]
    supertrend_dir = df['supertrend_dir'].iloc[-1]   # 1 = Down, -1 = Up
    trend_signal = df['trend_signal'].iloc[-1]       # 1 = Buy, -1 = Sell
    adx = df['adx'].iloc[-1]

    # İşleme uygun minimum ADX seviyesi (trendin güçlü olduğu durum)
    min_adx = 20

    # Long Sinyali:
    if (
        rsi < 30 and
        supertrend_dir == -1 and
        trend_signal == 1 and
        adx > min_adx
    ):
        return {
            'signal': 'long',
            'entry': close,
            'tp': round(close * 1.015, 2),   # %1.5 TP
            'sl': round(close * 0.985, 2),   # %1.5 SL
            'risk': 0.15
        }

    # Short Sinyali:
    elif (
        rsi > 70 and
        supertrend_dir == 1 and
        trend_signal == -1 and
        adx > min_adx
    ):
        return {
            'signal': 'short',
            'entry': close,
            'tp': round(close * 0.985, 2),
            'sl': round(close * 1.015, 2),
            'risk': 0.15
        }

    # Sinyal yoksa:
    else:
        return None

# Adım 4: Ana izleme ve emir döngüsü
# 3 coin için sinyal tarar ve uygun sinyal bulursa otomatik emir gönderir
import time
import pandas as pd

# İşlem yapılacak coinler
SYMBOLS = ["BTC/USDT", "ETH/USDT", "SOL/USDT"]

# Parametreler
LEVERAGE = 5
RISK = 0.20
QTY_USD = 60  # Her pozisyon için sabit 60 USDT değerinde pozisyon

def fetch_ohlcv(symbol):
    # Örnek veri yapısı (gerçekte exchange.fetch_ohlcv() kullanılacaktır)
    # Simüle verileri döndürür
    return pd.DataFrame({
        "timestamp": [],
        "open": [],
        "high": [],
        "low": [],
        "close": [],
        "volume": [],
        "rsi": [],
        "supertrend_dir": [],
        "trend_signal": [],
        "adx": []
    })

while True:
    print("\n\U0001F50D Yeni sinyal taraması başladı...")

    for symbol in SYMBOLS:
        try:
            df = fetch_ohlcv(symbol)  # Gerçek veri ile değiştirilecek

            signal = get_signal(df)

            if signal:
                qty = round(QTY_USD / signal['entry'], 3)
                place_order(
                    symbol=symbol,
                    side=signal['signal'],
                    entry_price=signal['entry'],
                    tp_price=signal['tp'],
                    sl_price=signal['sl'],
                    qty=qty,
                    leverage=LEVERAGE
                )
            else:
                print(f"{symbol}: Uygun sinyal YOK")

        except Exception as e:
            print(f"{symbol} için hata: {e}")

    print("\u23F1 60 saniye bekleniyor...")
    time.sleep(60)

# Adım 5: Emir Gönderme Fonksiyonu
import ccxt

# Bybit API bilgileri
bybit = ccxt.bybit({
    "apiKey": "YOUR_API_KEY",
    "secret": "YOUR_API_SECRET",
    "enableRateLimit": True,
    "options": {
        "defaultType": "future",
        "adjustForTimeDifference": True
    }
})

def place_order(symbol, side, entry_price, tp_price, sl_price, qty, leverage):
    try:
        # 1. Kaldıraç ayarla
        market = bybit.market(symbol)
        bybit.set_leverage(leverage, symbol)

        # 2. Pozisyon yönüne göre emir tipi belirle
        side_str = "buy" if side == "long" else "sell"
        opposite_side = "sell" if side == "long" else "buy"

        print(f"\n🚀 {symbol} için {side.upper()} pozisyon açılıyor...")

        # 3. Piyasa emri ile gir
        order = bybit.create_order(
            symbol=symbol,
            type="market",
            side=side_str,
            amount=qty
        )

        # 4. TP ve SL emirlerini kur
        # TP
        bybit.create_order(
            symbol=symbol,
            type="take_profit_market",
            side=opposite_side,
            amount=qty,
            params={
                "stopPrice": tp_price,
                "reduce_only": True
            }
        )

        # SL
        bybit.create_order(
            symbol=symbol,
            type="stop_market",
            side=opposite_side,
            amount=qty,
            params={
                "stopPrice": sl_price,
                "reduce_only": True
            }
        )

        print(f"✅ Emir gönderildi: {side.upper()} | Giriş: {entry_price} | TP: {tp_price} | SL: {sl_price}")

    except Exception as e:
        print(f"❌ Emir gönderme hatası: {e}")

LEVERAGE_TABLE = {
    "BTC/USDT": 5,
    "ETH/USDT": 6,
    "SOL/USDT": 7
}

leverage = LEVERAGE_TABLE.get(symbol, 5)

# Adım 6: get_signal(df) fonksiyonu
# Gönderilen indikatörlerin analizine göre LONG / SHORT / NONE kararı verir

def get_signal(df):
    """
    df: DataFrame içerisinde şu sütunlar yer almalı:
        - rsi
        - supertrend_dir (1=up, -1=down)
        - trend_signal ("buy", "sell", "")
        - adx
        - wave_consolidation ("up", "down", "consolidate")
        - smart_money ("long", "short", "none")
    """
    try:
        row = df.iloc[-1]  # son satır (en güncel)

        long_score = 0
        short_score = 0

        # RSI
        if row["rsi"] < 30:
            long_score += 1
        elif row["rsi"] > 70:
            short_score += 1

        # Supertrend
        if row["supertrend_dir"] == 1:
            long_score += 1
        elif row["supertrend_dir"] == -1:
            short_score += 1

        # Trend Signal
        if row["trend_signal"] == "buy":
            long_score += 1
        elif row["trend_signal"] == "sell":
            short_score += 1

        # ADX (güçlü trend mi)
        if row["adx"] > 25:
            long_score += 0.5 if row["supertrend_dir"] == 1 else 0
            short_score += 0.5 if row["supertrend_dir"] == -1 else 0

        # Wave Consolidation
        if row["wave_consolidation"] == "up":
            long_score += 1
        elif row["wave_consolidation"] == "down":
            short_score += 1

        # Smart Money Concepts
        if row["smart_money"] == "long":
            long_score += 1.5
        elif row["smart_money"] == "short":
            short_score += 1.5

        # Karar
        if long_score >= 4 and long_score > short_score:
            return "long"
        elif short_score >= 4 and short_score > long_score:
            return "short"
        else:
            return "none"

    except Exception as e:
        print(f"get_signal hata: {e}")
        return "none"

def place_order(symbol, side, amount, leverage=5, sl_ratio=0.01, tp_ratio=0.02):
    """
    Emir açar ve TP/SL seviyelerini ayarlar.
    
    symbol: "BTC/USDT"
    side: "buy" veya "sell"
    amount: pozisyon miktarı (coin bazında)
    leverage: kaldıraç oranı
    sl_ratio: zarar durdur oranı (0.01 = %1)
    tp_ratio: kar al oranı (0.02 = %2)
    """
    try:
        market = exchange.market(symbol)
        symbol_id = market['id']

        # Kaldıraç ayarla
        exchange.set_leverage(leverage, symbol)

        # İşlem yönü belirle
        is_long = side == "buy"

        # Pozisyonu piyasa emri ile aç
        order = exchange.create_market_order(
            symbol=symbol,
            side=side,
            amount=amount
        )

        entry_price = order['average'] if order['average'] else order['price']

        # TP ve SL seviyelerini hesapla
        if is_long:
            sl_price = entry_price * (1 - sl_ratio)
            tp_price = entry_price * (1 + tp_ratio)
        else:
            sl_price = entry_price * (1 + sl_ratio)
            tp_price = entry_price * (1 - tp_ratio)

        # TP ve SL emirlerini gir
        params = {
            "stopLoss": round(sl_price, 2),
            "takeProfit": round(tp_price, 2)
        }

        exchange.create_order(
            symbol=symbol,
            type="market",
            side=side,
            amount=amount,
            params=params
        )

        print(f"\u2705 {symbol} - {side.upper()} emri gönderildi. Giriş: {entry_price:.2f} | TP: {tp_price:.2f} | SL: {sl_price:.2f}")

    except Exception as e:
        print(f"\u274C Emir gönderme hatası ({symbol}): {e}")

def place_order(symbol, side, amount, leverage=5, sl_ratio=0.01, tp_ratio=0.02):
    """
    Emir açar ve TP/SL seviyelerini ayarlar.

    symbol: "BTC/USDT"
    side: "buy" veya "sell"
    amount: pozisyon miktarı (coin bazında)
    leverage: kaldıraç oranı
    sl_ratio: zarar durdur oranı (0.01 = %1)
    tp_ratio: kar al oranı (0.02 = %2)
    """
    try:
        market = exchange.market(symbol)
        symbol_id = market['id']

        # Kaldıraç ayarla
        exchange.set_leverage(leverage, symbol)

        # İşlem yönü belirle
        is_long = side == "buy"

        # Pozisyonu piyasa emri ile aç
        order = exchange.create_market_order(
            symbol=symbol,
            side=side,
            amount=amount
        )

        entry_price = order['average'] if order['average'] else order['price']

        # TP ve SL seviyelerini hesapla
        if is_long:
            sl_price = entry_price * (1 - sl_ratio)
            tp_price = entry_price * (1 + tp_ratio)
        else:
            sl_price = entry_price * (1 + sl_ratio)
            tp_price = entry_price * (1 - tp_ratio)

        # TP ve SL emirlerini gir
        params = {
            "stopLoss": round(sl_price, 2),
            "takeProfit": round(tp_price, 2)
        }

        exchange.create_order(
            symbol=symbol,
            type="market",
            side=side,
            amount=amount,
            params=params
        )

        print(f"\u2705 {symbol} - {side.upper()} emri gönderildi. Giriş: {entry_price:.2f} | TP: {tp_price:.2f} | SL: {sl_price:.2f}")

    except Exception as e:
        print(f"\u274C Emir gönderme hatası ({symbol}): {e}")

def main_loop():
    symbols = ["BTC/USDT", "ETH/USDT", "SOL/USDT"]
    interval = "15m"
    amount = 0.05  # coin bazında sabit miktar

    while True:
        try:
            for symbol in symbols:
                df = get_ohlcv(symbol, interval)
                df = enrich_indicators(df)
                signal = get_signal(df)

                if signal == "long":
                    place_order(symbol, "buy", amount)

                elif signal == "short":
                    place_order(symbol, "sell", amount)

                else:
                    print(f"⏸ {symbol} için sinyal yok.")

            time.sleep(60)  # 1 dakika bekle, sonra tekrar tara

        except Exception as e:
            print(f"\u26a0\ufe0f Döngü hatası: {e}")
            time.sleep(60)

def place_order(symbol, side, amount, leverage=5, sl_ratio=0.01, tp_ratio=0.02):
    """
    Emir açar ve TP/SL seviyelerini ayarlar.

    symbol: "BTC/USDT"
    side: "buy" veya "sell"
    amount: pozisyon miktarı (coin bazında)
    leverage: kaldıraç oranı
    sl_ratio: zarar durdur oranı (0.01 = %1)
    tp_ratio: kar al oranı (0.02 = %2)
    """
    try:
        market = exchange.market(symbol)
        symbol_id = market['id']

        # Kaldıraç ayarla
        exchange.set_leverage(leverage, symbol)

        # İşlem yönü belirle
        is_long = side == "buy"

        # Pozisyonu piyasa emri ile aç
        order = exchange.create_market_order(
            symbol=symbol,
            side=side,
            amount=amount
        )

        entry_price = order['average'] if order['average'] else order['price']

        # TP ve SL seviyelerini hesapla
        if is_long:
            sl_price = entry_price * (1 - sl_ratio)
            tp_price = entry_price * (1 + tp_ratio)
        else:
            sl_price = entry_price * (1 + sl_ratio)
            tp_price = entry_price * (1 - tp_ratio)

        # TP ve SL emirlerini gir
        params = {
            "stopLoss": round(sl_price, 2),
            "takeProfit": round(tp_price, 2)
        }

        exchange.create_order(
            symbol=symbol,
            type="market",
            side=side,
            amount=amount,
            params=params
        )

        msg = (
            f"✅ <b>{symbol}</b> - <b>{side.upper()}</b> emri gönderildi.\n"
            f"Giriş: <b>{entry_price:.2f}</b> | TP: <b>{tp_price:.2f}</b> | SL: <b>{sl_price:.2f}</b>"
        )
        print(msg)
        send_telegram_message(msg)

    except Exception as e:
        err_msg = f"❌ Emir gönderme hatası ({symbol}): {e}"
        print(err_msg)
        send_telegram_message(err_msg)

def place_order(symbol, side, amount, leverage=5, sl_ratio=0.01, tp_ratio=0.02):
    """
    Emir açar ve TP/SL seviyelerini ayarlar.

    symbol: "BTC/USDT"
    side: "buy" veya "sell"
    amount: pozisyon miktarı (coin bazında)
    leverage: kaldıraç oranı
    sl_ratio: zarar durdur oranı (0.01 = %1)
    tp_ratio: kar al oranı (0.02 = %2)
    """
    try:
        market = exchange.market(symbol)
        symbol_id = market['id']

        # Kaldıraç ayarla
        exchange.set_leverage(leverage, symbol)

        # İşlem yönü belirle
        is_long = side == "buy"

        # Pozisyonu piyasa emri ile aç
        order = exchange.create_market_order(
            symbol=symbol,
            side=side,
            amount=amount
        )

        entry_price = order['average'] if order['average'] else order['price']

        # TP ve SL seviyelerini hesapla
        if is_long:
            sl_price = entry_price * (1 - sl_ratio)
            tp_price = entry_price * (1 + tp_ratio)
        else:
            sl_price = entry_price * (1 + sl_ratio)
            tp_price = entry_price * (1 - tp_ratio)

        # TP ve SL emirlerini gir
        params = {
            "stopLoss": round(sl_price, 2),
            "takeProfit": round(tp_price, 2)
        }

        exchange.create_order(
            symbol=symbol,
            type="market",
            side=side,
            amount=amount,
            params=params
        )

        msg = (
            f"✅ <b>{symbol}</b> - <b>{side.upper()}</b> emri gönderildi.\n"
            f"Giriş: <b>{entry_price:.2f}</b> | TP: <b>{tp_price:.2f}</b> | SL: <b>{sl_price:.2f}</b>"
        )
        print(msg)
        send_telegram_message(msg)

    except Exception as e:
        err_msg = f"❌ Emir gönderme hatası ({symbol}): {e}"
        print(err_msg)
        send_telegram_message(err_msg)

import csv
from datetime import datetime

LOG_FILE = "trade_log.csv"

def log_trade(symbol, side, amount, entry, tp, sl):
    try:
        with open(LOG_FILE, mode='a', newline='') as file:
            writer = csv.writer(file)
            writer.writerow([
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                symbol,
                side.upper(),
                amount,
                round(entry, 2),
                round(tp, 2),
                round(sl, 2)
            ])
    except Exception as e:
        print(f"Log yazma hatasi: {e}")

import csv
from datetime import datetime

LOG_FILE = "trade_log.csv"

# Mevcut işlemleri izlemek için aktif pozisyonu kaydet
active_trade = {
    "symbol": None,
    "side": None,
    "entry_price": None,
    "amount": None,
    "tp": None,
    "sl": None
}

def log_trade(symbol, side, amount, entry, tp, sl):
    try:
        with open(LOG_FILE, mode='a', newline='') as file:
            writer = csv.writer(file)
            writer.writerow([
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                symbol,
                side.upper(),
                amount,
                round(entry, 2),
                round(tp, 2),
                round(sl, 2)
            ])
    except Exception as e:
        print(f"Log yazma hatasi: {e}")

def set_active_trade(symbol, side, amount, entry, tp, sl):
    global active_trade
    active_trade = {
        "symbol": symbol,
        "side": side,
        "entry_price": entry,
        "amount": amount,
        "tp": tp,
        "sl": sl
    }

def check_position_status(current_price):
    """
    Fiyat TP veya SL'ye ulaşmış mı kontrol eder. Ulaşmışsa pozisyonu kapatır ve bildirir.
    """
    if not active_trade["symbol"]:
        return  # açık işlem yok

    reached_tp = False
    reached_sl = False

    if active_trade["side"] == "buy":
        reached_tp = current_price >= active_trade["tp"]
        reached_sl = current_price <= active_trade["sl"]
    else:
        reached_tp = current_price <= active_trade["tp"]
        reached_sl = current_price >= active_trade["sl"]

    if reached_tp or reached_sl:
        result = "✅ Kar" if reached_tp else "❌ Zarar"
        msg = (
            f"📉 <b>{active_trade['symbol']}</b> işlemi kapandı.\n"
            f"Yön: <b>{active_trade['side'].upper()}</b>\n"
            f"Giriş: <b>{active_trade['entry_price']:.2f}</b>\n"
            f"Kapanış: <b>{current_price:.2f}</b>\n"
            f"Durum: <b>{result}</b>"
        )
        print(msg)
        send_telegram_message(msg)
        active_trade["symbol"] = None

# Pozisyon kontrolü fonksiyonu

def can_open_new_position():
    """
    Aktif pozisyon olup olmadığını kontrol eder.
    Dönüş: True ise yeni işlem açılabilir, False ise beklenmeli.
    """
    return active_trade["symbol"] is None

# Kullanım örneği (sinyal geldikten sonra kontrol edilecek)

def process_signal(symbol, side, amount, entry_price, tp, sl):
    if can_open_new_position():
        # Pozisyon açılabilir
        place_order(symbol, side, amount, entry_price, tp, sl)
        set_active_trade(symbol, side, amount, entry_price, tp, sl)
        log_trade(symbol, side, amount, entry_price, tp, sl)
        send_telegram_message(
            f"✅ Yeni işleme girildi
<b>{symbol}</b> | {side.upper()}
Giriş: <b>{entry_price:.2f}</b> | TP: <b>{tp:.2f}</b> | SL: <b>{sl:.2f}</b>"
        )
    else:
        print("Aktif bir işlem mevcut, yeni pozisyon açılamaz.")


import time

# Ana döngü fonksiyonu
def main_loop():
    while True:
        try:
            if active_trade["symbol"]:
                # Pozisyon açıksa TP/SL kontrolü
                current_price = get_current_price(active_trade["symbol"])
                check_position_status(current_price)
            else:
                # Pozisyon yoksa analiz yap ve gerekiyorsa işlem aç
                analyze_market_and_process_signal()

        except Exception as e:
            print(f"Hata oluştu: {e}")

        time.sleep(15)  # 15 saniyede bir tekrar çalışır

# Burada kullanacağımız fonksiyonlar:
# - get_current_price(symbol): sembolün anlık fiyatını döner
# - check_position_status(price): pozisyon kapanmış mı kontrol eder
# - analyze_market_and_process_signal(): sinyal varsa işlem açar

import ccxt

# Bybit API ile fiyat çekme fonksiyonu
def get_current_price(symbol):
    try:
        # ccxt formatı ile borsayı tanımlıyoruz
        bybit = ccxt.bybit({
            'apiKey': 'YOUR_API_KEY',
            'secret': 'YOUR_SECRET_KEY',
            'enableRateLimit': True,
        })

        # Bybit sembolü için ccxt formatında düzeltme
        market = bybit.market(symbol)
        ticker = bybit.fetch_ticker(market['symbol'])
        return ticker['last']

    except Exception as e:
        print(f"❗Fiyat alınırken hata oluştu: {e}")
        return None

# Ana döngü - Adım 14
import time
from indikatörler import get_signal  # Örnek: "buy" veya "sell" döndürür
from bybit_api import get_current_price, place_order
from trade_logger import log_trade, set_active_trade, check_position_status

# Takip edilecek coin listesi
symbols = ["BTC/USDT", "ETH/USDT", "SOL/USDT"]

# Sabit ayarlar
LEVERAGE = 5
TP_RATIO = 0.02
SL_RATIO = 0.01

while True:
    try:
        for symbol in symbols:
            # Eğer aktif pozisyon varsa ve bu coin ile ilgiliyse, TP/SL kontrolü yap
            if active_trade["symbol"] == symbol:
                current_price = get_current_price(symbol)
                check_position_status(current_price)
                continue  # Bu coin için yeni pozisyon açma

            # Eğer pozisyon açık değilse, yeni sinyal kontrolü yap
            signal = get_signal(symbol)  # buy / sell / none
            if signal in ["buy", "sell"]:
                price = get_current_price(symbol)
                tp = price * (1 + TP_RATIO) if signal == "buy" else price * (1 - TP_RATIO)
                sl = price * (1 - SL_RATIO) if signal == "buy" else price * (1 + SL_RATIO)
                amount = 10  # Test için sabit miktar

                # Emir gönder
                success = place_order(symbol, signal, amount, LEVERAGE)
                if success:
                    set_active_trade(symbol, signal, amount, price, tp, sl)
                    log_trade(symbol, signal, amount, price, tp, sl)

        time.sleep(15)  # Her 15 saniyede bir çalış

    except Exception as e:
        print(f"Bot hatası: {e}")
        time.sleep(15)


# === Çoklu Coin Aktif Pozisyon Yönetimi ===
from datetime import datetime
import csv

LOG_FILE = "trade_log.csv"

# 3 Coin için ayrı ayrı aktif pozisyon yapıları
active_trades = {
    "BTCUSDT": None,
    "ETHUSDT": None,
    "SOLUSDT": None
}

def log_trade(symbol, side, amount, entry, tp, sl):
    try:
        with open(LOG_FILE, mode='a', newline='') as file:
            writer = csv.writer(file)
            writer.writerow([
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                symbol,
                side.upper(),
                amount,
                round(entry, 2),
                round(tp, 2),
                round(sl, 2)
            ])
    except Exception as e:
        print(f"Log yazma hatasi: {e}")

def set_active_trade(symbol, side, amount, entry, tp, sl):
    active_trades[symbol] = {
        "symbol": symbol,
        "side": side,
        "entry_price": entry,
        "amount": amount,
        "tp": tp,
        "sl": sl
    }

def check_position_status(symbol, current_price):
    trade = active_trades.get(symbol)
    if not trade:
        return

    reached_tp = False
    reached_sl = False

    if trade["side"] == "buy":
        reached_tp = current_price >= trade["tp"]
        reached_sl = current_price <= trade["sl"]
    else:
        reached_tp = current_price <= trade["tp"]
        reached_sl = current_price >= trade["sl"]

    if reached_tp or reached_sl:
        result = "✅ Kar" if reached_tp else "❌ Zarar"
        msg = (
            f"📉 <b>{trade['symbol']}</b> islemi kapandi.\n"
            f"Yön: <b>{trade['side'].upper()}</b>\n"
            f"Giris: <b>{trade['entry_price']:.2f}</b>\n"
            f"Kapanis: <b>{current_price:.2f}</b>\n"
            f"Durum: <b>{result}</b>"
        )
        print(msg)
        send_telegram_message(msg)
        active_trades[symbol] = None

# Pozisyon açık mı kontrol fonksiyonu

def is_trade_open(symbol):
    return active_trades.get(symbol) is not None

# Yeni pozisyon açma kararı verecek sinyaller için örnek (placeholder)

def example_signal_generator(symbol):
    # burada gönderdiğimiz indikatör analizleri çalışır
    # örneğin: if rsi_buy and macd_cross_up and supertrend_long:
    # return ("buy", entry, tp, sl)
    return None  # sinyal yok

# Ana bot döngüsünde örnek kullanım:

def bot_main_loop():
    for symbol in ["BTCUSDT", "ETHUSDT", "SOLUSDT"]:
        price = get_latest_price(symbol)
        check_position_status(symbol, price)

        if not is_trade_open(symbol):
            signal = example_signal_generator(symbol)
            if signal:
                side, entry, tp, sl = signal
                amount = calculate_amount(entry)
                set_active_trade(symbol, side, amount, entry, tp, sl)
                log_trade(symbol, side, amount, entry, tp, sl)
                place_order(symbol, side, amount, entry, tp, sl)
                send_telegram_message(f"🚀 <b>{symbol}</b> için yeni pozisyon açıldı!\nYön: <b>{side}</b>\nEntry: <b>{entry}</b>\nTP: <b>{tp}</b>\nSL: <b>{sl}</b>")

# Adım 16 - İndikatörlerden sinyal üretimini başlat
# Örnek olarak RSI + Supertrend + ADX kullanıyoruz
# Bu modül, sinyal üretir ve trade başlatılması için çağrılır

import ccxt
import time
import pandas as pd
import numpy as np
from datetime import datetime

# --- INDICATOR FONKSIYONLARI ---
def calculate_rsi(close_prices, period=14):
    delta = np.diff(close_prices)
    gain = np.where(delta > 0, delta, 0)
    loss = np.where(delta < 0, -delta, 0)

    avg_gain = pd.Series(gain).rolling(window=period).mean()
    avg_loss = pd.Series(loss).rolling(window=period).mean()

    rs = avg_gain / avg_loss
    rsi = 100 - (100 / (1 + rs))
    return rsi.iloc[-1]

def calculate_supertrend(df, period=10, multiplier=3):
    hl2 = (df['high'] + df['low']) / 2
    atr = df['high'].combine(df['low'], max) - df['low'].combine(df['high'], min)
    atr = atr.rolling(window=period).mean()

    upper_band = hl2 + (multiplier * atr)
    lower_band = hl2 - (multiplier * atr)

    direction = []
    for i in range(len(df)):
        if i < period:
            direction.append(None)
        elif df['close'][i] > upper_band[i - 1]:
            direction.append("long")
        elif df['close'][i] < lower_band[i - 1]:
            direction.append("short")
        else:
            direction.append(direction[-1] if direction[-1] else None)
    return direction[-1]

def calculate_adx(df, period=14):
    up_move = df['high'].diff()
    down_move = df['low'].diff()
    plus_dm = np.where((up_move > down_move) & (up_move > 0), up_move, 0)
    minus_dm = np.where((down_move > up_move) & (down_move > 0), down_move, 0)

    tr1 = df['high'] - df['low']
    tr2 = abs(df['high'] - df['close'].shift())
    tr3 = abs(df['low'] - df['close'].shift())
    tr = pd.concat([tr1, tr2, tr3], axis=1).max(axis=1)
    atr = tr.rolling(window=period).mean()

    plus_di = 100 * pd.Series(plus_dm).rolling(window=period).mean() / atr
    minus_di = 100 * pd.Series(minus_dm).rolling(window=period).mean() / atr
    dx = 100 * abs(plus_di - minus_di) / (plus_di + minus_di)
    adx = dx.rolling(window=period).mean()
    return adx.iloc[-1]

# --- SİNYAL ÜRETİCİ ---
def generate_signal(df):
    rsi = calculate_rsi(df['close'])
    trend = calculate_supertrend(df)
    adx = calculate_adx(df)

    if trend == "long" and rsi < 70 and adx > 25:
        return "buy"
    elif trend == "short" and rsi > 30 and adx > 25:
        return "sell"
    else:
        return None

import time
from signal_generator import get_signal
from trade_execution import open_trade, close_trade
from telegram_notify import send_telegram_message, check_position_status
from trade_logger import set_active_trade, active_trade
from price_fetcher import get_current_price, get_symbols

INTERVAL = 15  # saniye

print("Bot başlatıldı...")
while True:
    try:
        # Aktif işlem açık mı kontrol et
        if active_trade["symbol"]:
            current_price = get_current_price(active_trade["symbol"])
            check_position_status(current_price)
        
        else:
            for symbol in get_symbols():
                signal_data = get_signal(symbol)
                if signal_data:
                    open_trade(**signal_data)
                    break  # Aynı anda sadece 1 coin işlemde olacak

    except Exception as e:
        send_telegram_message(f"Hata oluştu: {e}")
        print(f"Hata: {e}")

    time.sleep(INTERVAL)

# Adım 18: Performans Loglama ve Raporlama
# =========================================
# Gün sonunda yapılan işlemlerin raporlanması, toplam kar/zarar, başarı oranı ve işlem sayısı gibi veriler hesaplanır

import pandas as pd
import os
from datetime import datetime

LOG_FILE = "trade_log.csv"


def generate_daily_report():
    if not os.path.exists(LOG_FILE):
        print("Henüz kayıtlı işlem bulunmamaktadır.")
        return

    df = pd.read_csv(LOG_FILE, header=None)
    df.columns = ["datetime", "symbol", "side", "amount", "entry", "tp", "sl"]

    # Tarihsel filtreleme (bugünün işlemleri)
    df["datetime"] = pd.to_datetime(df["datetime"])
    today = pd.Timestamp.today().normalize()
    df_today = df[df["datetime"] >= today]

    if df_today.empty:
        print("Bugün için işlem bulunmamaktadır.")
        return

    # Kazanç/zarar hesabı
    def calc_result(row):
        if row["side"] == "BUY":
            return row["tp"] - row["entry"]
        else:
            return row["entry"] - row["tp"]

    df_today["result"] = df_today.apply(calc_result, axis=1)
    df_today["status"] = df_today["result"].apply(lambda x: "WIN" if x > 0 else "LOSS")

    total_trades = len(df_today)
    wins = len(df_today[df_today["status"] == "WIN"])
    losses = len(df_today[df_today["status"] == "LOSS"])
    winrate = (wins / total_trades) * 100
    total_profit = df_today["result"].sum()

    report = (
        f"📊 <b>Günlük Performans Raporu</b>\n"
        f"İşlem Sayısı: <b>{total_trades}</b>\n"
        f"Kazanılan: <b>{wins}</b> | Kaybedilen: <b>{losses}</b>\n"
        f"Başarı Oranı: <b>{winrate:.2f}%</b>\n"
        f"Toplam Kar/Zarar: <b>{total_profit:.2f}</b>"
    )

    print(report)
    send_telegram_message(report)


# Bu fonksiyon örneğin gün sonunda bir scheduler ile çağrılabilir
# generate_daily_report()


# Günlük performans raporunu 3 kez gönderecek zamanlayıcı ayarları (8 saatte bir)
import schedule
import time
from telegram import send_telegram_message
from trade_report import generate_daily_report

# Her 8 saatte bir gönderilecek fonksiyon

def send_periodic_report():
    try:
        report = generate_daily_report()
        send_telegram_message(f"📊 8 Saatlik Performans Özeti\n\n{report}")
    except Exception as e:
        print(f"Rapor gönderim hatası: {e}")

# Zamanlama ayarları
schedule.every().day.at("08:00").do(send_periodic_report)
schedule.every().day.at("16:00").do(send_periodic_report)
schedule.every().day.at("00:00").do(send_periodic_report)

# Sonsuz döngü
while True:
    schedule.run_pending()
    time.sleep(60)

# Pozisyon kontrolü fonksiyonu

def can_open_new_position():
    """
    Aktif pozisyon olup olmadığını kontrol eder.
    Dönüş: True ise yeni işlem açılabilir, False ise beklenmeli.
    """
    return active_trade["symbol"] is None

# Kullanım örneği (sinyal geldikten sonra kontrol edilecek)

def process_signal(symbol, side, amount, entry_price, tp, sl):
    if can_open_new_position():
        # Pozisyon açılabilir
        place_order(symbol, side, amount, entry_price, tp, sl)
        set_active_trade(symbol, side, amount, entry_price, tp, sl)
        log_trade(symbol, side, amount, entry_price, tp, sl)
        send_telegram_message(
            f"✅ Yeni işleme girildi
<b>{symbol}</b> | {side.upper()}
Giriş: <b>{entry_price:.2f}</b> | TP: <b>{tp:.2f}</b> | SL: <b>{sl:.2f}</b>"
        )
    else:
        print("Aktif bir işlem mevcut, yeni pozisyon açılamaz.")


import time

# Ana döngü fonksiyonu
def main_loop():
    while True:
        try:
            if active_trade["symbol"]:
                # Pozisyon açıksa TP/SL kontrolü
                current_price = get_current_price(active_trade["symbol"])
                check_position_status(current_price)
            else:
                # Pozisyon yoksa analiz yap ve gerekiyorsa işlem aç
                analyze_market_and_process_signal()

        except Exception as e:
            print(f"Hata oluştu: {e}")

        time.sleep(15)  # 15 saniyede bir tekrar çalışır

# Burada kullanacağımız fonksiyonlar:
# - get_current_price(symbol): sembolün anlık fiyatını döner
# - check_position_status(price): pozisyon kapanmış mı kontrol eder
# - analyze_market_and_process_signal(): sinyal varsa işlem açar

import ccxt

# Bybit API ile fiyat çekme fonksiyonu
def get_current_price(symbol):
    try:
        # ccxt formatı ile borsayı tanımlıyoruz
        bybit = ccxt.bybit({
            'apiKey': 'YOUR_API_KEY',
            'secret': 'YOUR_SECRET_KEY',
            'enableRateLimit': True,
        })

        # Bybit sembolü için ccxt formatında düzeltme
        market = bybit.market(symbol)
        ticker = bybit.fetch_ticker(market['symbol'])
        return ticker['last']

    except Exception as e:
        print(f"❗Fiyat alınırken hata oluştu: {e}")
        return None

# Ana döngü - Adım 14
import time
from indikatörler import get_signal  # Örnek: "buy" veya "sell" döndürür
from bybit_api import get_current_price, place_order
from trade_logger import log_trade, set_active_trade, check_position_status

# Takip edilecek coin listesi
symbols = ["BTC/USDT", "ETH/USDT", "SOL/USDT"]

# Sabit ayarlar
LEVERAGE = 5
TP_RATIO = 0.02
SL_RATIO = 0.01

while True:
    try:
        for symbol in symbols:
            # Eğer aktif pozisyon varsa ve bu coin ile ilgiliyse, TP/SL kontrolü yap
            if active_trade["symbol"] == symbol:
                current_price = get_current_price(symbol)
                check_position_status(current_price)
                continue  # Bu coin için yeni pozisyon açma

            # Eğer pozisyon açık değilse, yeni sinyal kontrolü yap
            signal = get_signal(symbol)  # buy / sell / none
            if signal in ["buy", "sell"]:
                price = get_current_price(symbol)
                tp = price * (1 + TP_RATIO) if signal == "buy" else price * (1 - TP_RATIO)
                sl = price * (1 - SL_RATIO) if signal == "buy" else price * (1 + SL_RATIO)
                amount = 10  # Test için sabit miktar

                # Emir gönder
                success = place_order(symbol, signal, amount, LEVERAGE)
                if success:
                    set_active_trade(symbol, signal, amount, price, tp, sl)
                    log_trade(symbol, signal, amount, price, tp, sl)

        time.sleep(15)  # Her 15 saniyede bir çalış

    except Exception as e:
        print(f"Bot hatası: {e}")
        time.sleep(15)


# === Çoklu Coin Aktif Pozisyon Yönetimi ===
from datetime import datetime
import csv

LOG_FILE = "trade_log.csv"

# 3 Coin için ayrı ayrı aktif pozisyon yapıları
active_trades = {
    "BTCUSDT": None,
    "ETHUSDT": None,
    "SOLUSDT": None
}

def log_trade(symbol, side, amount, entry, tp, sl):
    try:
        with open(LOG_FILE, mode='a', newline='') as file:
            writer = csv.writer(file)
            writer.writerow([
                datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
                symbol,
                side.upper(),
                amount,
                round(entry, 2),
                round(tp, 2),
                round(sl, 2)
            ])
    except Exception as e:
        print(f"Log yazma hatasi: {e}")

def set_active_trade(symbol, side, amount, entry, tp, sl):
    active_trades[symbol] = {
        "symbol": symbol,
        "side": side,
        "entry_price": entry,
        "amount": amount,
        "tp": tp,
        "sl": sl
    }

def check_position_status(symbol, current_price):
    trade = active_trades.get(symbol)
    if not trade:
        return

    reached_tp = False
    reached_sl = False

    if trade["side"] == "buy":
        reached_tp = current_price >= trade["tp"]
        reached_sl = current_price <= trade["sl"]
    else:
        reached_tp = current_price <= trade["tp"]
        reached_sl = current_price >= trade["sl"]

    if reached_tp or reached_sl:
        result = "✅ Kar" if reached_tp else "❌ Zarar"
        msg = (
            f"📉 <b>{trade['symbol']}</b> islemi kapandi.\n"
            f"Yön: <b>{trade['side'].upper()}</b>\n"
            f"Giris: <b>{trade['entry_price']:.2f}</b>\n"
            f"Kapanis: <b>{current_price:.2f}</b>\n"
            f"Durum: <b>{result}</b>"
        )
        print(msg)
        send_telegram_message(msg)
        active_trades[symbol] = None

# Pozisyon açık mı kontrol fonksiyonu

def is_trade_open(symbol):
    return active_trades.get(symbol) is not None

# Yeni pozisyon açma kararı verecek sinyaller için örnek (placeholder)

def example_signal_generator(symbol):
    # burada gönderdiğimiz indikatör analizleri çalışır
    # örneğin: if rsi_buy and macd_cross_up and supertrend_long:
    # return ("buy", entry, tp, sl)
    return None  # sinyal yok

# Ana bot döngüsünde örnek kullanım:

def bot_main_loop():
    for symbol in ["BTCUSDT", "ETHUSDT", "SOLUSDT"]:
        price = get_latest_price(symbol)
        check_position_status(symbol, price)

        if not is_trade_open(symbol):
            signal = example_signal_generator(symbol)
            if signal:
                side, entry, tp, sl = signal
                amount = calculate_amount(entry)
                set_active_trade(symbol, side, amount, entry, tp, sl)
                log_trade(symbol, side, amount, entry, tp, sl)
                place_order(symbol, side, amount, entry, tp, sl)
                send_telegram_message(f"🚀 <b>{symbol}</b> için yeni pozisyon açıldı!\nYön: <b>{side}</b>\nEntry: <b>{entry}</b>\nTP: <b>{tp}</b>\nSL: <b>{sl}</b>")

# Adım 16 - İndikatörlerden sinyal üretimini başlat
# Örnek olarak RSI + Supertrend + ADX kullanıyoruz
# Bu modül, sinyal üretir ve trade başlatılması için çağrılır

import ccxt
import time
import pandas as pd
import numpy as np
from datetime import datetime

# --- INDICATOR FONKSIYONLARI ---
def calculate_rsi(close_prices, period=14):
    delta = np.diff(close_prices)
    gain = np.where(delta > 0, delta, 0)
    loss = np.where(delta < 0, -delta, 0)

    avg_gain = pd.Series(gain).rolling(window=period).mean()
    avg_loss = pd.Series(loss).rolling(window=period).mean()

    rs = avg_gain / avg_loss
    rsi = 100 - (100 / (1 + rs))
    return rsi.iloc[-1]

def calculate_supertrend(df, period=10, multiplier=3):
    hl2 = (df['high'] + df['low']) / 2
    atr = df['high'].combine(df['low'], max) - df['low'].combine(df['high'], min)
    atr = atr.rolling(window=period).mean()

    upper_band = hl2 + (multiplier * atr)
    lower_band = hl2 - (multiplier * atr)

    direction = []
    for i in range(len(df)):
        if i < period:
            direction.append(None)
        elif df['close'][i] > upper_band[i - 1]:
            direction.append("long")
        elif df['close'][i] < lower_band[i - 1]:
            direction.append("short")
        else:
            direction.append(direction[-1] if direction[-1] else None)
    return direction[-1]

def calculate_adx(df, period=14):
    up_move = df['high'].diff()
    down_move = df['low'].diff()
    plus_dm = np.where((up_move > down_move) & (up_move > 0), up_move, 0)
    minus_dm = np.where((down_move > up_move) & (down_move > 0), down_move, 0)

    tr1 = df['high'] - df['low']
    tr2 = abs(df['high'] - df['close'].shift())
    tr3 = abs(df['low'] - df['close'].shift())
    tr = pd.concat([tr1, tr2, tr3], axis=1).max(axis=1)
    atr = tr.rolling(window=period).mean()

    plus_di = 100 * pd.Series(plus_dm).rolling(window=period).mean() / atr
    minus_di = 100 * pd.Series(minus_dm).rolling(window=period).mean() / atr
    dx = 100 * abs(plus_di - minus_di) / (plus_di + minus_di)
    adx = dx.rolling(window=period).mean()
    return adx.iloc[-1]

# --- SİNYAL ÜRETİCİ ---
def generate_signal(df):
    rsi = calculate_rsi(df['close'])
    trend = calculate_supertrend(df)
    adx = calculate_adx(df)

    if trend == "long" and rsi < 70 and adx > 25:
        return "buy"
    elif trend == "short" and rsi > 30 and adx > 25:
        return "sell"
    else:
        return None

import time
from signal_generator import get_signal
from trade_execution import open_trade, close_trade
from telegram_notify import send_telegram_message, check_position_status
from trade_logger import set_active_trade, active_trade
from price_fetcher import get_current_price, get_symbols

INTERVAL = 15  # saniye

print("Bot başlatıldı...")
while True:
    try:
        # Aktif işlem açık mı kontrol et
        if active_trade["symbol"]:
            current_price = get_current_price(active_trade["symbol"])
            check_position_status(current_price)
        
        else:
            for symbol in get_symbols():
                signal_data = get_signal(symbol)
                if signal_data:
                    open_trade(**signal_data)
                    break  # Aynı anda sadece 1 coin işlemde olacak

    except Exception as e:
        send_telegram_message(f"Hata oluştu: {e}")
        print(f"Hata: {e}")

    time.sleep(INTERVAL)

# Adım 18: Performans Loglama ve Raporlama
# =========================================
# Gün sonunda yapılan işlemlerin raporlanması, toplam kar/zarar, başarı oranı ve işlem sayısı gibi veriler hesaplanır

import pandas as pd
import os
from datetime import datetime

LOG_FILE = "trade_log.csv"


def generate_daily_report():
    if not os.path.exists(LOG_FILE):
        print("Henüz kayıtlı işlem bulunmamaktadır.")
        return

    df = pd.read_csv(LOG_FILE, header=None)
    df.columns = ["datetime", "symbol", "side", "amount", "entry", "tp", "sl"]

    # Tarihsel filtreleme (bugünün işlemleri)
    df["datetime"] = pd.to_datetime(df["datetime"])
    today = pd.Timestamp.today().normalize()
    df_today = df[df["datetime"] >= today]

    if df_today.empty:
        print("Bugün için işlem bulunmamaktadır.")
        return

    # Kazanç/zarar hesabı
    def calc_result(row):
        if row["side"] == "BUY":
            return row["tp"] - row["entry"]
        else:
            return row["entry"] - row["tp"]

    df_today["result"] = df_today.apply(calc_result, axis=1)
    df_today["status"] = df_today["result"].apply(lambda x: "WIN" if x > 0 else "LOSS")

    total_trades = len(df_today)
    wins = len(df_today[df_today["status"] == "WIN"])
    losses = len(df_today[df_today["status"] == "LOSS"])
    winrate = (wins / total_trades) * 100
    total_profit = df_today["result"].sum()

    report = (
        f"📊 <b>Günlük Performans Raporu</b>\n"
        f"İşlem Sayısı: <b>{total_trades}</b>\n"
        f"Kazanılan: <b>{wins}</b> | Kaybedilen: <b>{losses}</b>\n"
        f"Başarı Oranı: <b>{winrate:.2f}%</b>\n"
        f"Toplam Kar/Zarar: <b>{total_profit:.2f}</b>"
    )

    print(report)
    send_telegram_message(report)


# Bu fonksiyon örneğin gün sonunda bir scheduler ile çağrılabilir
# generate_daily_report()


# Günlük performans raporunu 3 kez gönderecek zamanlayıcı ayarları (8 saatte bir)
import schedule
import time
from telegram import send_telegram_message
from trade_report import generate_daily_report

# Her 8 saatte bir gönderilecek fonksiyon

def send_periodic_report():
    try:
        report = generate_daily_report()
        send_telegram_message(f"📊 8 Saatlik Performans Özeti\n\n{report}")
    except Exception as e:
        print(f"Rapor gönderim hatası: {e}")

# Zamanlama ayarları
schedule.every().day.at("08:00").do(send_periodic_report)
schedule.every().day.at("16:00").do(send_periodic_report)
schedule.every().day.at("00:00").do(send_periodic_report)

# Sonsuz döngü
while True:
    schedule.run_pending()
    time.sleep(60)

# === Günlük sıfırlama ===
last_reset_day = datetime.utcnow().day

def reset_daily_risk():
    global daily_loss_total, loss_streak, last_reset_day
    today = datetime.utcnow().day
    if today != last_reset_day:
        daily_loss_total = 0.0
        loss_streak = 0
        last_reset_day = today

# === Zarar kontrol fonksiyonu ===
def risk_control(current_pnl, entry_time):
    global loss_streak, daily_loss_total, bot_locked_until, last_trade_time

    reset_daily_risk()
    now = time.time()

    if now - last_trade_time < MIN_TRADE_INTERVAL:
        return False, "Pozisyonlar arası minimum süreye uyulmadı."

    if now < bot_locked_until:
        return False, f"Bot geçici olarak kilitli. {int((bot_locked_until - now)/60)} dk sonra tekrar dene."

    if daily_loss_total >= MAX_DAILY_LOSS:
        return False, "Günlük zarar limiti aşıldı. Bugün için yeni işlem açılmaz."

    if current_pnl < 0:
        abs_loss = abs(current_pnl)
        daily_loss_total += abs_loss
        loss_streak += 1

        if abs_loss >= MAX_LOSS_PER_TRADE:
            return False, "Bu işlemde zarar limiti aşıldı. Pozisyon kapatılmalı."

        if loss_streak >= LOSS_STREAK_LIMIT:
            bot_locked_until = now + COOLDOWN_AFTER_LOSS
            return False, "Art arda 3 zarar alındı. Bot 1 saat kilitlendi."

    last_trade_time = now
    return True, "Risk kontrolü geçildi. İşlem açılabilir."

# === Sinyal çakışması kontrolü ===
def check_signal_conflict(current_signal):
    global last_signal, last_position

    # Aynı anda hem long hem short sinyali gelirse
    if last_signal and last_signal != current_signal:
        return False, "Sinyal çakışması: Son sinyal ile mevcut sinyal çelişiyor."

    # Mevcut açık pozisyon varsa ve sinyal tersse işlem açılmaz
    if last_position and last_position != current_signal:
        return False, f"Açık pozisyon ile çelişen sinyal algılandı. Önce mevcut pozisyon kapatılmalı."

    last_signal = current_signal
    return True, "Sinyal tutarlı. İşlem yapılabilir."

# === Kullanım örneği ===
# result, reason = risk_control(current_pnl=-0.02, entry_time=time.time())
# if not result:
#     print("İşlem engellendi:", reason)
# else:
#     print("İşleme devam.")

# conflict_result, reason2 = check_signal_conflict("long")
# if not conflict_result:
#     print("Sinyal engellendi:", reason2)
# else:
#     print("Sinyal onaylandı.")


# Sinyal Kaynağının Dinamik Tanımlanması ve Önceliklendirilmesi

# Örnek sinyal kaynakları (güven puanı 0-100 aralığında)
signal_sources = {
    'LuxAlgo': 90,
    'TrendAlgo': 75,
    'PivotPoints': 60,
    'Supertrend': 70,
    'SmartMoneyConcepts': 85,
    'WaveConsolidation': 65,
}

# Her kaynaktan gelen sinyal: {source_name: 'long' / 'short' / None}
example_signals = {
    'LuxAlgo': 'long',
    'TrendAlgo': 'short',
    'PivotPoints': 'long',
    'SmartMoneyConcepts': None,
    'Supertrend': 'long',
    'WaveConsolidation': 'short',
}

# Sinyalleri puana göre tartıp ağırlıklı karar alma
def prioritize_signals(signals):
    vote_score = {'long': 0, 'short': 0}
    for source, signal in signals.items():
        if signal in vote_score:
            vote_score[signal] += signal_sources.get(source, 50)  # default 50 puan

    if vote_score['long'] > vote_score['short']:
        return 'long'
    elif vote_score['short'] > vote_score['long']:
        return 'short'
    else:
        return None  # Eşitlik durumunda işlem yapma

# === Kullanım Örneği ===
# direction = prioritize_signals(example_signals)
# print(f"Ağırlıklı sinyal yönü: {direction}")


# Dynamic Position Sizing Module

def calculate_position_size(account_balance, risk_per_trade, entry_price, stop_price):
    """
    account_balance: toplam kasa miktarı (örnek: 300 USDT)
    risk_per_trade: işlem başına risk oranı (örnek: 0.02)
    entry_price: işleme giriş fiyatı
    stop_price: stop loss seviyesi
    """
    stop_loss_distance = abs(entry_price - stop_price)
    if stop_loss_distance == 0:
        return 0, "Stop mesafesi 0 olamaz."

    risk_amount = account_balance * risk_per_trade
    position_size = risk_amount / stop_loss_distance

    return round(position_size, 2), "Pozisyon boyutu hesaplandı."


# === Kullanım örneği ===
# pozisyon, mesaj = calculate_position_size(300, 0.02, 100, 99)
# print("Pozisyon:", pozisyon, mesaj)

# Bu fonksiyon bot sistemine entegre edilerek emir miktarı dinamik belirlenebilir.



# Kâr Realizasyonu Modülü
import math

def calculate_take_profit(entry_price, direction, tp_ratio_1=0.015, tp_ratio_2=0.03):
    """
    Pozisyona göre 2 kademe kâr alma noktası hesaplar.
    entry_price: Pozisyon açılış fiyatı
    direction: 'long' veya 'short'
    tp_ratio_1: 1. kademede hedeflenen kâr oranı (vars. %1.5)
    tp_ratio_2: 2. kademede hedeflenen kâr oranı (vars. %3)
    Dönüş: (tp1, tp2)
    """
    if direction == 'long':
        tp1 = entry_price * (1 + tp_ratio_1)
        tp2 = entry_price * (1 + tp_ratio_2)
    else:
        tp1 = entry_price * (1 - tp_ratio_1)
        tp2 = entry_price * (1 - tp_ratio_2)
    return round(tp1, 4), round(tp2, 4)

# Kullanım örneği
# entry = 105000
# direction = 'long'
# tp1, tp2 = calculate_take_profit(entry, direction)
# print(f"TP1: {tp1} | TP2: {tp2}")



# Take Profit (Kâr Al) Modülü

def calculate_take_profits(entry_price, direction, tp1_perc=0.015, tp2_perc=0.03):
    '''
    Pozisyon açılış fiyatı ve yönüne göre iki TP seviyesi hesaplar.
    direction: 'long' veya 'short'
    tp1_perc: İlk kâr al seviyesi (örn: 0.015 = %1.5)
    tp2_perc: İkinci kâr al seviyesi (örn: 0.03 = %3)
    '''
    if direction == 'long':
        tp1 = entry_price * (1 + tp1_perc)
        tp2 = entry_price * (1 + tp2_perc)
    elif direction == 'short':
        tp1 = entry_price * (1 - tp1_perc)
        tp2 = entry_price * (1 - tp2_perc)
    else:
        raise ValueError('direction long veya short olmalı!')
    return round(tp1, 2), round(tp2, 2)

# Pozisyon kapatma (örnek fonksiyon; emir entegrasyonu Bybit API ile sağlanacak)
def close_position_at_tp(api, symbol, position_id, tp_level, amount):
    '''
    Burada API ile ilgili coin ve miktar için pozisyonun bir kısmı veya tamamı kapatılır.
    '''
    # Bybit API'de position/close veya order/market-close fonksiyonu çağrılacak
    # api.close_position(symbol=symbol, position_id=position_id, price=tp_level, amount=amount)
    pass

# Kullanım örneği:
# entry = 105000
# direction = 'long'
# tp1, tp2 = calculate_take_profits(entry, direction)
# print(f"TP1: {tp1} | TP2: {tp2}")


# Take Profit (TP) Modülü
# Bu modül pozisyon açıldıktan sonra otomatik olarak kar al emirlerini yönetir.

def calculate_tp_levels(entry_price, position_side, tp1_perc=0.015, tp2_perc=0.03):
    """
    Pozisyon yönüne göre iki TP seviyesi hesaplar.
    entry_price: float - Pozisyon açılış fiyatı
    position_side: 'long' veya 'short'
    tp1_perc: float - 1. TP yüzdesi (örn: 0.015 => %1.5)
    tp2_perc: float - 2. TP yüzdesi (örn: 0.03  => %3)
    return: (tp1_fiyatı, tp2_fiyatı)
    """
    if position_side == 'long':
        tp1 = entry_price * (1 + tp1_perc)
        tp2 = entry_price * (1 + tp2_perc)
    elif position_side == 'short':
        tp1 = entry_price * (1 - tp1_perc)
        tp2 = entry_price * (1 - tp2_perc)
    else:
        raise ValueError('Pozisyon yönü "long" veya "short" olmalı.')
    return round(tp1, 4), round(tp2, 4)

# Bybit API ile kar al emirleri gönderme fonksiyonu (sahte örnek, gerçek API fonksiyonu yerine kendi ana fonksiyonunu yazmalısın)
def place_take_profit_orders(symbol, position_side, entry_price, qty, api_client):
    tp1, tp2 = calculate_tp_levels(entry_price, position_side)
    qty1 = qty * 0.5  # %50'lik ilk TP
    qty2 = qty - qty1  # Kalanı ikinci TP

    # 1. TP için emir gönder
    api_client.create_order(
        symbol=symbol,
        side='sell' if position_side=='long' else 'buy',
        qty=qty1,
        price=tp1,
        reduce_only=True,
        type='limit'
    )
    # 2. TP için emir gönder
    api_client.create_order(
        symbol=symbol,
        side='sell' if position_side=='long' else 'buy',
        qty=qty2,
        price=tp2,
        reduce_only=True,
        type='limit'
    )
    return tp1, tp2

# Kullanım örneği:
# tp1, tp2 = place_take_profit_orders('BTCUSDT', 'long', 105000, 0.01, bybit_client)
# print(f'TP1: {tp1}, TP2: {tp2}')



# Python İndikatör Analiz Modülü (Demo)
import pandas as pd
import numpy as np

# === KLASİK İNDİKATÖRLER ===
# --- RSI ---
def calc_rsi(series, period=14):
    delta = series.diff()
    gain = (delta.where(delta > 0, 0)).rolling(window=period).mean()
    loss = (-delta.where(delta < 0, 0)).rolling(window=period).mean()
    rs = gain / loss
    return 100 - (100 / (1 + rs))

# --- EMA ---
def calc_ema(series, period=14):
    return series.ewm(span=period, adjust=False).mean()

# --- SuperTrend ---
def calc_supertrend(df, period=10, multiplier=3):
    hl2 = (df['High'] + df['Low']) / 2
    atr = df['High'].combine(df['Low'], np.maximum) - df['Low'].combine(df['High'], np.minimum)
    atr = atr.rolling(period).mean()
    upperband = hl2 + (multiplier * atr)
    lowerband = hl2 - (multiplier * atr)
    final_upperband = upperband.copy()
    final_lowerband = lowerband.copy()
    trend = [True]
    for i in range(1, len(df)):
        if df['Close'].iloc[i] > final_upperband.iloc[i-1]:
            trend.append(True)
        elif df['Close'].iloc[i] < final_lowerband.iloc[i-1]:
            trend.append(False)
        else:
            trend.append(trend[-1])
            if trend[-1] and lowerband.iloc[i] > final_lowerband.iloc[i-1]:
                final_lowerband.iloc[i] = final_lowerband.iloc[i-1]
            if not trend[-1] and upperband.iloc[i] < final_upperband.iloc[i-1]:
                final_upperband.iloc[i] = final_upperband.iloc[i-1]
    return pd.Series(trend, index=df.index)

# --- ADX ---
def calc_adx(df, period=14):
    plus_dm = df['High'].diff()
    minus_dm = df['Low'].diff().abs()
    plus_dm[plus_dm < 0] = 0
    minus_dm[minus_dm < 0] = 0
    tr = pd.concat([df['High'] - df['Low'],
                    (df['High'] - df['Close'].shift()).abs(),
                    (df['Low'] - df['Close'].shift()).abs()], axis=1).max(axis=1)
    atr = tr.rolling(period).mean()
    plus_di = 100 * (plus_dm.rolling(period).mean() / atr)
    minus_di = 100 * (minus_dm.rolling(period).mean() / atr)
    dx = (abs(plus_di - minus_di) / abs(plus_di + minus_di)) * 100
    adx = dx.rolling(period).mean()
    return adx

# --- Basit ZigZag (local min/max tespiti) ---
def find_zigzag(df, depth=5):
    local_min = (df['Low'].rolling(window=depth*2+1, center=True).min() == df['Low'])
    local_max = (df['High'].rolling(window=depth*2+1, center=True).max() == df['High'])
    zigzag_points = np.where(local_min | local_max, df['Close'], np.nan)
    return pd.Series(zigzag_points, index=df.index)

# === ÖRNEK KULLANIM ===
# Demo veri (Gerçek veriyle değiştirilebilir)
dates = pd.date_range('2024-01-01', periods=200)
data = pd.DataFrame({
    'Open': np.random.uniform(100, 120, 200),
    'High': np.random.uniform(120, 130, 200),
    'Low': np.random.uniform(90, 100, 200),
    'Close': np.random.uniform(100, 120, 200),
}, index=dates)

data['RSI'] = calc_rsi(data['Close'])
data['EMA_14'] = calc_ema(data['Close'])
data['SuperTrend'] = calc_supertrend(data)
data['ADX'] = calc_adx(data)
data['ZigZag'] = find_zigzag(data)

# Özet gösterim
data[['Close','RSI','EMA_14','SuperTrend','ADX','ZigZag']].tail(10)

# Bybit Otomatik Trade Botu (Ana Modül)
# --- Tufan Bey için, indikatör bazlı tam otomatik sistem ---

import ccxt
import pandas as pd
import time
from datetime import datetime
from risk_management import risk_control  # Gelişmiş risk modülü
from indicator_module import *            # İndikatör analizleri burada
from telegram_notify import send_telegram # Telegram bildirimi

# === Kullanıcı parametreleri ===
API_KEY = 'BURAYA_KEY'
API_SECRET = 'BURAYA_SECRET'
SYMBOLS = ['BTC/USDT', 'ETH/USDT', 'SOL/USDT']
LEVERAGE = 5
POSITION_SIZE = 60     # USDT
TIMEFRAME = '15m'      # Zaman aralığı

exchange = ccxt.bybit({
    'apiKey': API_KEY,
    'secret': API_SECRET,
    'enableRateLimit': True,
    'options': {'defaultType': 'future'},
})

positions = {s: None for s in SYMBOLS}

# --- Ana Bot Döngüsü ---
def main_loop():
    while True:
        for symbol in SYMBOLS:
            # 1. Veri çekme
            df = get_ohlcv(exchange, symbol, TIMEFRAME, limit=200)
            
            # 2. İndikatör analizleri
            signals = get_all_signals(df)
            
            # 3. Açık pozisyon var mı kontrol
            if positions[symbol] is not None:
                # Pozisyonu izle, stop/TP/risk yönetimini uygula
                manage_position(exchange, symbol, positions, signals)
                continue
            
            # 4. Trade sinyali oluştu mu?
            side = decide_signal(signals)
            if side:
                # 5. Risk kontrol
                pnl = 0.0  # Örn: gerçek PnL entegrasyonu eklenebilir
                risk_ok, reason = risk_control(pnl, time.time())
                if not risk_ok:
                    print(f"{symbol} - İşlem reddedildi: {reason}")
                    continue
                # 6. Emir gönder
                order = open_position(exchange, symbol, side, POSITION_SIZE, LEVERAGE)
                positions[symbol] = order
                send_telegram(f"{symbol} için {side.upper()} pozisyon açıldı!")
            
        time.sleep(60 * 8)   # 8 dakikada bir (isteğe göre)

# --- Modül fonksiyonlarını ekle ---
# get_ohlcv, get_all_signals, decide_signal, open_position, manage_position gibi fonksiyonlar

if __name__ == '__main__':
    main_loop()

# Bybit Otomatik Trade Botu - TP/SL Emirli Sistem
# --- TP/SL Emirleri Borsaya Gönderilir ---

import ccxt
import pandas as pd
import time
from datetime import datetime
from risk_management import risk_control  # Gelişmiş risk modülü
from indicator_module import *            # İndikatör analizleri burada
from telegram_notify import send_telegram # Telegram bildirimi

# === Kullanıcı parametreleri ===
API_KEY = 'BURAYA_KEY'
API_SECRET = 'BURAYA_SECRET'
SYMBOLS = ['BTC/USDT', 'ETH/USDT', 'SOL/USDT']
LEVERAGE = 5
POSITION_SIZE = 60     # USDT
TIMEFRAME = '15m'      # Zaman aralığı

exchange = ccxt.bybit({
    'apiKey': API_KEY,
    'secret': API_SECRET,
    'enableRateLimit': True,
    'options': {'defaultType': 'future'},
})

positions = {s: None for s in SYMBOLS}

def open_position_with_tp_sl(exchange, symbol, side, amount, leverage, tp, sl):
    """
    İşlem açılır açılmaz hem pozisyon açılır hem de TP ve SL emirleri borsaya gönderilir.
    """
    # Pozisyon aç
    order_side = 'buy' if side == 'long' else 'sell'
    params = {'reduce_only': False, 'leverage': leverage}
    order = exchange.create_market_order(symbol, order_side, amount, params=params)
    order_id = order['id']

    # TP/SL emirlerini gönder
    if side == 'long':
        # Take profit
        exchange.create_order(symbol, 'take_profit_market', 'sell', amount, tp, {
            'reduce_only': True,
            'stop_loss': sl,
            'leverage': leverage
        })
    else:
        exchange.create_order(symbol, 'take_profit_market', 'buy', amount, tp, {
            'reduce_only': True,
            'stop_loss': sl,
            'leverage': leverage
        })
    return order

# --- Ana Bot Döngüsü ---
def main_loop():
    while True:
        for symbol in SYMBOLS:
            # 1. Veri çekme
            df = get_ohlcv(exchange, symbol, TIMEFRAME, limit=200)
            
            # 2. İndikatör analizleri
            signals = get_all_signals(df)
            
            # 3. Açık pozisyon var mı kontrol
            if positions[symbol] is not None:
                # Pozisyonu izle, raporla
                manage_position(exchange, symbol, positions, signals)
                continue
            
            # 4. Trade sinyali oluştu mu?
            side, tp, sl = decide_signal(signals)  # Hem yön, hem TP/SL noktaları alınır
            if side:
                # 5. Risk kontrol
                pnl = 0.0  # Örn: gerçek PnL entegrasyonu eklenebilir
                risk_ok, reason = risk_control(pnl, time.time())
                if not risk_ok:
                    print(f"{symbol} - İşlem reddedildi: {reason}")
                    continue
                # 6. Emir gönder (TP/SL dahil)
                order = open_position_with_tp_sl(exchange, symbol, side, POSITION_SIZE, LEVERAGE, tp, sl)
                positions[symbol] = order
                send_telegram(f"{symbol} için {side.upper()} pozisyon açıldı! TP: {tp}, SL: {sl}")
        time.sleep(60 * 8)   # 8 dakikada bir (isteğe göre)

if __name__ == '__main__':
    main_loop()

# Bybit Otomatik Trade Botu (Ana Modül)
# --- Tufan Bey için, indikatör bazlı tam otomatik sistem ---

import ccxt
import pandas as pd
import time
from datetime import datetime
from risk_management import risk_control  # Gelişmiş risk modülü
from indicator_module import *            # İndikatör analizleri burada
from telegram_notify import send_telegram # Telegram bildirimi

# === Kullanıcı parametreleri ===
API_KEY = 'BURAYA_KEY'
API_SECRET = 'BURAYA_SECRET'
SYMBOLS = ['BTC/USDT', 'ETH/USDT', 'SOL/USDT']
LEVERAGE = 5
POSITION_SIZE = 60     # USDT
TIMEFRAME = '15m'      # Zaman aralığı

exchange = ccxt.bybit({
    'apiKey': API_KEY,
    'secret': API_SECRET,
    'enableRateLimit': True,
    'options': {'defaultType': 'future'},
})

positions = {s: None for s in SYMBOLS}

# --- Ana Bot Döngüsü ---
def main_loop():
    while True:
        for symbol in SYMBOLS:
            # 1. Veri çekme
            df = get_ohlcv(exchange, symbol, TIMEFRAME, limit=200)
            
            # 2. İndikatör analizleri
            signals = get_all_signals(df)
            
            # 3. Açık pozisyon var mı kontrol
            if positions[symbol] is not None:
                # Pozisyonu izle, stop/TP/risk yönetimini uygula
                manage_position(exchange, symbol, positions, signals)
                continue
            
            # 4. Trade sinyali oluştu mu?
            side = decide_signal(signals)
            if side:
                # 5. Risk kontrol
                pnl = 0.0  # Örn: gerçek PnL entegrasyonu eklenebilir
                risk_ok, reason = risk_control(pnl, time.time())
                if not risk_ok:
                    print(f"{symbol} - İşlem reddedildi: {reason}")
                    continue
                # 6. Emir gönder
                order = open_position(exchange, symbol, side, POSITION_SIZE, LEVERAGE)
                positions[symbol] = order
                send_telegram(f"{symbol} için {side.upper()} pozisyon açıldı!")
            
        time.sleep(60 * 8)   # 8 dakikada bir (isteğe göre)

# --- Modül fonksiyonlarını ekle ---
# get_ohlcv, get_all_signals, decide_signal, open_position, manage_position gibi fonksiyonlar

if __name__ == '__main__':
    main_loop()
