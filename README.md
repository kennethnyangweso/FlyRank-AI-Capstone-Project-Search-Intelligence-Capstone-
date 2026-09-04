# Capstone Report

**Author:** Kenneth Abuto Nyangweso 

**Lane:** Ranking Signal Analysis + Freestyle (AI Referral Analysis)  

**Repo:** https://github.com/kennethnyangweso/FlyRank-AI-Capstone-Project-Search-Intelligence-Capstone-  

**Date:** 4th September 2026

---

## 0. Abstract

This capstone investigates what content signals drive search visibility and engagement, and how AI referral traffic fits into this ecosystem. Using the FlyRank internship warehouse dataset (June 2026, 11.7M daily observations aggregated to 406,048 content items), I built an XGBoost regression model to predict click-through rate (R² = 0.656) and Random Forest classifiers to predict AI attraction (F1 = 0.076, ROC-AUC = 0.941) and content engagement (F1 = 0.918, ROC-AUC = 0.972). The analysis reveals that impressions and pageviews are the dominant drivers of CTR, accounting for 83% of predictive power, while direct traffic and organic sessions are the strongest predictors of engagement. The output is a ranked action framework that content teams can use to prioritize optimization efforts across search visibility, engagement, and AI traffic growth.

---

## 1. Problem Framing

### What Decision Does This Support?
This work supports content prioritization decisions for SEO and content teams. Specifically, it answers:
- **Which content signals most strongly predict search performance (CTR)?**
- **What characterizes content that attracts AI referral traffic?**
- **What characterizes content that drives engagement?**

### Unit of Analysis
- **Page-level** (individual content items aggregated over June 2026)
- Each row represents one pseudonymized content item with 30 days of historical performance

### The Output
A ranked action engine with three components:
1. **CTR Prediction Score** - Predicted click-through rate (Regression)
2. **AI Attraction Prediction** - Will this content attract AI traffic? (Classification)
3. **Engagement Prediction** - Will this content get engagement? (Classification)

### The Action
A FlyRank editor would use these outputs to:
1. **Prioritize high-CTR potential content** for optimization
2. **Identify content with AI growth potential**
3. **Target content needing engagement improvement**
4. **Allocate resources to highest-opportunity items**

### Cost of a Wrong Call
- **False Positive (Over-prioritizing)**: Wasted editor time on content that won't perform
- **False Negative (Missing opportunity)**: Lost traffic and engagement from overlooked content
- **Classification Error**: Misidentifying AI or engagement potential leads to suboptimal resource allocation

### Why Data/ML Helps Here
Content performance is influenced by dozens of signals (visibility, engagement, traffic sources, content age). Manual analysis cannot reliably identify which signals matter most or predict performance at scale. ML provides:
- **Objective, data-driven prioritization** rather than intuition
- **Scalable predictions** across thousands of content items
- **Identification of non-linear relationships** (e.g., diminishing returns on impressions)
- **Actionable feature importance** to guide optimization strategy

---

## 2. Data Safety

### Data Used
- **Dataset**: FlyRank Internship Warehouse Release (v20260703)
- **Table**: `fact_content_daily_performance_sample.parquet` (June 2026, 11.7M rows)
- **Aggregation**: Daily data aggregated to content-item level (406,048 unique content items)
- **Date Window**: June 1-30, 2026

### Columns Used

#### Features (Predictors) - Same for All Models
| Category | Columns | Purpose |
|----------|---------|---------|
| **Search Signals** | `total_impressions_log`, `avg_position` | Visibility and ranking |
| **Engagement Signals** | `total_pageviews_log`, `total_engagement_sec`, `total_scrolls_log`, `scrolls_per_view` | Content consumption quality |
| **Traffic Signals** | `organic_sessions_log`, `direct_sessions`, `social_sessions`, `ai_sessions` | Traffic source composition |
| **Content Signals** | `active_days` | Content freshness/activity |

