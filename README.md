# ORCID Publication Harvester

Tool to retrieve and analyze publications for a list of authors identified by their ORCID iDs. It reads public works via the ORCID Public API and can optionally enrich with Crossref and OpenAlex. Outputs per‑author CSVs, a combined CSV, and gap/metrics reports.

---

## Quick Start

```bash
# 1) Clone and enter
git clone https://github.com/atjohnson1994/orcid-publication-harvester.git
cd orcid-publication-harvester

# 2) Create & activate a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate  # Windows: .venv\Scripts\activate

# 3) Install deps
pip install -r requirements.txt

# 4) Prepare your ORCID CSV (first column = ORCID iD)
cp examples/orcids.csv .
# edit or replace with your file

# 5) Run (ORCID only)
python orcid_harvest_csv.py \
  --orcid-csv orcids.csv \
  --client-id $ORCID_CLIENT_ID \
  --client-secret $ORCID_CLIENT_SECRET

# 6) Run with enrichment & DOI backfill
python orcid_harvest_csv.py \
  --orcid-csv orcids.csv \
  --client-id $ORCID_CLIENT_ID \
  --client-secret $ORCID_CLIENT_SECRET \
  --with-crossref --with-openalex --with-backfill
```

Outputs are written to `./output/`:
- `Family_Given_<ORCID>.csv` per author
- `all_orcid_publications.csv` (combined)
- `gap_report.csv` (coverage vs enrichment)
- `author_metrics.csv` (activity last 12/24 months, citations, OA share)

---

## Getting ORCID API Credentials

1. Register for the **ORCID Public API** (free): https://info.orcid.org/register-for-a-public-api-client-application/
2. Choose **Sandbox** for testing (fake data) or **Production** for real data.
3. After approval, you’ll get a **Client ID** and **Client Secret**.
4. Export them as env vars or pass via flags:

```bash
export ORCID_CLIENT_ID=APP-XXXXXX
export ORCID_CLIENT_SECRET=xxxxxxxx
```

## Repository Structure

```
.
├─ orcid_harvest_csv.py            # main script
├─ requirements.txt                # dependencies
├─ README.md                       # this file
├─ LICENSE                         # MIT by default
├─ .gitignore
└─ examples/
   └─ orcids.csv                   # example input (first column = ORCID)
```

---

## Configuration & Flags

```
--orcid-csv <path>         Path to CSV with ORCID iDs in the first column (header OK)
--client-id <id>           ORCID Public API client_id
--client-secret <secret>   ORCID Public API client_secret
--sandbox                  Use ORCID sandbox token endpoint
--output-dir <dir>         Output directory (default: output)
--combined-name <name>     Combined CSV file name (default: all_orcid_publications.csv)
--gap-report <name>        Gap report CSV name (default: gap_report.csv)
--metrics-report <name>    Metrics report CSV name (default: author_metrics.csv)
--with-crossref            Enrich bibliographic data from Crossref (by DOI)
--with-openalex            Add citations/OA/concepts from OpenAlex (by DOI)
--with-backfill            Attempt DOI backfill for items missing DOI (title search)
--sleep <seconds>          Delay between authors (default: 0.4)
```

---

## File: `orcid_harvest_csv.py`

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
ORCID Works → CSVs with optional Crossref/OpenAlex enrichment.

Now reads ORCID iDs from a CSV file (first column).

Features
- Read ORCIDs from CSV (first column, header OK)
- Per-author CSV + combined CSV
- Optional enrichment flags:
    --with-crossref      add bibliographic detail from Crossref (by DOI)
    --with-openalex      add citations/OpenAccess/concepts from OpenAlex (by DOI)
    --with-backfill      try to find missing DOIs by title search (Crossref, fallback OpenAlex)
- Dedupe + merge priority:
    Bibliography (title/journal/vol/issue/pages): Crossref > OpenAlex > ORCID
    Analytics (citations/OA): OpenAlex
    Identifiers: union; prefer DOI as primary key
