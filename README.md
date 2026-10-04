# Amazon ML Challenge 2026 — Business Entity Resolution

**Team Name:** `Train.Test.Split`

## Team Members

* **Divya Nehete**
* **Vanshika Agrawal**
* **Sanjana Bhat**
* **Khushee Dengale**

---

An end-to-end **Business Entity Resolution** system developed for the **Amazon ML Challenge 2026**.

The project identifies whether business entities across multiple data sources refer to the same real-world business. It combines **text normalization, blocking, fuzzy matching, deterministic rules, hard-negative mining, and machine learning** to generate high-quality entity matches while keeping memory usage manageable on large datasets.

---


## Project Overview

Business Entity Resolution is the task of identifying records from different data sources that represent the same underlying entity.

For this challenge, the system matches entities from **Source 1** against entities from **Source 2 and Source 3** using information such as:

* Business name
* Business address
* Country
* Entity IDs

Instead of comparing every Source-1 entity against the entire target universe, the pipeline first generates a smaller set of likely candidates and then applies similarity-based matching.

### Key objectives

* Improve matching precision
* Reduce unnecessary pairwise comparisons
* Handle variations in business names and addresses
* Work efficiently with millions of records
* Generate competition-ready submission files
* Make the pipeline restart-safe through checkpointing

---

## Approach

The pipeline follows a multi-stage entity-resolution architecture:

```text
Training / Test Data
        │
        ▼
Data Loading
        │
        ▼
Text Normalization
        │
        ├── Business Name
        ├── Business Address
        └── Country
        │
        ▼
Blocking / Candidate Generation
        │
        ▼
Fuzzy Similarity Features
        │
        ├── Name Similarity
        ├── Address Similarity
        ├── Token Similarity
        ├── Jaccard Similarity
        └── Exact Match Signals
        │
        ▼
Hard-Negative Mining
        │
        ▼
Machine Learning Model
        │
        ▼
Probability / Score
        │
        ▼
Threshold Optimization
        │
        ▼
Deterministic Strong-Match Rules
        │
        ▼
Candidate & Match Generation
        │
        ▼
Submission Validation
        │
        ▼
matching_results.tsv
candidate_pairs.tsv
```

---

## Dataset

The notebook expects the Amazon ML Challenge dataset containing:

### Training files

```text
train_source1.tsv
train_source2.tsv
train_source3.tsv
train_ground_truth.tsv
```

### Test files

```text
test_source1.tsv
test_source2.tsv
test_source3.tsv
```

The ground-truth file is used to construct positive and negative training examples and evaluate the matching pipeline.

---

## Data Preprocessing

The project performs extensive normalization before generating matches.

### Business name normalization

The pipeline:

* Converts text to lowercase
* Applies Unicode normalization
* Removes punctuation
* Normalizes whitespace
* Standardizes common corporate terms

Examples of standardized terms include:

```text
private limited → pvt ltd
private → pvt
limited → ltd
corporation → corp
company → co
incorporated → inc
```

### Address normalization

Common address terms are standardized:

```text
road → rd
street → st
avenue → ave
boulevard → blvd
lane → ln
building → bldg
apartment → apt
floor → flr
number → no
```

This helps the system recognize records that differ only because of formatting or common abbreviations.

---

## Candidate Generation & Blocking

Comparing every Source-1 entity with every Source-2/Source-3 entity would be computationally expensive.

The project therefore uses **blocking** to restrict comparisons to potentially relevant records.

Blocking keys are generated using combinations of:

* Country
* Business-name tokens
* First and last meaningful name words
* Address tokens
* Exact normalized names
* Name-address combinations

This significantly reduces the number of unnecessary comparisons while retaining likely matches.

---

## Similarity Features

The matching model uses **13 features**.

### Business name features

* `name_ratio`
* `name_token_ratio`
* `name_wratio`
* `name_jaccard`

### Address features

* `address_ratio`
* `address_token_ratio`
* `address_wratio`
* `address_jaccard`

### Additional matching signals

* `country_same`
* `name_exact`
* `address_exact`
* `name_addr_avg`
* `strong_exact`

