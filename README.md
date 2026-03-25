# LRO PostgreSQL Docker Setup

## Prerequisites
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) installed and running
- Docker Image file: https://drive.google.com/file/d/104fkVMOSoPA-O5F7ZxqdQ3L6YrLpBtR_/view?usp=sharing

## Quick Start

### 1. Build & Start the Container
```bash
docker compose up -d --build
```

### 2. Connect to the Database
| Parameter | Value            |
|-----------|------------------|
| Host      | `localhost`      |
| Port      | `5432`           |
| Database  | `database`   |
| User      | `admin`      |
| Password  | `password`   |

**psql:**
```bash
docker exec -it postgres psql -U admin -d database
```

**Python (psycopg2):**
```python
import psycopg2
conn = psycopg2.connect(
    host="localhost", port=5432,
    dbname="database", user="admin", password="password"
)
```

### 3. Stop the Container
```bash
docker compose down        # keeps data volume
docker compose down -v     # removes data volume too
```

---

## Sharing the Database Image (437MB .tar)

Because the resulting `lro_postgres_image.tar` is larger than GitHub's 100MB file limit, you cannot push it directly to the repository codebase.

Instead, please use one of these methods to share the image:

### Option 1: GitHub Releases (Recommended)
1. Go to your repository on GitHub.
2. Click **Releases** on the right side, then **Draft a new release**.
3. Create a tag (e.g., `v1.0`).
4. Drag and drop your `lro_postgres_image.tar` into the "Attach binaries" box at the bottom.
5. Click **Publish release**.

*(Other team members can then download the `.tar` file directly from the GitHub Releases page).*

### Option 2: Cloud Storage
1. Upload the `.tar` file to a shared Google Drive, OneDrive, or Dropbox folder.
2. Share the download link with your team members.

---

### Import (Receiver's End)
Once a team member downloads the `.tar` file, they can load it locally:
```bash
docker load -i lro_postgres_image.tar
docker compose up -d
```

---

## Adding More Init Scripts
Drop additional `.sql` files into `init-scripts/`. They execute in alphabetical order on the **first** container startup only. To re-run them, remove the volume:
```bash
docker compose down -v
docker compose up -d --build
```

---

## Database Normalization (ETL)

After ingesting raw CSV data into the `staging` table using `ingest_csv.py`, you can normalize it into the structured schema:

1. **Run the Normalization Script**:
   ```bash
   docker exec postgres psql -U admin -d database -f /normalize_data.sql
   ```
2. **Verify the Results**:
   Open the `explore_staging.ipynb` notebook. The second section ("2. Normalized Data Verification") contains cells to count and preview the data in the structured tables (`site`, `variable`, `datastream`, etc.).

---

## Data Exploration (Jupyter Notebook)

To explore the ingested data:
1. Ensure your container is running: `docker compose up -d`
2. Install notebook dependencies:
   ```bash
   pip install pandas sqlalchemy psycopg2-binary notebook
   ```
3. Launch Jupyter:
   ```bash
   jupyter notebook explore_staging.ipynb
   ```
4. Run the cells to see the first 10 rows of the staging table.

---

## Project Structure
```
Final Project/
├── Dockerfile                  # PostgreSQL 16 image definition
├── docker-compose.yml          # Service orchestration
├── .dockerignore               # Build context exclusions
├── init-scripts/
│   └── 01-create-schema.sql    # Auto-runs on first startup
└── README.md
```

## Adding More Init Scripts
Drop additional `.sql` files into `init-scripts/`. They execute in alphabetical order on the **first** container startup only. To re-run them, remove the volume:
```bash
docker compose down -v
docker compose up -d --build
```
