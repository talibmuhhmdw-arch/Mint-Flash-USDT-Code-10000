import pandas as pd
import pandas_ta as ta
import requests, time
from datetime import datetime, timedelta, timezone
from flask import Flask
from threading import Thread

# --- [ 1. إعداد سيرفر الويب لإبقاء البوت حياً ] ---
app = Flask('')

@app.route('/')
def home():
    return "✅ Bot is Online and Running 24/7!"

def run_web_server():
    # المنصات السحابية تحتاج تشغيل السيرفر على Port 8080 لتوليد رابط
    app.run(host='0.0.0.0', port=8080)

# --- [ 2. إعدادات البوت الأساسية ] ---
TOKEN = "8419527364:AAGjoA0tvSsjmZqVwsq3xA30-zwswoHqQnM"
CHAT_ID = "8356595702"

# رموز الذهب والفضة مقابل الدولار (Spot Gold & Silver)
SYMBOLS = {
    "XAUUSD=X": "الذهب 🟡",
    "XAGUSD=X": "الفضة ⚪"
}
TIMEFRAMES = ["1m", "5m", "15m"]
last_sent_signal = {"XAUUSD=X": None, "XAGUSD=X": None}

# --- [ 3. الوظائف التقنية ] ---

def get_data(symbol, interval):
    try:
        url = f"https://query1.finance.yahoo.com/v8/finance/chart/{symbol}?interval={interval}&range=1d"
        res = requests.get(url, headers={'User-Agent': 'Mozilla/5.0'}, timeout=15)
        data = res.json()['chart']['result'][0]
        df = pd.DataFrame(data['indicators']['quote'][0])
        # تعبئة القيم المفقودة (Forward Fill)
        df[['open', 'high', 'low', 'close']] = df[['open', 'high', 'low', 'close']].ffill()
        return df.dropna()
    except Exception as e:
        print(f"⚠️ خطأ في جلب بيانات {symbol}: {e}")
        return None

def check_news():
    """فحص أخبار الدولار عالية التأثير"""
    try:
        url = "https://nfs.faireconomy.media/ff_calendar_thisweek.json"
        events = requests.get(url, timeout=10).json()
        now = datetime.now(timezone.utc)
        for e in events:
            if e['country'] == 'USD' and e['impact'] == 'High':
                e_time = datetime.strptime(e['date'], '%Y-%m-%dT%H:%M:%S%z')
                if e_time - timedelta(minutes=30) <= now <= e_time + timedelta(minutes=15):
                    return True, e['title']
        return False, None
    except: return False, None

def analyze_logic(df):
    """تحليل فني للفريم المختار"""
    if df is None or len(df) < 25: return "Neutral", 0, 0

    # حساب المؤشرات (RSI, ATR, Bollinger Bands)
    rsi = ta.rsi(df['close'], length=14).iloc[-1]
    atr = ta.atr(df['high'], df['low'], df['close'], length=14).iloc[-1]
    bb = ta.bbands(df['close'], length=20, std=2)

    last_close = df['close'].iloc[-1]
    lower_b = bb['BBL_20_2.0'].iloc[-1]
    upper_b = bb['BBU_20_2.0'].iloc[-1]

    # كشف الانفجار السعري
    candle_size = abs(df['close'].iloc[-1] - df['open'].iloc[-1])

    if candle_size > (atr * 2):
        if last_close > upper_b: return "EXPLOSION_BUY", rsi, atr
        if last_close < lower_b: return "EXPLOSION_SELL", rsi, atr

    if rsi < 32 and last_close <= lower_b: return "BUY", rsi, atr
    if rsi > 68 and last_close >= upper_b: return "SELL", rsi, atr

    return "Neutral", rsi, atr

def send_telegram(msg):
    url = f"https://api.telegram.org/bot{TOKEN}/sendMessage"
    try:
        requests.post(url, data={"chat_id": CHAT_ID, "text": msg, "parse_mode": "Markdown"})
    except:
        print("❌ فشل في إرسال رسالة تليجرام")

# --- [ 4. الحلقة الرئيسية للبوت ] ---

def main_bot_loop():
    print("🚀 نظام قناص الذهب والفضة بدأ العمل...")
    send_telegram("🔔 *نظام المراقبة مفعل*\nالذهب والفضة مقابل الدولار (24/7)")

    while True:
        try:
            is_news, news_name = check_news()

            for symbol, name in SYMBOLS.items():
                results, atrs, rsis = {}, {}, {}

                # تحليل جميع الفريمات
                for tf in TIMEFRAMES:
                    df = get_data(symbol, tf)
                    res, r_rsi, r_atr = analyze_logic(df)
                    results[tf], rsis[tf], atrs[tf] = res, r_rsi, r_atr

                # السعر الحالي
                df_1m = get_data(symbol, "1m")
                if df_1m is None or df_1m.empty: continue
                current_price = round(df_1m['close'].iloc[-1], 2)

                # منطق الإشارة الموحدة
                final_signal = None
                if "BUY" in results['15m'] and "BUY" in results['5m']:
                    final_signal = "شراء قوي 🟢 (BUY)"
                elif "SELL" in results['15m'] and "SELL" in results['5m']:
                    final_signal = "بيع قوي 🔴 (SELL)"

                if "EXPLOSION" in results['5m'] or "EXPLOSION" in results['1m']:
                    final_signal = "⚠️ انفجار سعري 🔥 (Breakout)"

                # إرسال التنبيه إذا تغيرت الإشارة
                if final_signal and final_signal != last_sent_signal[symbol]:
                    target_atr = atrs['15m'] if atrs['15m'] > 0 else 0.5
                    sl = current_price - (target_atr * 2) if "BUY" in final_signal or "EXPLOSION_BUY" in results['5m'] else current_price + (target_atr * 2)
                    tp = current_price + (target_atr * 4) if "BUY" in final_signal or "EXPLOSION_BUY" in results['5m'] else current_price - (target_atr * 4)

                    news_tag = f"\n📢 *خبر مؤثر:* {news_name}" if is_news else ""

                    msg = (f"🎯 *إشارة قناص جديدة*\n"
                           f"━━━━━━━━━━━━━━━\n"
                           f"📦 الأصل: {name}\n"
                           f"📢 النوع: {final_signal}\n"
                           f"💰 السعر: {current_price}\n"
                           f"⏱ (15m): {results['15m']}\n"
                           f"⏱ (5m): {results['5m']}\n"
                           f"━━━━━━━━━━━━━━━\n"
                           f"🎯 الهدف: {round(tp, 2)}\n"
                           f"🛑 الوقف: {round(sl, 2)}\n"
                           f"⚖️ إدارة مخاطر: لوت 0.01 لكل 50$\n"
                           f"{news_tag}")

                    send_telegram(msg)
                    last_sent_signal[symbol] = final_signal
                    print(f"✅ تم إرسال إشارة لـ {name}")

                # إعادة التصفير عند هدوء السوق
                if 45 < rsis['5m'] < 55:
                    last_sent_signal[symbol] = None

        except Exception as e:
            print(f"❌ خطأ في الحلقة الرئيسية: {e}")

        time.sleep(60) # فحص كل دقيقة

# --- [ 5. تشغيل النظام ] ---
if __name__ == "__main__":
    # تشغيل سيرفر الويب في الخلفية لضمان استمرار الخدمة
    web_thread = Thread(target=run_web_server)
    web_thread.daemon = True
    web_thread.start()

    # تشغيل البوت
    main_bot_loop()
