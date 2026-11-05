"""
import argparse
import numpy as np
import pandas as pd
import yfinance as yf
 
# Yahoo-tickers för några stora bolag i Helsingfors 
TICKERS = {
    "NOKIA.HE": "Nokia", "KNEBV.HE": "Kone", "SAMPO.HE": "Sampo",
    "NESTE.HE": "Neste", "FORTUM.HE": "Fortum", "UPM.HE": "UPM",
    "STERV.HE": "Stora Enso", "NDA-FI.HE": "Nordea", "WRT1V.HE": "Wärtsilä",
    "ELISA.HE": "Elisa", "KESKOB.HE": "Kesko", "OUT1V.HE": "Outokumpu",
    "ORNBV.HE": "Orion", "VALMT.HE": "Valmet", "METSO.HE": "Metso",
    "CGCBV.HE": "Cargotec", "TYRES.HE": "Nokian Tyres", "KEMIRA.HE": "Kemira",
    "FIA1S.HE": "Finnair", "KCR.HE": "Konecranes", "TIETO.HE": "TietoEVRY",
    "FSKRS.HE": "Fiskars",
}
 
# Svenska namn -> finska namn som Yahoo använder för stad
STADSALIAS = {"helsingfors": "helsinki", "grankulla": "kauniainen", "esbo": "espoo",
              "tammerfors": "tampere", "åbo": "turku", "vanda": "vantaa", "uleåborg": "oulu"}
_stad_cache = {}
 
 
def hamta_stad(tick: str) -> str:
    """Bolagets säte enligt Yahoo Finance (kan saknas för vissa bolag)."""
    if tick not in _stad_cache:
        try:
            _stad_cache[tick] = (yf.Ticker(tick).info.get("city") or "okänd").strip()
        except Exception:
            _stad_cache[tick] = "okänd"
    return _stad_cache[tick]
 
 
def analysera(dagar_flode: int = 5) -> pd.DataFrame:
    data = yf.download(list(TICKERS), period="3mo", auto_adjust=True,
                       progress=False, group_by="ticker")
    rader = []
    for tick, namn in TICKERS.items():
        try:
            df = data[tick].dropna()
        except KeyError:
            print(f"Hoppar över {tick}: ingen data")
            continue
        if len(df) < 25:
            continue
        normal_vol = df["Volume"].iloc[-26:-1].mean()
        dag_andring = (df["Close"].iloc[-1] / df["Close"].iloc[-2] - 1) * 100
        omsattning = (df["Close"] * df["Volume"]).iloc[-1] / 1e6  # miljoner euro
        signerad = np.sign(df["Close"].diff()) * df["Volume"]
        flode = signerad.iloc[-dagar_flode:].sum() / normal_vol
        rader.append({
            "Bolag": namn,
            "Stad": hamta_stad(tick),
            "Kursändring %": round(dag_andring, 2),
            "Omsättning M€": round(omsattning, 1),
            "Volym vs normal": round(df["Volume"].iloc[-1] / normal_vol, 2),
            "Flöde (köp+/sälj-)": round(flode, 2),
        })
    return pd.DataFrame(rader)
 
 
if __name__ == "__main__":
    ap = argparse.ArgumentParser(description="Aktieflöde per stad (bolagets säte)")
    ap.add_argument("--stad", help="t.ex. helsingfors, tampere, grankulla, esbo")
    args = ap.parse_args()
 
    tabell = analysera()
    print("\nBolag per stad i datan:", tabell["Stad"].value_counts().to_dict())
    if args.stad:
        onskad = STADSALIAS.get(args.stad.lower(), args.stad.lower())
        tabell = tabell[tabell["Stad"].str.lower() == onskad]
        if tabell.empty:
            raise SystemExit(f"Inga bolag i listan har sitt säte i '{args.stad}'.")
    print("\nMEST HANDLADE (omsättning idag):")
    print(tabell.sort_values("Omsättning M€", ascending=False).head(5).to_string(index=False))
    print("\nSTÖRST KÖPTRYCK (senaste 5 dagarna):")
    print(tabell.sort_values("Flöde (köp+/sälj-)", ascending=False).head(3).to_string(index=False))
    print("\nSTÖRST SÄLJTRYCK (senaste 5 dagarna):")
    print(tabell.sort_values("Flöde (köp+/sälj-)").head(3).to_string(index=False))
 
