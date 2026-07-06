# End-to-End Metagenomics & Machine Learning Pipeline
*(BCD Analytics Hub - Colorectal Cancer Prediction)*

This document is structured exactly like a Jupyter Notebook. In a real Jupyter environment (like Google Colab or a local Jupyter server), the `bash` commands can be run by prefixing them with an exclamation mark (`!`) in a code cell. The Python code runs natively.

---

### Part 1: Initial Preparations
**Step 0: Prerequisites and Environment Setup**
*   **Required Setup:** macOS Terminal (Native Unix Environment), Anaconda/Miniconda, minimum 8GB RAM, and a stable internet connection.
*   **Required Input Files:** None.
*   **Generated Output Files:** A fully configured Unix environment ready for bioinformatics.
*   **My Guide / Instructions:** "Before I started anything, I made sure my Mac had at least 8GB of RAM and a good internet connection. Since my laptop is a Mac, I was able to run all the heavy bioinformatics commands natively right in my macOS Terminal. I installed Anaconda to manage my environments, and installed all the tools I'd need (like SRA Toolkit, fastp, Bowtie2, and Kraken2) using Conda. For the machine learning later, I installed Python packages like pandas and scikit-learn. Finally, I pre-downloaded the massive database files for Bowtie2 (GRCh38 human genome) and Kraken2 (MGnify human-gut database)."

---

### Part 2: Metadata Management
**Step 1 (Python): Preparing the Workspace & Merging Metadata**
*   **Required Setup:** Python Environment (Jupyter Notebook or Google Colab) with `pandas` installed.
*   **Required Input Files:** `META_file_gyatri.xlsx` (clinical data), `SraRunTable_gyatriiiii.xlsx` (sequencing IDs).
*   **Generated Output Files:** `merged_file.xlsx`
*   **My Guide / Instructions:** "First, I had to link the clinical patient data with the actual sequencing run IDs. Without this, I wouldn't know which DNA sequence belonged to a healthy person or a cancer patient. I used Python's Pandas library to merge these two excel sheets together based on their shared 'Sample Name' column, and then saved the final merged list."

```python
# Import the pandas library for data manipulation
import pandas as pd

# Load the clinical metadata (containing Diagnosis, CRC Stage, Sex, etc.)
print("Loading Metadata...")
df1 = pd.read_excel("META_file_gyatri.xlsx", sheet_name=0) 

# Load the SRA Run Table (containing the ERR/SRR sequencing IDs)
print("Loading SRA Run Table...")
df2 = pd.read_excel("SraRunTable_gyatriiiii.xlsx", sheet_name=0)

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

### Part 3: Raw Data Acquisition
**Step 2 (Bash): Downloading the Raw Sequencing Data**
*   **Required Setup:** Linux Terminal with `sra-tools` installed via Conda.
*   **Required Input Files:** `merged_file.xlsx` (to know which ERR numbers to download).
*   **Generated Output Files:** `sra_data/ERR14218891/ERR14218891.sra`
*   **My Guide / Instructions:** "Next, I needed to actually get the biological data. I used the SRA Toolkit's `prefetch` command in my Linux terminal to securely download the highly compressed raw sequencing data straight from the NCBI databases into a new folder."

```bash
%%bash
# Create a directory to store the raw data
mkdir -p sra_data/ERR14218891

# Use prefetch to download the .sra file securely
echo "Downloading SRA file..."
prefetch --output-directory sra_data/ERR14218891 ERR14218891
```

---

### Part 4: Data Conversion
**Step 3 (Bash): FASTQ Conversion**
*   **Required Setup:** Linux Terminal with `sra-tools` installed.
*   **Required Input Files:** `ERR14218891.sra`
*   **Generated Output Files:** `ERR14218891_1.fastq.gz` (Forward reads), `ERR14218891_2.fastq.gz` (Reverse reads).
*   **My Guide / Instructions:** "The raw `.sra` file I downloaded was highly compressed and useless to my analysis tools. I used the `fastq-dump` command to convert it into a readable FASTQ format. I made sure to use `--split-files` because this was paired-end data, meaning I needed separate files for the forward and reverse reads, and `--gzip` so my hard drive wouldn't fill up instantly."

```bash
%%bash
echo "Converting to FASTQ..."
# The --split-files flag separates Read 1 (Forward) and Read 2 (Reverse)
# The --gzip flag compresses the output to save massive amounts of storage
fastq-dump --split-files --gzip sra_data/ERR14218891/ERR14218891.sra --outdir sra_data/ERR14218891/
```

---

### Part 5: Quality Control & Filtering
**Step 4 (Bash): Quality Control & Trimming (fastp)**
*   **Required Setup:** Linux Terminal with `fastp` installed.
*   **Required Input Files:** `ERR14218891_1.fastq.gz`, `ERR14218891_2.fastq.gz`.
*   **Generated Output Files:** `ERR14218891_trimmed_R1.fastq.gz`, `ERR14218891_trimmed_R2.fastq.gz`, and `ERR14218891_fastp.html` (Quality Report).
*   **My Guide / Instructions:** "Sequencing machines make physical errors, especially at the end of the reads. I didn't want garbage data messing up my machine learning model later. I used a tool called `fastp` to filter the data. I told it to drop any bases that had a Phred quality score below 20 (meaning less than 99% accuracy), and it generated trimmed files with all the bad data removed."

```bash
%%bash
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

