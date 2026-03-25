# Batch Article Import Guide

How to import multiple articles (with PDFs) into Heero Publishers at once.

---

## Overview

Instead of submitting articles one by one through the website, you can batch import them using a CSV spreadsheet + PDF files. This uses the **Janeway Imports Plugin**.

---

## For the Client (Article Owner)

### Step 1: Fill out the spreadsheet

Open `heero_import_template.xlsx` (included in this folder) in Excel or Google Sheets.

**Each row = one article.** If you have 50 PDFs, fill 50 rows.

The columns marked **REQUIRED** (green) must be filled in:

| Column | What to enter | Example |
|--------|--------------|---------|
| Article title | Full title | Mushroom Cultivation in West Africa |
| Author surname | Last name | Isikhuemhen |
| Author given name | First name | Omoanghe |
| Author email | Email address | omon@ncat.edu |
| Author is primary (Y/N) | Y for main author | Y (pre-filled) |
| Date published | YYYY-MM-DD format | 2025-06-15 |
| Article section | Category | Research Articles (pre-filled) |
| Stage | Always Published | Published (pre-filled) |
| Article filename | Exact PDF name | mushroom_study.pdf |
| Journal Code | Always heero | heero (pre-filled) |

Optional columns (yellow) — fill if you have the info:
- Author institution, Author Salutation, Keywords, Volume/Issue numbers, DOI

### Step 2: Prepare the ZIP

1. Create a folder (any name, e.g. `heero_import`)
2. Save the spreadsheet as **CSV UTF-8**: File → Save As → select "CSV UTF-8 (Comma delimited)"
3. Put the CSV file in the folder
4. Put ALL the PDF files in the same folder
5. **Important**: Each PDF filename must match exactly what you typed in the "Article filename" column
6. ZIP the folder

Example folder structure:
```
heero_import/
  articles.csv
  mushroom_study.pdf
  bioconversion_waste.pdf
  tropical_fungi_review.pdf
  soil_analysis_2024.pdf
  ...
```

### Step 3: Send the ZIP

Send the ZIP file to your admin for upload.

---

## For the Admin (Uploading to Server)

### Prerequisites

The Imports plugin must be installed on the server:
```bash
cd /home/janeway/janeway/src/plugins
sudo -u janeway git clone https://github.com/openlibhums/imports.git
sudo /home/janeway/janeway/venv/bin/pip install python-wordpress-xmlrpc==2.3
sudo systemctl restart janeway
```

**Status: Already installed on heeropublishers.org (March 2026)**

### Step 1: Access the import tool

Go to: `https://heeropublishers.org/heero/plugins/imports/`

Or: Journal Manager → look for "Imports" or "All Articles" in the sidebar.

### Step 2: Upload the CSV

1. Select **Import Articles** (or similar option)
2. Upload the CSV file from the client's ZIP
3. Review the preview — check that titles, authors, and dates look correct
4. Confirm the import

### Step 3: Upload the PDFs as galleys

After the metadata is imported, each article needs its PDF attached:

1. Go to each imported article in the manager
2. Navigate to the Production/Galley section
3. Upload the corresponding PDF as a **Galley** (label: "PDF")
4. The article is now published and downloadable

**Note:** The CSV import creates the article records (metadata). The PDF files (galleys) may need to be uploaded separately through the article manager, depending on the plugin version. Check the import tool interface — some versions support file upload directly with the CSV.

### Step 4: Verify

- Check the journal page: `https://heeropublishers.org/heero/`
- Verify articles appear in the article list
- Click on a few articles to confirm PDFs are downloadable

---

## Troubleshooting

### CSV won't import
- Make sure it's saved as **CSV UTF-8** (not regular CSV)
- Check for special characters in titles or names
- Ensure date format is YYYY-MM-DD
- Verify the Journal Code column says "heero" for every row

### Articles imported but no PDF
- PDFs must be uploaded as galleys after the metadata import
- Go to each article → Production → Add Galley → Upload PDF

### Wrong metadata
- You can edit any article after import through the journal manager
- Go to: `https://heeropublishers.org/heero/manager/` → find the article → edit

### Need to undo the import
- Restore from the most recent database backup:
  ```bash
  ls -lt /backups/janeway/*.dump | head -5
  sudo -u janeway pg_restore -h localhost -d janeway --clean /backups/janeway/db_YYYYMMDD_020001.dump
  sudo systemctl restart janeway
  ```

---

## Plugin Documentation

Official docs: https://janeway-imports.readthedocs.io/en/latest/

Plugin source: https://github.com/openlibhums/imports
