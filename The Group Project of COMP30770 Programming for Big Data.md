# The Group Project of COMP30770 Programming for Big Data

**Project Title:** Amazon Electronics Review Analysis: Identifying Top-Rated Products Using Big Data Pipelines

| Name        | UCD Student ID |
| ----------- | -------------- |
| Yiming Chen | 22209465       |
| Xiaoyan Li  | 22207129       |
| Enzhe Shen  | 22212419       |
| Peitao Lin  | 22206283       |

**Code Link:** https://github.com/MUGU1Z1/comp30770-project

------

## Section 1. Introduction

This project uses the Amazon Electronics Reviews 2023 dataset, comprising approximately 2 GB of structured user review data (`Electronics.jsonl`) alongside 1 GB of semi-structured product metadata (`meta_Electronics.jsonl`), totalling over 500,000 review records in our prototype. On our development hardware (Intel i5-12600KF, 32 GB DDR5 RAM, Windows 11), loading and processing the review file with a single-threaded Python prototype takes over 36 seconds for data ingestion and over 100 seconds for metadata joining — confirming that the dataset qualifies as Big Data in terms of processing demands relative to available hardware. The two datasets exhibit distinct structures (Variety): the review file is line-delimited JSON with structured fields (`rating`, `text`, `timestamp`), while the metadata file contains semi-structured nested objects (`categories`, `images`, `details`) with inconsistent field presence across records.

The overall objective of this project is to analyse Amazon Electronics customer reviews to identify the highest-rated products and most frequent sentiment keywords across the catalogue. This analysis provides actionable insights for consumers seeking reliable product recommendations and for sellers aiming to understand quality benchmarks — insights only feasible to extract at scale through a Big Data pipeline.

------

## Section 2. Traditional Solution

Before developing the big data pipeline, we built a single-threaded Python prototype to validate the processing logic on a 500,000-row subset of the data and establish performance baselines. No parallelism was used at this stage. The overall objective was decomposed into five steps:

### Step 1: Data Loading and Cleaning

We read the review JSONL file line by line, filtering out records missing a rating, product ID, or review text. HTML tags embedded in review texts (e.g. `<br/>`) were stripped using a regular expression, producing 499,993 clean records.

```python
def clean_text(text):
    text = re.sub(r'<[^>]+>', ' ', text)   # remove HTML tags
    return re.sub(r'\s+', ' ', text).strip()

reviews = []
with open("Electronics.jsonl", "r", encoding="utf-8") as f:
    for i, line in enumerate(f):
        if i >= MAX_ROWS:
            break
        r = json.loads(line)
        if r.get("rating") and r.get("parent_asin") and r.get("text"):
            reviews.append({
                "rating": float(r["rating"]),
                "asin": r["parent_asin"],
                "text": clean_text(r["text"]),
            })
```

> **Time: 36.23s | Peak memory: 331.5 MB**

------

### Step 2: Aggregate Ratings per Product

For each unique product (identified by `parent_asin`), we accumulated total rating and review count using a `defaultdict`, then computed the average rating. This produced 177,724 unique products.

```python
asin_stats = defaultdict(lambda: {"count": 0, "total_rating": 0.0})
for r in reviews:
    asin_stats[r["asin"]]["count"] += 1
    asin_stats[r["asin"]]["total_rating"] += r["rating"]

asin_avg = {
    asin: {"count": v["count"], "avg_rating": round(v["total_rating"] / v["count"], 2)}
    for asin, v in asin_stats.items()
}
```

> **Unique products: 177,724 | Time: 1.33s | Peak memory: 88.3 MB**

------

### Step 3: Word Frequency Analysis

All review texts were tokenised by whitespace, common stopwords removed, and word frequencies counted using Python's `Counter`. This CPU-bound step processes millions of tokens sequentially and was identified as a primary bottleneck.

```python
word_counter = Counter()
for r in reviews:
    words = r["text"].lower().split()
    filtered = [w.strip(".,!?\"'") for w in words
                if w.strip(".,!?\"'") not in stopwords and len(w) > 2]
    word_counter.update(filtered)
```

> **Top 10 words:** great(149,010), very(124,083), one(123,014), use(112,360), good(108,195), just(98,595), like(97,960), these(95,473), can(95,052), all(92,840)
>
> **Time: 35.41s | Peak memory: 23.0 MB**

------

### Step 4: Join Reviews with Product Metadata

