# Genome Explorers: Colorectal Cancer Prediction via Microbiome Analysis
*(A BCD Analytics Hub Workshop Project)*

## 🔬 Project Overview
This project investigates the capability to diagnose Colorectal Cancer (CRC) non-invasively by analyzing the human gut microbiome. By processing raw stool metagenomes through a rigorous computational pipeline (Dry Lab), I successfully profiled bacterial communities and deployed Machine Learning algorithms to discover distinct microbial biomarkers that quantitatively discriminate between healthy controls and CRC patients.

## 🎯 Key Objectives
*   **Establish a Dry Lab Pipeline:** Bridge the gap between physical DNA sequencing and computational analysis by configuring a robust Linux bioinformatics environment.
*   **Data Sanitization:** Rigorously download, filter, and decontaminate raw sequencing data (FASTQ) to remove machine errors and human host DNA.
*   **Taxonomic Profiling:** Identify the exact bacterial species present in the purified samples using highly specific reference databases.
*   **Predictive Modeling:** Train a Random Forest Machine Learning classifier to predict CRC status based on microbial abundance and evaluate its clinical viability using AUC-ROC metrics.

## 🛠️ Technologies & Tools Used
**Environment Setup:**
*   macOS Terminal (Native Unix Environment)
*   Anaconda / Miniconda (Environment Management)
*   Google Colab / Jupyter Notebooks

**Bioinformatics Command-Line Tools (Linux):**
*   **SRA Toolkit:** (`prefetch`, `fastq-dump`) For downloading and extracting raw sequences.
*   **FastQC & fastp:** For visualizing and trimming low-quality Phred scores from raw reads.
*   **Bowtie2:** For mapping reads against the human genome (`GRCh38_noalt_as`) to remove host contamination.
*   **Kraken2:** For rapid k-mer based taxonomic classification against the EBI MGnify database.

**Machine Learning (Python):**
*   **Pandas & NumPy:** For complex metadata merging and matrix sanitization.
*   **Scikit-Learn:** For implementing the `RandomForestClassifier` and calculating `roc_auc_score`.
*   **Matplotlib:** For visualizing the ROC curve and top microbial biomarkers.

## 📊 The Bioinformatics Pipeline
This project follows a strict 9-step end-to-end pipeline:

1.  **Environment Configuration:** Establishing the Linux/Conda architecture and pre-downloading massive reference databases (Bowtie2 Index & MGnify human-gut).
2.  **Metadata Merging:** Using Pandas to link clinical patient diagnoses (Cancer vs. Healthy) to their exact sequencing Run IDs.
3.  **Data Acquisition:** Securely downloading highly compressed `.sra` files from the NCBI Sequence Read Archive.
4.  **FASTQ Conversion:** Splitting the raw `.sra` data into readable, paired-end FASTQ files.
5.  **Quality Filtering (fastp):** Computationally dropping DNA bases with Phred scores below 20 (99% accuracy) to eliminate machine sequencing errors.
6.  **Host Decontamination (Bowtie2):** Filtering out human DNA by saving only the reads that *failed* to map to the human reference genome (`--un-conc-gz`).
7.  **Taxonomic Profiling (Kraken2):** Matching the pure microbial reads against the MGnify database to identify the exact bacterial species and their abundances.
8.  **ML Data Preparation:** Aligning the final species abundance matrix with the clinical target vectors, dropping missing values, and binary-encoding the diagnoses.
9.  **Machine Learning:** Training a Random Forest algorithm, manually tuning randomness parameters, evaluating the model via AUC-ROC, and extracting the top 10 bacterial biomarkers driving the predictions.

## 🏆 Results & Conclusion
*   **Model Performance:** The Random Forest classifier successfully differentiated between healthy and cancerous samples, evaluated via a robust AUC-ROC score (mitigating the bias of imbalanced medical datasets).
*   **Biomarker Discovery:** The model successfully identified key taxa (e.g., *Fusobacterium nucleatum*) as primary drivers for CRC classification.
*   **Personal Takeaway:** This project proved that a highly accurate CRC prediction model is entirely dependent on the successful, manual execution of upstream dry-lab tasks. From configuring Linux environments to managing massive storage constraints and outputs, this pipeline demonstrates true, hands-on mastery of computational metagenomics without reliance on automated AI "black boxes".

## 🚀 How to Replicate
The complete, executable code for this project (including both Bash terminal commands and Python scripts) is thoroughly documented step-by-step. Please refer to the `BCD_NGS_Pipeline.md` file (or corresponding `.ipynb` Notebook) in this repository for explicit instructions, required input files, and expected outputs for every step.