- Gap/metrics report per author
"""

import os
import re
import csv
import time
import json
import argparse
import requests
from typing import Dict, Any, List, Optional, Tuple
from dataclasses import dataclass, field
from rapidfuzz import fuzz
from dateutil import parser as dtp
from collections import defaultdict

# --------------------------
# Config
# --------------------------
DEFAULT_OUTPUT_DIR = "output"
DEFAULT_COMBINED_CSV = "all_orcid_publications.csv"
DEFAULT_GAP_REPORT = "gap_report.csv"
DEFAULT_METRICS_REPORT = "author_metrics.csv"

ORCID_TOKEN_URL_PROD = "https://orcid.org/oauth/token"
ORCID_TOKEN_URL_SANDBOX = "https://sandbox.orcid.org/oauth/token"
ORCID_API_BASE = "https://api.orcid.org/v3.0"

CROSSREF_WORKS = "https://api.crossref.org/works/"
CROSSREF_QUERY = "https://api.crossref.org/works"
OPENALEX_WORKS = "https://api.openalex.org/works/"

REQUEST_DELAY_SECONDS = 0.4  # polite default
ORCID_REGEX = re.compile(r"^\d{4}-\d{4}-\d{4}-\d{3}[\dX]$", re.I)


# --------------------------
# Utilities
# --------------------------
def safe_filename(name: str) -> str:
    return re.sub(r'[^A-Za-z0-9_.-]+', "_", str(name)).strip("_") or "unknown"


def http_get_json(url: str, params: dict | None = None, headers: dict | None = None, timeout: int = 30, retries: int = 4, backoff_base: int = 2) -> Optional[dict]:
    h = {"Accept": "application/json"}
    if headers:
        h.update(headers)
    last_err = None
    for i in range(retries):
        try:
            r = requests.get(url, params=params, headers=h, timeout=timeout)
            if r.status_code in (429, 500, 502, 503, 504):
                time.sleep(backoff_base ** i)
                continue
            r.raise_for_status()
            return r.json()
        except Exception as e:
            last_err = e
            time.sleep(backoff_base ** i)
    if last_err:
        print(f"[WARN] GET failed {url}: {last_err}")
    return None


def http_post_json(url: str, data: dict, headers: dict | None = None, timeout: int = 30) -> dict:
    h = {"Accept": "application/json"}
    if headers:
        h.update(headers)
    r = requests.post(url, data=data, headers=h, timeout=timeout)
    r.raise_for_status()
    return r.json()


def extract_first_value(d: Optional[Dict[str, Any]], path: List[str]) -> Optional[str]:
    cur = d
    for key in path:
        if not isinstance(cur, dict) or key not in cur:
            return None
        cur = cur[key]
    return cur.strip() if isinstance(cur, str) and cur.strip() else None


def normalize_doi(doi: Optional[str]) -> Optional[str]:
    if not doi:
        return None
    doi = doi.strip()
    doi = re.sub(r'^(https?://(dx\.)?doi\.org/)', '', doi, flags=re.I)
    return doi.lower()


def parse_year_from_crossref(item: dict) -> Optional[str]:
    parts = (item.get("issued", {}) or {}).get("date-parts") or []
    if parts and parts[0] and parts[0][0]:
        return str(parts[0][0])
    return None


def parse_year_from_openalex(item: dict) -> Optional[str]:
    y = item.get("publication_year")
    return str(y) if y else None


def best_date(*vals: Optional[str]) -> Optional[str]:
    # Return earliest parsed date ISO (YYYY-MM-DD) among provided values
    dt_min = None
    for v in vals:
        if not v:
            continue
        try:
            dt = dtp.parse(str(v), default=None)
            if not dt_min or dt < dt_min:
                dt_min = dt
        except Exception:
            continue
    return dt_min.date().isoformat() if dt_min else None


def read_orcids_from_csv(path: str) -> List[str]:
    """Read ORCID iDs from the first column of a CSV. Skips blanks, de-duplicates, tolerates header."""
    orcids: List[str] = []
    with open(path, "r", encoding="utf-8-sig", newline="") as f:
        reader = csv.reader(f)
        for row in reader:
            if not row:
                continue
            cell = (row[0] or "").strip()
            # tolerate header or malformed rows
            if not ORCID_REGEX.match(cell):
                continue
            orcids.append(cell)
    # Deduplicate preserving order
    seen = set()
    ordered = []
    for oid in orcids:
        if oid not in seen:
            ordered.append(oid)
            seen.add(oid)
    return ordered


# --------------------------
# Data model
# --------------------------
@dataclass
class WorkRow:
    orcid: str
    title: Optional[str] = None
    journal: Optional[str] = None
    year: Optional[str] = None
    type: Optional[str] = None
    put_code: Optional[int] = None
    doi: Optional[str] = None
    url: Optional[str] = None

    # Enrichment
    publisher: Optional[str] = None
    volume: Optional[str] = None
    issue: Optional[str] = None
    page: Optional[str] = None
    published_date: Optional[str] = None  # earliest of print/online/etc.
    citations: Optional[int] = None
    oa_status: Optional[bool] = None
    oa_url: Optional[str] = None
    concepts: Optional[str] = None  # comma-joined OpenAlex concepts

    # Provenance flags
    source_orcid_claimed: bool = False
    source_crossref_hit: bool = False
    source_openalex_hit: bool = False
    found_via_backfill: bool = False

    # Raw for debug (optional)
    raw_sources: Dict[str, Any] = field(default_factory=dict)

    def to_dict(self) -> Dict[str, Any]:
        return {
            "orcid": self.orcid,
            "title": self.title,
            "journal": self.journal,
            "year": self.year,
            "type": self.type,
            "put_code": self.put_code,
            "doi": self.doi,
            "url": self.url,
            "publisher": self.publisher,
            "volume": self.volume,
            "issue": self.issue,
            "page": self.page,
            "published_date": self.published_date,
            "citations": self.citations,
            "oa_status": self.oa_status,
            "oa_url": self.oa_url,
            "concepts": self.concepts,
            "source_orcid_claimed": self.source_orcid_claimed,
            "source_crossref_hit": self.source_crossref_hit,
            "source_openalex_hit": self.source_openalex_hit,
            "found_via_backfill": self.found_via_backfill,
        }


# --------------------------
# ORCID functions
# --------------------------
def get_orcid_access_token(client_id: str, client_secret: str, sandbox: bool) -> str:
    token_url = ORCID_TOKEN_URL_SANDBOX if sandbox else ORCID_TOKEN_URL_PROD
    data = {"client_id": client_id, "client_secret": client_secret, "grant_type": "client_credentials", "scope": "/read-public"}
    j = http_post_json(token_url, data)
    token = j.get("access_token")
    if not token:
        raise RuntimeError("Failed to obtain ORCID access token.")
    return token


def fetch_preferred_name(orcid: str, token: str) -> Tuple[str, str]:
    url = f"{ORCID_API_BASE}/{orcid}/person"
    headers = {"Authorization": f"Bearer {token}"}
    data = http_get_json(url, headers=headers)
    if not data:
        return (orcid, safe_filename(orcid))
    given = extract_first_value(data, ["name", "given-names", "value"]) or ""
    family = extract_first_value(data, ["name", "family-name", "value"]) or ""
    parts = [p for p in (given, family) if p]
    if not parts:
        return (orcid, safe_filename(orcid))
    display = " ".join(parts)
    stub = safe_filename(f"{family}_{given}") if (given or family) else safe_filename(orcid)
    return display, stub


def fetch_first_author_family(orcid: str, token: str) -> Optional[str]:
    url = f"{ORCID_API_BASE}/{orcid}/person"
    headers = {"Authorization": f"Bearer {token}"}
    data = http_get_json(url, headers=headers)
    return extract_first_value(data or {}, ["name", "family-name", "value"])


def fetch_works_for_orcid(orcid: str, token: str) -> List[WorkRow]:
    url = f"{ORCID_API_BASE}/{orcid}/works"
    headers = {"Authorization": f"Bearer {token}"}
    data = http_get_json(url, headers=headers)
    rows: List[WorkRow] = []
    if not data:
        return rows
    groups = data.get("group", []) or []
    for g in groups:
        summaries = g.get("work-summary", []) or []
        for ws in summaries:
            title = extract_first_value(ws, ["title", "title", "value"])
            journal = extract_first_value(ws, ["journal-title", "value"])
            year = extract_first_value(ws, ["publication-date", "year", "value"])
            type_ = ws.get("type")
            put_code = ws.get("put-code")
            # identifiers
            doi = None
            url_pref = None
            ext_ids = (ws.get("external-ids", {}) or {}).get("external-id", []) or []
            for eid in ext_ids:
                t = (eid.get("external-id-type") or "").lower()
                v = eid.get("external-id-value") or ""
                ext_url = extract_first_value(eid, ["external-id-url", "value"])
                if t == "doi" and v:
                    doi = normalize_doi(v)
                if not url_pref and ext_url:
                    url_pref = ext_url
            if doi and not url_pref:
                url_pref = f"https://doi.org/{doi}"
            row = WorkRow(
                orcid=orcid,
                title=title,
                journal=journal,
                year=year,
                type=type_,
                put_code=put_code,
                doi=doi,
                url=url_pref,
                source_orcid_claimed=True,
            )
            rows.append(row)
    return rows


# --------------------------
# Crossref & OpenAlex enrichment
# --------------------------
def enrich_crossref_by_doi(doi: str) -> Optional[dict]:
    if not doi:
        return None
    j = http_get_json(CROSSREF_WORKS + doi.lower())
    if j and "message" in j:
        return j["message"]
    return None


def enrich_openalex_by_doi(doi: str) -> Optional[dict]:
    if not doi:
        return None
    j = http_get_json(OPENALEX_WORKS + "https://doi.org/" + doi.lower())
    return j


def crossref_lookup_by_title(title: str, year: Optional[str], first_author_last: Optional[str]) -> Optional[str]:
    if not title:
        return None
    params = {"query.bibliographic": title, "rows": 5}
    j = http_get_json(CROSSREF_QUERY, params=params)
    if not j:
        return None
    items = (j.get("message", {}) or {}).get("items", []) or []
    best = None
    best_score = 0
    for it in items:
        t = (it.get("title") or [""])[0]
        y = parse_year_from_crossref(it)
        authors = it.get("author", []) or []
        first_last = (authors[0].get("family") if authors else None)
        score = fuzz.token_sort_ratio(title, t)
        if year and y and str(y) == str(year):
            score += 5
        if first_author_last and first_last and first_author_last.lower() == str(first_last).lower():
            score += 5
        if score > best_score:
            best, best_score = it, score
    if best and best_score >= 85:
        doi = best.get("DOI")
        return normalize_doi(doi)
    return None


def openalex_lookup_by_title(title: str, year: Optional[str]) -> Optional[str]:
    if not title:
        return None
    params = {"search": title, "per_page": 5}
    j = http_get_json("https://api.openalex.org/works", params=params)
    if not j:
        return None
    items = j.get("results", []) or []
    best = None
    best_score = 0
    for it in items:
        t = it.get("title") or ""
        y = parse_year_from_openalex(it)
        score = fuzz.token_sort_ratio(title, t)
        if year and y and str(y) == str(year):
            score += 5
        if score > best_score:
            best, best_score = it, score
    if best and best_score >= 85:
        doi = best.get("doi")
        return normalize_doi(doi)
    return None


# --------------------------
# Merge logic
# --------------------------
@dataclass
class MergePriority:
    # Bibliography: Crossref > OpenAlex > ORCID
    pass

def merge_with_crossref(row: WorkRow, c: dict) -> None:
    if not c:
        return
    row.source_crossref_hit = True
    title = (c.get("title") or [""])[0] or None
    container = (c.get("container-title") or [""])[0] or None
    row.title = title or row.title
    row.journal = container or row.journal
    row.publisher = c.get("publisher") or row.publisher
    row.volume = c.get("volume") or row.volume
    row.issue = c.get("issue") or row.issue
    page = c.get("page") or c.get("fpage") or c.get("lpage")
    row.page = page or row.page

    p_date = None
    if "published-print" in c and c["published-print"].get("date-parts"):
        part = c["published-print"]["date-parts"][0]
        p_date = "-".join(str(x) for x in part if x)
    o_date = None
    if "published-online" in c and c["published-online"].get("date-parts"):
        part = c["published-online"]["date-parts"][0]
        o_date = "-".join(str(x) for x in part if x)
    row.published_date = best_date(p_date, o_date, row.published_date)

    y = parse_year_from_crossref(c)
    row.year = y or row.year

    cr_doi = normalize_doi(c.get("DOI"))
    row.doi = cr_doi or row.doi
    if row.doi and not row.url:
        row.url = f"https://doi.org/{row.doi}"


def merge_with_openalex(row: WorkRow, o: dict) -> None:
    if not o:
        return
    row.source_openalex_hit = True
    if "cited_by_count" in o:
        try:
            row.citations = int(o.get("cited_by_count") or 0)
        except Exception:
            pass
    oa = o.get("open_access") or {}
    is_oa = oa.get("is_oa")
    row.oa_status = bool(is_oa) if is_oa is not None else row.oa_status
    bol = o.get("best_oa_location") or {}
    if not row.oa_url and isinstance(bol, dict):
        row.oa_url = bol.get("url") or row.oa_url

    concepts = o.get("concepts") or []
    if concepts and isinstance(concepts, list):
        names = [c.get("display_name") for c in concepts if c.get("display_name")]
        if names:
            row.concepts = ", ".join(names[:6])

    biblio = o.get("biblio") or {}
    row.volume = row.volume or biblio.get("volume")
    row.issue = row.issue or biblio.get("issue")
    row.page = row.page or biblio.get("first_page") or biblio.get("pages")

    row.year = row.year or parse_year_from_openalex(o)

    odoi = o.get("doi")
    if odoi:
        odoi = normalize_doi(odoi)
        if odoi and not row.doi:
            row.doi = odoi
        if row.doi and not row.url:
            row.url = f"https://doi.org/{row.doi}"


def dedupe_rows(rows: List[WorkRow]) -> List[WorkRow]:
    by_doi: Dict[str, WorkRow] = {}
    pending: List[WorkRow] = []

    for r in rows:
        if r.doi:
            key = r.doi.lower()
            if key not in by_doi:
                by_doi[key] = r
            else:
                base = by_doi[key]
                for f in vars(base).keys():
                    if getattr(base, f) in (None, "", False) and getattr(r, f):
                        setattr(base, f, getattr(r, f))
        else:
            pending.append(r)

    def norm_title(t: Optional[str]) -> str:
        return re.sub(r'\s+', ' ', (t or "").strip().lower())

    used = set()
    result = list(by_doi.values())
    for i, r in enumerate(pending):
        if i in used:
            continue
        merged = r
        for j in range(i + 1, len(pending)):
            if j in used:
                continue
            s = pending[j]
            t1, t2 = norm_title(merged.title), norm_title(s.title)
            score = fuzz.token_sort_ratio(t1, t2) if (t1 and t2) else 0
            if score >= 90 and (merged.year == s.year or not merged.year or not s.year):
                for f in vars(merged).keys():
                    if getattr(merged, f) in (None, "", False) and getattr(s, f):
                        setattr(merged, f, getattr(s, f))
                used.add(j)
        result.append(merged)
    return result


# --------------------------
# CSV writers
# --------------------------
PER_AUTHOR_HEADERS = [
    "orcid","title","journal","year","type","put_code","doi","url",
    "publisher","volume","issue","page","published_date",
    "citations","oa_status","oa_url","concepts",
    "source_orcid_claimed","source_crossref_hit","source_openalex_hit","found_via_backfill"
]

def write_csv(path: str, rows: List[WorkRow]) -> None:
    os.makedirs(os.path.dirname(path), exist_ok=True)
    with open(path, "w", newline="", encoding="utf-8") as f:
        writer = csv.DictWriter(f, fieldnames=PER_AUTHOR_HEADERS)
        writer.writeheader()
        for r in rows:
            writer.writerow(r.to_dict())


def write_gap_report(path: str, author_summary: Dict[str, Dict[str, Any]]) -> None:
    os.makedirs(os.path.dirname(path), exist_ok=True)
    headers = [
        "orcid","display_name","claimed_count","with_doi_count","enriched_count",
        "backfilled_count","likely_incomplete"
    ]
    with open(path, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=headers)
        w.writeheader()
        for orcid, s in author_summary.items():
            w.writerow({
                "orcid": orcid,
                "display_name": s.get("display_name"),
                "claimed_count": s.get("claimed_count", 0),
                "with_doi_count": s.get("with_doi_count", 0),
                "enriched_count": s.get("enriched_count", 0),
                "backfilled_count": s.get("backfilled_count", 0),
                "likely_incomplete": s.get("likely_incomplete", False),
            })


def write_metrics_report(path: str, metrics_rows: List[Dict[str, Any]]) -> None:
    os.makedirs(os.path.dirname(path), exist_ok=True)
    headers = [
        "orcid","display_name","pubs_last_12m","pubs_last_24m",
        "most_recent_pub_date","total_citations","oa_share_last_24m"
    ]
    with open(path, "w", newline="", encoding="utf-8") as f:
        w = csv.DictWriter(f, fieldnames=headers)
        w.writeheader()
        for row in metrics_rows:
            w.writerow(row)


# --------------------------
# Metrics & summaries
# --------------------------

def compute_author_metrics(orcid: str, display_name: str, works: List[WorkRow]) -> Dict[str, Any]:
    from datetime import date
    today = date.today()
    d12 = today.replace(year=today.year - 1)
    d24 = today.replace(year=today.year - 2)

    def parse_any_date(w: WorkRow) -> Optional[str]:
        if w.published_date:
            return w.published_date
        if w.year:
            try:
                return f"{int(w.year):04d}-01-01"
            except Exception:
                return None
        return None

    pubs_12 = pubs_24 = 0
    most_recent = None
    total_citations = 0
    oa_24_count = 0
    total_24 = 0

    for w in works:
        dt_iso = parse_any_date(w)
        if dt_iso:
            try:
                dt = dtp.parse(dt_iso).date()
            except Exception:
                dt = None
        else:
            dt = None

        if dt:
            if dt >= d12:
                pubs_12 += 1
            if dt >= d24:
                pubs_24 += 1
                total_24 += 1
                if w.oa_status:
                    oa_24_count += 1
            if (not most_recent) or (dt > dtp.parse(most_recent).date()):
                most_recent = dt.isoformat()

        if isinstance(w.citations, int):
            total_citations += w.citations

    oa_share_24m = (oa_24_count / total_24) if total_24 else 0.0
    return {
        "orcid": orcid,
        "display_name": display_name,
        "pubs_last_12m": pubs_12,
        "pubs_last_24m": pubs_24,
        "most_recent_pub_date": most_recent,
        "total_citations": total_citations,
        "oa_share_last_24m": round(oa_share_24m, 3),
    }


# --------------------------
# Main
# --------------------------

def main():
    ap = argparse.ArgumentParser(description="Fetch ORCID works with optional Crossref/OpenAlex enrichment (ORCIDs from CSV first column).")
    ap.add_argument("--orcid-csv", required=True, help="Path to CSV containing ORCID iDs in the first column. Header allowed.")
    ap.add_argument("--client-id", default=os.getenv("ORCID_CLIENT_ID"), help="ORCID API client_id")
    ap.add_argument("--client-secret", default=os.getenv("ORCID_CLIENT_SECRET"), help="ORCID API client_secret")
    ap.add_argument("--sandbox", action="store_true", help="Use ORCID sandbox token endpoint")
    ap.add_argument("--output-dir", default=DEFAULT_OUTPUT_DIR)
    ap.add_argument("--combined-name", default=DEFAULT_COMBINED_CSV)
    ap.add_argument("--gap-report", default=DEFAULT_GAP_REPORT)
    ap.add_argument("--metrics-report", default=DEFAULT_METRICS_REPORT)
    ap.add_argument("--with-crossref", action="store_true", help="Enrich bibliographic data from Crossref")
    ap.add_argument("--with-openalex", action="store_true", help="Enrich citations/open access from OpenAlex")
    ap.add_argument("--with-backfill", action="store_true", help="Attempt DOI backfill for items missing DOI (title search)")
    ap.add_argument("--sleep", type=float, default=REQUEST_DELAY_SECONDS, help="Delay between authors (seconds)")
    args = ap.parse_args()

    if not args.client_id or not args.client_secret:
        raise SystemExit("Please provide ORCID --client-id and --client-secret (or set env vars ORCID_CLIENT_ID/ORCID_CLIENT_SECRET).")

    # Load ORCIDs from CSV (first column)
    orcid_list = read_orcids_from_csv(args.orcid_csv)
    if not orcid_list:
        raise SystemExit(f"No valid ORCID iDs found in first column of {args.orcid_csv}.")

    token = get_orcid_access_token(args.client_id, args.client_secret, args.sandbox)

    combined_rows: List[WorkRow] = []
    author_summary: Dict[str, Dict[str, Any]] = {}
    metrics_rows: List[Dict[str, Any]] = []

    os.makedirs(args.output_dir, exist_ok=True)

    for idx, oid in enumerate(orcid_list, start=1):
        print(f"[{idx}/{len(orcid_list)}] {oid}: fetching name & works...")
        display_name, name_stub = fetch_preferred_name(oid, token)
        first_author_last = fetch_first_author_family(oid, token)

        # 1) Seed from ORCID
        works = fetch_works_for_orcid(oid, token)

        # 2) Optional backfill by title → DOI
        if args.with_backfill:
            for w in works:
                if not w.doi and w.title:
                    doi = crossref_lookup_by_title(w.title, w.year, first_author_last)
                    if not doi:
                        doi = openalex_lookup_by_title(w.title, w.year)
                    if doi:
                        w.doi = doi
                        w.url = w.url or f"https://doi.org/{doi}"
                        w.found_via_backfill = True

        # 3) Optional enrichment by DOI
        for w in works:
            if args.with_crossref and w.doi:
                c = enrich_crossref_by_doi(w.doi)
                if c:
                    merge_with_crossref(w, c)
            if args.with_openalex and w.doi:
                o = enrich_openalex_by_doi(w.doi)
                if o:
                    merge_with_openalex(w, o)

        # 4) Dedupe
        works = dedupe_rows(works)

        # 5) Write per-author CSV
        per_author_name = f"{name_stub}_{safe_filename(oid)}.csv"
        per_author_path = os.path.join(args.output_dir, per_author_name)
        write_csv(per_author_path, works)
        print(f"  -> wrote {len(works)} rows to {per_author_path}")

        # 6) Build summaries
        claimed = sum(1 for w in works if w.source_orcid_claimed)
        with_doi = sum(1 for w in works if w.doi)
        enriched = sum(1 for w in works if (w.source_crossref_hit or w.source_openalex_hit))
        backfilled = sum(1 for w in works if w.found_via_backfill)
        likely_incomplete = claimed < with_doi  # heuristic

        author_summary[oid] = {
            "display_name": display_name,
            "claimed_count": claimed,
            "with_doi_count": with_doi,
            "enriched_count": enriched,
            "backfilled_count": backfilled,
            "likely_incomplete": likely_incomplete,
        }

        metrics_rows.append(compute_author_metrics(oid, display_name, works))
        combined_rows.extend(works)

        time.sleep(args.sleep)

    # 7) Write combined CSV
    combined_path = os.path.join(args.output_dir, args.combined_name)
    write_csv(combined_path, combined_rows)
    print(f"\nCombined: wrote {len(combined_rows)} rows to {combined_path}")

    # 8) Gap/metrics reports
    gap_path = os.path.join(args.output_dir, args.gap_report)
    write_gap_report(gap_path, author_summary)
    print(f"Gap report: {gap_path}")

    metrics_path = os.path.join(args.output_dir, args.metrics_report)
    write_metrics_report(metrics_path, metrics_rows)
    print(f"Metrics report: {metrics_path}\n")

    print("Done.")


if __name__ == "__main__":
    main()
```

---

## File: `requirements.txt`

```
requests
rapidfuzz
python-dateutil
```

---

## File: `.gitignore`

```
# venvs
.venv/
venv/

# macOS & Windows
.DS_Store
Thumbs.db

# Python
__pycache__/
*.pyc
*.pyo
*.pyd

# outputs
output/

# editor
.vscode/
.idea/
```

## File: `examples/orcids.csv`

```
orcid,author_name
0000-0002-1825-0097,Example Author A
0000-0001-5109-3700,Example Author B
```

---