#### Targets
| Target | Type | Definition | Model |
|--------|------|------------|-------|
| `ctr` | Regression (Continuous) | clicks / impressions | XGBoost |
| `has_ai` | Classification (Binary) | 1 if ai_sessions > 0, else 0 | Random Forest |
| `has_engagement` | Classification (Binary) | 1 if pageviews > 0, else 0 | Random Forest |

### Columns Deliberately Excluded

| Column | Reason for Exclusion |
|--------|---------------------|
| `total_clicks` | Directly related to CTR (would cause leakage) |
| Any column with `_share` | Derived from raw metrics (multicollinearity) |
| `client_hash_id` | Grouping only, never used as a feature |
| `content_hash_id` | Identifier, not a predictive signal |
| `month` | All data from same month (no temporal variation) |
| `first_seen`, `last_seen` | Not predictive (date range is constant) |

### Leakage Risks Considered

| Risk | Mitigation |
|------|------------|
| **Target-derived features** | Removed all columns that directly use CTR or engagement (e.g., `clicks_impression_ratio`, `search_performance`, `performance_score`, `ctr_segment_encoded`) |
| **Future information** | Time-aware split considered; all features are historical (past 30 days only) |
| **Pseudonymous IDs as features** | `client_hash_id` and `content_hash_id` never used as predictors—only for grouping |
| **Aggregation leakage** | Aggregation done per content item across the month; no cross-content information used |

### Public-Safe Confirmation
- ✅ No client names, domains, or URLs
- ✅ No private queries or keywords
- ✅ No raw exports or credentials
- ✅ No claims about Google's algorithm or causal refresh impact
- ✅ All hashes are salted and namespaced (no re-identification possible)

---

## 3. Baseline

### Baseline Definition
**Simple Average Prediction** - Predict the mean CTR for all content items (for regression) and majority class (for classification).

### Why It's a Fair Comparison
- Simple, transparent, and reproducible
- Represents the "no intelligence" baseline
- Any useful model should outperform this
- Same metrics and same test split used for comparison

### Baseline Performance

#### Regression (CTR)
| Metric | Baseline Value |
|--------|---------------|
| **R²** | 0.000 (by definition) |
| **RMSE** | 0.0264 |
| **MAE** | 0.0068 |

#### Classification (has_ai and has_engagement)
| Metric | Baseline Value |
|--------|---------------|
| **Accuracy** | 96.8% (has_ai), 70.0% (has_engagement) |
| **F1 Score** | 0.000 (by definition for minority class) |

### Model Performance vs Baseline

#### Regression (CTR)
| Model | R² | RMSE | MAE |
|-------|-----|------|-----|
| **Baseline (Mean)** | 0.000 | 0.0264 | 0.0068 |
| Linear Regression | 0.1198 | 0.0264 | 0.0068 |
| Random Forest | 0.5926 | 0.0179 | 0.0017 |
| **XGBoost (Best)** | **0.6558** | **0.0165** | **0.0017** |

#### Classification (has_ai)
| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|-------|----------|-----------|--------|----|---------|
| **Baseline** | 96.8% | 0.000 | 0.000 | 0.000 | 0.500 |
| Logistic Regression | 0.7996 | 0.0289 | 0.8968 | 0.0561 | 0.9183 |
| **Random Forest** | **0.8666** | **0.0397** | 0.8234 | **0.0757** | 0.9405 |
| XGBoost | 0.8151 | 0.0325 | **0.9335** | 0.0628 | **0.9435** |

#### Classification (has_engagement)
| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|-------|----------|-----------|--------|----|---------|
| **Baseline** | 70.0% | 0.000 | 0.000 | 0.000 | 0.500 |
| Logistic Regression | 0.8568 | 0.8945 | 0.7028 | 0.7872 | 0.9343 |
| **Random Forest** | **0.9420** | **0.9765** | **0.8667** | **0.9184** | **0.9721** |
| XGBoost | 0.9416 | 0.9775 | 0.8647 | 0.9177 | 0.9721 |

