import os
import re
import time
import threading
import requests

TELEGRAM_BOT_TOKEN = os.getenv("TELEGRAM_BOT_TOKEN", "").strip()
TELEGRAM_CHAT_ID = os.getenv("TELEGRAM_CHAT_ID", "").strip()
CHECK_INTERVAL = int(os.getenv("CHECK_INTERVAL", "60"))

SUCCESS_PHRASES = (
    "payment success",
    "thank you for choosing tamara",
)

NOT_FOUND_PHRASES = (
    "this page isn’t available",
    "this page isn't available",
    "the request url was not found on the server",
    "go to homepage",
)

tracked_items: dict[str, dict] = {}
lock = threading.Lock()


def send_telegram_message(chat_id: str, text: str) -> None:
    if not TELEGRAM_BOT_TOKEN or not chat_id:
        print("تنبيه: لم يتم ضبط TELEGRAM_BOT_TOKEN أو chat_id")
        return

    api_url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/sendMessage"
    payload = {
        "chat_id": chat_id,
        "text": text,
    }
    try:
        response = requests.post(api_url, json=payload, timeout=15)
        response.raise_for_status()
    except requests.RequestException as e:
        print(f"خطأ أثناء إرسال رسالة تيليجرام: {e}")


def extract_url_and_label(message_text: str) -> tuple[str | None, str]:
    url_match = re.search(r"(https?://\S+)", message_text)
    if not url_match:
        return None, ""
    url = url_match.group(1).strip()
    label = message_text.replace(url, "").strip(" -:\n\t")
    return url, label


def detect_target_state(url: str) -> str:
    headers = {
        "User-Agent": (
            "Mozilla/5.0 (Linux; Android 14; Honor Pad X9a) "
            "AppleWebKit/537.36 (KHTML, like Gecko) "
            "Chrome/128.0.0.0 Mobile Safari/537.36"
        )
    }
    response = requests.get(url, headers=headers, timeout=20, allow_redirects=True)
    text_lower = response.text.lower()

    if any(phrase in text_lower for phrase in SUCCESS_PHRASES):
        return "PAYMENT_SUCCESS"

    if response.status_code == 404 or any(phrase in text_lower for phrase in NOT_FOUND_PHRASES):
        return "PAGE_404"

    return f"OTHER_{response.status_code}"


def telegram_listener() -> None:
    offset = 0
    base_url = f"https://api.telegram.org/bot{TELEGRAM_BOT_TOKEN}/getUpdates"

    while True:
        try:
            resp = requests.get(base_url, params={"timeout": 25, "offset": offset}, timeout=30)
            data = resp.json()
            if not data.get("ok"):
                time.sleep(3)
                continue

            for update in data.get("result", []):
                offset = update["update_id"] + 1
                msg = update.get("message") or {}
                text = (msg.get("text") or "").strip()
                chat_id = str((msg.get("chat") or {}).get("id", ""))

                if not text or not chat_id:
                    continue

                if TELEGRAM_CHAT_ID and chat_id != TELEGRAM_CHAT_ID:
                    continue

                if text == "/list":
                    with lock:
                        if not tracked_items:
                            send_telegram_message(chat_id, "لا توجد روابط قيد المراقبة حالياً.")
                        else:
                            lines = [
                                f"- {info['label'] or 'بدون رقم'}: {u}"
                                for u, info in tracked_items.items()
                            ]
                            send_telegram_message(chat_id, "الروابط قيد المراقبة:\n" + "\n".join(lines))
                    continue

                if text.startswith("/del"):
                    url_to_del, _ = extract_url_and_label(text)
                    with lock:
                        if url_to_del and url_to_del in tracked_items:
                            del tracked_items[url_to_del]
                            send_telegram_message(chat_id, f"تم حذف الرابط من المراقبة:\n{url_to_del}")
                        else:
                            send_telegram_message(chat_id, "أرسلي /del متبوعاً بالرابط لحذفه.")
                    continue

                url, label = extract_url_and_label(text)
                if url:
                    with lock:
                        tracked_items[url] = {
                            "label": label,
                            "last_state": None,
                            "chat_id": chat_id,
                        }
                    label_display = label if label else "غير محدد"
                    send_telegram_message(
                        chat_id,
                        f"تمت إضافة الرابط للمراقبة بنجاح.\nالرقم: {label_display}\nالرابط: {url}",
                    )
                else:
                    send_telegram_message(
                        chat_id,
                        "أرسلي الرابط مع الرقم في نفس الرسالة ليتم مراقبته فوراً.",
                    )

        except Exception as e:
            print(f"خطأ في استقبال رسائل تيليجرام: {e}")
            time.sleep(5)


def monitor_loop() -> None:
    while True:
        with lock:
            items_snapshot = list(tracked_items.items())

        for url, info in items_snapshot:
            try:
                current_state = detect_target_state(url)
                previous_state = info["last_state"]
                chat_id = info.get("chat_id") or TELEGRAM_CHAT_ID
                label_text = info["label"] if info["label"] else "بدون رقم"

                if current_state in ("PAYMENT_SUCCESS", "PAGE_404") and current_state != previous_state:
                    status_ar = (
                        "تم السداد بنجاح (Payment success)"
                        if current_state == "PAYMENT_SUCCESS"
                        else "الصفحة غير متاحة (404)"
                    )
                    alert_msg = (
                        f"تم السداد / تحديث الحالة: {status_ar}\n"
                        f"الرقم: {label_text}\n"
                        f"الرابط: {url}"
                    )
                    send_telegram_message(chat_id, alert_msg)

                with lock:
                    if url in tracked_items:
                        tracked_items[url]["last_state"] = current_state

            except requests.RequestException as e:
                print(f"تعذر فحص الرابط ({url}): {e}")

        time.sleep(CHECK_INTERVAL)


if __name__ == "__main__":
    if not TELEGRAM_BOT_TOKEN:
        print("يرجى ضبط TELEGRAM_BOT_TOKEN في متغيرات البيئة.")
    else:
        listener_thread = threading.Thread(target=telegram_listener, daemon=True)
        listener_thread.start()
        monitor_loop()
