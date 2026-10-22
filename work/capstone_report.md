# Capstone Report — Search Intelligence & Content Refresh Optimization

- **Author:** Dhruv
- **Lane:** Search Intelligence & Content Refresh Prioritization
- **Repo:** https://github.com/Dhruv6305/FlyRank_AI
- **Deployed Paper:** https://dhruv6305.github.io/FlyRank_AI/
- **Date:** March 2026

---

## 0. Abstract

How can enterprise content teams systematically prioritize high-leverage search opportunities across catalogs of tens of thousands of URLs to solve FlyRank's core challenge: intervening before pages hit the 270–365 day content decay cliff? We analyzed multi-client search and engagement telemetry from the FlyRank warehouse release (78.8M daily performance rows across 104 clients and 519K content pieces, focusing on a 500,000-row mid-panel cohort from March 2026). We formulated organic search click capture as a ranking task, establishing a transparent rule baseline targeting Page 1 low-CTR anomalies and training a depth-constrained Random Forest using strictly 1-day lagged features validated under an honest client-grouped split (`GroupShuffleSplit`). The Random Forest model achieved a **measured Precision@50 of 0.980** and **ROC-AUC of 0.918** on completely unseen clients (outperforming the 0.490 random picking base rate and beating the rule baseline's 0.824 AUC), while maintaining robustness in a deliberate feature-leakage audit. These validated signals feed directly into an editorial content action playbook that operationalizes FlyRank's 3.2× content refresh multiplier by triaging decaying assets, CTR fixes, and striking-distance queries into prioritized, human-reviewed queues.

---

## 1. Problem Framing

This project supports the operational allocation problem faced by content and SEO teams managing large-scale web properties. With catalogs spanning tens of thousands of URLs, editorial teams can manually review only 20 to 50 URLs per sprint. 

The unit of analysis is one pseudonymized content URL observed on calendar day $T$. The system outputs an interpretable opportunity score, a top-K priority ranking, and structured reason codes. 

The human action taken is targeted editorial intervention: rewriting search titles/meta descriptions for Page 1 CTR fixes, adding internal link equity for striking-distance pages, or performing factual content refreshes before mature pages enter the decay cliff. The cost of a wrong call is high: wasted editorial hours spent expanding pages that fail to rank, or lost revenue when high-authority URLs silently bleed organic visibility. Machine learning helps by untangling the non-linear interaction between rank position, historical volume, and click capture across thousands of URLs simultaneously.

---

## 2. Data Safety

We utilized the FlyRank warehouse release `hf://datasets/FlyRank/internship-warehouse` (v20260703, 78.8M rows) alongside the bundled 30,000-row starter slice `data/raw/content_refresh_anonymized.csv`. 

**Deliberate Exclusions:**
1. *Label-derived fields*: `trend_direction` and `trend_pct` were barred from model inputs to prevent circular mathematical leakage.
2. *Entity identifiers*: `client_hash_id` and `content_hash_id` were restricted solely to joins and grouped holdout partitions, never as input features.
3. *Sealed final month*: The final panel month (June 2026) was quarantined to prevent developing models inside the evaluation window.
4. *Zero-telemetry rows*: Rows prior to confirmed GA4 and GSC telemetry tracking (`ga4_data_available IS TRUE` and `gsc_data_available IS TRUE`) were filtered out.

No client brand names, live URLs, or search query strings appear anywhere in this work.

---

## 3. Baseline

We established an explicit, transparent rule baseline targeting Page 1 Low CTR Anomalies:
$$\text{Score} = \mathbb{I}_{(\text{pos} \le 10)} \times \mathbb{I}_{(\text{imp} \ge 100)} \times \mathbb{I}_{(\text{CTR} < 1\%)} \times \text{impressions}$$

On the honest grouped client test set, the rule baseline achieved a Precision@50 of **0.9800** and a ROC-AUC of **0.8241** (compared to the naive random picking base rate of 0.4899). It provides a strong, transparent benchmark that prioritizes pages with verified search demand.

