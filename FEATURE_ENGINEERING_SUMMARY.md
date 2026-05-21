# Feature Engineering Summary - HBAAC Demand Forecasting

## Overview
Total features used: **23 features** organized into 7 categories

### Feature List by Category

---

## 1. **CALENDAR FEATURES** (9 features)
Extract temporal patterns and seasonality from dates

| Feature | Type | Range | Meaning | Impact |
|---------|------|-------|---------|--------|
| `dayofweek` | Integer | 0-6 | Day of week (0=Monday, 6=Sunday) | Captures **weekly demand patterns** (e.g., higher sales on weekdays vs weekends) |
| `month` | Integer | 1-12 | Month of year | Captures **monthly/seasonal patterns** (e.g., higher demand in certain months) |
| `quarter` | Integer | 1-4 | Quarter of year | Captures **quarterly business cycles** (e.g., fiscal quarter effects) |
| `weekofyear` | Integer | 1-53 | ISO week number | Captures **weekly positioning** within the year; identifies holiday periods |
| `year` | Integer | 2024-2025 | Calendar year | Accounts for **year-over-year trends** and long-term growth |
| `dayofmonth` | Integer | 1-31 | Day within the month | Captures **end-of-month effects** (e.g., payment cycles, promotions) |
| `is_weekend` | Binary | 0 or 1 | Boolean: is Saturday or Sunday (1=yes, 0=no) | Captures **weekend vs weekday demand differences** |
| `is_tet_season` | Binary | 0 or 1 | Boolean: is January or February (Vietnamese Lunar New Year) | Captures **seasonal surge** during Tet holidays (currently disabled in code) |

**Impact**: Helps model recognize **time-based demand patterns** beyond just historical quantities. Auto parts may have different demand on weekends vs weekdays, and during holidays.

---

## 2. **LAG FEATURES** (5 features)
Recent historical quantities at different time intervals

| Feature | Type | Meaning | Impact |
|---------|------|---------|--------|
| `lag_1` | Float | Quantity sold **1 day ago** | Captures **immediate momentum** and day-to-day persistence |
| `lag_7` | Float | Quantity sold **7 days ago** | Captures **weekly pattern repetition** (same day of week effect) |
| `lag_14` | Float | Quantity sold **14 days ago** | Captures **bi-weekly cyclicity** and mid-range trends |
| `lag_28` | Float | Quantity sold **28 days ago** | Captures **monthly pattern repetition** and seasonal demand |
| `lag_56` | Float | Quantity sold **56 days ago** | Captures **bi-monthly trends** and longer cyclicity |

**Calculation**: `lag_X = quantity from X days ago` (shifted backward in time)

**Impact**: Directly uses **past demand as a strong predictor** of future demand. Most time series data shows strong autocorrelation (today's demand is related to yesterday's). These are typically the most important features.

---

## 3. **ROLLING MEAN FEATURES** (4 features)
Average quantity over recent windows (shifted to avoid data leakage)

| Feature | Type | Window | Meaning | Impact |
|---------|------|--------|---------|--------|
| `roll_mean_7` | Float | 7 days | Average quantity over past 7 days | Captures **short-term demand trend**, smooths daily volatility |
| `roll_mean_14` | Float | 14 days | Average quantity over past 14 days | Captures **2-week trend**, provides smoother signal |
| `roll_mean_28` | Float | 28 days | Average quantity over past 28 days | Captures **monthly trend**, identifies growth/decline patterns |
| `roll_mean_56` | Float | 56 days | Average quantity over past 56 days | Captures **2-month trend**, identifies long-term patterns |

**Calculation**: For each day, compute the **mean of yesterday + 6 previous days** (shift=1 to prevent leakage)

**Impact**: Provides **smoothed trend information** without using future data. Helps identify if demand is trending up/down/stable. Reduces noise compared to single-day lags.

---

## 4. **SKU ENCODING & PROFIT WEIGHTS** (2 features)
Product identifier and business importance

| Feature | Type | Value | Meaning | Impact |
|---------|------|-------|---------|--------|
| `sku_enc` | Integer | 0 to 15,971 | Encoded product ID (LabelEncoder) | Allows model to **differentiate between SKUs** and learn product-specific patterns |
| `weight` | Float | 0.0 to ~0.0001 | Profit contribution ratio per SKU | **Emphasizes high-profit products** in training; used for weighted RMSSE calculation in competition |

**Calculation**: 
- `sku_enc`: Sequential integer mapping of ItemCode (0=first SKU, 15971=last SKU)
- `weight`: (Total profit for SKU) / (Total profit across all SKUs)

**Impact**: 
- `sku_enc`: Allows model to **learn different demand patterns per product** (e.g., brake pads may have different patterns than oil filters)
- `weight`: Ensures model **prioritizes accuracy on high-profit items** where errors matter most

---

## 5. **INTERMITTENCY/SPARSITY FEATURES** (2 features)
Characteristics of products with intermittent (sporadic) demand

