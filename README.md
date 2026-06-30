# CEO-Bench Framework Copy

> Princeton arXiv:2606.18543 | https://ceobench.com | github.com/zlab-princeton/ceobench-src

**模擬**: Agent 經營 SaaS 公司 NovaMind，500 天，起始 $1M。成績 = 期末現金。
**界面**: Python package `novamind_api` → 34 tools + SQL 查 19 表 + 自訂腳本系統。

---

## 1. 19 張資料庫表（完整 Schema）

### customers
所有客戶（個人 + 企業）
```
customer_id          INTEGER PRIMARY KEY
customer_type        TEXT — 'small' / 'large' (enterprise)
group_id             TEXT — 'S1','S2','S3','E1','E2','E3','D_S01'-'D_S10','D_E01'-'D_E10'
created_day          INTEGER
persona_industry     TEXT — e.g. creative, legal, manufacturing
persona_role         TEXT — e.g. freelancer, managing-partner
persona_experience   TEXT — e.g. early-career, veteran
persona_work_style   TEXT — e.g. scrappy, methodical, strategic
persona_tech_savvy   TEXT — e.g. basic, expert
company_size_descriptor  TEXT — enterprise only (e.g. regional, prestigious)
company_culture          TEXT — enterprise only
company_decision_style   TEXT — enterprise only
company_primary_concern  TEXT — enterprise only
persona_description      TEXT — human-readable brief
email               TEXT — enterprise only
contract_start_day   INTEGER — enterprise only
acquisition_source   TEXT — 'word_of_mouth' or ad channel ID
--- HIDDEN (每客戶 sampling params) ---
steepness_left/right — 不對稱 sigmoid 曲線參數
c_max — 硬預算上限 ($/mo, per-seat for enterprise)
q_max — 品質天花板（可感知的最大品質）
q_min — 品質地板（免費時仍需的最低品質）
usage_demand — 期望用量 (units/day)
quality_sensitivity, price_sensitivity, willingness_to_pay
usage_scale, patience
ads_quality_sensitivity, ads_return_sensitivity
contract_lockin_penalty
reply_delay_mean/std, negotiation_rate, initial_offer_factor, max_negotiation_turns  (enterprise only)
```

### subscriptions
訂閱（當前 + 歷史）
```
subscription_id     INTEGER PRIMARY KEY
customer_id         INTEGER FK → customers
plan                TEXT — 'A','B','C'
listed_price        REAL — 列表價 / 協商價
promotion           REAL — 當前折扣
effective_price     REAL — 實際支付 = listed_price − promotion (≥0)
start_day           INTEGER
end_day             INTEGER — NULL if active
status              TEXT — 'lead','subscribed','cancelled','lost'
billing_day_mod30   INTEGER — 計費日 (0-29)
seat_count          INTEGER
pending_plan        TEXT — 計劃中的方案變更
pending_price       REAL — 變更後價格
contract_months     INTEGER — 承諾長度 (1=月付)
contract_end_day    INTEGER
--- HIDDEN ---
daily_usage_rate, billing_period_usage, effective_c_max
churn_reason, first_billing_done
```

### daily_usage
```
day, customer_id, usage_units
```

### ledger
總帳流水
```
id       INTEGER PK
day      INTEGER
category TEXT — 'subscription_payment','compute','capacity','advertising',
          'operations','development','lead_acquisition_cost',
          'initial_funding','market_research','group_research',
          'research_project','ad_revenue'
amount   REAL — positive=income, negative=expense
note     TEXT
```

### service_day
每日服務指標
```
day               INTEGER PK
total_usage_units INTEGER
p95_ms            REAL
error_rate        REAL (0-1)
downtime_minutes  INTEGER
capacity_tier     INTEGER (0-7)
capacity_units    INTEGER
```

### config_history
每日配置快照
```
day                INTEGER PK
price_A/B/C        REAL
tier_A/B/C         INTEGER
spend_advertising  REAL
spend_operations   REAL
spend_development  REAL
capacity_tier      INTEGER
ad_spend_social_media / search_ads / linkedin / content_marketing / referral_program  REAL
quota_A/B/C        INTEGER
```

### social_media_posts
客戶社群反饋
```
post_id    INTEGER PK
day        INTEGER
content    TEXT
--- HIDDEN ---
customer_id, likes, shares, virality_score, sentiment, reputation_impact, influence_score
```