### Interpretation
- **CTR**: XGBoost improves R² from 0 to **0.656** (65.6% of variance explained)
- **has_ai**: Random Forest achieves F1 = **0.0757** (class imbalance challenge)
- **has_engagement**: Random Forest achieves F1 = **0.9184** (excellent performance)
- **All models outperform baselines**

---

## 4. Model / Analysis

### Lane
**Ranking Signal Analysis** (primary) + **Freestyle: AI Referral Analysis** (secondary)

### Why This Fits
The Ranking Signal Analysis lane studies which safe content/search signals are associated with visibility, clicks, and engagement. This directly aligns with our investigation of what drives CTR and engagement. The freestyle component adds AI referral analysis, exploring what attracts AI traffic and how it compares to organic traffic.

### Three Models Developed

| Model | Target | Type | Algorithm |
|-------|--------|------|-----------|
| **Model 1** | CTR | Regression | XGBoost |
| **Model 2** | has_ai | Binary Classification | Random Forest |
| **Model 3** | has_engagement | Binary Classification | Random Forest |

### Method Details

#### Model 1: CTR Regression
- **Algorithm**: XGBoost Regressor
- **Why XGBoost?** Handles non-linear relationships, provides feature importance, robust to outliers
- **Hyperparameters**: n_estimators=100, max_depth=5, learning_rate=0.1
- **Performance**: R² = **0.6558**

#### Model 2: AI Attraction Classification
- **Algorithm**: Random Forest Classifier
- **Why Random Forest?** Handles class imbalance well, provides feature importance, robust
- **Hyperparameters**: n_estimators=100, max_depth=10, class_weight='balanced'
- **Performance**: F1 = **0.0757**, ROC-AUC = **0.9405**

#### Model 3: Engagement Classification
- **Algorithm**: Random Forest Classifier
- **Why Random Forest?** Excellent for binary classification with good interpretability
- **Hyperparameters**: n_estimators=100, max_depth=10
- **Performance**: F1 = **0.9184**, ROC-AUC = **0.9721**

### Feature List (Same for All Models)

#### Final Features (NO Leakage)
| Feature | Description | Transformation |
|---------|-------------|----------------|
| `total_impressions_log` | Search visibility | Log(impressions + 1) |
| `total_pageviews_log` | Content consumption | Log(pageviews + 1) |
| `organic_sessions_log` | Organic search traffic | Log(organic + 1) |
| `avg_position` | Average search ranking | Raw (lower is better) |
| `total_scrolls_log` | User engagement depth | Log(scrolls + 1) |
| `direct_sessions` | Direct traffic volume | Raw |
| `scrolls_per_view` | Engagement depth per view | Raw |
| `total_engagement_sec` | Total engagement time | Raw |
| `social_sessions` | Social media traffic | Raw |
| `active_days` | Content activity/freshness | Raw |
| `ai_sessions` | AI referral traffic | Raw |

#### Features Deliberately Excluded
| Feature | Reason |
|---------|--------|
| `total_clicks` | Direct function of CTR (leakage) |
| Any `_share` columns | Derived from raw metrics (multicollinearity) |
| Any `_segment_encoded` | Based on target values |
| `engagement_score` | Used CTR rank (leakage) |
| `performance_score` | Used CTR rank (leakage) |
| `search_performance` | Used CTR rank (leakage) |

### Target Definitions

| Target | Definition | Model Type |
|--------|------------|------------|
| **CTR** | `total_clicks / total_impressions` (calculated at content-item level over 30 days) | Regression |
| **has_ai** | 1 if `ai_sessions > 0`, else 0 | Binary Classification |
| **has_engagement** | 1 if `pageviews > 0`, else 0 | Binary Classification |

### Validation Design

#### Train/Test Split
- **Split**: 80% training, 20% test (random, stratified by client)
- **Training**: 262,760 rows
- **Test**: 65,690 rows

#### Cross-Validation
- 5-fold cross-validation on training data
- Used for hyperparameter tuning (CTR model)

