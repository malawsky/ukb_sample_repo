# UK Biobank Research Analysis Platform (RAP) Guide

## Overview

The RAP is hosted on DNAnexus and provides secure cloud-based access to UK Biobank data. All computation runs on DNAnexus infrastructure (AWS eu-west-2, London region). Data cannot be downloaded locally; all analyses must run on the platform.

## dx CLI Reference

### Installation & Authentication

```bash
pip install dxpy
dx --version

dx login --token <TOKEN> --noprojects
dx select <PROJECT_ID>
dx whoami
```

### Navigation

```bash
dx pwd                                    # Current working directory
dx cd /folder                             # Change directory
dx ls                                     # List current directory
dx ls -l                                  # Detailed listing (shows file IDs)
dx ls /path/to/folder                     # List specific folder
dx tree /folder                           # Directory tree
```

### File Operations

```bash
# Upload (local → RAP)
dx upload local_file.txt                  # Upload single file
dx upload local_file.txt --path /dest/    # Upload to specific folder
dx upload -r local_dir/                   # Upload directory recursively
dx upload file.txt --brief                # Print only file ID

# Download (RAP → local)
dx download file_name                     # Download by name
dx download file-xxxxxxxxxxxx             # Download by file ID
dx download -r /folder/                   # Download folder recursively

# Manage
dx mkdir -p /path/to/folder               # Create folders
dx mv old_name new_name                   # Rename/move
dx rm file_name                           # Delete file
dx rm -r /folder/                         # Delete folder recursively
dx cp file project-yyyy:/dest/            # Copy across projects
```

### Search

```bash
dx find data --name "*.bam"               # Search by filename pattern
dx find data --tag my_tag                 # Search by tag
dx find data --class file --path /folder/ # Search in specific folder
dx describe file_name                     # Detailed metadata for any object
```

---

## Running Jobs with Swiss Army Knife (SAK)

The Swiss Army Knife app launches a VM with preinstalled bioinformatics tools and runs a user-specified command.

**Preinstalled tools**: plink, plink2, regenie, bcftools, samtools, bedtools, tabix, bgzip, vcftools, bgenix, qctool, bolt-lmm, and more.

### ⚠️ Common Mistakes That Will Fail

**❌ MISTAKE 1: Using dxfuse paths in your command**

```bash
# THIS FAILS — /dm/ paths don't exist inside SAK containers:
dx run swiss-army-knife \
  -icmd="cd /dm/project/scripts && bash run.sh" \
  --destination /output/
```

**Why**: SAK jobs start in `/home/dnanexus/`. Local dxfuse mounts (`/dm/`, `/mnt/project/`) are not available inside the container.

**❌ MISTAKE 2: Forgetting to pass scripts via `-iin=`**

```bash
# THIS FAILS — scripts must be explicitly passed:
dx run swiss-army-knife \
  -icmd="Rscript analysis.R" \
  --destination /output/
```

**Why**: Files on RAP aren't automatically in the container. You must pass them via `-iin=`.

**❌ MISTAKE 3: Passing duplicate filenames**

```bash
# THIS FAILS — same filename from two different file IDs:
dx run swiss-army-knife \
  -iin=file-XXXXX1 \    # config.R (old version)
  -iin=file-XXXXX2 \    # config.R (new version)
  ...
```

**Why**: SAK downloads all inputs to the working directory. Duplicate filenames cause `mv` conflicts.

### ✅ Correct SAK Pattern

```bash
# Step 1: Get file IDs for scripts
dx ls -l /path/to/scripts/

# Step 2: Submit with file IDs for scripts, project-qualified paths for data
dx run swiss-army-knife \
  -icmd="<command_using_basenames_only>" \
  -iin=file-XXXXX1 \                              # Scripts: file IDs
  -iin=file-XXXXX2 \
  -iin="<PROJECT_ID>:/path/to/data/file.csv" \     # Data: project-qualified paths
  --destination /output/folder/ \
  --instance-type <instance> \
  --priority normal \
  --cost-limit 5.00 \
  --name "Job description" \
  --brief --yes
```

**Key rules**:

1. **Scripts → file IDs** (`file-XXXXX`) — get from `dx ls -l`
2. **Data → project-qualified paths** (`<PROJECT_ID>:/path/...`) — needed for `dx download` at job start
3. **Command → basenames only** (`bash run.sh`, NOT `bash /dm/scripts/run.sh`)
4. **No duplicate filenames** — if you re-uploaded a script, use only the latest file ID
5. **Verify paths first** — run `dx ls <path>` for every file before submitting

### Example: PLINK2 Frequency Calculation

```bash
dx run swiss-army-knife \
  -icmd="plink2 --bfile input --freq --out results" \
  -iin=/genotypes/HM3/ukb_hm3.bed \
  -iin=/genotypes/HM3/ukb_hm3.bim \
  -iin=/genotypes/HM3/ukb_hm3.fam \
  --destination /output/ \
  --instance-type mem1_ssd1_v2_x4 \
  --priority low \
  --name "HM3 allele frequencies" \
  --brief --yes
```

### Example: R Analysis with Multiple Scripts

```bash
dx run swiss-army-knife \
  -icmd="bash run_wrapper.sh" \
  -iin=file-XXXXXXXXX1 \    # run_wrapper.sh
  -iin=file-XXXXXXXXX2 \    # 00_config.R
  -iin=file-XXXXXXXXX3 \    # 01_analysis.R
  -iin="<PROJECT_ID>:/path/to/data/phenotypes.csv" \
  --destination /path/to/results/ \
  --instance-type mem2_ssd1_v2_x2 \
  --priority normal \
  --cost-limit 10.00 \
  --name "R_analysis" \
  --brief --yes
```

---

## Instance Types

Format: `mem{N}_ssd{N}_v{N}_x{cores}`

| Instance | Cores | RAM | SSD | Use Case |
|----------|-------|-----|-----|----------|
| mem1_ssd1_v2_x2 | 2 | 4 GB | 40 GB | Light tasks, file conversion |
| mem1_ssd1_v2_x4 | 4 | 8 GB | 80 GB | Small GWAS, QC |
| mem1_ssd1_v2_x8 | 8 | 16 GB | 160 GB | Medium GWAS |
| mem1_ssd1_v2_x16 | 16 | 32 GB | 320 GB | Large GWAS, regenie step 1 |
| mem2_ssd1_v2_x2 | 2 | 8 GB | 40 GB | Memory-intensive small jobs |
| mem2_ssd1_v2_x8 | 8 | 32 GB | 160 GB | Large data processing |
| mem3_ssd1_v2_x4 | 4 | 32 GB | 80 GB | High-memory tasks |
| mem3_ssd1_v2_x16 | 16 | 128 GB | 320 GB | Very large analyses |

### Choosing Instance Size

**Rule of thumb**: RAM should be ~5–10× the size of your largest data file when doing statistical analyses.

- **Small data** (<100 MB): `mem1_ssd1_v2_x2` (4 GB)
- **Medium data** (100–500 MB): `mem1_ssd1_v2_x8` (16 GB)
- **Large data** (500 MB+, full UKB cohort): `mem1_ssd1_v2_x16` (32 GB) or `mem2_ssd1_v2_x8` (32 GB)

**Symptom of insufficient RAM**: Job killed with "Out of memory" (OOM). Solution: resubmit with 2–4× more RAM.

```bash
dx run --instance-type-help   # List all available instance types
```

---

## Priority Levels

| Priority | Description | Cost |
|----------|-------------|------|
| `low` | Spot instances (may be preempted mid-run) | Cheapest |
| `normal` | Tries spot for 15 min, then falls back to on-demand | Mid-range |
| `high` | On-demand only (guaranteed) | Most expensive |

**Recommendation**: Use `--priority normal` for most work. Use `low` only for quick/cheap jobs you don't mind restarting.

---

## Monitoring Jobs

```bash
dx find jobs -n 20                        # List recent jobs
dx watch job-xxxxxxxxxxxx                 # Stream live logs
dx watch job-xxxx --get-stdout            # Stream stdout only
dx describe job-xxxxxxxxxxxx              # Job details, status, cost
dx terminate job-xxxxxxxxxxxx             # Kill a running job
```

