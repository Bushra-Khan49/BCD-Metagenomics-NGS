# 🟦 End-to-End Metagenomics & NGS Pipeline
*(BCD Analytics Hub - Colorectal Cancer Prediction)*

---

### 🔸 Part 1: Initial Preparations
**Step 0: Prerequisites and Environment Setup**
*   **Required Setup:** macOS Terminal (Native Unix Environment), minimum 8GB RAM, and a stable internet connection.
*   **Required Input Files:** None.
*   **Generated Output Files:** A fully configured Unix environment ready for bioinformatics.

Before I started anything, I had to ensure my hardware and software were properly prepared for heavy bioinformatics processing. I made sure my Mac had at least 8GB of RAM and a stable internet connection because the databases I needed to download were massive. Since my laptop is a Mac, I was able to bypass complex virtual machines and run all the heavy bioinformatics commands natively right in my macOS Terminal. I installed all the necessary bioinformatics tools (SRA Toolkit for downloading data, fastp for quality control, Bowtie2 for decontamination, and Kraken2 for taxonomic profiling) directly in my macOS environment. For the Machine Learning phase, I installed Python packages like `pandas`, `numpy`, and `scikit-learn`. Finally, I pre-downloaded the massive reference database files: the GRCh38 human genome for Bowtie2 (to filter out human DNA) and the highly specific MGnify human-gut database for Kraken2 (to identify bacteria).

---

### 🟢 Part 2: Metadata Management
**Step 1 (Python): Preparing the Workspace & Merging Metadata**
*   **Required Setup:** Native macOS Python Environment with `pandas` installed.
*   **Required Input Files:** `META_file_bushra.xlsx` (clinical data), `SraRunTable_bushra.xlsx` (sequencing IDs).
*   **Generated Output Files:** `merged_file.xlsx`

In Machine Learning, raw data is useless without context. First, I had to link the clinical patient data (which tells me if a patient is healthy or has Colorectal Cancer) with the actual sequencing run IDs (the unique codes pointing to their specific DNA samples on the internet). Without this critical step, I wouldn't know which DNA sequence belonged to a healthy person or a cancer patient, making prediction impossible. I wrote a Python script utilizing the Pandas library to seamlessly merge these two Excel sheets together. By matching them perfectly on their shared 'Sample Name' column, I created a unified 'source of truth' document. I then saved this final merged list as `merged_file.xlsx` so I could reference it throughout the rest of the project.

```python
# Import the pandas library for data manipulation
import pandas as pd

# Load the clinical metadata (containing Diagnosis, CRC Stage, Sex, etc.)
print("Loading Metadata...")
df1 = pd.read_excel("META_file_bushra.xlsx", sheet_name=0) 

# Load the SRA Run Table (containing the ERR/SRR sequencing IDs)
print("Loading SRA Run Table...")
df2 = pd.read_excel("SraRunTable_bushra.xlsx", sheet_name=0)

# Merge the two datasets on the shared 'Sample Name' column
print("Merging Datasets...")
merged = pd.merge(df2, df1, on="Sample Name")

# Save the finalized, merged target vector to a new Excel file
merged.to_excel("merged_file.xlsx", index=False)
print("Merge Complete! Saved as merged_file.xlsx")

# Display the first few rows to verify
merged.head()
```

---

### 🔹 Part 3: Raw Data Acquisition
**Step 2 (Bash): Downloading the Raw Sequencing Data**
*   **Required Setup:** macOS Terminal with `sra-tools` installed via Conda.
*   **Required Input Files:** `merged_file.xlsx` (to know which ERR numbers to download).
*   **Generated Output Files:** `sra_data/ERR14218891/ERR14218891.sra`

Next, I needed to actually acquire the biological data for analysis. The NCBI Sequence Read Archive (SRA) hosts petabytes of genomic data. Instead of trying to download these files manually through a web browser, which is slow and prone to corruption, I used the SRA Toolkit's `prefetch` command directly in my macOS terminal. This specialized command securely and efficiently downloads the highly compressed raw sequencing data straight from the NCBI databases into a dedicated folder on my Mac, ensuring the data integrity remains completely intact.

```bash
# Create a directory to store the raw data
mkdir -p sra_data/ERR14218891

# Use prefetch to download the .sra file securely
echo "Downloading SRA file..."
prefetch --output-directory sra_data/ERR14218891 ERR14218891
```

---

### 🟣 Part 4: Data Conversion
**Step 3 (Bash): FASTQ Conversion**
*   **Required Setup:** macOS Terminal with `sra-tools` installed.
*   **Required Input Files:** `ERR14218891.sra`
*   **Generated Output Files:** `ERR14218891_1.fastq.gz` (Forward reads), `ERR14218891_2.fastq.gz` (Reverse reads).

