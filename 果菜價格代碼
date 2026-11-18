"""每天抓取農業部批發市場蔬果行情，整理成 data/latest.json。只用 Python 內建模組。"""
import datetime as dt
import json
import os
import sys
import time
import urllib.parse
import urllib.request
from collections import defaultdict

API = "https://data.moa.gov.tw/api/v1/AgriProductsTransType/"
DAYS_BACK = 14          # 往回抓幾天（用來算近一週均價，也能跨過休市日）
TZ = dt.timezone(dt.timedelta(hours=8))


def roc(d):
    return f"{d.year - 1911}.{d.month:02d}.{d.day:02d}"


def pick(rec, *keys):
    for k in keys:
        v = rec.get(k)
        if v not in (None, ""):
            return v
    return None


def num(v):
    try:
        return float(str(v).replace(",", ""))
    except (TypeError, ValueError):
        return 0.0


def fetch_day(d):
    qs = urllib.parse.urlencode({"Start_time": roc(d), "End_time": roc(d)})
    req = urllib.request.Request(API + "?" + qs, headers={"User-Agent": "veg-prices/1.0"})
    for attempt in range(3):
        try:
            with urllib.request.urlopen(req, timeout=60) as r:
                body = json.load(r)
            break
        except Exception as e:  # noqa: BLE001
            print(f"  第 {attempt + 1} 次失敗：{e}")
            time.sleep(3)
    else:
        return []
    if isinstance(body, dict):
        body = body.get("Data") or body.get("data") or []
    return body if isinstance(body, list) else []


def day_prices(rows):
    """把一天的所有市場資料，合併成 {品名: (類別, 加權均價, 總交易量)}"""
    acc = defaultdict(lambda: [0.0, 0.0, 0.0, 0])  # 價*量, 量, 價合計, 筆數
    cat = {}
    for r in rows:
        code = str(pick(r, "TcType", "種類代碼") or "")
        c = "v" if code.startswith("N04") else "f" if code.startswith("N05") else None
        name = pick(r, "CropName", "作物名稱")
        price = num(pick(r, "Avg_Price", "平均價"))
        qty = num(pick(r, "Trans_Quantity", "交易量"))
        if not c or not name or price <= 0:
            continue
        a = acc[name]
        a[0] += price * qty
        a[1] += qty
        a[2] += price
        a[3] += 1
        cat[name] = c
    return {n: (cat[n], (a[0] / a[1] if a[1] > 0 else a[2] / a[3]), a[1]) for n, a in acc.items()}


def main():
    today = dt.datetime.now(TZ).date()
    days = {}
    for i in range(DAYS_BACK):
        d = today - dt.timedelta(days=i)
        rows = fetch_day(d)
        print(roc(d), len(rows), "筆")
        if rows and not days:
            print("  欄位範例：", list(rows[0].keys()))
        got = day_prices(rows)
        if got:
            days[d] = got
        time.sleep(0.5)

    if not days:
        sys.exit("抓不到任何蔬果資料，已停止（不會覆蓋舊資料）。")

    latest = max(days)
    older = [days[d] for d in sorted(days, reverse=True) if d < latest][:7]
    items = []
    for name, (c, price, qty) in days[latest].items():
        hist = [o[name][1] for o in older if name in o]
        avg = sum(hist) / len(hist) if hist else None
        items.append({"n": name, "c": c, "p": round(price, 1),
                      "a": round(avg, 1) if avg else None, "q": round(qty)})
    items.sort(key=lambda x: x["n"])

    if len(items) < 20:
        sys.exit(f"品項只有 {len(items)} 個，資料可能不完整，已停止。")

    os.makedirs("data", exist_ok=True)
    out = {"date": latest.isoformat(),
           "updated": dt.datetime.now(TZ).isoformat(timespec="minutes"),
           "items": items}
    with open("data/latest.json", "w", encoding="utf-8") as f:
        json.dump(out, f, ensure_ascii=False, separators=(",", ":"))
    print(f"完成：{latest}，共 {len(items)} 個品項")


if __name__ == "__main__":
    main()