### agent_social_media_posts
CEO 發文
```
agent_post_id     INTEGER PK
day               INTEGER
content           TEXT (max 280 chars)
reply_to_post_id  INTEGER — NULL if original post
views             INTEGER
comment_post_ids  TEXT — JSON list of customer comment post_ids
--- HIDDEN ---
effect_by_group   TEXT — JSON {group_id: effect_score [-1,1]}
views_by_group    TEXT — JSON {group_id: view_count}
```

### predictions
現金預測記錄
```
submit_day       INTEGER
horizon_days     INTEGER — 7, 28, 84, 182
metric           TEXT — 'cash'
predicted_value  REAL
predicted_lower  REAL — 95% CI lower
predicted_upper  REAL — 95% CI upper
submitted_at     REAL — epoch seconds
```

### enterprise_turns
企業協商對話
```
message_id       INTEGER PK
customer_id      INTEGER FK
thread_type      TEXT — 'new_lead','plan_change','churn_prevention','renegotiation','renewal','general'
turn_number      INTEGER — 0-indexed
sender           TEXT — 'customer','agent','system'
message_text     TEXT
offer_json       TEXT — structured offer data
day              INTEGER
email            TEXT
seat_count       INTEGER
closed           INTEGER — 0=open, 1=closed
close_reason     TEXT — 'accepted','agent_rejected'
--- HIDDEN ---
next_reply_day, current_offer_price, _internal_status
```

### notifications
收件匣
```
notification_id  INTEGER PK
day              INTEGER
type             TEXT
message          TEXT
```

### research_projects
R&D 專案
```
project_id              TEXT PK — e.g. 't1_1','t1_2'
tier                    INTEGER (1-20)
status                  TEXT — 'in_progress','completed'
started_day             INTEGER
expected_completion_day INTEGER
expected_quality_boost  REAL — sampled
quality_boost_applied   REAL — actual on completion
--- HIDDEN ---
actual_completion_day   INTEGER
```

### macroeconomic_conditions
總經條件（ISM PMI，~30天發布延遲）
```
day           INTEGER PK
pmi_value     REAL (30-70 scale)
pmi_trend     TEXT — 'strong_expansion'(>58),'expansion'(52-58),'neutral'(48-52),
              'contraction'(42-48),'severe_contraction'(<42)
pmi_change    REAL
cycle_phase   TEXT — 'peak','declining','trough','recovering'
description   TEXT — human-readable summary
```

### ad_channel_leads
廣告成效歷史
```
id              INTEGER PK
day             INTEGER
channel_id      TEXT
group_id        TEXT
leads_generated INTEGER
spend           REAL
```

### group_info_levels
群組研究等級
```
group_id         TEXT PK
info_level       INTEGER — 0=undiscovered, 1-5=researched
is_discoverable  INTEGER — 1=discoverable, 0=initial
discovered_day   INTEGER
last_research_day INTEGER
```

### segment_discovery
市場研究歷史
```
id                     INTEGER PK
day                    INTEGER
cost                   REAL
success                INTEGER — 1 if new segment found
discovered_group_id    TEXT — NULL if failed
--- HIDDEN ---
remaining_undiscovered INTEGER
```

### issues
支援案件
```
issue_id        INTEGER PK
customer_id     INTEGER FK
group_id        TEXT
open_day        INTEGER
days_open       INTEGER
status          TEXT — 'open','resolved'
resolved_day    INTEGER — NULL if open
resolution_type TEXT — 'ops_resolved'
```

### ads_revenue
廣告收入明細
```
day           INTEGER
customer_id   INTEGER FK
group_id      TEXT
ads_strength  REAL (0-1)
revenue       REAL
--- HIDDEN ---
seat_count    INTEGER
sensitivity   REAL
```

### config_overrides
進階配置變更歷史
```
id              INTEGER PK
day             INTEGER
tool_name       TEXT — e.g. 'set_promotion','set_ads_strength','set_lead_promotion'
setting_type    TEXT — 設定類別
settings_json   TEXT — 全量 JSON 快照
```

---

## 2. 34 個工具（完整參數）

### 2.1 商業配置 (4)