---

## Batch Jobs (e.g. per-chromosome GWAS)

```bash
for chr in $(seq 1 22); do
  dx run swiss-army-knife \
    -icmd="plink2 --bgen chr${chr}.bgen ref-first --sample ukb.sample --pheno pheno.txt --glm --out chr${chr}_gwas" \
    -iin=/genotypes/chr${chr}.bgen \
    -iin=/genotypes/ukb.sample \
    -iin=/input/pheno.txt \
    --destination /output/gwas/ \
    --instance-type mem1_ssd1_v2_x8 \
    --priority low \
    --name "GWAS chr${chr}" \
    --brief --yes
done
```

---

## Extracting Phenotype Data (Table Exporter)

UK Biobank phenotype data is stored in a database on the platform. To extract fields:

```bash
dx run table-exporter \
  -idataset_or_cohort_or_dashboard=record-XXXXX \
  -ifield_names_file_txt=file-XXXXX \
  -ioutput="output_name" \
  -icoding_option="REPLACE" \
  -ientity="participant" \
  --priority normal \
  --instance-type mem2_ssd1_v2_x2 \
  --cost-limit 5.00 \
  --brief --yes
```

**Field file format**: one field per line, ending with a blank line:
```
eid
p23400_i0
p23401_i0

```

**Common mistakes that cause failures**:
- Missing `coding_option` or `entity` → job stalls indefinitely with no error
- Using `field_names` instead of `field_names_file_txt` → error
- Adding `output_format` or `header_style` → may cause issues

**Note**: For GP clinical/prescription data, use `-ientity="gp_clinical"` or `-ientity="gp_scripts"` instead of the participant entity.

---

## R Scripts on SAK: The Download Pattern

R scripts that read from `/dm/` paths directly **will fail** in SAK jobs. You must download files explicitly.

### ❌ This Fails

```r
df <- fread("/dm/project/data/file.csv")
# Error: File does not exist or is non-readable
```

### ✅ This Works

```r
# Define at top of every R script
RAP_PROJECT <- "<YOUR_PROJECT_ID>"

dx_download <- function(rap_path) {
  local_name <- basename(rap_path)
  if (!file.exists(local_name)) {
    full_path <- paste0(RAP_PROJECT, ":", rap_path)
    cmd <- paste0("dx download '", full_path, "' -o '", local_name, "' -f --no-progress")
    cat("Downloading:", rap_path, "\n")
    system(cmd)
  } else {
    cat("Using cached:", local_name, "\n")
  }
  return(local_name)
}

# Use it
local_file <- dx_download("/path/to/data/phenotypes.csv")
df <- fread(local_file)
```

### Wrapper Script Pattern

```bash
#!/bin/bash
# run_analysis.sh

# Install R packages (use Posit Package Manager for fast binary installs)
Rscript -e "install.packages(c('data.table','survival'), repos='https://packagemanager.posit.co/cran/__linux__/noble/latest', Ncpus=8, quiet=TRUE)"

# Run R script (which internally downloads the files it needs)
Rscript -e "source('01_analysis.R')"

# Upload results (use SIMPLE paths, not project-qualified)
dx mkdir -p /path/to/results/ || true
dx upload *.csv --destination /path/to/results/
```

---

## Gotchas & Lessons Learned

### SAK Container Path Resolution

SAK jobs run inside a container. `dx download "path/to/file.csv"` resolves against the container's project, not yours.

**Fix**: Always use project-qualified paths for downloads:
```bash
dx download "<PROJECT_ID>:/path/to/file.csv" -o file.csv -f --no-progress
```

### dx Commands Inside SAK Jobs

Use **simple paths** (not project-qualified) for `dx mkdir`, `dx upload`, and `dx mv`:

```bash
# WRONG — causes permission errors:
dx mkdir -p "<PROJECT_ID>:/path/to/results/"
dx upload file.csv --destination "<PROJECT_ID>:/path/to/results/"

# CORRECT — job already runs in project context:
dx mkdir -p "/path/to/results/" || true
dx upload file.csv --destination "/path/to/results/"
```

