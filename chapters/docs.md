# Shotgun metagenomics preprocessing and profiling workflow with KneadData and Meteor2

## Overview

This document describes a reproducible workflow for processing paired-end shotgun metagenomic reads using **KneadData** for host-read depletion and quality control, followed by **Meteor2** for taxonomic, functional, and optional strain-level profiling. The workflow is designed for host-associated microbiome datasets, particularly **human gut shotgun metagenomes**, and reflects the command-line interfaces and recommendations documented in the official KneadData and Meteor repositories.

The pipeline is structured in two major stages:

1. **Preprocessing and host-read removal with KneadData**
2. **Metagenomic profiling with Meteor2**

The expected final outputs are:

* host-depleted, high-quality paired FASTQ files from KneadData
* per-sample gene count tables from Meteor2 mapping
* per-sample taxonomic and functional abundance profiles from Meteor2 profiling
* merged multi-sample abundance tables for downstream analysis

---

# Part I. Host read removal and quality control with KneadData

## 1. Purpose

KneadData is a quality-control and decontamination tool designed for metagenomic and metatranscriptomic sequencing data, especially microbiome datasets. In host-associated samples, sequencing libraries often contain a substantial fraction of host-derived reads. KneadData performs **in silico separation of microbial reads from contaminant reads**, typically using:

* **Trimmomatic** for adapter/quality trimming
* **TRF** for optional low-complexity/repetitive sequence filtering
* **Bowtie2** for contaminant/host read removal against a reference database

In this workflow, KneadData is used to remove **human reads** before downstream profiling with Meteor2.

---

## 2. Installation

### 2.1 Recommended installation with Conda

A Conda installation is recommended because it resolves the main dependencies required by KneadData.

```bash
conda create -n kneaddata -c bioconda kneaddata -y
conda activate kneaddata
```

This installation typically provides the core software required to run the workflow, including:

* KneadData
* Bowtie2
* Trimmomatic
* Java runtime support

Alternative installation methods exist, including `pip` and Docker, but Conda is the preferred option for a reproducible local CLI workflow.

---

## 3. Download the human host reference database

KneadData provides helper commands to download reference databases compatible with Bowtie2.

```bash
kneaddata_database --download human_genome bowtie2 $DIR
```

Where:

* `$DIR` is the directory where the reference database will be stored

### Notes

* For human host depletion, the recommended database is the **human hg39 reference** supplied by KneadData
* The database corresponds to **T2T-CHM13v2.0**
* Approximate size: **3.6 GB**

Example:

```bash
kneaddata_database --download human_genome bowtie2 /home/dani/kneaddata_test/database
```

---

## 4. Input requirements

This workflow assumes **paired-end FASTQ** input files.

Generic paired input:

```text
sample_R1.fastq.gz
sample_R2.fastq.gz
```

Example real inputs:

```text
/home/dani/kneaddata_test/input/36522947_R1.fastq.gz
/home/dani/kneaddata_test/input/36522947_R2.fastq.gz
```

---

## 5. Run KneadData on paired-end reads

### 5.1 Basic paired-end command

```bash
kneaddata \
  --input1 seq1.fastq.gz \
  --input2 seq2.fastq.gz \
  -db $DATABASE \
  --output kneaddata_output
```

### Argument description

| Argument   | Description                                        |
| ---------- | -------------------------------------------------- |
| `--input1` | Forward mate (R1)                                  |
| `--input2` | Reverse mate (R2)                                  |
| `-db`      | Path to the KneadData/Bowtie2 contaminant database |
| `--output` | Directory where output files will be written       |

---

## 6. Real execution examples

### Sample 1

```bash
kneaddata \
  --input1 /home/dani/kneaddata_test/input/36522947_R1.fastq.gz \
  --input2 /home/dani/kneaddata_test/input/36522947_R2.fastq.gz \
  -db /home/dani/kneaddata_test/database \
  --output /home/dani/kneaddata_test/output
```

### Sample 2

```bash
kneaddata \
  --input1 /home/dani/kneaddata_test/input/36525128_R1.fastq.gz \
  --input2 /home/dani/kneaddata_test/input/36525128_R2.fastq.gz \
  -db /home/dani/kneaddata_test/database \
  --output /home/dani/kneaddata_test/output
```

