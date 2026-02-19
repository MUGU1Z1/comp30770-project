# COMP30770 Group Project — Amazon Electronics Review Analysis

> Identifying top-rated products and sentiment keywords from Amazon Electronics reviews using Big Data pipelines.

## Group Members

| Name | UCD Student ID |
|------|---------------|
| Yiming Chen | 22209465 |
| Xiaoyan Li | 22207129 |
| Enzhe Shen | 22212419 |
| Peitao Lin | 22206283 |

---

## Project Overview

This project analyses the [Amazon Reviews 2023](https://huggingface.co/datasets/McAuley-Lab/Amazon-Reviews-2023) dataset (Electronics category) to:

- Identify the **top 20 highest-rated products** based on verified customer reviews
- Extract the **most frequent sentiment keywords** from review texts
- Demonstrate the performance improvement of **Spark RDD (MapReduce)** over a traditional single-threaded Python solution

---

## Datasets

| File | Size | Description |
|------|------|-------------|
| `Electronics.jsonl` | ~2 GB | Structured review data (rating, text, timestamp, verified_purchase) |
| `meta_Electronics.jsonl` | ~1 GB | Semi-structured product metadata (title, category, price, images) |

Both files are downloaded automatically by the notebook. Not included in this repository due to size.

---

## Repository Structure

```
comp30770-project/
├── COMP30770_Project.ipynb     # Main notebook (all code)
├── COMP30770_Group_Report.md   # Project report
└── README.md                   # This file
```

---

## Requirements

- Python 3.11
- PySpark 3.5.1
- Java 17 (required by Spark)
- pandas, pyarrow, jupyter

Install dependencies:

```bash
pip install pyspark==3.5.1 pandas pyarrow jupyter
```

---

## How to Run

1. Clone this repository:
```bash
git clone https://github.com/your-username/comp30770-project.git
cd comp30770-project
```

2. Start Jupyter Notebook:
```bash
jupyter notebook
```

3. Open `COMP30770_Project.ipynb` and run all cells in order.
   - The notebook will automatically download the datasets on first run (~3 GB total)
   - Tested on Windows 11 with Intel i5-12600KF and 32 GB RAM

---

## Results Summary

### Top 3 Highest-Rated Products (≥100 reviews)

| Rank | Avg Rating | Product |
|------|-----------|---------|
| 1 | 4.89 | Amazon Basics High-Speed HDMI Cable 2-Pack |
| 2 | 4.85 | Crucial RAM 16GB Kit DDR3 1600 MHz CL11 |
| 3 | 4.85 | Lifetime 28240 Adjustable Folding Laptop Table |

### Performance Comparison

| Task | Traditional Python | Spark RDD | Speedup |
|------|--------------------|-----------|---------|
| Word Frequency | 35.41s | 36.32s | 0.97x |
| Metadata Join | 104.62s | 58.83s | 1.78x |

---

## Notes

- The Spark optimisation was run in `local[*]` mode on a single machine (16 logical cores).
- Below-expectation speedup for word frequency is due to `sc.parallelize()` serialisation overhead in local mode. In a distributed HDFS environment, 3–5x speedup would be expected.
- This repository will remain public until at least May 2026 as required by the project submission guidelines.