#### `set_prices(A, B, C)`
設定月訂閱價（A/$float, B/$float, C/$float）→ 下週生效
- 影響：獲客（高價→少註冊）、續約、營收
- 內部：asymmetric sigmoid Q_req(price)，分左右半坡

#### `set_model_tiers(A=1-5, B=1-5, C=1-5)`
品質乘數 × 成本：
```
Tier 1: $0.30/Mtok → 0.60×  Flash-Lite/4o-mini class
Tier 2: $2.00/Mtok → 0.75×  Haiku/Flash class
Tier 3: $6.00/Mtok → 0.90×  Sonnet/GPT-4o class
Tier 4: $12.00/Mtok → 1.00× Opus/GPT-5 class
Tier 5: $30.00/Mtok → 1.10× o1/o3 reasoning class
```
delivered_quality = product_quality × tier_multiplier

#### `set_capacity_tier(tier=0-7)`
```
Tier 0:   50K units/day  $85/d    — serverless API
Tier 1:  200K units/day  $215/d   — 1× H100 dedicated
Tier 2:  800K units/day  $530/d   — 4× H100 reserved
Tier 3:  2.5M units/day  $1,330/d — 8× H100 + auto-scaling
Tier 4:    8M units/day  $4,000/d — 16-32 H100 hyperscale
Tier 5:   25M units/day  $10,000/d — 64× H100 cluster
Tier 6:   80M units/day  $28,000/d — 256× H100 pod
Tier 7:  300M units/day  $75,000/d — 1024+ GPU fleet
```
超載公式：overload = max(0, total_usage/capacity − 1) → p95↑ error_rate↑
outage_prob = 0.1 × overload²

#### `set_usage_quotas(A=int, B=int, C=int)`
每人每日用量配額。超額→品質懲罰

### 2.2 行銷支出 (7)

#### `set_daily_spend(operations=$, development=$)`
全域每日預算
- **ops**: outage_prob = 0.03 × exp(−0.002 × ops_spend)。$0→~3%/d 停機風險
- **dev**: q_improvement = 0.006 × ln(1 + spend/5000) per day

#### `set_targeted_ad_spend(targeted_spend={channel: {group: $/day}})`
唯一廣告管道。5渠道×26群組，精準投放。
渠道清單：social_media, search_ads, linkedin, content_marketing, referral_program

#### `set_targeted_ops_spend(by_group={}, by_plan={}, by_group_plan={}, by_customer={})`
額外運維：4 種 scope 獨立 Poisson 解析池
- Individual groups (S*,D_S*): scale=0.3 issues/$/day
- Enterprise groups (E*,D_E*): scale=0.05 issues/$/day

#### `set_targeted_dev_spend(targeted_spend={group: $/day})`
累積性群組品質加成（5× 效率 vs 全域 dev）
q_bonus += 0.005 × log(1 + spend/1000) per day

#### `set_ads_strength(global=0-1, by_group={}, by_customer={})`
應用內廣告強度。log-scaling: effective = log(1+9×strength)/log(10)
效果：quality_penalty = sensitivity × effective
      revenue = return_sensitivity × effective (per customer/day)

#### `set_lead_promotion(global=$, by_group={}, by_channel={}, by_channel_group={})`
新客戶首月折扣（疊加）。第一計費期自動應用

#### `set_promotion(global=$, by_group={}, by_customer={}, by_group_plan={})`
既有客戶折扣（下計費週期生效）

### 2.3 企業銷售 (2)

#### `send_enterprise_deal(deals=[{customer_id, plan, price, seat_count, contract_months}])`
多輪議價。返回對話 thread，客戶在 stochastic delay 後回覆

#### `reject_enterprise_deal(deals=[{customer_id}])`
拒絕企業提案

### 2.4 資訊分析 (7)

#### `get_social_posts(days=None, limit=50)`
讀取社群貼文（客訴/競爭者/總經趨勢）

#### `get_cost_info()`
當前所有成本/層級資訊

#### `get_market_overview()`
市場概覽 + 總經條件（PMI, cycle phase）

#### `get_group_insights(group_id)`
群組洞察（含參數估計、聲譽影響網絡、網絡效應矩陣）

#### `describe_tables(table_names=[...])`
SQL schema 查詢

#### `list_all_tables()`
列出19張表名