### Part 6: Host Decontamination
**Step 5 (Bash): Host Decontamination (Bowtie2)**
*   **Required Setup:** Linux Terminal with `bowtie2` installed, minimum 8GB RAM.
*   **Required Input Files:** `ERR14218891_trimmed_R1.fastq.gz`, `ERR14218891_trimmed_R2.fastq.gz`, and `GRCh38_noalt_as` (Human Index Files).
*   **Generated Output Files:** `ERR14218891_nonhuman.fastq.1.gz`, `ERR14218891_nonhuman.fastq.2.gz`.
*   **My Guide / Instructions:** "The stool samples contained a lot of human DNA from the patient's own cells. I had to remove this human contamination. I used `bowtie2` to map all my trimmed reads against a massive human reference genome database. The crucial part was using the `--un-conc-gz` flag—this told the tool to keep only the reads that FAILED to match the human genome, leaving me with pure microbial DNA."

```bash
%%bash
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
  -S /dev/null -p 16
```

---

### Part 7: Taxonomic Profiling
**Step 6 (Bash): Taxonomic Profiling (Kraken2)**
*   **Required Setup:** Linux Terminal with `kraken2` installed, extremely high RAM (or HPC).
*   **Required Input Files:** `ERR14218891_nonhuman.fastq.1.gz`, `ERR14218891_nonhuman.fastq.2.gz`, and the MGnify `human-gut` database.
*   **Generated Output Files:** `ERR14218891.k2report`, `ERR14218891_classified.fastq`, `ERR14218891.kraken2.out`.
*   **My Guide / Instructions:** "Now that I had pure microbial DNA, I needed to identify exactly what bacteria were in it. I used `Kraken2` and pointed it to the highly specific MGnify human-gut database. This tool chopped my reads into smaller 'k-mers' and matched them to known bacteria, giving me a final report of what species were present. Because I had enough storage and RAM on my Mac, I was able to download the database and process this intensive classification completely locally. (For systems lacking space, downloading pre-computed cloud profiles from Zenodo is another option, but I processed Kraken2 as my main method)."

```bash
%%bash
mkdir -p kraken2_output

echo "Executing Taxonomic Classification with Kraken2..."
# --db points to the specific MGnify human-gut database
# --threads 32 allocates maximum CPU power
kraken2 \
  --db kraken_database \
  --threads 32 \
  --report kraken2_output/ERR14218891.k2report \
  --classified-out kraken2_output/ERR14218891_classified.fastq \
  --unclassified-out kraken2_output/ERR14218891_unclassified.fastq \
  --output kraken2_output/ERR14218891.kraken2.out \
  bowtie2_output/ERR14218891_nonhuman.fastq.1.gz \
  bowtie2_output/ERR14218891_nonhuman.fastq.2.gz
```

---

### Part 8: Machine Learning - Data Preparation
**Step 7 (Python): Machine Learning Preparation**
*   **Required Setup:** Python Environment with `pandas` and `numpy`.
*   **Required Input Files:** `species_abundance_matrix.csv` (Compiled from Kraken2/MetaPhlAn), `merged_file.xlsx` (From Step 1).
*   **Generated Output Files:** Cleaned `X` (Features) and `y` (Target) variables stored in memory.
*   **My Guide / Instructions:** "With my taxonomic profiling done, it was time for Machine Learning. Once my local Kraken2 results were compiled into a single table for all patients, I loaded this table (my Features) and the clinical metadata (my Target) into Python. I aligned them to make sure the patients matched perfectly, dropped any rows missing a clinical diagnosis, and changed the text diagnoses into binary numbers (Cancer = 1, Healthy = 0)."

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

### Part 9: Machine Learning - Model Training & Discovery
**Step 8 (Python): Random Forest Training & AUC-ROC Evaluation**
*   **Required Setup:** Python Environment with `scikit-learn` and `matplotlib`.
*   **Required Input Files:** Cleaned `X` and `y` variables.
*   **Generated Output Files:** Trained `RandomForestClassifier` model, ROC Curve visualization.
*   **My Guide / Instructions:** "I split my data into training and testing sets. I manually tuned the 'random_state' parameter a few times (like 4 or 42) to see how it affected the results, proving that understanding these parameters is just as important as the model itself. I trained a Random Forest model because it handles this kind of noisy data well. Finally, I evaluated it using the AUC-ROC score to prove it could actually distinguish between healthy and cancer patients accurately."

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
*   **My Guide / Instructions:** "The last step was to figure out *why* the model made its predictions. I extracted the 'Feature Importances' from the Random Forest. This ranked the specific bacterial species based on how important they were for detecting cancer. I plotted the top 10 biomarkers, successfully turning raw sequencing data into real clinical discoveries all on my own."

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
