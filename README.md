# Purview-Data-Risk-Assessment-analyzer
A python script that will automatically label your sharepoint sites with Low, Medium or High for oversharing. It looks for keywords, sensitive info vs how many people are accessing the information etc.

import pandas as pd
import numpy as np

# ============================================================
#  CONFIGURATION
# ============================================================

INPUT_FILE = "Report_DataOversharingworkshop.xlsx"
OUTPUT_FILE = "Purview_DataRisk_scored.xlsx"
SHEET_NAME = "Data risk assessment results"

# Department / function keywords (same as before)
HIGH_DEPT_KEYWORDS = [
    # ---- Executive / leadership ----
    "ceo", "cfo", "coo", "cio", "cto", "ciso",
    "executive", "leadership", "board", "board of directors",
    "president", "chancellor", "provost", "dean",

    # ---- HR / people / identity ----
    "hr", "human resources", "people team", "people ops",
    "payroll", "benefits", "compensation", "salary", "salaries",
    "talent", "recruiting", "recruitment", "performance reviews",
    "personnel", "employee relations",

    # ---- Finance / accounting / payments ----
    "finance", "financial", "general ledger", "gl",
    "accounts payable", "ap-", "payables", "accounts receivable",
    "receivables", "billing", "invoicing", "treasury",
    "cash management", "tax", "taxes", "budget office",
    "controller", "controllership",

    # ---- Legal / compliance / risk ----
    "legal", "contracts", "contracting", "nda", "non-disclosure",
    "litigation", "discovery", "compliance", "ethics",
    "regulatory", "regulation", "audit", "auditor", "risk management",

    # ---- EDU: student identifiable data / core systems ----
    "student records", "studentrecords", "student services",
    "sis", "student information system", "registrar",
    "grades", "transcripts", "discipline", "special education",
    "iep", "504", "ell", "esl", "counseling records",
    "financial aid", "bursar",

    # ---- Healthcare: PHI / clinical data ----
    "patient", "patients", "phi", "ephi", "hipaa",
    "ehr", "emr", "epic", "cerner", "meditech", "clinical",
    "radiology", "lab results", "laboratory", "pharmacy",
    "oncology", "cardiology", "icu", "unit", "care team",
    "claims", "medical records",

    # ---- Government: citizen records / casework ----
    "case management", "case files", "child welfare",
    "social services", "benefits administration",
    "unemployment", "public assistance",
    "veterans services", "dmv", "drivers license",
    "court records", "prosecution", "public defender",
    "district attorney", "sheriff", "police", "law enforcement",
    "corrections", "parole", "probation",

    # ---- Anything explicitly labeled confidential/sensitive ----
    "restricted", "confidential", "secret", "internal only",
]

MED_DEPT_KEYWORDS = [
    # ---- Commercial / general business functions ----
    "sales", "account management", "customer success",
    "crm", "marketing", "campaigns", "events", "partners",
    "channel", "product management",

    # ---- Operations / supply chain / projects ----
    "operations", "supply chain", "logistics",
    "warehouse", "inventory", "procurement", "purchasing",
    "projects", "program management", "pm office", "pmo",

    # ---- EDU: teaching & curriculum (usually some sensitivity) ----
    "curriculum", "assessment", "learning resources",
    "instructional materials", "faculty", "department chair",
    "gradebook", "class rosters", "course planning",

    # ---- Healthcare: scheduling / non-clinical ops ----
    "scheduling", "clinic admin", "referrals", "intake",
    "provider relations", "network management",

    # ---- Government: permitting / licensing / planning ----
    "licensing", "permits", "planning", "zoning",
    "land records", "property records", "tax assessor",
    "elections", "voter registration",

    # ---- Security / IT / misc important shared areas ----
    "security", "information security", "infosec",
    "it operations", "infrastructure", "identity",
    "helpdesk", "service desk", "support desk",
]

# ============================================================
#  HELPER FUNCTIONS
# ============================================================

