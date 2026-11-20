# DVC Workflow Documentation: Dataset Tracking and Versioning

This document outlines the Data Version Control (DVC) workflow and architecture implemented for the `mlops-iris-classifier` repository as part of Experiment 03.

---

## 1. Remote Configuration

A local storage remote was configured to simulate remote object storage (such as AWS S3, Google Cloud Storage, or an Azure Blob container) without incurring cloud infrastructure dependencies:

- **Remote Name**: `myremote`
- **Remote Storage Path**: `~/dvc-remote-storage` (configured as default remote via `dvc remote add -d myremote ~/dvc-remote-storage`)
- **Remote Configuration File**: `.dvc/config`

```ini
[core]
    remote = myremote
['remote "myremote"']
    url = ~/dvc-remote-storage
```

---

## 2. Git + DVC Synchronization Cycle

DVC decouples metadata tracking from large file storage:
- **Git** tracks lightweight `.dvc` metafile pointers (containing content hash, byte size, and relative file path) alongside the source code.
- **DVC** computes content MD5 hashes, stores data objects in the local `.dvc/cache`, and synchronizes objects with the remote storage (`dvc push` / `dvc pull`).

### The Standard Iteration Cycle:
For every dataset creation or modification:
1. **Track with DVC**:
   ```bash
   dvc add data/raw/iris_v1.csv
   ```
   *Computes MD5 hash and updates `data/raw/iris_v1.csv.dvc`.*
2. **Stage Metafiles in Git**:
   ```bash
   git add data/raw/iris_v1.csv.dvc data/raw/.gitignore
   ```
   *Only commits the small YAML pointer, keeping Git repository size minimal.*
3. **Commit Metadata in Git**:
   ```bash
   git commit -m "<descriptive commit message>"
   ```
4. **Push Dataset to Remote Storage**:
   ```bash
   dvc push
   ```
   *Uploads the hashed data object from `.dvc/cache` to the remote backend.*

---

## 3. Dataset Versions and Hashes

| Version | Git Commit | Row Count (incl. header) | File Size | Content MD5 Hash | Description |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Version 1** | `81aace2` | 151 lines (150 data rows) | 2,928 bytes | `21d441a28bce4417276097df955afc50` | Raw Iris dataset generated from `sklearn.datasets.load_iris` |
| **Version 2** | `606a95d` | 171 lines (170 data rows) | 4,462 bytes | `674c8c36bb7c4ba8d851dee9e6ee67af` | Augmented dataset with 20 Gaussian-perturbed synthetic rows |

---

## 4. Comparing Dataset Versions

To compare dataset versions across Git commits:

1. **Commit History of Data Pointer**:
   ```bash
   git log --oneline -- data/raw/iris_v1.csv.dvc
   ```
2. **Inspect DVC Differences**:
   ```bash
   dvc diff <v1-commit-hash>
   ```
   *Output:*
   ```text
   Modified:
       data/raw/iris_v1.csv

   files summary: 1 modified
   ```
3. **Side-by-Side Metafile Comparison**:
   ```bash
   git show <v1-commit-hash>:data/raw/iris_v1.csv.dvc
   git show HEAD:data/raw/iris_v1.csv.dvc
   ```

---

## 5. Reproducing Historical Experiments

To restore any previous historical version of the dataset without manual file copies:

### Restore Version 1:
```bash
# 1. Checkout the metafile pointer from the historical Git commit
git checkout 81aace2 -- data/raw/iris_v1.csv.dvc

# 2. Instruct DVC to sync the working copy with the checked-out hash
dvc checkout data/raw/iris_v1.csv.dvc

# 3. Verify row count
wc -l data/raw/iris_v1.csv  # 151 lines (150 rows + header)
```

### Restore Version 2 (Latest HEAD):
```bash
# 1. Return the metafile pointer to HEAD
git checkout HEAD -- data/raw/iris_v1.csv.dvc

# 2. Sync dataset to latest hash
dvc checkout data/raw/iris_v1.csv.dvc

# 3. Verify row count
wc -l data/raw/iris_v1.csv  # 171 lines (170 rows + header)
```

---

## 6. Verification Summary

- **DVC Version**: 3.67.1
- **DVC Cache & Remote Storage Verified**: Hashed objects stored under `md5/21/d441...` and `md5/67/4c8c...`.
- **Git Log Verified**: Exactly two distinct dataset version commits tracked.
- **Reproducibility**: Switching between versions verified with exact row counts (150 and 170).