The fuzzy matching features are generated using **RapidFuzz**.

---

## Fuzzy Matching

The project uses several complementary similarity measures:

### Ratio

Measures character-level similarity between two strings.

### Token Set Ratio

Handles differences in word ordering and extra words.

For example:

```text
ABC Technologies Pvt Ltd
ABC Technologies Limited
```

can still receive a high similarity score despite textual differences.

### WRatio

Provides a flexible fuzzy similarity score that works well across different string structures.

### Jaccard Similarity

Compares the overlap between the token sets of two strings.

Using multiple similarity measures provides a richer representation of entity similarity than relying on a single metric.

---

## Machine Learning

The project uses tree-based machine learning models for classification.

The pipeline imports and utilizes:

* `ExtraTreesClassifier`
* `RandomForestClassifier`

The classifier learns from the similarity features to distinguish:

```text
Match
vs.
Non-match
```

The model produces a probability/score that is subsequently used for match decisioning.

---

## Hard-Negative Mining

Random negative examples are often too easy for an entity-resolution model.

For example, two businesses from completely different countries or with completely unrelated names are obvious non-matches.

Instead, this project mines **hard negatives** from the same blocking neighborhoods.

These are records that look similar but represent different entities.

This helps the model learn difficult distinctions such as:

```text
Similar business names
+
Similar addresses
+
Same country
≠
necessarily the same entity
```

---

## Precision-First Decisioning

The project is designed with a **precision-first** objective.

Instead of blindly accepting every high-scoring fuzzy match, the pipeline combines:

1. Machine-learning scores
2. Exact matching rules
3. Strong deterministic signals
4. Threshold optimization

The threshold is tuned using **F0.5**, with attention to:

* Positive-class F0.5
* Macro F0.5

F0.5 gives more importance to precision than recall, which is useful when incorrect entity matches are particularly costly.

---

## Training Strategy

The training pipeline samples Source-1 entities while maintaining matched and unmatched examples.

The notebook uses:

```text
60,000 sampled Source-1 entities
```

with matched and unmatched examples used for model development.

A grouped train/validation strategy is used to reduce leakage between related entities.

---

## Restart-Safe Pipeline

A major design feature of the project is its **checkpoint-based architecture**.

Expensive stages save intermediate results under:

```text
/kaggle/working/ber_checkpoints
```

This allows the notebook to resume from previously completed stages rather than recomputing the entire pipeline after a restart.

Checkpointing is especially useful because entity resolution over millions of records can be computationally expensive.

---

## Memory Optimization

The target Source-2 and Source-3 data is processed in **chunks** rather than loading the entire target universe into memory.

The pipeline also uses:

* Chunked processing
* Compact block indexes
* Pickle checkpoints
* Garbage collection
* Streaming candidate processing

This makes the solution more practical for Kaggle environments with limited RAM.

---

## Submission Generation

The notebook generates two main TSV files:

```text
candidate_pairs.tsv
matching_results.tsv
```

### candidate_pairs.tsv

Contains:

```text
source1_entity_id
candidate_entity_ids
```

It stores the candidate target entities considered for each Source-1 entity.

### matching_results.tsv

Contains:

```text
source1_entity_id
matched_entity_ids
```

It stores the final predicted entity matches.

---

## Submission Validation

Before finalizing the output, the notebook checks:

* Required headers
* Duplicate Source-1 IDs
* Missing Source-1 IDs
* Unknown Source-1 IDs
* Valid target entity prefixes
* Matches appearing inside the candidate set

This helps catch formatting and consistency errors before submission.

---

## Final Output

The notebook creates:

```text
candidate_pairs.tsv
matching_results.tsv
model.joblib
amazon_entity_resolution_submission.zip
```

The final ZIP contains:

```text
matching_results.tsv
candidate_pairs.tsv
model.joblib
```

---

## Results

The executed pipeline produced:

| Metric                    |     Result |
| ------------------------- | ---------: |
| Source-1 entities         |  1,732,544 |
| S1 entities with ≥1 match |  1,500,159 |
| Total candidates          | 36,123,427 |
| Total predicted matches   |  5,481,994 |
| Match rate                |     86.59% |

The generated submission artifacts were successfully created by the notebook.

> Note: The notebook itself explicitly states that a hidden-test leaderboard score cannot be guaranteed without access to the hidden labels. The pipeline is optimized toward high precision and the target F0.5 objective.

---

## Tech Stack

### Programming Language

* Python

### Libraries

* Pandas
* NumPy
* Scikit-learn
* RapidFuzz
* Joblib

### Machine Learning

* Extra Trees
* Random Forest
* Group-based train/validation splitting

### Environment

* Kaggle Notebook
* Python 3
* CPU-based execution

---

## Project Structure

A recommended GitHub repository structure is:

```text
amazon-entity-resolution/
│
├── train-test-slay.ipynb
├── README.md
│
├── outputs/
│   ├── matching_results.tsv
│   ├── candidate_pairs.tsv
│   └── model.joblib
│
└── .gitignore
```

Large competition datasets and generated TSV files should generally **not** be committed directly to GitHub.

---

## How to Run

### 1. Open the notebook

Open:

```text
train-test-slay.ipynb
```

in Kaggle.

### 2. Attach the competition dataset

Make sure the following files are available:

```text
train_source1.tsv
train_source2.tsv
train_source3.tsv
train_ground_truth.tsv

test_source1.tsv
test_source2.tsv
test_source3.tsv
```

The notebook automatically searches the Kaggle input and working directories for these files.

### 3. Install dependencies

The notebook installs RapidFuzz and imports the required Python libraries.

```bash
pip install rapidfuzz
```

### 4. Run the notebook

Execute the cells sequentially.

Checkpoint files will be created automatically in:

```text
/kaggle/working/ber_checkpoints
```

### 5. Retrieve the outputs

Final files will be available in:

```text
/kaggle/working/output
```

---

## Key Highlights

### Hybrid Entity Resolution

Combines deterministic rules with machine learning rather than relying exclusively on fuzzy matching.

### Precision-Oriented Matching

Optimizes the decision threshold around F0.5 to emphasize precision.

### Hard-Negative Mining

Uses difficult, highly similar non-matches to improve classifier discrimination.

### Efficient Blocking

Reduces the number of entity pairs requiring expensive similarity calculations.

### Large-Scale Processing

Designed to operate on millions of business records using chunked processing.

### Restart-Safe Architecture

Checkpointing prevents expensive stages from needing to be recomputed after interruptions.

### Submission-Safe Outputs

Automatically generates and validates the required TSV files.

---

## Limitations

* Final leaderboard performance depends on hidden test labels.
* Blocking can potentially exclude a true match if the correct entity does not fall into any generated block.
* Fuzzy similarity alone cannot completely resolve ambiguous businesses.
* Large output files can require substantial disk space.
* The final submission should always be validated against the competition's latest submission requirements.

---

## Future Improvements

Potential improvements include:

* More sophisticated multilingual name normalization
* Character n-gram similarity
* TF-IDF based similarity
* Learned embeddings for business names and addresses
* ANN/vector-based candidate retrieval
* More advanced ensemble models
* Better calibration of match probabilities
* Adaptive blocking strategies
* Additional domain-specific address features
* Improved handling of transliteration and multilingual entities

---

## Conclusion

This project demonstrates a scalable approach to **large-scale business entity resolution** by combining traditional record-linkage techniques with machine learning.

The core pipeline:

```text
Normalize
    ↓
Block
    ↓
Generate Candidates
    ↓
Calculate Similarity Features
    ↓
Mine Hard Negatives
    ↓
Train ML Model
    ↓
Optimize Threshold
    ↓
Apply Strong Matching Rules
    ↓
Generate Matches
    ↓
Validate Submission
```

The result is a **memory-conscious, restart-safe and precision-focused entity-resolution pipeline** capable of processing millions of records and producing competition-ready outputs.

---


Built as part of the **Amazon ML Challenge 2026**.