#### `get_tool_documentation(tool_names=[...])`
任一工具的完整文檔（含 sample_io）

### 2.5 市場研究 (2)

#### `research_market()`
發現新客戶群（20個可發現：D_S01-10, D_E01-10）。順序由 seed 預先確定。
成本遞增，第1次最便宜

#### `research_group(group_id, target_level=1-5)`
提升群組資訊層級。Level 越高→agent 看到的 hidden param 越精確

### 2.6 R&D (2)

#### `start_research_project(tier=1-20)`
Tier 1-10: 標準 R&D。Tier 11-20: 前沿（高成本、長週期、高方差、高品質回報）
每個專案有隨機完成時間(Normal)和品質增益(Normal)。可重複啟動同 tier

#### `list_research_projects()`
列出所有研發中/已完成專案

### 2.7 社群媒體 (1)

#### `post_social_media(content, reply_to_post_id=None)`
發文（≤280字）或回覆。效果經 LLM judge 評估影響各群組聲譽

### 2.8 執行腳本 (7)

#### `python_exec(code)`
執行任意 Python code + SQL。這是最強工具——可分析資料庫後自訂策略邏輯

#### `register_daily_calculation(name, code)`
註冊每日自動計算函數（回傳值自動寫入記憶檔案）

#### `remove_daily_calculation(name)` / `list_daily_calculations()`

#### `register_script(name, code)` / `run_script(name)` / `list_scripts()` / `delete_script(name)`
命名腳本系統。腳本可透過 `register_daily_calculation` 每日排程執行

### 2.9 模擬控制 (1)

#### `next_week(rationale, dashboard_notes, cash_pred_7d, cash_pred_28d, cash_pred_84d, cash_pred_182d, cash_pred_7d_lower, cash_pred_7d_upper, cash_pred_28d_lower, ...)`
每週必呼叫一次→前進 7 天。需提交 4 個 horizon 的現金預測點估計 + 95% CI 上下界。
預測準確度是論文分析的關鍵技能指標之一。

---

## 3. 26 客戶群（Hidden Parameters）

### 3.1 初始 6 群

| 群 | 名稱 | q_min±σ | q_range±σ | c_max±σ | slope±σ | usage±σ | base_market_cap | churn/mo | Lock-in |
|:---|:-----|:--------|:----------|:--------|:--------|:--------|:---------------|:---------|:--------|
| S1 | 價格敏感個人 | 0.10±0.05 | 0.45±0.10 | $50±27 | 0.010±0.004 | 80±50 | 272K | 5.5% | 16%/mo |
| S2 | 品質導向專業 | 0.30±0.08 | 0.55±0.08 | $140±60 | 0.003±0.0015 | 180±90 | 136K | 3.5% | 10%/mo |
| S3 | 重度使用者 | 0.25±0.07 | 0.55±0.10 | $180±75 | 0.004±0.002 | 450±250 | 85K | 4.5% | 12%/mo |
| E1 | 成本削減企業 | 0.50±0.06 | 0.45±0.10 | $55/seat±23 | 0.008±0.003 | 60/seat±38 | 1.19K | 1.5% | 10%/mo |
| E2 | 品質優先企業 | 0.70±0.08 | 0.50±0.06 | $120/seat±45 | 0.002±0.0015 | 150/seat±75 | 510 | 0.8% | 6%/mo |
| E3 | 戰略夥伴 | 0.75±0.10 | 0.35±0.08 | $100/seat±38 | 0.003±0.0015 | 100/seat±50 | 136 | 0.5% | 4%/mo |

**個人群座席**: S1:1, S2:1, S3:1 | **企業群座席**: E1:50-500, E2:100-1000, E3:200-2000

### 3.2 可發現個人群 D_S01-10