def load_assessment(path: str) -> pd.DataFrame:
    """Load the Purview DSPM export and locate the real header row."""
    raw = pd.read_excel(path, sheet_name=SHEET_NAME, header=None)
    header_row_index = raw.index[raw.iloc[:, 0] == "Data source ID"][0]
    headers = raw.iloc[header_row_index]
    df = raw.iloc[header_row_index + 1 :].copy()
    df.columns = headers
    return df.reset_index(drop=True)


def get_dept_hint(value: str) -> str:
    """Return High/Medium/None based on presence of department keywords."""
    if not isinstance(value, str):
        return "None"
    text = value.lower()
    for kw in HIGH_DEPT_KEYWORDS:
        if kw and kw.lower() in text:
            return "High"
    for kw in MED_DEPT_KEYWORDS:
        if kw and kw.lower() in text:
            return "Medium"
    return "None"


def compute_thresholds(series: pd.Series, default_low: float, default_high: float):
    """
    Compute median and 75th percentile thresholds for a metric.
    Fallback to defaults if there isn't enough data.
    """
    s = series.replace([np.inf, -np.inf], np.nan).dropna()
    s = s[s > 0]  # ignore zeros, they’re not “real” activity/sensitivity
    if len(s) == 0:
        return default_low, default_high

    med = s.quantile(0.5)
    hi = s.quantile(0.75)

    # Ensure hi >= med, with some basic sanity
    if hi < med:
        hi = med

    return float(med), float(hi)


def add_calculated_columns_and_thresholds(df: pd.DataFrame):
    """Compute derived metrics and dynamic thresholds."""
    numeric_cols = [
        "Total items",
        "Total items accessed",
        "Total scanned items",
        "Total unscanned data",
        "Total sensitive items",
        "Times users accessed items",
        "Unique users accessing items",
    ]
    for col in numeric_cols:
        if col in df.columns:
            df[col] = pd.to_numeric(df[col], errors="coerce").fillna(0)
        else:
            df[col] = 0

    # Percent sensitive
    df["SensitivePct"] = df["Total sensitive items"] / df["Total scanned items"].replace(0, pd.NA)

    # Percent unscanned
    total_scanned = df["Total scanned items"]
    total_unscanned = df["Total unscanned data"]
    df["UnscannedPct"] = total_unscanned / (total_scanned + total_unscanned).replace(0, pd.NA)

    # External sharing flag
    df["ExternalSharing"] = df["Items shared with"].astype(str).str.contains(
        "everyone|anyone|external|public", case=False, na=False
    )

    # DeptHint from ID + optional name
    src_id_col = "Data source ID"
    name_col = "Data source name" if "Data source name" in df.columns else None

    def compute_dept(row):
        id_val = str(row.get(src_id_col, ""))
        name_val = str(row.get(name_col, "")) if name_col else ""
        hint_id = get_dept_hint(id_val)
        hint_name = get_dept_hint(name_val)
        if "High" in (hint_id, hint_name):
            return "High"
        if "Medium" in (hint_id, hint_name):
            return "Medium"
        return "None"

    df["DeptHint"] = df.apply(compute_dept, axis=1)

    # ---- Dynamic thresholds (range-based) -------------------
    thresholds = {}

    thresholds["sens_med"], thresholds["sens_high"] = compute_thresholds(
        df["SensitivePct"], default_low=0.05, default_high=0.20
    )
    thresholds["sens_items_med"], thresholds["sens_items_high"] = compute_thresholds(
        df["Total sensitive items"], default_low=5, default_high=25
    )
    thresholds["users_med"], thresholds["users_high"] = compute_thresholds(
        df["Unique users accessing items"], default_low=2, default_high=10
    )
    thresholds["access_med"], thresholds["access_high"] = compute_thresholds(
        df["Times users accessed items"], default_low=5, default_high=25
    )
    thresholds["unscanned_med"], thresholds["unscanned_high"] = compute_thresholds(
        df["UnscannedPct"], default_low=0.10, default_high=0.40
    )

    return df, thresholds


