# Amazon Electronics Review Analysis

**COMP30770 – Programming for Big Data**

---

## 📌 Project Overview

This project analyses the **Amazon Electronics Reviews 2023** dataset to identify:

* ⭐ Top-rated products in the Electronics category
* 💬 Most frequent sentiment keywords in customer reviews

We first implemented a **single-threaded baseline pipeline** in Python to establish performance metrics, and then optimised the bottleneck stages using **Spark Core (RDD-based) MapReduce**.

---

## 🎯 Project Objectives

The primary goal of this project is to demonstrate how big data techniques can improve the scalability of review analytics pipelines.

Specifically, we aim to:

* Build a reproducible single-machine baseline
* Identify computational bottlenecks
* Apply MapReduce optimisation using Spark RDD
* Compare performance before and after optimisation

---

## 🗂 Dataset

We use the **Amazon Electronics Reviews 2023** dataset:

| File                     | Size  | Description                      |
| ------------------------ | ----- | -------------------------------- |
| `Electronics.jsonl`      | ~2 GB | Structured customer review data  |
| `meta_Electronics.jsonl` | ~1 GB | Semi-structured product metadata |

**Big Data Justification**

* Volume: Processing 500,000 reviews already incurs significant runtime on our hardware
* Variety: The two datasets have different structures requiring different parsing strategies

---

## 🖥 Environment

Experiments were conducted on:

* CPU: Intel i5-12600KF
* RAM: 32 GB DDR5
* OS: Windows 11
* Python: 3.x
* PySpark: 3.5.1

---

## ⚙️ Project Pipeline

### Section 2 — Traditional (Single-Threaded)

The baseline pipeline consists of five steps:

1. **Data Loading & Cleaning**
2. **Aggregate Ratings per Product**
3. **Word Frequency Analysis**
4. **Join Reviews with Metadata**
5. **Output Top-20 Product Ranking**

**Identified Bottlenecks**

* Step 3: Word frequency (CPU-bound)
* Step 4: Metadata join (I/O-bound)

---

### Section 3 — MapReduce Optimisation

We reimplemented the bottleneck stages using **Spark Core (RDD API)**.

**Optimisations**

* 🔹 Word frequency via distributed token counting
* 🔹 Metadata join via distributed hash join

---

## 📊 Performance Summary

| Task           | Traditional | Spark RDD | Speedup |
| -------------- | ----------- | --------- | ------- |
| Word Frequency | 35.41s      | 36.32s    | 0.97×   |
| Metadata Join  | 104.62s     | 58.83s    | 1.78×   |

**Key Insight**

The metadata join shows meaningful improvement, while the word frequency task is affected by **Python–JVM serialisation overhead** in local mode.

---

## 🚀 How to Run

### 1️⃣ Install dependencies

```bash
pip install pyspark pandas pyarrow
```

### 2️⃣ Run the notebook

Open:

```
comp30770-project.ipynb
```

and execute all cells in order.

---

## 📁 Repository Structure

```
.
├── comp30770-project.ipynb   # Main analysis notebook
├── data/                     # (not included in repo)
├── report.pdf                # Final project report
└── README.md
```

---

## 👥 Team Members

* Yiming Chen
* Xiaoyan Li
* **Enzhe Shen**
* Peitao Lin

---

## 📜 Notes

* All experiments were conducted in local mode (`local[*]`).
* In a distributed HDFS environment, further speedup is expected.
* The repository is intended for academic use only.

---

## ✅ Reproducibility

To reproduce the results:

1. Download the Amazon Electronics dataset
2. Place JSONL files in the working directory
3. Run the notebook sequentially

---

⭐ *This project was developed as part of UCD COMP30770 Programming for Big Data.*