| ID | 群組 | q_min | c_max | usage | market_cap | churn | 廣告最優渠道 |
|:---|:-----|:------|:------|:------|:-----------|:------|:----------|
| D_S01 | Niche Creators | 0.15 | $80 | 120 | 68K | 5.5% | referral(267), linkedin(249) |
| D_S02 | Academic Researchers | 0.30 | $110 | 150 | 34K | 2.5% | social(185), referral(160) |
| D_S03 | Non-Profit Workers | 0.18 | $55 | 90 | 68K | 4.5% | social(214), search(100) |
| D_S04 | Small Agency Teams | 0.20 | $100 | 200 | 34K | 4.0% | search(64), referral(135) |
| D_S05 | Indie Game Devs | 0.25 | $130 | 350 | 17K | 4.5% | referral(231), social(114) |
| D_S06 | Freelance Writers | 0.12 | $65 | 100 | 51K | 5.0% | search(167), content(114) |
| D_S07 | Data Analysts | 0.28 | $120 | 280 | 25K | 3.5% | search(125), referral(167) |
| D_S08 | Social Media Managers | 0.15 | $70 | 150 | 34K | 4.5% | social(285), referral(249) |
| D_S09 | UX Designers | 0.25 | $110 | 180 | 17K | 3.5% | content(107), search(185) |
| D_S10 | Music Producers | 0.12 | $60 | 80 | 17K | 5.5% | content(214), linkedin(206) |

### 3.3 可發現企業群 D_E01-10

| ID | 群組 | q_min | c_max/seat | usage/seat | seat_range | market_cap | churn |
|:---|:-----|:------|:-----------|:-----------|:-----------|:-----------|:------|
| D_E01 | Government Agencies | 0.65 | $45 | 50 | 100-1000 | 85 | 1.0% |
| D_E02 | Educational Institutions | 0.50 | $35 | 80 | 100-800 | 170 | 2.0% |
| D_E03 | Healthcare Networks | 0.70 | $75 | 120 | 200-2000 | 51 | 0.8% |
| D_E04 | Regional Banks | 0.65 | $55 | 90 | 150-1500 | 68 | 0.5% |
| D_E05 | Insurance Brokers | 0.55 | $50 | 70 | 50-500 | 170 | 1.0% |
| D_E06 | Construction Firms | 0.40 | $40 | 40 | 50-400 | 340 | 2.5% |
| D_E07 | Telecom Operators | 0.60 | $70 | 130 | 500-5000 | 34 | 0.5% |
| D_E08 | Energy Companies | 0.55 | $60 | 60 | 200-2000 | 34 | 0.8% |
| D_E09 | Real Estate Groups | 0.35 | $45 | 100 | 50-500 | 255 | 2.0% |
| D_E10 | Shipping Lines | 0.40 | $50 | 50 | 200-1500 | 34 | 1.5% |

### 3.4 廣告渠道 × 群組 yields (leads/$1000/day)

```
                 social_media   search_ads   linkedin   content_mkt  referral
S1 (Price-Sens)    124.58       124.58        49.83      320.36       302.56
S2 (Quality)       149.50        64.07       160.18      231.37       135.26
S3 (Power User)    106.79        81.87        35.60       99.67       170.86
E1 (Cost-Cut)        0.10         0.10         0.13        0.24         0.09
E2 (Quality-1st)     0.22         0.15         0.09        0.05         0.09
E3 (Strategic)       0.07         0.05         0.03        0.09         0.17
```
Enterprise leads 是 account-level（每個 lead = 一個組織，含數十到數千 seats）
Referral 最便宜（病毒效應），LinkedIn 對企業最有效，Content 對 S1/S2 最強

---

## 4. 世界機制核心公式

### 每日現金變化
```
Δcash = 訂閱收入 + 廣告收入 − 容量成本 − 算力成本 − 運維支出 − 研發支出
        − 精準運維 − 精準研發 − 廣告支出 − 線索獲取成本
        − 市場研究 − 群組研究 − 研發專案
```

### 新客戶期望 (per group per day)
```
E[新線索] = 聲譽 × 市場飽和度 × 日曆週期 × 總經週期 
          × 社群反應 × 需求暴衝 × (廣告線索 + 網絡效應)
```
- 廣告線索 = Σ(spend × leads_per_1000 / 1000)
- 網絡效應 = Σ(現有客戶 × network_leads_per_1000 / 1000)
- Poisson 採樣

### 客戶感知品質
```
Q_perc = model_tier × (q0 + q_shared_bonus + q_group_bonus)
       − overload_penalty − outage_penalty
       + relationship_bonus + stickiness_bonus
       − open_issues_penalty − quota_saturation_penalty
       − inapp_ads_penalty
```
客戶的參與約束：Q_perc ≥ Q_min + slope × price，且 price ≤ c_max
否則退訂

