# Yelp Suspicious Review & Friend-Correlation Mining (PySpark)

This repository contains a PySpark-based analysis pipeline over the Yelp Academic Dataset. The workflow focuses on identifying **potentially suspicious low-rated reviews (1–2 stars)** by looking for cases where a reviewer’s **friends also reviewed the same businesses**, then ranking users by “suspicious friend review overlap.” A second phase performs **frequent itemset mining** (pairs/triplets) over suspect-centered friend groups to surface **recurring reviewer clusters**.

> Date in notebook: **December 8, 2024**  
> Stack: **Python + Apache Spark (PySpark)**, JSON-line Yelp dataset files

---

## What this project does

### 1) Load Yelp JSON datasets into Spark RDDs
The code reads these JSONL files:

- `data/yelp_academic_dataset_review.json`
- `data/yelp_academic_dataset_user.json`
- `data/yelp_academic_dataset_business.json`

It parses and projects each into a smaller “fields RDD”:

- **Reviews** → `(review_id, business_id, user_id, stars, text)`
- **Users** → `(user_id, name, yelping_since, friends)`
- **Businesses** → `(business_id, name, review_count, stars)`

It also saves small samples to `output/desAnalysis/*Sample` to confirm schema.

---

### 2) Filter for low-rated reviews (1–2 stars)
Low ratings are used as a signal to examine potential “coordinated negativity”:

- `lowRatedReviewsRDD = reviewFieldsRDD.filter(lambda r: r[3] in [1, 2])`

A sample is saved at:
- `output/desAnalysis/lowRatedReviewsSample`

---

### 3) Join low-rated reviewers with their friends and find overlaps
For each low-rated review, the code checks whether any of the reviewer’s friends also reviewed the **same business** (within the low-rated subset used in the join logic).

High-level steps:

1. Build `(user_id -> set(friend_ids))`
2. Attach friend sets to low-rated reviewers
3. Group low-rated reviews by business
4. For each reviewer + business, collect **friend reviews** on that business
5. Keep only cases where `len(friendsReviews) > 0`
6. Group by user and keep users with overlaps across **multiple businesses**

The final structure per user resembles:

```json
{
  "userId": "...",
  "businessReviews": [
    {
      "businessId": "...",
      "userReviewId": "...",
      "friendsReviews": [["friendUserId", "friendReviewId"], ...]
    },
    ...
  ]
}