def classify_row(row, thresholds) -> str:
    """
    Classify a data source as High / Medium / Low / None using:
    - Dynamic quantile-based thresholds
    - DeptHint bumps
    - External sharing
    """

    pct = row["SensitivePct"]
    total_sensitive = row["Total sensitive items"]
    unique_users = row["Unique users accessing items"]
    times_accessed = row["Times users accessed items"]
    unscanned_pct = row["UnscannedPct"]
    external = row["ExternalSharing"]
    total_items = row["Total items"]
    scanned = row["Total scanned items"]
    dept_hint = row["DeptHint"]

    # Completely empty = None
    if total_items == 0 and scanned == 0 and total_sensitive == 0:
        return "None"

    # baseline: if there is content at all, start as Low
    score = 0  # 0 = Low, 1 = Medium, 2 = High

    sens_med = thresholds["sens_med"]
    sens_high = thresholds["sens_high"]
    sens_items_med = thresholds["sens_items_med"]
    sens_items_high = thresholds["sens_items_high"]
    users_med = thresholds["users_med"]
    users_high = thresholds["users_high"]
    access_med = thresholds["access_med"]
    access_high = thresholds["access_high"]
    unscan_med = thresholds["unscanned_med"]
    unscan_high = thresholds["unscanned_high"]

    # Normalize NaN to 0 for comparison
    pct_val = float(pct) if pd.notna(pct) else 0.0
    unscan_val = float(unscanned_pct) if pd.notna(unscanned_pct) else 0.0

    # ---------------- HIGH conditions ----------------
    high_sens = (pct_val >= sens_high) or (total_sensitive >= sens_items_high)
    high_exposed = (unique_users >= users_med) or (times_accessed >= access_med)
    high_unscanned = (unscan_val >= unscan_high) and (total_items > 0)

    # 1) Very sensitive + some exposure
    if high_sens and high_exposed:
        score = max(score, 2)

    # 2) Sensitive + external sharing
    if ((pct_val >= sens_med) or (total_sensitive >= sens_items_med)) and external:
        score = max(score, 2)

    # 3) Large blind spot
    if high_unscanned:
        score = max(score, 2)

    # ---------------- MEDIUM conditions ----------------
    med_sens = (pct_val >= sens_med) or (total_sensitive >= sens_items_med)
    med_exposed = (unique_users >= users_med) or (times_accessed >= access_med)
    med_unscanned = (unscan_val >= unscan_med) and (total_items > 0)

    if (med_sens or med_exposed or med_unscanned or external) and score < 2:
        score = max(score, 1)

    # ---------------- DeptHint bump ----------------
    # High departments (HR/Exec/Finance/etc) should *at least* be Medium if they have any content.
    if dept_hint == "High" and total_items > 0:
        score = max(score, 1)
    elif dept_hint == "Medium" and total_items > 0:
        score = max(score, 1)

    # ---------------- Final mapping ----------------
    if score <= 0:
        return "Low"
    elif score == 1:
        return "Medium"
    else:
        return "High"


# ============================================================
#  MAIN
# ============================================================

def main():
    df = load_assessment(INPUT_FILE)
    df, thresholds = add_calculated_columns_and_thresholds(df)
    df["RiskBand"] = df.apply(lambda r: classify_row(r, thresholds), axis=1)

    # Optional: bring key columns to the front
    preferred_order = [
        "Data source ID",
        "Source type",
        "DeptHint",
        "Total items",
        "Total items accessed",
        "Total sensitive items",
        "Total scanned items",
        "Total unscanned data",
        "Times users accessed items",
        "Unique users accessing items",
        "Items shared with",
        "SensitivePct",
        "UnscannedPct",
        "ExternalSharing",
        "RiskBand",
    ]
    cols = [c for c in preferred_order if c in df.columns] + [
        c for c in df.columns if c not in preferred_order
    ]
    df = df[cols]

    df.to_excel(OUTPUT_FILE, sheet_name="Scored results", index=False)

    print("✔ Done! Output saved to:", OUTPUT_FILE)
    print("Thresholds used:")
    for k, v in thresholds.items():
        print(f"  {k}: {v:.4f}")


if __name__ == "__main__":
    main()