### Stochastic 分佈
| 分佈 | 用途 | Motivation |
|:-----|:-----|:-----------|
| Normal | R&D 品質增益 | Captures uncertain payoff |
| Poisson | 每日新線索數 | Rate-based counts |
| Bernoulli | 非自願退訂事件 | Binary shocks |
| Uniform | 聲譽損害噪音 | Bounded uncertainty |
| Log-normal | 競爭者品質跳升 | Skewed positive shocks |

### 競爭者機制
- Stationary: 預設序列品質提升事件
- Adaptive: 每期提升 u·I，u~U[0.2,0.5]，I=agent累積品質
- 競爭者品質提升→客戶期望上升→不投入 dev 則品質落後→churn

### 聲譽傳播
聲譽在群組間擴散。reputation_influence_matrix 定義群組間影響權重。
一個群組的品質失敗→影響相關群組聲譽→連鎖反應

---

## 5. R&D Research Tiers (1-20)

| Tier | 成本 | 預期完成 | 品質增益範圍 | 標籤 |
|:-----|:-----|:---------|:------------|:------|
| 1 | $1K | 7d | +0.02 | Quick Win |
| 2 | $3K | 12d | +0.04 | Small Improvement |
| 3 | $8K | 18d | +0.06 | Feature Enhancement |
| 4 | $15K | 25d | +0.09 | Major Feature |
| 5 | $25K | 33d | +0.12 | Breakthrough |
| 6 | $40K | 42d | +0.16 | Platform Upgrade |
| 7 | $60K | 52d | +0.20 | Architecture Redesign |
| 8 | $85K | 63d | +0.25 | Next Generation Engine |
| 9 | $115K | 75d | +0.30 | Revolutionary |
| 10 | $150K | 90d | +0.35 | Paradigm Shift |
| 11 | $200K | 100d | +0.40+ | Frontier Moonshot |
| ... | ... | ... | ... | (Tier 11-20: 前緣研發) |
| 20 | $1.5M | 200d | +0.80+ | Defining Moonshot |

---

## 6. Rule-Based 最佳策略

論文實測最好的 rule-based 啟發式演算法（期末 $15.76M）：

**配置**:
- 固定價格: ~$9.99/~$29.99/~$99.99（A/B/C 的經典定價梯度）
- 固定廣告: ~$20K/週，集中在 S2 + E1 的 high-leads channel 組合
- 固定研發: ~$30K/週 globals
- 固定運維: 約營收 15%
- 容量層級: Tier 2-3（視用戶數觸發調整）
- Model tier: Tier 3-4 固定不做優化

**不做的**:
- 不 research_market（不探索新群組）
- 不調整價格
- 不調整 model tier
- 不用 promotion 系統
- 不用 enterprise 協商
- 不用 ads_strength
- 不用 targeted_dev_spend

**關鍵 insight**: 策略越穩定→churn 越穩定→聲譽累積→獲客自然增長。
頻繁調整策略是最常見的破產原因。

---

## 7. Paper Citations Reference

| 模型 | 破產數 | 最高現金 | 平均存活天數 |
|:-----|:-------|:---------|:------------|
| Claude Opus 4.8 | 0/3 | $27.78M | 500±0 |
| GPT-5.5 | 2/3 | $21.30M | 334±230 |
| Claude Opus 4.7 | 0/3 | $0.39M | 500±0 |
| Kimi K2.6 | 1/3 | $0.10M | 343±110 |
| Claude Sonnet 4.6 | 2/3 | $0.07M | 282±136 |
| GLM 5.1 | 3/3 | $0 | 215±91 |
| Claude Haiku 4.5 | 3/3 | $0 | 145±71 |
| Gemini 3 Flash | 3/3 | $0 | 154±37 |
| DeepSeek V4 Pro | 3/3 | $0 | 114±39 |
| Grok 4.20 | 3/3 | $0 | 28±9 |
| **Rule-based** | — | $15.76M | 500 |

Estimated upper bound of attainable cash: ~$2.2B (far from saturated).

---

*Source: github.com/zlab-princeton/ceobench-src | arXiv:2606.18543 | ceobench.com*
*This is a framework copy for reference — all parameters extracted from source code at commit f69b6f3.*
