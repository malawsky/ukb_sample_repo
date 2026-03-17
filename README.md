# UKB-RAP Starter Repository

A local workspace template for managing analyses on the UK Biobank Research Analysis Platform (DNAnexus).

## What This Is

A minimal setup for working with UK Biobank data on the Research Analysis Platform (RAP). It includes a CLI reference guide, recommended project structure, and common gotchas that will save you hours of debugging.

## Repository Structure

```
ukb-rap/
├── README.md                   # This file
├── docs/
│   └── RAP_GUIDE.md            # Platform guide, dx CLI reference, common pitfalls
├── projects/                   # Analysis-specific project folders
│   └── <project_name>/         # Each analysis gets its own folder
│       ├── README.md           # What this analysis does
│       ├── scripts/            # Scripts to run on RAP (bash, R, python)
│       └── results/            # Downloaded results (local copies)
└── scripts/
    └── common/                 # Shared utility scripts
```

## Quick Start

```bash
# 1. Login
dx login --token <YOUR_TOKEN> --noprojects
dx select <YOUR_PROJECT_ID>

# 2. Browse data
dx ls /Bulk/
dx ls /genotypes/

# 3. Run a test job
dx run swiss-army-knife \
  -icmd="plink2 --version" \
  --instance-type mem1_ssd1_v2_x2 \
  --priority low \
  --brief --yes

# 4. Monitor jobs
dx find jobs -n 10
dx watch <job-id>
```

## Recommended RAP Project Structure

Keep your RAP directories organized and mirroring this local repo:

```
<your-project>/
├── data/              # Raw data (treat as read-only after creation)
├── scripts/           # Analysis scripts (versioned)
├── results/           # All outputs
│   ├── tables/
│   ├── figures/
│   └── logs/
├── phenotypes/        # Phenotype files (if running GWAS)
└── gwas/              # GWAS results (if applicable)
    ├── step1/
    └── step2/
```

## Key Principles

1. **All computation runs on RAP** — data cannot be downloaded locally
2. **Mirror your local repo structure on RAP** — makes it obvious where files live
3. **Version your analyses** — use `v1/`, `v2/` subdirectories for major revisions
4. **Never delete, only archive** — move obsolete files to an `archive/` folder
5. **Read `docs/RAP_GUIDE.md`** before submitting your first job — the common mistakes section alone will save you hours

## Useful Links

- [RAP Web UI](https://ukbiobank.dnanexus.com/)
- [DNAnexus Documentation](https://documentation.dnanexus.com/)
- [RAP Documentation](https://dnanexus.gitbook.io/uk-biobank-rap/)
- [UKB RAP Notebooks](https://github.com/UK-Biobank/UKB-RAP-Notebooks-Access)
- [UKB Genomics Notebooks](https://github.com/UK-Biobank/UKB-RAP-Notebooks-Genomics)
