# Purview-Data-Risk-Assessment-analyzer
A python script that will automatically label your sharepoint sites with Low, Medium or High
Code is below. It will check various things. You can edit it any way you like for your needs.

import pandas as pd

# ============================================================
#  CONFIGURATION
#  (change INPUT_FILE / OUTPUT_FILE / thresholds as you like)
# ============================================================

INPUT_FILE = "Report_DataOversharingworkshop.xlsx"
OUTPUT_FILE = "Purview_DataRisk_scored.xlsx"
SHEET_NAME = "Data risk assessment results"

# --- Risk thresholds (tweak these per customer) --------------

SENSITIVE_HIGH_PCT = 0.30      # 30%+ sensitive = high
SENSITIVE_MED_PCT = 0.10       # 10–29% sensitive = medium
SENSITIVE_ITEM_HIGH = 50       # 50+ sensitive items = high
SENSITIVE_ITEM_MED = 10        # 10–49 sensitive items = medium

UNIQUE_USERS_MED = 5           # 5+ unique users = collaboration signal
TIMES_ACCESSED_MED = 10        # 10+ touches = medium activity

UNSCANNED_HIGH_PCT = 0.40      # >40% unscanned = blind-spot risk

# ------------------------------------------------------------------
# Department / function keywords
# These are intentionally broad and cover:
# - Commercial (Exec, HR, Finance, Legal, Sales, R&D, etc.)
# - Education (Student records, SIS, Registrar, Financial aid, etc.)
# - Healthcare (PHI, clinical, EHR, billing, etc.)
# - Government (tax, licensing, police, courts, welfare, etc.)
#
# "High" = locations that *usually* hold highly sensitive data
# "Medium" = important but not always high-risk by default
# ------------------------------------------------------------------

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
    """
    Load the Purview DSPM export and locate the real header row.
    We look for the row whose first column is 'Data source ID'.
    """
    raw = pd.read_excel(path, sheet_name=SHEET_NAME, header=None)

    header_row_index = raw.index[raw.iloc[:, 0] == "Data source ID"][0]
    headers = raw.iloc[header_row_index]
    df = raw.iloc[header_row_index + 1 :].copy()
    df.columns = headers

    return df.reset_index(drop=True)


def get_dept_hint(value: str) -> str:
    """
    Inspect a string (site path or name) and return:
    - "High" if it contains any high-risk keywords (HR, exec, finance, etc.)
    - "Medium" if it contains any medium-risk keywords
    - "None" otherwise
    """
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


def add_calculated_columns(df: pd.DataFrame) -> pd.DataFrame:
    """Normalize numeric fields and compute percentages / flags / dept hints."""
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

    # Percentage of scanned items that are sensitive
    df["SensitivePct"] = df["Total sensitive items"] / df["Total scanned items"].replace(0, pd.NA)

    # Percentage of content that is unscanned
    total_scanned = df["Total scanned items"]
    total_unscanned = df["Total unscanned data"]
    df["UnscannedPct"] = total_unscanned / (total_scanned + total_unscanned).replace(0, pd.NA)

    # External / broad sharing flag based on "Items shared with"
    # (Everyone/Anyone/External/Public considered risky)
    df["ExternalSharing"] = df["Items shared with"].astype(str).str.contains(
        "everyone|anyone|external|public", case=False, na=False
    )

    # ---- Department hint based on Data source ID and optional name ----
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

    return df


def classify_row(row) -> str:
    """
    Classify a data source as High / Medium / Low / None
    using a scoring model + department hints.

    Notes:
    - Completely empty locations (no items, no scans, no sensitive items) → "None".
    - Everything else starts at "Low". If no conditions are hit, result stays "Low"
      (this replaced the old 'Unknown' state).
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

    # --- Empty locations are None ---
    if total_items == 0 and scanned == 0 and total_sensitive == 0:
        return "None"

    # Start with Low as a baseline if there's any content
    score = 0   # 0 = Low, 1 = Medium, 2 = High

    # --- High risk conditions ---
    high_cond_1 = (
        (pd.notna(pct) and pct >= SENSITIVE_HIGH_PCT)
        or (total_sensitive >= SENSITIVE_ITEM_HIGH)
    ) and (unique_users >= UNIQUE_USERS_MED or times_accessed >= TIMES_ACCESSED_MED)

    high_cond_2 = (
        (pd.notna(pct) and pct >= SENSITIVE_MED_PCT)
        or (total_sensitive >= SENSITIVE_ITEM_MED)
    ) and external

    high_cond_3 = (
        pd.notna(unscanned_pct)
        and unscanned_pct >= UNSCANNED_HIGH_PCT
        and total_items > 0
    )

    if high_cond_1 or high_cond_2 or high_cond_3:
        score = max(score, 2)

    # --- Medium risk conditions ---
    med_cond = (
        (pd.notna(pct) and pct >= SENSITIVE_MED_PCT)
        or (total_sensitive >= SENSITIVE_ITEM_MED)
        or (unique_users >= UNIQUE_USERS_MED)
        or (times_accessed >= TIMES_ACCESSED_MED)
        or external
    )

    if med_cond and score < 2:
        score = max(score, 1)

    # --- Department-based bump --------------------------
    if dept_hint == "High" and total_items > 0:
        # HR/Exec/Finance/etc → at least Medium
        score = max(score, 1)
    elif dept_hint == "Medium" and total_items > 0:
        # Important but not obviously critical → at least Medium
        score = max(score, 1)
    # ----------------------------------------------------

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
    df = add_calculated_columns(df)
    df["RiskBand"] = df.apply(classify_row, axis=1)

    # Optional: bring key columns to the front for readability
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
    print(f"✔ Done! Output saved to: {OUTPUT_FILE}")


if __name__ == "__main__":
    main()