The metadata JSONL file (~1 GB) was read sequentially to build a lookup dictionary keyed by `parent_asin`. Each product's aggregated review stats were then enriched with title, category, and price. This I/O-bound step was the largest bottleneck due to file size.

```python
meta = {}
with open("meta_Electronics.jsonl", "r", encoding="utf-8") as f:
    for line in f:
        m = json.loads(line)
        asin = m.get("parent_asin")
        if asin:
            meta[asin] = {
                "title": m.get("title", "Unknown"),
                "category": m.get("main_category", "Unknown"),
                "price": m.get("price")
            }

enriched = {asin: {**stats, **meta[asin]}
            for asin, stats in asin_avg.items() if asin in meta}
```

> **Joined: 177,724 products | Time: 104.62s | Peak memory: 850.2 MB**

------

### Step 5: Output Top 20 Product Ranking

Products with at least 100 reviews were filtered and sorted by average rating in descending order.

```python
top20 = sorted(
    [(asin, info) for asin, info in enriched.items() if info["count"] >= 100],
    key=lambda x: x[1]["avg_rating"], reverse=True
)[:20]
```

> **Time: 0.01s**

**Table 1: Top 20 highest-rated Electronics products (minimum 100 reviews)**

| Rank | Avg Rating | Reviews | Product Title                                     |
| ---- | ---------- | ------- | ------------------------------------------------- |
| 1    | 4.89       | 122     | Amazon Basics High-Speed HDMI Cable 2-Pack        |
| 2    | 4.85       | 113     | Crucial RAM 16GB Kit DDR3 1600 MHz CL11           |
| 3    | 4.85       | 110     | Lifetime 28240 Adjustable Folding Laptop Table    |
| 4    | 4.81       | 133     | Lamicall Tablet Stand Adjustable                  |
| 5    | 4.78       | 125     | Fintie Slimshell Case for 6" Kindle Paperwhite    |
| 6    | 4.77       | 113     | Amazon Basics USB 2.0 Extension Cable             |
| 7    | 4.77       | 154     | Amazon Basics USB-A to USB-B 2.0 Cable            |
| 8    | 4.77       | 133     | SAMSUNG 64GB 100MB/s (U3) MicroSD                 |
| 9    | 4.76       | 454     | Amazon Basics HDMI Cable 18Gbps High-Speed        |
| 10   | 4.75       | 193     | Cat 6 Ethernet Cable 1Ft (6Pack)                  |
| 11   | 4.75       | 142     | BlueRigger HDMI Cable 4K (6FT, 4K 60Hz HDR)       |
| 12   | 4.74       | 276     | Amazon Kindle Paperwhite Leather Case, Onyx Black |
| 13   | 4.74       | 147     | SanDisk 32GB Cruzer USB 2.0 Flash Drive           |
| 14   | 4.74       | 171     | Amazon Basics External Hard Drive Portable Case   |
| 15   | 4.73       | 163     | Echo Dot (3rd Gen) - Smart speaker with Alexa     |
| 16   | 4.73       | 260     | MagicFiber Extra Large Microfiber Cleaning Cloth  |
| 17   | 4.73       | 142     | VELCRO Brand ONE-WRAP Cable Ties 100Pk            |
| 18   | 4.73       | 245     | Chicka Chicka Boom Boom (Chicka Chicka Book)      |
| 19   | 4.72       | 120     | SAMSUNG 32GB 95MB/s (U1) MicroSD                  |
| 20   | 4.72       | 106     | Ceptics Brazil Travel Adapter Plug with Dual USB  |

------

### Traditional Solution Performance Summary

| Step      | Description               | Time        | Peak Memory |
| --------- | ------------------------- | ----------- | ----------- |
| Step 1    | Load & Clean Data         | 36.23s      | 331.5 MB    |
| Step 2    | Aggregate Ratings         | 1.33s       | 88.3 MB     |
| Step 3    | Word Frequency Analysis ⚠️ | 35.41s      | 23.0 MB     |
| Step 4    | Join with Metadata ⚠️      | 104.62s     | 850.2 MB    |
| Step 5    | Top 20 Ranking Output     | 0.01s       | —           |
| **Total** |                           | **177.60s** |             |

> ⚠️ Steps 3 and 4 identified as bottlenecks for MapReduce optimisation.

------

## Section 3. MapReduce Optimisation