---

## 7. Parallelization and performance tuning

KneadData allows control over computational resources.

```bash
kneaddata \
  --input1 seq1.fastq.gz \
  --input2 seq2.fastq.gz \
  -db $DATABASE \
  --output kneaddata_output \
  -t 16 \
  -p 8
```

### Relevant options

| Option | Meaning                                                                |
| ------ | ---------------------------------------------------------------------- |
| `-t`   | Number of threads used by underlying tools such as Bowtie2/Trimmomatic |
| `-p`   | Number of parallel processes                                           |

These parameters should be adjusted according to available CPU and memory resources.

---

## 8. Batch execution over all samples in an input directory

If all paired-end reads are stored in the same input directory with a consistent naming convention (`*_R1.fastq.gz` and `*_R2.fastq.gz`), processing can be automated.

```bash
INPUT_DIR=/home/dani/kneaddata_test/input
OUTPUT_DIR=/home/dani/kneaddata_test/output
DB_DIR=/home/dani/kneaddata_test/database

for r1 in ${INPUT_DIR}/*_R1.fastq.gz; do
  base=$(basename "${r1}" _R1.fastq.gz)

  kneaddata \
    --input1 "${INPUT_DIR}/${base}_R1.fastq.gz" \
    --input2 "${INPUT_DIR}/${base}_R2.fastq.gz" \
    -db "${DB_DIR}" \
    --output "${OUTPUT_DIR}"
done
```

This assumes each sample has exactly one forward file and one reverse file.

---

## 9. Main output files produced by KneadData

KneadData creates several intermediate and final files. The **primary outputs of interest for downstream metagenomic profiling** are the paired clean files.

### 9.1 Main paired outputs

Typical filenames:

```text
36522947_R1_kneaddata_paired_1.fastq
36522947_R1_kneaddata_paired_2.fastq
```

These files contain reads that:

* remain properly paired after trimming
* passed quality control
* did **not** align to the host reference database

These are the **clean paired-end reads** that should be used as input for Meteor2.

### 9.2 Unmatched outputs

Typical filenames:

```text
36522947_R1_kneaddata_unmatched_1.fastq
36522947_R1_kneaddata_unmatched_2.fastq
```

These contain reads that survived filtering but lost their mate during trimming or decontamination. They may be useful in some contexts, but for the workflow documented here they are **not used downstream in Meteor2**.

### 9.3 Intermediate files

KneadData also generates multiple intermediate files, for example:

* decompressed FASTQ files
* reformatted identifier files
* trimmed paired and trimmed single files
* Bowtie2 clean intermediate files

These are useful for auditing and troubleshooting, but they are not the final files intended for profiling.

---

## 10. Important naming caveat in KneadData outputs

KneadData uses the basename of `--input1` as the prefix for all generated outputs. This means that output filenames may look confusing at first sight.

Example:

```text
36522947_R1_kneaddata_paired_2.fastq
```

Even though the file ends in `_paired_2.fastq` and corresponds to the reverse mate, its prefix still contains `R1` because KneadData derives the output prefix from the first input file.

This does **not** indicate a technical problem, but the filenames are not ideal for downstream clarity.

---

## 11. Recommended renaming strategy

To standardize the naming and make downstream processing easier, it is advisable to rename or symlink the paired clean files.

### Example renaming

```bash
mv 36522947_R1_kneaddata_paired_1.fastq 36522947_clean_R1.fastq
mv 36522947_R1_kneaddata_paired_2.fastq 36522947_clean_R2.fastq
```

A consistent naming convention is strongly recommended before importing the files into Meteor2.

---

## 12. Optional use of `--output-prefix`

KneadData also supports explicit output naming through an output prefix.

```bash
kneaddata \
  --input1 seq1.fastq.gz \
  --input2 seq2.fastq.gz \
  -db $DATABASE \
  --output kneaddata_output \
  --output-prefix sample_name
```

This can make the generated filenames more predictable and easier to standardize.

---

## 13. Practical summary of the KneadData stage

At the end of the KneadData stage, the files that should be retained for downstream profiling are the paired clean FASTQ files:

```text
*_kneaddata_paired_1.fastq
*_kneaddata_paired_2.fastq
```

These represent:

* high-quality reads
* host-depleted reads
* paired-end reads suitable for microbial profiling

---

# Part II. Metagenomic profiling with Meteor2

## 14. Purpose

Meteor2 is a platform for quantitative shotgun metagenomic profiling of complex microbial ecosystems. It relies on **ecosystem-specific microbial gene catalogues** and supports:

* taxonomic profiling
* functional profiling
* optional strain-level analysis

For the present workflow, Meteor2 is used after KneadData to profile **human gut microbiome** shotgun data.

---

## 15. Installation

Meteor2 can be installed in a dedicated Conda environment.

```bash
conda create --name meteor -c conda-forge -c bioconda meteor -y
conda activate meteor
```

A separate environment from KneadData is recommended to avoid dependency conflicts.

---

## 16. Catalogue download

Meteor2 requires a gene catalogue matching the ecosystem under study.

For **human gut** samples, the corresponding catalogue is:

```text
hs_10_4_gut
```

### 16.1 Recommended catalogue download with Meteor2

Meteor provides an internal command to download, verify, and prepare catalogues.

#### Full catalogue

```bash
meteor download -i hs_10_4_gut -o /home/dani/meteor2_test/catalogues -c
```

#### Light / fast catalogue

```bash
meteor download -i hs_10_4_gut --fast -o /home/dani/meteor2_test/catalogues -c
```

### Why this is the preferred method

Using `meteor download` is preferable because it:

* downloads the correct catalogue version
* verifies file integrity with checksum validation
* prepares the catalogue automatically

---

## 17. Alternative manual catalogue download and extraction

If the catalogue has already been downloaded manually, the corresponding files may look like this:

```text
hs_10_4_gut.tar.xz
hs_10_4_gut_taxo.tar.xz
```

In that case, they must be extracted manually.

```bash
cd /home/dani/meteor2_test/catalogues

tar -xvf hs_10_4_gut.tar.xz
tar -xvf hs_10_4_gut_taxo.tar.xz
```

This step is only necessary if the catalogue was not obtained through `meteor download`.

---

## 18. Input requirements for Meteor2

Meteor2 does not start directly from a single FASTQ pair passed to a generic top-level command. Instead, the workflow is divided into multiple subcommands.

The first step is **FASTQ import/indexing** with `meteor fastq`.

Meteor expects a directory containing input FASTQ files. For paired-end data, filenames must follow one of the accepted patterns:

```text
sample_R1.fastq and sample_R2.fastq
```

or

```text
sample_1.fastq and sample_2.fastq
```

### Important implication for KneadData outputs

KneadData output names such as:

```text
36522947_R1_kneaddata_paired_1.fastq
36522947_R1_kneaddata_paired_2.fastq
```

can still be compatible with Meteor2 because the suffix pattern `_1.fastq` / `_2.fastq` is present. However, for clarity, traceability, and standardization, it is strongly recommended to rename or symlink the KneadData outputs into a simpler, explicit paired-end convention before running `meteor fastq`.

### Suggested Meteor input naming

```text
36522947_R1.fastq
36522947_R2.fastq
36525128_R1.fastq
36525128_R2.fastq
```

---

## 19. Avoiding file duplication with symbolic links

Since KneadData outputs may already consume substantial disk space, one practical strategy is to create symbolic links instead of copying files.

### Example

```bash
mkdir -p /home/dani/meteor2_test/input
cd /home/dani/meteor2_test/input

ln -s /path/to/36522947_R1_kneaddata_paired_1.fastq 36522947_R1.fastq
ln -s /path/to/36522947_R1_kneaddata_paired_2.fastq 36522947_R2.fastq

ln -s /path/to/36525128_R1_kneaddata_paired_1.fastq 36525128_R1.fastq
ln -s /path/to/36525128_R1_kneaddata_paired_2.fastq 36525128_R2.fastq
```

This preserves storage and keeps the Meteor input directory clean and standardized.

---

## 20. Step 1: import and index FASTQ files with `meteor fastq`

This command prepares a Meteor-compatible FASTQ repository from the input directory.

```bash
meteor fastq \
  -i /home/dani/meteor2_test/input \
  -p \
  -o /home/dani/meteor2_test/results/fastq
```

### Meaning of the arguments