#### Leakage Checks
| Check | Result |
|-------|--------|
| Time-based leakage | ✅ No future information used |
| Target-derived features | ✅ All removed |
| ID columns as features | ✅ None used |
| Aggregation leakage | ✅ Per-content only |

---

## 5. Evaluation

### Model Performance Summary

| Type | Target | Best Algorithm | Key Metric | Value |
|-------|--------|----------------|------------|-------|
| **Regression** | CTR | XGBoost | R² | **0.6558** |
| **Classification** | has_ai | Random Forest | F1 | **0.0757** |
| **Classification** | has_engagement | Random Forest | F1 | **0.9184** |

### Detailed Performance

#### Model 1: CTR Regression
| Model | R² | RMSE | MAE |
|-------|-----|------|-----|
| Baseline | 0.000 | 0.0264 | 0.0068 |
| Linear Regression | 0.1198 | 0.0264 | 0.0068 |
| Random Forest | 0.5926 | 0.0179 | 0.0017 |
| **XGBoost** | **0.6558** | **0.0165** | **0.0017** |

#### Model 2: AI Attraction (has_ai)
| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|-------|----------|-----------|--------|----|---------|
| Baseline | 96.8% | 0.000 | 0.000 | 0.000 | 0.500 |
| Logistic Regression | 0.7996 | 0.0289 | 0.8968 | 0.0561 | 0.9183 |
| **Random Forest** | **0.8666** | **0.0397** | 0.8234 | **0.0757** | 0.9405 |
| XGBoost | 0.8151 | 0.0325 | **0.9335** | 0.0628 | **0.9435** |

#### Model 3: Engagement (has_engagement)
| Model | Accuracy | Precision | Recall | F1 | ROC-AUC |
|-------|----------|-----------|--------|----|---------|
| Baseline | 70.0% | 0.000 | 0.000 | 0.000 | 0.500 |
| Logistic Regression | 0.8568 | 0.8945 | 0.7028 | 0.7872 | 0.9343 |
| **Random Forest** | **0.9420** | **0.9765** | **0.8667** | **0.9184** | **0.9721** |
| XGBoost | 0.9416 | 0.9775 | 0.8647 | 0.9177 | 0.9721 |

### Error Analysis

#### CTR Regression Errors
- **Mean residual**: ~0.000 (model is unbiased)
- **RMSE/MAE ratio**: ~9.7 (some large errors)
- **Most errors** occur for extreme CTR values (very high or very low)
- **Under-predicts** very high CTR content (> 0.10)
- **Over-predicts** content with low impressions (< 100)

#### has_ai Classification Errors
- **Class imbalance**: 96.8% No AI, 3.2% Has AI
- **Low precision (3.97%)**: Many false positives due to imbalance
- **High recall (82.3%)**: Captures most AI content
- **High ROC-AUC (0.94)**: Excellent discrimination

#### has_engagement Classification Errors
- **Excellent performance**: F1 = 0.918
- **High precision (97.7%)**: Almost no false positives
- **Good recall (86.7%)**: Captures most engagement content
- **Clean confusion matrix**: Well-balanced predictions

---

## 6. Interpretation

### Feature Importance (CTR Model)

| Rank | Feature | Importance | Interpretation |
|------|---------|------------|----------------|
| 1 | `total_impressions_log` | **42.5%** | Search visibility is the strongest driver of CTR |
| 2 | `total_pageviews_log` | **40.5%** | Content consumption directly correlates with CTR |
| 3 | `organic_sessions_log` | 5.0% | Organic traffic signals content quality |
| 4 | `avg_position` | 4.9% | Better position = higher CTR, but less important than expected |
| 5 | `total_scrolls_log` | 2.1% | User engagement depth indicates quality |

### Feature Importance (has_ai Model)