The raw `.sra` file I just downloaded is a highly compressed, proprietary binary format that downstream bioinformatics tools simply cannot read. I needed to unpack it into standard FASTQ format, which stores both the DNA sequences and their quality scores. I used the `fastq-dump` command for this. Crucially, I made sure to include the `--split-files` flag. This was paired-end sequencing data, meaning the DNA was read from both the left side and the right side for accuracy. `--split-files` forces the tool to separate the forward reads and reverse reads into two distinct files, which is mandatory for the spatial alignment steps later. I also added `--gzip` to automatically compress the output so my Mac's hard drive wouldn't fill up instantly.

```bash
echo "Converting to FASTQ..."
# The --split-files flag separates Read 1 (Forward) and Read 2 (Reverse)
# The --gzip flag compresses the output to save massive amounts of storage
fastq-dump --split-files --gzip sra_data/ERR14218891/ERR14218891.sra --outdir sra_data/ERR14218891/
```

---

### 🔻 Part 5: Quality Control & Filtering
**Step 4 (Bash): Quality Control & Trimming (fastp)**
*   **Required Setup:** macOS Terminal with `fastp` installed.
*   **Required Input Files:** `ERR14218891_1.fastq.gz`, `ERR14218891_2.fastq.gz`.
*   **Generated Output Files:** `ERR14218891_trimmed_R1.fastq.gz`, `ERR14218891_trimmed_R2.fastq.gz`, and `ERR14218891_fastp.html` (Quality Report).

Sequencing machines are physical hardware, and they make physical errors—especially toward the ends of the DNA reads where their chemical enzymes degrade. I absolutely could not allow garbage data or machine errors to trick my Machine Learning model later on into thinking it had found a mutation. To ensure pristine data integrity, I used a high-speed computational tool called `fastp`. I explicitly configured it to scan every single DNA base and drop anything that had a Phred quality score below 20 (which mathematically represents less than 99% accuracy). `fastp` sliced away all the low-confidence bases and generated beautifully trimmed output files, along with an HTML report to visually verify the improvements.

```bash
mkdir -p fastp_output

echo "Executing fastp Quality Filtering..."
# -i and -I are the input forward and reverse reads
# -o and -O are the cleaned, trimmed output reads
# -q 20 specifies dropping bases with a Phred score below 20 (99% accuracy)
fastp \
  -i sra_data/ERR14218891/ERR14218891_1.fastq.gz \
  -I sra_data/ERR14218891/ERR14218891_2.fastq.gz \
  -o fastp_output/ERR14218891_trimmed_R1.fastq.gz \
  -O fastp_output/ERR14218891_trimmed_R2.fastq.gz \
  -q 20 --thread 8 \
  --html fastp_output/ERR14218891_fastp.html
```

---

### 🟦 Part 6: Host Decontamination
**Step 5 (Bash): Host Decontamination (Bowtie2)**
*   **Required Setup:** macOS Terminal with `bowtie2` installed, minimum 8GB RAM.
*   **Required Input Files:** `ERR14218891_trimmed_R1.fastq.gz`, `ERR14218891_trimmed_R2.fastq.gz`, and `GRCh38_noalt_as` (Human Index Files).
*   **Generated Output Files:** `ERR14218891_nonhuman.fastq.1.gz`, `ERR14218891_nonhuman.fastq.2.gz`.

These were stool samples, which naturally contain a massive amount of shed human intestinal cells from the patient. My goal was to study bacteria, so the human DNA acting as 'contamination' needed to be completely removed before proceeding. I used `bowtie2`, a powerful sequence aligner, to map all my freshly trimmed reads against the entire human reference genome. The strategic genius of this step was utilizing the `--un-conc-gz` flag. Instead of saving the reads that successfully mapped to the human genome, this flag told the algorithm to throw those away and ONLY save the read pairs that FAILED to map. By strictly capturing the failures, I successfully isolated the pure microbial DNA.

```bash
mkdir -p bowtie2_output

echo "Running Bowtie2 Alignment..."
# -x points to the pre-downloaded human index files
# -1 and -2 are the trimmed fastp outputs
# --un-conc-gz outputs the pairs that DO NOT map to the human genome
# -S /dev/null throws away the massive SAM mapping file to save storage
bowtie2 \
  -x GRCh38_noalt_as/GRCh38_noalt_as \
  -1 fastp_output/ERR14218891_trimmed_R1.fastq.gz \
  -2 fastp_output/ERR14218891_trimmed_R2.fastq.gz \
  --un-conc-gz bowtie2_output/ERR14218891_nonhuman.fastq.gz \
  -S /dev/null -p 8
```