| Argument | Description                                      |
| -------- | ------------------------------------------------ |
| `-i`     | Directory containing FASTQ files                 |
| `-p`     | Indicates that the FASTQ files are paired-end    |
| `-o`     | Output directory for the Meteor FASTQ repository |

### Notes

* Meteor2 expects the **directory** of FASTQ files, not an individual FASTQ file here
* The output will contain one subdirectory per imported sample/library
* If multiple sequencing runs belong to the same library, the `-m` option can be used to group them using a regular expression

---

## 21. Step 2: mapping against the gene catalogue with `meteor mapping`

After FASTQ import, each sample must be mapped separately against the selected catalogue.

### Example for one sample

```bash
meteor mapping \
  -i /home/dani/meteor2_test/results/fastq/36522947_R1_kneaddata_paired \
  -r /home/dani/meteor2_test/catalogues/hs_10_4_gut \
  -o /home/dani/meteor2_test/results/mapping \
  -t 6
```

### Important note

The input of `meteor mapping` is **not** the original FASTQ file. It is the **sample directory generated by `meteor fastq`** for a given sample.

### Meaning of the arguments

| Argument | Description                                                                   |
| -------- | ----------------------------------------------------------------------------- |
| `-i`     | Directory corresponding to one imported sample in the Meteor FASTQ repository |
| `-r`     | Catalogue/reference directory                                                 |
| `-o`     | Output directory for mapping results                                          |
| `-t`     | Number of threads                                                             |

---

## 22. Automating the mapping step across all samples

Because `meteor mapping` is run per sample directory, it should be automated when several samples are present.

```bash
FASTQ_REPO=/home/dani/meteor2_test/results/fastq
REF_DIR=/home/dani/meteor2_test/catalogues/hs_10_4_gut
MAP_DIR=/home/dani/meteor2_test/results/mapping

for sample_dir in ${FASTQ_REPO}/*; do
  if [ -d "${sample_dir}" ]; then
    meteor mapping \
      -i "${sample_dir}" \
      -r "${REF_DIR}" \
      -o "${MAP_DIR}" \
      -t 6
  fi
done
```

This loop processes all imported samples one by one.

---

## 23. Step 3: taxonomic and functional profiling with `meteor profile`

Once mapping has been completed for a sample, taxonomic and functional profiling can be performed.

### Example

```bash
meteor profile \
  -i /home/dani/meteor2_test/results/mapping \
  -r /home/dani/meteor2_test/catalogues/hs_10_4_gut \
  -o /home/dani/meteor2_test/results/profile \
  -n coverage
```

### Meaning of the arguments

| Argument      | Description                                                          |
| ------------- | -------------------------------------------------------------------- |
| `-i`          | Directory containing mapped sample output                            |
| `-r`          | Catalogue/reference directory                                        |
| `-o`          | Output directory for profiled sample results                         |
| `-n coverage` | Length-normalized abundance calculation using coverage normalization |

### Notes on normalization

The `-n coverage` option performs gene-length normalization and is the recommended mode in the standard Meteor workflow.

If omitted, no normalization is applied to the gene table.

### Typical profile outputs

Depending on the catalogue type, this step can generate:

* species abundance tables
* ARD abundance tables
* dbCAN abundance tables
* gut metabolic module abundance tables
* gut brain module abundance tables

---

## 24. Automating the profiling step across all mapped samples

As with mapping, profiling is conceptually sample-based and should be applied to each mapped sample directory when needed.

```bash
MAP_DIR=/home/dani/meteor2_test/results/mapping
REF_DIR=/home/dani/meteor2_test/catalogues/hs_10_4_gut
PROFILE_DIR=/home/dani/meteor2_test/results/profile

for sample_dir in ${MAP_DIR}/*; do
  if [ -d "${sample_dir}" ]; then
    meteor profile \
      -i "${sample_dir}" \
      -r "${REF_DIR}" \
      -o "${PROFILE_DIR}" \
      -n coverage
  fi
done
```

---

## 25. Step 4: merge all profiled samples with `meteor merge`

Once all samples have been profiled, the outputs can be merged into multi-sample abundance tables.

```bash
meteor merge \
  -i /home/dani/meteor2_test/results/profile \
  -r /home/dani/meteor2_test/catalogues/hs_10_4_gut \
  -o /home/dani/meteor2_test/results/merge
```

