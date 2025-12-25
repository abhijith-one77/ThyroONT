# ThyroONT
#Script summary in detail for the minion pipeline

````markdown
# Oxford Nanopore Variant Calling Pipeline  
### Basecalling → Filtering → Alignment → Coverage → Small Variants → Structural Variants

This pipeline processes Oxford Nanopore **R10.4.1** sequencing data and generates:

- Filtered FASTQ reads
- Aligned BAM file
- Coverage statistics
- SNP/Indel variants (Clair3)
- Structural variants (Sniffles2)

Designed for:
✔ Slurm HPC  
✔ ONT long-read data  
✔ Human GRCh38 reference  

---

## 🔧 Software Requirements

Installed via Conda:

- minimap2
- samtools
- mosdepth
- clair3
- sniffles2
- python ≥ 3.9
- bash

---

## 📁 Input Files

| Path | Description |
|------|-------------|
| `fastq_pass/*.fastq.gz` | ONT basecalled reads |
| `reference.fasta` | GRCh38 genome |
| `targets.bed` *(optional)* | Regions for variant calling |

---

## 🧪 Output Files

| Output | Description |
|--------|-------------|
| `minion_filtered.fastq` | Reads ≥1000bp |
| `minion_sorted.bam` | Aligned reads |
| `minion_depth.*` | Coverage |
| `clair3_output/*.vcf.gz` | SNP/Indel calls |
| `minion_inversion_results.vcf` | Structural variants |
| `inversions_PASS.vcf` | High-confidence inversions |

---

## 🚀 Usage

```bash
bash pipeline.sh
````

Logs appear in the terminal.

---

## 📌 Notes

* `fastq_pass` reads are already quality-filtered
* Filtering to ≥1000bp improves accuracy
* Clair3 model is ONT-trained
* Sniffles2 detects large rearrangements (like inversions)

---

## 🧬 Citation

If you use this workflow, please cite:

* Dorado (Oxford Nanopore)
* minimap2
* samtools
* Clair3
* Sniffles2
* mosdepth

`````

---

# 🟩 B. FULL PIPELINE SCRIPT  
## `pipeline.sh`  
### (This is COMPLETE — nothing missing)

> Replace project path if needed.

````bash
#!/usr/bin/env bash
set -euo pipefail

###############################################
# USER CONFIGURATION
###############################################
PROJECT=/rds/projects/d/dixonh-02/PROMSW1736celllineAS
FASTQ_DIR=$PROJECT/fastq_pass
THREADS=32

cd $FASTQ_DIR

###############################################
echo "STEP 1 — Merge FASTQ files"
###############################################

# Combine all pass fastq.gz files
zcat FAZ62778_pass_*.fastq.gz > minion_pass.fastq

# Check size
ls -lh minion_pass.fastq


###############################################
echo "STEP 2 — Filter reads ≥1000bp"
###############################################

awk 'BEGIN {FS="\n"; RS="@"; ORS=""} length($2) >= 1000 {print "@"$0}' \
 minion_pass.fastq > minion_filtered.fastq

ls -lh minion_filtered.fastq


###############################################
echo "STEP 3 — Download GRCh38 Reference"
###############################################

wget -c https://hgdownload.cse.ucsc.edu/goldenpath/hg38/bigZips/analysisSet/hg38.analysisSet.chroms.tar.gz
tar -xzvf hg38.analysisSet.chroms.tar.gz
cat hg38.analysisSet.chroms/*.fa > reference.fasta

# Index reference
samtools faidx reference.fasta


###############################################
echo "STEP 4 — Align reads using minimap2"
###############################################

minimap2 -ax map-ont -t $THREADS --secondary=no \
 reference.fasta minion_filtered.fastq \
 | samtools view -bS - > minion_aligned.bam


###############################################
echo "STEP 5 — Sort & index BAM"
###############################################

samtools sort -@ $THREADS -m 2G minion_aligned.bam -o minion_sorted.bam
samtools index minion_sorted.bam


###############################################
echo "STEP 6 — Compute coverage with mosdepth"
###############################################

mosdepth -t $THREADS --fast-mode --by 500 \
 minion_depth minion_sorted.bam

mkdir -p minion_mosdepth
mv minion_depth* minion_mosdepth/


###############################################
echo "STEP 7 — Create Clair3 Environment (first run only)"
###############################################

conda create -n clair3_env -c conda-forge -c bioconda clair3 python=3.9 -y || true
source $(conda info --base)/etc/profile.d/conda.sh
conda activate clair3_env


###############################################
echo "STEP 8 — Locate ONT Model"
###############################################

MODEL_PATH=$(dirname $(which run_clair3.sh))/../bin/models/ont


###############################################
echo "STEP 9 — Run Clair3 Variant Calling"
###############################################

run_clair3.sh \
  --bam_fn=minion_sorted.bam \
  --ref_fn=reference.fasta \
  --threads=$THREADS \
  --platform="ont" \
  --model_path="${MODEL_PATH}" \
  --output=clair3_output


###############################################
echo "STEP 10 — Variant Statistics"
###############################################

echo "Total variants:"
zgrep -v "^#" clair3_output/merge_output.vcf.gz | wc -l

echo "Variants with QUAL>20:"
zgrep -v "^#" clair3_output/merge_output.vcf.gz | awk '$6 > 20' | wc -l


###############################################
echo "STEP 11 — Install Sniffles2 (once)"
###############################################

pip install sniffles==2.2


###############################################
echo "STEP 12 — Run Sniffles2"
###############################################

sniffles \
 --input minion_sorted.bam \
 --reference reference.fasta \
 --vcf minion_inversion_results.vcf \
 --threads $THREADS


###############################################
echo "STEP 13 — Extract high confidence inversions"
###############################################

grep "SVTYPE=INV" minion_inversion_results.vcf | grep PASS > inversions_PASS.vcf


###############################################
echo "🎉 PIPELINE COMPLETE"
###############################################
`````

---

# 🟨 C. GPU Session Script (Optional)

## `gpu_session.sh`

```bash
srun \
 --account=dixonh-02 \
 --qos=bbgpu \
 --gres=gpu:1 \
 --time=03:00:00 \
 --mem=64G \
 --cpus-per-task=32 \
 --pty /bin/bash
```

Then run:

```bash
bash pipeline.sh
```

---

# 📖 STEP-BY-STEP EXPLANATION (Plain English)

### **Step 1 — Merge FASTQ**

ONT outputs many small FASTQ files
We combine them into one:

```bash
zcat *.fastq.gz > minion_pass.fastq
```

---

### **Step 2 — Filter reads**

Keep only long reads:

```bash
length(sequence) ≥ 1000 bp
```

Reason: improves accuracy.

---

### **Step 3 — Download GRCh38**

We use the Analysis Set (standard for variant calling).

Index it for fast lookup.

---

### **Step 4 — Align**

ONT requires `map-ont` preset.

`--secondary=no` avoids duplicate alignments.

---

### **Step 5 — Sort & Index**

Required for downstream tools.

---

### **Step 6 — Coverage**

500bp windows = smoother plot.

---

### **Step 7-10 — Clair3**

Neural-network caller for small variants.

We count:

* total variants
* high-confidence variants (QUAL > 20)

---

### **Step 11-13 — Sniffles2**

Detects big variants:

* deletions
* insertions
* inversions
* duplications

We extract **inversions only**.