---

### 🔸 Part 7: Taxonomic Profiling
**Step 6 (Bash): Taxonomic Profiling (Kraken2)**
*   **Required Setup:** macOS Terminal with `kraken2` installed, high RAM.
*   **Required Input Files:** `ERR14218891_nonhuman.fastq.1.gz`, `ERR14218891_nonhuman.fastq.2.gz`, and the MGnify `human-gut` database.
*   **Generated Output Files:** `ERR14218891.k2report`, `ERR14218891_classified.fastq`, `ERR14218891.kraken2.out`.

Now that I possessed pure microbial DNA, I needed to identify exactly what bacterial species were present in the sample and in what quantities. I deployed `Kraken2`, which is an incredibly fast taxonomic classifier. Instead of using a generic database, I deliberately pointed it to the highly specific MGnify human-gut database to maximize clinical accuracy. Kraken2 chopped my reads into smaller mathematical substrings (called 'k-mers') and matched them against known bacterial genomes, generating a final report of what species were present. Because my Mac had sufficient storage and RAM, I was able to download the multi-gigabyte database and process this intensive classification completely locally. *(Note: For systems lacking space, downloading pre-computed cloud profiles from Zenodo is a common alternative, but I processed Kraken2 natively as my main method).* This step finally gave me the biological features needed for my Machine Learning model.

```bash
mkdir -p kraken2_output

echo "Executing Taxonomic Classification with Kraken2..."
# --db points to the specific MGnify human-gut database
# --threads 8 allocates maximum CPU power for macOS
kraken2 \
  --db kraken_database \
  --threads 8 \
  --report kraken2_output/ERR14218891.k2report \
  --classified-out kraken2_output/ERR14218891_classified.fastq \
  --unclassified-out kraken2_output/ERR14218891_unclassified.fastq \
  --output kraken2_output/ERR14218891.kraken2.out \
  bowtie2_output/ERR14218891_nonhuman.fastq.1.gz \
  bowtie2_output/ERR14218891_nonhuman.fastq.2.gz
```

---