### Meaning of the arguments

| Argument | Description                                         |
| -------- | --------------------------------------------------- |
| `-i`     | Directory containing profiled sample subdirectories |
| `-r`     | Catalogue/reference directory                       |
| `-o`     | Output directory for merged tables                  |

### Output purpose

This step produces merged matrices suitable for downstream statistical analysis across samples.

---

## 26. Optional downstream strain analysis

Meteor2 also supports strain-level analysis, although this is optional and outside the core taxonomic/functional workflow described above.

### Strain inference

```bash
meteor strain \
  -i <mapping_sample_dir> \
  -o <strain_dir> \
  -r <refdir>
```

### Strain tree computation

```bash
meteor tree \
  -i <strain_dir> \
  -o <tree_dir>
```

These steps are only relevant if strain-resolved comparisons are part of the study design.

---

# Part III. Recommended directory organization

A clear directory structure improves traceability and reproducibility.

## Example layout

```text
project/
├── kneaddata/
│   ├── input/
│   ├── database/
│   └── output/
├── meteor2/
│   ├── catalogues/
│   ├── input/
│   ├── output/
│   └── results/
│       ├── fastq/
│       ├── mapping/
│       ├── profile/
│       └── merge/
```

---

# Part IV. Practical guidance and best practices

## 27. General recommendations

### 27.1 Keep the paired clean FASTQ files from KneadData

For this workflow, the correct files to propagate downstream are:

```text
*_kneaddata_paired_1.fastq
*_kneaddata_paired_2.fastq
```

Do not use the unmatched files unless there is a specific analytical reason to process single-end survivors separately.

### 27.2 Standardize names before Meteor import

Even if Meteor can recognize `_1` and `_2`, cleaner names simplify the workflow and reduce ambiguity.

### 27.3 Prefer symbolic links over duplication when possible

Large shotgun datasets quickly become storage-intensive. Symlinks allow a clean and reproducible organization without copying files.

### 27.4 Keep environments separate

Use independent Conda environments for KneadData and Meteor2.

### 27.5 Document the exact catalogue version used

Catalogue versioning is critical for reproducibility because Meteor2 relies on environment-specific references.

### 27.6 Record all parameters used

At minimum, document:

* KneadData database version
* KneadData thread/process settings
* Meteor catalogue name and version
* Meteor normalization mode
* software versions and execution dates

---

# Part V. End-to-end workflow summary

## 28. Conceptual flow

```text
Raw paired-end shotgun FASTQ
        ↓
KneadData
  - trimming
  - repetitive sequence filtering
  - human read removal
        ↓
Clean paired FASTQ
        ↓
Meteor fastq
        ↓
Meteor mapping
        ↓
Meteor profile
        ↓
Meteor merge
        ↓
Multi-sample taxonomic and functional abundance tables
```

---

# Part VI. Minimal command checklist

## 29. KneadData

```bash
conda create -n kneaddata -c bioconda kneaddata -y
conda activate kneaddata
kneaddata_database --download human_genome bowtie2 /path/to/database

kneaddata \
  --input1 sample_R1.fastq.gz \
  --input2 sample_R2.fastq.gz \
  -db /path/to/database \
  --output /path/to/kneaddata_output
```

## 30. Meteor2

```bash
conda create --name meteor -c conda-forge -c bioconda meteor -y
conda activate meteor

meteor download -i hs_10_4_gut -o /path/to/catalogues -c

meteor fastq \
  -i /path/to/input_fastq_dir \
  -p \
  -o /path/to/results/fastq

meteor mapping \
  -i /path/to/results/fastq/sample_dir \
  -r /path/to/catalogues/hs_10_4_gut \
  -o /path/to/results/mapping \
  -t 6

meteor profile \
  -i /path/to/results/mapping/sample_dir \
  -r /path/to/catalogues/hs_10_4_gut \
  -o /path/to/results/profile \
  -n coverage

meteor merge \
  -i /path/to/results/profile \
  -r /path/to/catalogues/hs_10_4_gut \
  -o /path/to/results/merge
```

---

# References

* KneadData official repository
* KneadData bioBakery wiki/tutorial
* Meteor official repository
