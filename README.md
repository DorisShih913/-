# -
傳統換算需自行查詢與計算，容易出現匯率過期或計算失誤。​  利用最新匯率資料自動計算、即時回饋，並以OLED顯示結果
import network
import urequests
import utime
from machine import Pin, I2C
from ssd1306 import SSD1306_I2C

# ==== Wi-Fi 連線設定 ====
WIFI_SSID = "DorisShih"
WIFI_PASS = "ssy91913"

def wifi_connect():
    wlan = network.WLAN(network.STA_IF)
    wlan.active(True)
    wlan.connect(WIFI_SSID, WIFI_PASS)
    print('正在連接WiFi...')
    while not wlan.isconnected():
        utime.sleep(1)
    print('已連接，IP:', wlan.ifconfig()[0])
    
# ==== Telegram Bot 設定 ====
BOT_TOKEN = "7835488288:AAGq4HkPF5HhmOg4TekOE-Rm0MJeGW_RC6U"
TELEGRAM_URL = f"https://api.telegram.org/bot{BOT_TOKEN}"

def get_updates(offset=None):
    url = TELEGRAM_URL + "/getUpdates"
    if offset:
        url += f"?offset={offset}"
    try:
        resp = urequests.get(url)
        result = resp.json()
        resp.close()
        return result
    except Exception as e:
        print("取得Telegram訊息錯誤：", e)
        return None

def send_message(chat_id, text):
    url = TELEGRAM_URL + f"/sendMessage?chat_id={chat_id}&text={text}"
    try:
        urequests.get(url).close()
    except Exception as e:
        print("發送訊息錯誤：", e)
        
# ==== SSD1306 OLED 初始化 ====
i2c = I2C(0, scl=Pin(22), sda=Pin(21))  # 依照你的接腳設定
oled = SSD1306_I2C(128, 64, i2c)

def oled_show_multiline(lines):
    oled.fill(0)
    for i, line in enumerate(lines):
        oled.text(line, 0, i * 16)  # 每行 16 pixel
    oled.show()

# ==== 匯率查詢 ====
def fetch_rates():
    url = "https://open.er-api.com/v6/latest/TWD"
    try:
        resp = urequests.get(url)
        if resp.status_code == 200:
            data = resp.json()
            resp.close()
            return data['rates']
        else:
            resp.close()
            return None
    except Exception as e:
        print("取得匯率時錯誤：", e)
        return None

def convert_to_twd(amount, currency, rates):
    if currency in rates:
        return amount / rates[currency]
    else:
        return None

def convert_from_twd(amount, currency, rates):
    if currency in rates:
        return amount * rates[currency]
    else:
        return None
        
# ==== 主程式 ====
wifi_connect()
oled_show_multiline(["Telegram Bot", "Ready..."])

offset = None
rates = fetch_rates()
oled_show_multiline(["Fetch Rate", "OK"])

while True:
    result = get_updates(offset)
    if result and 'result' in result and len(result['result']) > 0:
        for update in result['result']:
            offset = update['update_id'] + 1
            message = update['message']
            chat_id = message['chat']['id']
            text = message.get('text', '').strip()
            try:
                parts = text.split()
                if len(parts) == 2:
                    # 外幣換台幣 (ex: USD 100)
                    currency = parts[0].upper()
                    amount = float(parts[1])
                    if not rates or currency not in rates:
                        rates = fetch_rates()
                    twd = convert_to_twd(amount, currency, rates)
                    if twd:
                        reply = f"{amount} {currency}\n= {twd:.2f} TWD"
                        send_message(chat_id, reply.replace('\n', ' '))
                        oled_show_multiline([f"{amount} {currency}", f"= {twd:.2f} TWD"])
                    else:
                        send_message(chat_id, "找不到此幣別")
                        oled_show_multiline(["無法換算"])
                elif len(parts) == 3:
                    # 台幣換外幣 (ex: TWD USD 100)
                    from_currency = parts[0].upper()
                    to_currency = parts[1].upper()
                    amount = float(parts[2])
                    if from_currency == "TWD" and to_currency in rates:
                        if not rates or to_currency not in rates:
                            rates = fetch_rates()
                        result_amt = convert_from_twd(amount, to_currency, rates)
                        if result_amt:
                            reply = f"{amount} TWD\n= {result_amt:.2f} {to_currency}"
                            send_message(chat_id, reply.replace('\n', ' '))
                            oled_show_multiline([f"{amount} TWD", f"= {result_amt:.2f} {to_currency}"])
                        else:
                            send_message(chat_id, "找不到此幣別")
                            oled_show_multiline(["無法換算"])
                    else:
                        send_message(chat_id, "格式錯誤，請用 'TWD USD 100'")
                        oled_show_multiline(["格式錯誤"])
                else:
                    send_message(chat_id, "格式錯誤，請用 'USD 100' 或 'TWD USD 100'")
                    oled_show_multiline(["格式錯誤"])
            except Exception as e:
                send_message(chat_id, f"錯誤：{e}")
                oled_show_multiline(["發生錯誤"])
    utime.sleep(2)
