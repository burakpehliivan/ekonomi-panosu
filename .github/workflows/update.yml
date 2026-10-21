#!/usr/bin/env python3
"""Ekonomi panosu veri toplayıcı. Eksik veriyi ASLA tahmin etmez; hata olursa kaydeder, son geçerli veriyi korur."""
import os, sys, json, ssl, datetime as dt, xml.etree.ElementTree as ET
import requests
from requests.adapters import HTTPAdapter

OUT = "data/data.json"
# EVDS seri kodları: "discover" çalıştırması sonrası doldurulacak (tahminle yazılmaz)
EVDS = {"policy": "", "ticari": "", "ihtiyac": "", "konut": ""}
ALTIN_LIMIT = 28      # Altınapi ücretsiz deneme: ayda 30 istek; 2 istek pay bırakılır
EMA_TEYIT_GUN = 3     # kesişimden sonra aynı tarafta kalınması gereken işlem günü
now = dt.datetime.now(dt.timezone.utc)


class Legacy(HTTPAdapter):  # TCMB sunucusu eski SSL ayarı ister
    def init_poolmanager(self, *a, **k):
        c = ssl.create_default_context(); c.options |= 0x4
        k["ssl_context"] = c
        super().init_poolmanager(*a, **k)


S = requests.Session(); S.mount("https://", Legacy())
S.headers["User-Agent"] = "ekonomi-panosu/1.0"


def get(url, **k):
    r = S.get(url, timeout=30, **k); r.raise_for_status(); return r


def redact(s):  # anahtarlar hata mesajıyla dosyaya sızmasın
    for n in ("FRED_API_KEY", "EVDS_API_KEY", "ALTINAPI_KEY"):
        if os.environ.get(n): s = s.replace(os.environ[n], "***")
    return s


def iso(t):
    for f in ("%d-%m-%Y", "%Y-%m-%d", "%Y-%m"):
        try: return dt.datetime.strptime(str(t), f).date().isoformat()
        except ValueError: pass
    raise ValueError("tarih okunamadı: " + str(t))


def fred(sid, n=1):
    r = get("https://api.stlouisfed.org/fred/series/observations", params=dict(
        series_id=sid, api_key=os.environ["FRED_API_KEY"], file_type="json",
        sort_order="desc", limit=n + 15)).json()
    o = [(x["date"], float(x["value"])) for x in r["observations"] if x["value"] != "."][:n]
    if not o: raise ValueError(sid + ": veri yok")
    return o[::-1]  # eski -> yeni


def f_fed():
    u, l = fred("DFEDTARU")[-1], fred("DFEDTARL")[-1]
    return dict(upper=u[1], lower=l[1]), u[0]


def f_evds(key):
    code = EVDS[key]
    if not code: raise ValueError("EVDS seri kodu henüz girilmedi")
    d = (now - dt.timedelta(days=150)).strftime("%d-%m-%Y"); e = now.strftime("%d-%m-%Y")
    r = get(f"https://evds3.tcmb.gov.tr/service/evds/series={code}&startDate={d}&endDate={e}&type=json",
            headers={"key": os.environ["EVDS_API_KEY"]}).json()
    col = code.replace(".", "_")
    o = [(x["Tarih"], float(x[col])) for x in r.get("items", []) if x.get(col) not in (None, "")]
    if not o: raise ValueError(code + ": veri yok")
    return dict(value=o[-1][1], series=code), iso(o[-1][0])


def f_tcmb():
    x = ET.fromstring(get("https://www.tcmb.gov.tr/kurlar/today.xml").content)
    c = x.find("Currency[@CurrencyCode='USD']")
    return (dict(buy=float(c.findtext("ForexBuying")), sell=float(c.findtext("ForexSelling"))),
            dt.datetime.strptime(x.get("Tarih"), "%d.%m.%Y").date().isoformat())


def f_altin(meta):
    ind = meta.get("_prev_altin", {})
    if ind.get("last_ok", "")[:10] == now.date().isoformat() or now.weekday() > 4:
        return None  # bugün alındı ya da hafta sonu: istek harcama
    cnt = meta.setdefault("altin_requests", {}); m = now.strftime("%Y-%m")
    if cnt.get(m, 0) >= ALTIN_LIMIT: raise RuntimeError(f"Aylık istek koruması doldu ({ALTIN_LIMIT})")
    cnt[m] = cnt.get(m, 0) + 1  # istekten önce say
    from altinapi import AltinapiClient
    p = AltinapiClient(api_key=os.environ["ALTINAPI_KEY"]).get_price("USDTRY")
    ts = p.get("timestamp")
    try: d = (dt.datetime.fromtimestamp(float(ts) / (1000 if float(ts) > 1e11 else 1), dt.timezone.utc)
              if str(ts).replace(".", "").isdigit() else dt.datetime.fromisoformat(str(ts).replace("Z", "+00:00"))).date().isoformat()
    except Exception: d = now.date().isoformat()
    return dict(buy=float(p["bid"]), sell=float(p["ask"]), snapshot_utc=now.isoformat(timespec="minutes")), d


def f_eur():
    j = get("https://data-api.ecb.europa.eu/service/data/EXR/D.USD.EUR.SP00.A",
            params=dict(lastNObservations=30, format="jsondata")).json()
    ob = next(iter(j["dataSets"][0]["series"].values()))["observations"]
    ds = [v["id"] for v in j["structure"]["dimensions"]["observation"][0]["values"]]
    pts = [(ds[int(i)], v[0]) for i, v in sorted(ob.items(), key=lambda a: int(a[0]))]
    return dict(last=pts[-1][1], recent=pts[-5:]), pts[-1][0]