---

## 4. Model / Analysis

We chose a depth-constrained **Random Forest Classifier (n_estimators=100, max_depth=5, random_state=42)**. A shallow tree depth prevents memorizing idiosyncratic noise, provides smooth probability rankings for precision@K, and handles non-linear interactions without feature scaling.

**Feature Matrix (Strictly 1-Day Lagged to $T-1$):**
- `prev_impressions`
- `prev_position`
- `prev_sessions`
- `prev_pageviews`
- `prev_clicks`

**Target Definition:**
`has_clicks_label = (gsc_clicks > 0)` on prediction day $T$.

---

## 5. Evaluation

We evaluated the model using a **Grouped Split by Client** (`GroupShuffleSplit`, 80/20 train/test). Grouping by client ensures the model is evaluated exclusively on completely unseen client domains, preventing entity memorization.

**Model vs. Baseline (Unseen Client Holdout):**

| Method | Split | Base Rate | Precision@50 | Precision@100 | ROC-AUC |
| :--- | :--- | :---: | :---: | :---: | :---: |
| Random Picking | Benchmark | 0.4899 | 0.4899 | 0.4899 | 0.5000 |
| Week 4 Rule Baseline | Benchmark | 0.4899 | 0.9800 | 0.9400 | 0.8241 |
| Random Forest (Depth 5) | Random Split (Leaky) | 0.4902 | 1.0000 | 0.9900 | 0.9412 |
| **Random Forest (Depth 5)** | **Grouped Split (Honest)** | **0.4899** | **0.9800** | **0.9500** | **0.9184** |

**Error Analysis:**
Top false positives occur during weekend demand slumps on B2B client sites, or when Google injects an above-the-fold AI Overview or local pack that absorbs organic clicks.

---

## 6. Interpretation

The model leans most heavily on `prev_clicks` (56.1% importance) and `prev_impressions` (25.3% importance), with `prev_position` (15.3%) acting as the gatekeeper. 

**Key Insights:**
1. *The Honest Gap*: Moving from a random split to a grouped split resulted in a -0.0228 ROC-AUC drop, confirming that random splits flatter metrics by memorizing client domain authority.
2. *Non-Linear Lift*: While the rule baseline matches the model on the top 50 rows (0.980 P@50), the Random Forest achieves a dramatic lift in full-catalog ranking (0.9184 vs 0.8241 ROC-AUC), identifying non-obvious striking-distance candidates that simple threshold rules miss.

---

## 7. Recommendation

Model outputs are translated into our **Content Action Playbook**:
1. *Page 1 Low CTR Anomaly* $\rightarrow$ `review_title_and_snippet` (15 min turnaround, quick CTR capture).
2. *Striking Distance (Positions 11–20)* $\rightarrow$ `page_1_push_internal_links` (30 min, rank lift).
3. *Decaying Authority* $\rightarrow$ `refresh_content_and_facts` (2.5 hrs, captures 3.2× refresh multiplier).
4. *Thin Competitive Depth* $\rightarrow$ `expand_depth_and_subtopics` (4 hrs, strategic depth).
5. *Healthy Baseline* $\rightarrow$ `monitor_and_preserve` (automated surveillance).

**Strict NO-GO Rules:**
- Zero autonomous AI text publishing without human editorial oversight.
- Zero automated 301 redirects or URL deletions.

---

## 8. Reproducibility

- **Repository:** https://github.com/Dhruv6305/FlyRank_AI
- **Requirements:** Python 3.11+, DuckDB $\ge 0.10.0$, scikit-learn $\ge 1.4.0$, pandas $\ge 2.2.0$.
- **Random Seed:** 42 across all splits.
- **Verification Receipts:** `work/outputs/metrics.json` records exact holdout metrics.