From Section 2, Steps 3 and 4 were identified as the primary bottlenecks: word frequency analysis (35.41s) and metadata join (104.62s), together accounting for 79% of total execution time. Both tasks are well-suited to MapReduce — word counting is embarrassingly parallel, and key-based joining maps directly to the MapReduce paradigm. We implemented both optimisations using **Spark Core API (RDDs)** with PySpark 3.5.1, running in `local[*]` mode across all 16 logical cores. We expected a 3–5x speedup on both tasks.

------

### MapReduce Optimisation 1: Word Frequency Analysis

Word frequency is embarrassingly parallel: each review can be tokenised independently with no inter-record dependencies. We mapped each review to a list of `(word, 1)` pairs, then used `reduceByKey` to sum counts across all partitions in parallel.

**Map function** — for each review, emit `(word, 1)` pairs:

```python
def map_words(review_dict):
    text = re.sub(r'<[^>]+>', ' ', review_dict["text"]).lower()
    return [
        (w.strip(".,!?\"'"), 1)
        for w in text.split()
        if len(w.strip(".,!?\"'")) > 2
        and w.strip(".,!?\"'") not in STOPWORDS
    ]
```

**Reduce function** — sum counts per word:

```python
top10 = sc.parallelize(reviews, sc.defaultParallelism) \
    .flatMap(map_words) \
    .reduceByKey(lambda a, b: a + b) \
    .sortBy(lambda x: x[1], ascending=False) \
    .take(10)
```

> **Results:** Identical top 10 words as traditional solution — correctness confirmed.
>
> **Spark time: 36.32s | Traditional: 35.41s | Speedup: 0.97x**

The speedup did not match our 3–5x expectation. The reason is that `sc.parallelize(reviews)` serialises the in-memory Python list and transfers it to Spark workers via py4j socket communication (~150 MB transfer for 500k records), which largely cancels out the parallelism benefit. In a production environment where data is read directly from HDFS by each worker independently, this serialisation overhead would not exist and the expected 3–5x speedup would be achievable. This result highlights an important lesson: **MapReduce efficiency depends not just on computation, but also on data locality**.

------

### MapReduce Optimisation 2: Metadata Join

The metadata join is a classic distributed hash join: both review stats and metadata are keyed by ASIN, allowing Spark to co-partition and join them in parallel.

**Map (reviews)** — emit `(asin, (rating, 1))`:

```python
def map_review_for_join(r):
    return (r["asin"], (r["rating"], 1))
```

**Reduce** — aggregate to average rating per product:

```python
review_stats_rdd = sc.parallelize(reviews, sc.defaultParallelism) \
    .map(map_review_for_join) \
    .reduceByKey(lambda a, b: (a[0]+b[0], a[1]+b[1])) \
    .mapValues(lambda x: round(x[0]/x[1], 2))
```

**Map (metadata)** — emit `(asin, (title, category, price))`:

```python
meta_rdd = sc.parallelize(meta_list, sc.defaultParallelism)
```

**Distributed hash join** by ASIN key:

```python
joined_rdd = review_stats_rdd.join(meta_rdd)
result_count = joined_rdd.count()
```

> **Joined: 177,724 products — identical to traditional, correctness confirmed.**
>
> **Spark time: 58.83s | Traditional: 104.62s | Speedup: 1.78x**

The 1.78x speedup is meaningful but below the 3–5x expectation. Similar to Step 3, `sc.parallelize()` introduces serialisation overhead. However, the join operation involves significant inter-partition shuffling (Spark redistributes records so matching ASINs land on the same partition), and parallelising this shuffle across 16 cores delivers a real improvement despite the serialisation cost. In a distributed environment with data pre-partitioned by ASIN on HDFS, the full 3–5x speedup would be expected.

------

### Final Performance Comparison

**Table 2: Performance comparison — Traditional Python vs Spark RDD**

| Task                    | Traditional | Spark RDD | Actual Speedup | Expected |
| ----------------------- | ----------- | --------- | -------------- | -------- |
| Word Frequency (Step 3) | 35.41s      | 36.32s    | 0.97x          | 3–5x     |
| Metadata Join (Step 4)  | 104.62s     | 58.83s    | 1.78x          | 3–5x     |

In summary, MapReduce delivered a meaningful improvement for the metadata join (1.78x) and demonstrated correctness on both tasks. The below-expectation speedup for word frequency is attributable to Python-JVM serialisation overhead inherent in the local development environment when using `sc.parallelize()` rather than reading from a distributed filesystem. This is a constraint of local mode, not a fundamental limitation of MapReduce — in a production HDFS cluster, both tasks would benefit from the full expected speedup.