**Exception**: `dx download` at job start SHOULD use project-qualified paths.

### SAK Auto-Upload Limitations

SAK auto-uploads files from `/home/dnanexus/out/` to `--destination`, but:
- Only flat structure allowed — subdirectories like `out/tables/` cause validation errors
- All files go to the `--destination` root

**Best practice**: Use explicit `dx upload` for organized output instead of relying on auto-upload.

### Duplicate Files Break `dx download`

If a file path resolves to multiple objects (e.g. from re-running a job), `dx download` fails with `ResolutionError`. **Fix**: download by file ID:

```bash
dx ls -l /output/tables/           # Shows file IDs
dx download file-xxxxxxxxxxxx -o local_name.csv
```

### R Package Installation on SAK

The SAK image is Ubuntu Noble. Use Posit Package Manager for fast pre-compiled binary installs:

```r
repos <- "https://packagemanager.posit.co/cran/__linux__/noble/latest"
install.packages("xgboost", repos = repos, Ncpus = 8, quiet = TRUE)
```

### R sprintf() Gotchas

- **No `%,d`** — R doesn't support comma separators in `sprintf()`. Use `format(x, big.mark = ",")`
- **Use `%.0f` not `%d`** for `min()`/`max()` output — they return numeric, not integer

### Low vs Normal Priority

`--priority low` uses spot instances that can be preempted mid-run. `--priority normal` tries spot for 15 minutes, then falls back to on-demand. Use `normal` for anything that takes more than ~10 minutes.

### Spot Instance Restarts

Even with `--priority normal`, the first 15 minutes run on spot. If interrupted, DNAnexus auto-retries on on-demand. Check `failureCounts` in `dx describe` if a job seems to have restarted.

### Table Exporter CLI Syntax

`--project` and `--destination` cannot be used together. Include the project in `--destination`:

```bash
# WRONG:
dx run app-table-exporter --project <PROJECT_ID> --destination /output/ ...

# RIGHT:
dx run app-table-exporter --destination "<PROJECT_ID>:/output/" ...
```

### UKB Data Tips

- **First Occurrences fields** (p130xxx–p132xxx) already combine HES + GP + death registry. You don't need to separately process these sources for ascertainment dates.
- **NMR QC flags** (p23700–p23780) are mostly empty. Use sample-level missingness instead.
- **HES ICD-10 array fields** (p41270, p41280) have up to 259 indices per participant — very wide CSVs. For specific ICD-10 codes, prefer First Occurrences or Spark SQL on the `hesin_diag` table.
- **p2976 vs p2986**: p2976 = age at diabetes diagnosis; p2986 = age started insulin. Commonly confused.

---

## Pre-Submission Checklist

Before submitting ANY SAK job:

1. **Verify all file paths exist**: `dx ls <path>` for every file referenced in your scripts
2. **Get file IDs for scripts**: `dx ls -l /path/to/scripts/` — check for duplicates
3. **Check config paths are complete**: abbreviated paths like `dm/file.csv` will fail if the real path is longer
4. **Build `-iin=` list correctly**: scripts as file IDs, data as project-qualified paths, command uses basenames only
5. **Check instance size vs data size**: RAM should be 5–10× your largest input file

---

## Cost Management

- Storage is billed monthly; compute is billed per-hour per instance type
- Set `--cost-limit` on every job to cap runaway costs
- Use the smallest instance that fits your task
- Clean up output files you no longer need
- Check spending in project Settings → Billing on the web UI

---

## Useful Links

- [RAP Web UI](https://ukbiobank.dnanexus.com/)
- [DNAnexus Documentation](https://documentation.dnanexus.com/)
- [RAP Documentation](https://dnanexus.gitbook.io/uk-biobank-rap/)
- [UKB RAP Notebooks](https://github.com/UK-Biobank/UKB-RAP-Notebooks-Access)
- [UKB Genomics Notebooks](https://github.com/UK-Biobank/UKB-RAP-Notebooks-Genomics)