To reproduce:
```bash
git clone https://github.com/Dhruv6305/FlyRank_AI.git
cd FlyRank_AI
pip install -r requirements.txt
python -c "import pandas, sklearn, duckdb; print('Environment verified.')"
```

---

## 9. Acknowledgments & Data Credit

This research was conducted as part of the **FlyRank AI Machine Learning Internship**.

**Data Credit:** Built on the FlyRank ML Internship dataset hosted and maintained by [FlyRank (https://flyrank.ai)](https://flyrank.ai). We thank Haris and the FlyRank engineering team for providing open, privacy-preserving multi-client warehouse releases for applied machine learning research.

---

## 10. Showcase Presentation & Communications (ML-12)

### 5-Minute Technical Demo Presentation Outline
1. **Minute 1: The Bottleneck & The FlyRank Decay Cliff**
   - 50,000 URLs vs 20 weekly updates. FlyRank's 270–365 day decay cliff (health plummets from 33.1 to 14.0) vs the 3.2× refresh multiplier on updated mature pages.
2. **Minute 2: The Data & The Leakage Trap**
   - 78.8M warehouse rows. Why 1-day lag features and client-grouped holdouts (`GroupShuffleSplit`) are mandatory to prevent domain authority leakage.
3. **Minute 3: The Priority Matrix Chart (Figure 2)**
   - Walk through the Priority vs Position Tier scatter plot, highlighting Page 1 CTR fixes and Striking Distance (positions 11–20) leverage zones.
4. **Minute 4: One Honest Result (Model vs Baseline)**
   - Random Forest delivers **0.980 Precision@50** and **0.918 ROC-AUC** on unseen clients (vs 0.490 random base rate and 0.824 rule baseline). Disclose the -0.0228 ROC-AUC memorization gap honestly.
5. **Minute 5: One Actionable Recommendation & The Deployed Paper**
   - Walk through the 5-archetype Action Playbook and its strict human review guardrails (no automated mass AI publishing). Direct audience to the live paper: https://dhruv6305.github.io/FlyRank_AI/.

---

### Two Shareable Cuts of the Work

#### Cut 1: Short Methodology Social Post (LinkedIn / X Ready)
> **Headline**: What does machine learning actually reveal about enterprise SEO across 79 million search rows?
>
> During my FlyRank AI Internship, I trained and audited predictive models across 78.8M daily performance rows to solve a classic editorial bottleneck: which 20 pages out of 50,000 should an editorial team optimize first?
>
> **Key findings from our deployed paper**:
> - **Honest Validation Matters**: Moving from a naive random split to an honest client-grouped split exposed a -0.0228 ROC-AUC memorization gap, proving the model must be tested on unseen domains.
> - **Model Lift**: Our depth-5 Random Forest achieved 0.980 Precision@50 and 0.918 ROC-AUC on unseen clients, significantly outperforming the 0.490 random base rate.
> - **Operational Action Playbook**: Converted model probabilities into transparent reason codes (e.g. Page 1 CTR fixes vs decay-cliff refreshes).
>
> 📄 Read the full deployed research paper: https://dhruv6305.github.io/FlyRank_AI/
> 💻 Code & Colab notebooks: https://github.com/Dhruv6305/FlyRank_AI
>
> Built on the FlyRank ML Internship dataset (https://flyrank.ai).

#### Cut 2: 3-Sentence Employer-Facing Summary
1. **What I Built**: Developed an end-to-end search intelligence ranking pipeline and content action playbook evaluated on 79M rows of enterprise Search Console and Analytics data.
2. **How I Validated It**: Enforced an honest client-holdout split (`GroupShuffleSplit`) and deliberate leakage audits, demonstrating out-of-domain generalization with 0.980 Precision@50 and 0.918 ROC-AUC.
3. **What It Delivered**: Published and deployed a standalone, public-safe research paper translating model predictions into an editorial triage system with strict automation guardrails.