| Feature | Type | Meaning | Impact |
|---------|------|---------|--------|
| `zero_sale_ratio_28` | Float | Proportion of zero-sale days in past 28 days (0.0-1.0) | Identifies **intermittent/sparse products** that only sell on certain days |
| `days_since_last_sale` | Integer | Days elapsed since last sale (capped at 999) | Indicates **demand dormancy**; high values = product rarely sells |

**Calculation**:
- `zero_sale_ratio_28`: Count zero-sale days in last 28 days / 28
- `days_since_last_sale`: Current day - (date of last sale > 0)

**Impact**: 
- Helps model handle **two-tier products**: intermittent (rarely sell) vs regular (sell most days)
- Intermittent products have **harder-to-predict demand** and need special treatment
- High `days_since_last_sale` → expect continued low/zero demand; low value → expect sales to resume

---

## 6. **TARGET ENCODING / HISTORICAL AGGREGATES** (2 features)
Average historical demand by time-of-period and season

| Feature | Type | Meaning | Impact |
|---------|------|---------|--------|
| `target_enc_dayofweek` | Float | Average quantity sold on **this day-of-week** historically | Captures **weekly seasonality** (e.g., Fridays typically higher than Mondays) |
| `target_enc_month` | Float | Average quantity sold in **this month** historically | Captures **monthly/seasonal demand** (e.g., higher in summer than winter) |

**Calculation**:
- `target_enc_dayofweek`: Mean quantity for (SKU, dayofweek) across all training data
- `target_enc_month`: Mean quantity for (SKU, month) across all training data
- **Computed only on training data to prevent future leakage**

**Impact**: 
- Provides **strong baseline seasonality signals** based on historical patterns
- For example, if a SKU always sells more on Fridays, this feature encodes that pattern
- Helps model **quickly adapt to seasonal changes**

---

## 7. **PRICE & PROFIT FEATURES** (4 features)
Unit economics and pricing dynamics

| Feature | Type | Meaning | Impact |
|---------|------|---------|--------|
| `unit_price_lag1` | Float | Unit selling price from **yesterday** | Captures **price sensitivity**; price changes may affect demand |
| `unit_cost_lag1` | Float | Unit cost from **yesterday** | Provides **cost context** for products; high-margin items may have different demand patterns |
| `profit_margin_lag1` | Float | Profit margin yesterday: `(price - cost) / price` (0.0-1.0) | Indicates **product profitability** yesterday; high margins = premium products |
| `price_discount_lag1` | Float | Price relative to max in past 56 days: `price / max_price_56days` (0.0-1.0) | Identifies **current discount level** (0.5 = 50% off max price) |

**Calculation**:
- Unit prices computed as: `SalesAmount / Quantity` for each transaction
- Lagged by 1 day: shifted backward to use yesterday's price
- `profit_margin_lag1`: (unit_price_lag1 - unit_cost_lag1) / unit_price_lag1
- `price_discount_lag1`: unit_price_lag1 / (max unit_price over past 56 days)

**Impact**:
- **Price sensitivity**: Products on sale (low `price_discount_lag1`) may sell more
- **Margin correlation**: High-margin products may have different demand elasticity
- **Promotional effects**: Captures price-driven spikes in demand
- **Supply chain signals**: Cost changes may precede volume adjustments

---

## Summary Table: Feature Groups & Their Impact

| Group | # Features | Primary Purpose | Model Impact |
|-------|-----------|-----------------|--------------|
| Calendar | 9 | Temporal patterns & seasonality | High - captures weekly, monthly cycles |
| Lags | 5 | Recent history & momentum | **Very High** - autoregressive foundation |
| Rolling Mean | 4 | Smoothed trends | High - trend identification, noise reduction |
| SKU Encoding | 1 | Product differentiation | Medium - enables product-specific learning |
| Profit Weights | 1 | Business importance | High - prioritizes valuable products |
| Intermittency | 2 | Demand sparsity patterns | Medium-High - handles sparse/intermittent products |
| Target Encoding | 2 | Historical seasonal patterns | High - seasonal baseline |
| Price/Profit | 4 | Economic dynamics | Medium - captures promotions & pricing effects |

---

## Feature Importance (from model training)

Based on the model's feature importance analysis:
- **Top performers**: Lag features (especially `lag_1`, `lag_7`) and rolling means are typically most important
- **Mid-level**: Calendar features and target encoding
- **Supporting**: Price features and intermittency indicators

---

## Key Design Decisions

1. **Lag shift timing**: Rolling features use `shift(1)` (yesterday's data) to prevent data leakage in recursive forecasting
2. **Target encoding computed on training data only**: Prevents using future information during validation
3. **Price features lagged by 1 day**: Avoids leakage (tomorrow's price shouldn't predict today's demand)
4. **Profit weights**: Used to emphasize high-profit products in training and scoring
5. **Recursive forecasting**: Predictions update lag features day-by-day during forecast generation

---

## Future Enhancement Opportunities

- Add **promotional calendar** features if available
- Add **competitor pricing** data
- Add **macro economic indicators** (GDP growth, unemployment)
- Add **weather data** (temperature, precipitation)
- Separate models for **intermittent vs regular demand** products
- Add **interaction features** (e.g., `price * dayofweek`)
- Time-decay weights (recent observations more important than old ones)