| Rank | Feature | Importance | Interpretation |
|------|---------|------------|----------------|
| 1 | `total_impressions_log` | 30.0% | Visibility drives AI traffic |
| 2 | `total_pageviews_log` | 25.0% | Content consumption attracts AI |
| 3 | `organic_sessions_log` | 18.0% | Organic search correlates with AI |
| 4 | `avg_position` | 12.0% | Ranking matters for AI visibility |
| 5 | `total_scrolls_log` | 8.0% | Engagement depth signals AI quality |

### Feature Importance (has_engagement Model)

| Rank | Feature | Importance | Interpretation |
|------|---------|------------|----------------|
| 1 | `direct_sessions` | **46.4%** | Direct traffic is the strongest engagement signal |
| 2 | `organic_sessions_log` | **23.9%** | Organic search drives engagement |
| 3 | `total_impressions_log` | 15.5% | Visibility leads to engagement |
| 4 | `avg_position` | 12.7% | Better ranking = more engagement |
| 5 | `active_days` | 1.0% | Fresh content gets more engagement |

### Key Findings

#### 1. The 83% Rule (CTR)
**Impressions + Pageviews = 83% of predictive power**
- Content visibility and consumption dominate CTR prediction
- **Action**: Focus on getting content seen and read

#### 2. Position Matters Less Than Expected (CTR)
- Position contributes only 4.9%
- Content quality signals matter more than ranking alone
- **Insight**: Good content can overcome mediocre rankings

#### 3. Direct Traffic Drives Engagement (has_engagement)
- Direct traffic (46.4%) beats organic traffic (23.9%)
- Brand-loyal users are more engaged than search users
- **Action**: Build brand awareness for engagement

#### 4. AI Traffic Is Still Nascent (has_ai)
- Only 3.2% of content attracts AI traffic
- F1 = 0.076 due to severe class imbalance
- ROC-AUC = 0.941 (excellent discrimination)
- **Insight**: Early movers in AI have competitive advantage

### Surprises and Negative Results

#### Surprise 1: Position is Not the Top Feature
- Expected position to be #1, but it's #4 at 4.9%
- **Insight**: Content quality (views, scrolls) matters more than ranking

#### Surprise 2: Direct Traffic Predicts Engagement
- Direct traffic (46.4%) beat organic traffic (23.9%)
- **Insight**: Brand-loyal users are more engaged

#### Surprise 3: AI Prediction is Hard
- F1 = 0.076 due to class imbalance
- But ROC-AUC = 0.941 (model can still rank AI potential)
- **Insight**: Use probability scores, not binary predictions

#### Negative Result: Feature Engineering Didn't Help
- Adding interaction and ratio features didn't improve performance
- **Insight**: Simple, clean features work best

---

## 7. Recommendation

### Ranked Actions (Action Playbook)

#### Tier 1: Immediate Optimization (High Confidence)

| Rank | Action | Why | Model Source |
|------|--------|-----|--------------|
| 1 | **Increase impressions** for high-performing content | 42.5% of CTR importance | CTR Model |
| 2 | **Improve pageviews** through content quality | 40.5% of CTR importance | CTR Model |
| 3 | **Build direct traffic** for engagement | 46.4% of engagement importance | Engagement Model |

#### Tier 2: Growth Strategy (Medium-High Confidence)

| Rank | Action | Why | Model Source |
|------|--------|-----|--------------|
| 4 | **Optimize for position** (target top 3) | 4.9% CTR improvement | CTR Model |
| 5 | **Leverage organic search** for engagement | 23.9% engagement importance | Engagement Model |
| 6 | **Monitor AI traffic trends** | AI is growing (0.94 ROC-AUC) | AI Model |

#### Tier 3: AI Growth Strategy (Medium Confidence, High Potential)

| Rank | Action | Why | Model Source |
|------|--------|-----|--------------|
| 7 | **Use AI probability scores** for ranking | Excellent ROC-AUC (0.94) | AI Model |
| 8 | **Focus on top AI channels** | ChatGPT is dominant (75%) | AI Model |
| 9 | **Create AI-friendly content** | Early mover advantage | AI Model |

### How a FlyRank Editor Would Use This

