# ORCID-Publication-Harvester
Tool to retrieve and analyze publications for a list of authors identified by their ORCID iDs. It reads public works via the ORCID Public API and can optionally enrich with Crossref and OpenAlex. Outputs per‑author CSVs, a combined CSV, and gap/metrics reports.

# Quick Start
# 1) Clone and enter
git clone https://github.com/your-username/orcid-publication-harvester.git
cd orcid-publication-harvester


# 2) Create & activate a virtual environment (recommended)
python -m venv .venv
source .venv/bin/activate # Windows: .venv\Scripts\activate


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