def f_vix():
    o = fred("VIXCLS", 6)
    days = [dict(date=o[i][0], value=o[i][1], change=round(o[i][1] - o[i - 1][1], 2)) for i in range(1, len(o))]
    diff = days[-1]["value"] - o[0][1]
    return dict(days=days, direction="yukari" if diff > 0 else "asagi" if diff < 0 else "yatay"), o[-1][0]


def ema(v, n):
    k = 2 / (n + 1); e = [sum(v[:n]) / n]  # SMA ile başlat
    for x in v[n:]: e.append(x * k + e[-1] * (1 - k))
    return e  # e[j] -> v[n-1+j]


def f_spx():
    o = fred("SP500", 400); d = [x[0] for x in o]; v = [x[1] for x in o]
    e9, e14 = ema(v, 9), ema(v, 14)
    df = [(d[i], e9[i - 8] - e14[i - 13]) for i in range(13, len(v))]
    side = [1 if x[1] > 0 else -1 for x in df]
    ch = [j for j in range(1, len(side)) if side[j] != side[j - 1]]
    mk = lambda j: dict(date=df[j][0], dir="yukari" if side[j] == 1 else "asagi")
    cross = mk(ch[-1]) if ch else None
    prev = mk(ch[-2]) if len(ch) > 1 else None
    held = len(df) - ch[-1] if ch else None
    state = ("kesisim_yok" if not ch else ("yukselis" if side[-1] == 1 else "dusus") if held >= EMA_TEYIT_GUN else "teyit_bekliyor")
    return dict(price=v[-1], ema9=e9[-1], ema14=e14[-1], above=side[-1] == 1, state=state, held=held,
                teyit_gun=EMA_TEYIT_GUN, cross=cross, prev_cross=prev, n_obs=len(v), first_date=d[0],
                freq="günlük kapanış (FRED SP500)"), d[-1]


JOBS = {
    "fed": ("FED hedef faiz aralığı", "https://fred.stlouisfed.org/series/DFEDTARU", f_fed),
    "policy": ("TCMB politika faizi", "https://evds3.tcmb.gov.tr", lambda: f_evds("policy")),
    "ticari": ("Ticari kredi faizi", "https://evds3.tcmb.gov.tr", lambda: f_evds("ticari")),
    "ihtiyac": ("İhtiyaç kredisi faizi", "https://evds3.tcmb.gov.tr", lambda: f_evds("ihtiyac")),
    "konut": ("Konut kredisi faizi", "https://evds3.tcmb.gov.tr", lambda: f_evds("konut")),
    "tcmb_usd": ("TCMB USD kuru", "https://www.tcmb.gov.tr/kurlar/today.xml", f_tcmb),
    "kapalicarsi": ("Harem Altın USD (Altınapi)", "https://altinapi.com", None),
    "eurusd": ("EUR/USD (ECB)", "https://data-api.ecb.europa.eu", f_eur),
    "vix": ("VIX", "https://fred.stlouisfed.org/series/VIXCLS", f_vix),
    "spx": ("S&P 500 EMA 9/14", "https://fred.stlouisfed.org/series/SP500", f_spx),
}


def run(mode="all"):  # all | main (Altınapi hariç) | altin (yalnız Altınapi)
    old = json.load(open(OUT)) if os.path.exists(OUT) else {}
    ind, meta = old.get("ind", {}), old.get("meta", {})
    for k, (label, url, fn) in JOBS.items():
        if (mode == "main" and k == "kapalicarsi") or (mode == "altin" and k != "kapalicarsi"): continue
        r = ind.setdefault(k, {}); r.update(label=label, url=url)
        try:
            if k == "kapalicarsi":
                meta["_prev_altin"] = {"last_ok": r.get("last_ok", "")}
                res = f_altin(meta); meta.pop("_prev_altin")
            else: res = fn()
            if res is None: continue
            r.update(payload=res[0], data_date=res[1], last_ok=now.isoformat(timespec="seconds"), error=None)
        except Exception as e:
            meta.pop("_prev_altin", None)
            r["error"] = redact(f"{type(e).__name__}: {e}")[:300]
            print("HATA", k, r["error"])
    os.makedirs("data", exist_ok=True)
    json.dump(dict(generated=now.isoformat(timespec="seconds"), ind=ind, meta=meta),
              open(OUT, "w"), ensure_ascii=False, indent=1)


def discover():  # EVDS seri adlarını listeler; kodları ben ya da sen buradan seçersin
    b = "https://evds3.tcmb.gov.tr/service/evds/"; h = {"key": os.environ["EVDS_API_KEY"]}
    for g in get(b + "datagroups/mode=0&type=json", headers=h).json():
        name = g.get("DATAGROUP_NAME", "")
        if any(w in name.casefold() for w in ("kredi faiz", "politika", "repo ihale")):
            print("GRUP", g["DATAGROUP_CODE"], "|", name)
            for s in get(b + "serieList/type=json&code=" + g["DATAGROUP_CODE"], headers=h).json():
                print("   ", s["SERIE_CODE"], "|", s["SERIE_NAME"])


if __name__ == "__main__":
    a = sys.argv[1] if len(sys.argv) > 1 else "all"
    discover() if a == "discover" else run(a)