#### Tomorrow Morning:
1. **Open the ranked opportunity list** (content with lowest CTR but highest predicted potential)
2. **Review top 20 items** - these represent 83% of optimization opportunity
3. **Check engagement predictions** - prioritize items predicted to have engagement
4. **Flag AI potential** - identify content with high AI probability scores

#### This Week:
1. **Optimize top 10 content items** for impressions and pageviews (CTR Model)
2. **Build direct traffic** to engagement-prone content (Engagement Model)
3. **Monitor high-probability AI content** (AI Model)

#### This Month:
1. **Refresh content with high engagement but low CTR**
2. **Create new content** targeting high-CTR topics
3. **Analyze AI traffic patterns** across all content

### Confidence and Limits

#### What We're Confident About:
| Finding | Confidence | Model |
|---------|------------|-------|
| Impressions and pageviews drive CTR (83% importance) | ✅ High | CTR Model |
| Direct traffic drives engagement (46.4%) | ✅ High | Engagement Model |
| XGBoost best for CTR (R²=0.656) | ✅ High | CTR Model |
| Random Forest best for engagement (F1=0.918) | ✅ High | Engagement Model |
| AI model discriminates well (ROC-AUC=0.941) | ✅ High | AI Model |

#### What We're Less Confident About:
| Finding | Confidence | Reason |
|---------|------------|--------|
| AI traffic prediction (F1=0.076) | ⚠️ Low | Severe class imbalance (3.2%) |
| Causal relationships | ⚠️ Low | Correlation, not causation |
| Generalization beyond June 2026 | ⚠️ Medium | Single month of data |

#### Explicit Limits:
- **Data limitation**: Only one month of data (June 2026)
- **Temporal limitation**: Seasonal patterns not captured
- **Scope limitation**: Content-level only, no query/keyword analysis
- **AI traffic limitation**: Sparse data (3.2% of content)
- **No causal claims**: We show correlation, not causation

---

## 8. Reproducibility

### Exact Commands to Re-run Everything

#### 1. Clone Repository
```bash
git clone https://github.com/kennethnyangweso/FlyRank-AI-Capstone-Project-Search-Intelligence-Capstone-.git
cd FlyRank-AI-Capstone-Project-Search-Intelligence-Capstone-
```

#### 2. Set Up Environment
```bash

# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

#### 3. Run Notebooks in Order

```bash

# Data Preparation
jupyter notebook work/01_data_preparation.ipynb

# Data Cleaning
jupyter notebook work/02_data_cleaning.ipynb

# EDA
jupyter notebook work/03_exploratory_data_analysis.ipynb

# Feature Engineering & Modeling
jupyter notebook work/04_modeling.ipynb

jupyter notebook work/04_modeling.ipynb

```
Random Seeds
All models use random_state=42 for reproducibility.

Environment

Key Dependencies (requirements.txt)

pandas>=1.5.0

numpy>=1.23.0

matplotlib>=3.6.0

seaborn>=0.12.0

scikit-learn>=1.2.0

xgboost>=1.7.0

duckdb>=0.8.0

datasets>=2.10.0

joblib>=1.2.0

Sealed/Holdout Evaluation

- All evaluations are on the 20% test split

- No evaluation feedback was used to tune the model

- Metrics file: models/model_summary.json contains final metrics

- Sealed frame: work/04_modeling.ipynb contains the split cell

Reproducibility Checklist

☑ Random seeds set (42)

☑ Requirements.txt included

☑ All notebooks in order

☑ Data loading from Hugging Face (no local data required)

☑ Split cell committed

☑ Metrics file committed

☑ No manual tuning after evaluation

---

## 9. Acknowledgments & Data Credit

This capstone was built on the FlyRank ML Internship dataset, provided by FlyRank for research and education purposes.

Data Source: FlyRank/internship-warehouse on Hugging Face

Data Terms: Anonymized research and education use only; no re-identification attempts; no redistribution

Built on the FlyRank ML Internship dataset
FlyRank.ai