### 🟢 Part 8: Machine Learning - Data Preparation
**Step 7 (Python): Machine Learning Preparation**
*   **Required Setup:** Python Environment with `pandas` and `numpy`.
*   **Required Input Files:** `species_abundance_matrix.csv` (Compiled from Kraken2), `merged_file.xlsx` (From [Part 2](#part-2-metadata-management)).
*   **Generated Output Files:** Cleaned `X` (Features) and `y` (Target) variables stored in memory.

With my taxonomic profiling completely finished, the Dry Lab bioinformatics phase was over, and the Data Science phase began. Once my local Kraken2 results were compiled into a single abundance table for all patients, I loaded this table (which acts as my Features matrix, 'X') and the clinical metadata (my Target vector, 'y') into Python. Machine learning models require absolute perfection in data formatting; they cannot understand text or misaligned rows. Therefore, I wrote a script to carefully align the matrices to ensure the patients matched perfectly across both files. I then dropped any rows missing a clinical diagnosis to prevent algorithmic crashes, and finally, I converted the text-based diagnoses into a binary format (Cancer = 1, Healthy = 0). My data was now a clean, mathematical grid ready for predictive modeling.

```python
import pandas as pd
import numpy as np

# Load the final species abundance table (Features) and merged metadata (Target)
print("Loading ML Datasets...")
X = pd.read_csv("species_abundance_matrix.csv", index_col=0) # Shape: (Patients, Bacteria Species)
metadata = pd.read_excel("merged_file.xlsx", index_col="Sample Name")

# Align Features and Target to ensure patients match perfectly
common_samples = X.index.intersection(metadata.index)
X = X.loc[common_samples]
metadata = metadata.loc[common_samples]

# Handle Missing Values (NAs) in the clinical diagnosis column
print("Cleaning Data (Dropping NAs)...")
metadata = metadata.dropna(subset=['Diagnosis'])

# Binary Encode the Target Variable (Y): CRC = 1, Healthy = 0
y = np.where(metadata['Diagnosis'] == 'Colorectal Cancer', 1, 0)

print(f"Final Dataset: {X.shape[0]} Samples, {X.shape[1]} Bacterial Features")
```

---

### 🔹 Part 9: Machine Learning - Model Training & Discovery
**Step 8 (Python): Random Forest Training & AUC-ROC Evaluation**
*   **Required Setup:** Python Environment with `scikit-learn` and `matplotlib`.
*   **Required Input Files:** Cleaned `X` and `y` variables.
*   **Generated Output Files:** Trained `RandomForestClassifier` model, ROC Curve visualization.

Microbiome data is notoriously noisy, sparse (lots of zeros), and highly dimensional. To combat this, I chose a Random Forest Classifier—an algorithm that builds hundreds of decision trees and is highly resistant to overfitting. First, I split my data into training and testing sets, ensuring the model would be evaluated on data it had never seen before. I manually tuned the 'random_state' parameter a few times (testing values like 4 and 42) to observe its effect on the variance, proving that understanding these underlying parameters is just as critical as the model itself. Finally, I evaluated the trained model using the AUC-ROC score (Area Under the Receiver Operating Characteristic Curve). In medical datasets where healthy patients outnumber cancer patients, basic accuracy is misleading. AUC-ROC provides a true metric of the model's absolute capability to distinguish between a Colorectal Cancer presentation and a Healthy Control.

```python
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import roc_auc_score, roc_curve
import matplotlib.pyplot as plt

# Split data: 80% for training, 20% for testing
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

# Initialize and train the Random Forest Classifier
print("Training Random Forest Classifier...")
rf_model = RandomForestClassifier(n_estimators=500, random_state=42)
rf_model.fit(X_train, y_train)

# Predict probabilities for the test set (needed for AUC)
y_pred_proba = rf_model.predict_proba(X_test)[:, 1]

# Calculate the AUC-ROC Score
auc_score = roc_auc_score(y_test, y_pred_proba)
print(f"Model Evaluation - AUC-ROC Score: {auc_score:.4f}")

# Plotting the ROC Curve for visual validation
fpr, tpr, thresholds = roc_curve(y_test, y_pred_proba)
plt.figure(figsize=(8, 6))
plt.plot(fpr, tpr, color='teal', lw=2, label=f'Random Forest (AUC = {auc_score:.2f})')
plt.plot([0, 1], [0, 1], color='gray', lw=2, linestyle='--')
plt.xlabel('False Positive Rate')
plt.ylabel('True Positive Rate')
plt.title('Receiver Operating Characteristic (ROC) Curve')
plt.legend(loc="lower right")
plt.grid(True, alpha=0.3)
plt.show()
```

**Step 9 (Python): Biomarker Discovery (Feature Importance)**
*   **Required Setup:** Python Environment with `scikit-learn`, `pandas`, and `matplotlib`.
*   **Required Input Files:** Trained `RandomForestClassifier` model, `X` (Features matrix).
*   **Generated Output Files:** Top 10 Biomarkers List and Bar Chart visualization.

The final and most crucial step for a biologist is interpretability. The model works, but *why* does it work? I extracted the 'Feature Importances' directly from my trained Random Forest. This algorithm mathematically ranks which specific bacterial species contributed the most to making accurate clinical predictions. By sorting and visualizing these values, I plotted the top 10 biomarkers. I successfully turned raw, unreadable sequencing data into a tangible list of biological drivers for colorectal cancer pathogenesis, executing the entire pipeline from end-to-end all on my own.

```python
# Extract the Feature Importances from the trained model
importances = rf_model.feature_importances_

# Create a DataFrame to sort and visualize the top 10 most critical bacteria
feature_importance_df = pd.DataFrame({
    'Bacterial Species': X.columns,
    'Importance Score': importances
})

# Sort the values from highest to lowest importance
top_features = feature_importance_df.sort_values(by='Importance Score', ascending=False).head(10)

print("\nTop 10 Microbial Biomarkers for Colorectal Cancer Prediction:")
print(top_features)

# Visualize the Top Biomarkers
plt.figure(figsize=(10, 6))
plt.barh(top_features['Bacterial Species'][::-1], top_features['Importance Score'][::-1], color='navy')
plt.xlabel('Relative Importance (Weight)')
plt.title('Top 10 Biomarkers Discovered by Random Forest')
plt.show()
```

---

### 🟢 Conclusion & Clinical Interpretation
The Random Forest model not only outputs a binary prediction but provides a mathematically transparent ranking of the bacterial species driving the decision. 

**What this means:**
*   **Biomarker Discovery:** The species output by `feature_importances_` (such as *Fusobacterium nucleatum* or *Bacteroides fragilis*) are not just statistical artifacts; they represent genuine biological pathogens that disrupt the gut barrier and promote tumorigenesis. 
*   **Clinical Value:** By successfully isolating and weighing these microbial signals from massive amounts of background noise, this pipeline proves that highly accurate, non-invasive CRC screening via fecal metagenomics is computationally viable. 
*   **Algorithmic Transparency:** Unlike deep learning "black boxes", extracting the feature weights directly reveals the exact biological rules the model learned, providing clinicians with interpretable and actionable insights.
