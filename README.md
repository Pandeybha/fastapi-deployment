from fastapi import FastAPI, HTTPException, Query
from pydantic import BaseModel
from typing import Optional, List
from pathlib import Path
import pandas as pd

app = FastAPI(title="Play Store Data API")

# === Load the Excel file sitting next to main.py ===
BASE_DIR = Path(__file__).parent
FILE_PATH = BASE_DIR / "Play Store Data (1).xlsx"   # change name if yours differs

try:
    df = pd.read_excel(FILE_PATH)
except FileNotFoundError:
    raise RuntimeError(
        f"Excel file not found at: {FILE_PATH}.\n"
        f"Place 'Play Store Data (1).xlsx' next to main.py or update FILE_PATH."
    )
except Exception as e:
    raise RuntimeError(f"Failed to read Excel: {e}")

# --- Helper: find columns regardless of exact spelling/case/spaces ---
norm = {c.lower().replace(" ", "_"): c for c in df.columns}
def getcol(*candidates: str) -> Optional[str]:
    for c in candidates:
        key = c.lower().replace(" ", "_")
        if key in norm:
            return norm[key]
    return None

APP_COL = getcol("App")
CAT_COL = getcol("Category")
RAT_COL = getcol("Rating")
REV_COL = getcol("Reviews")
INS_COL = getcol("Installs")
TYP_COL = getcol("Type")
PRC_COL = getcol("Price")
CR_COL  = getcol("Content_Rating", "Content Rating")

# --- Safe converters to avoid 500s on weird/empty cells ---
def to_int_safe(v) -> Optional[int]:
    try:
        if v is None or (isinstance(v, float) and pd.isna(v)) or pd.isna(v):
            return None
    except Exception:
        pass
    if isinstance(v, str):
        s = v.strip().replace(",", "")
        if s == "":
            return None
        # handles strings like "123.0"
        try:
            return int(float(s))
        except Exception:
            return None
    try:
        return int(v)
    except Exception:
        return None

def to_float_safe(v) -> Optional[float]:
    try:
        if v is None or (isinstance(v, float) and pd.isna(v)) or pd.isna(v):
            return None
    except Exception:
        pass
    if isinstance(v, str):
        s = v.strip()
        if s == "":
            return None
        try:
            return float(s)
        except Exception:
            return None
    try:
        return float(v)
    except Exception:
        return None

def to_str_safe(v) -> Optional[str]:
    try:
        if v is None or pd.isna(v):
            return None
    except Exception:
        pass
    s = str(v)
    return s if s != "" else None

# --- Response model (all optional so missing columns don’t crash) ---
class AppData(BaseModel):
    App: Optional[str] = None
    Category: Optional[str] = None
    Rating: Optional[float] = None
    Reviews: Optional[int] = None
    Installs: Optional[str] = None
    Type: Optional[str] = None
    Price: Optional[str] = None
    Content_Rating: Optional[str] = None

def row_to_appdata(rec: dict) -> AppData:
    return AppData(
        App            = to_str_safe(rec.get(APP_COL)) if APP_COL else None,
        Category       = to_str_safe(rec.get(CAT_COL)) if CAT_COL else None,
        Rating         = to_float_safe(rec.get(RAT_COL)) if RAT_COL else None,
        Reviews        = to_int_safe(rec.get(REV_COL)) if REV_COL else None,
        Installs       = to_str_safe(rec.get(INS_COL)) if INS_COL else None,
        Type           = to_str_safe(rec.get(TYP_COL)) if TYP_COL else None,
        Price          = to_str_safe(rec.get(PRC_COL)) if PRC_COL else None,
        Content_Rating = to_str_safe(rec.get(CR_COL))  if CR_COL  else None,
    )

@app.get("/")
def home():
    return {
        "message": "Play Store Data API is running ✅",
        "rows": int(len(df)),
        "columns": list(df.columns),
    }

from typing import Optional

@app.get("/apps", response_model=List[AppData])
def list_apps(limit: Optional[str] = Query("10")):
    # accept strings like "", "10", or bad text, and coerce to int safely
    try:
        n = int(limit) if limit is not None and str(limit).strip() != "" else 10
    except Exception:
        n = 10
    n = max(1, min(n, 100))  # clamp 1..100
    subset = df.head(n)
    return [row_to_appdata(rec) for rec in subset.to_dict(orient="records")]


@app.get("/apps/search", response_model=List[AppData])
def search_apps(q: str = Query(..., min_length=1), limit: int = Query(20, ge=1, le=100)):
    """Case-insensitive substring search by app name."""
    if not APP_COL:
        raise HTTPException(status_code=400, detail="App name column not found in Excel.")
    mask = df[APP_COL].astype(str).str.contains(q, case=False, na=False)
    subset = df[mask].head(limit)
    if subset.empty:
        raise HTTPException(status_code=404, detail="No matching apps.")
    return [row_to_appdata(rec) for rec in subset.to_dict(orient="records")]

@app.get("/apps/{app_name}", response_model=AppData)
def get_app(app_name: str):
    """Exact match by app name (case-insensitive)."""
    if not APP_COL:
        raise HTTPException(status_code=400, detail="App name column not found in Excel.")
    mask = df[APP_COL].astype(str).str.lower() == app_name.lower()
    subset = df[mask]
    if subset.empty:
        raise HTTPException(status_code=404, detail="App not found.")
    rec = subset.iloc[0].to_dict()
    return row_to_appdata(rec)
