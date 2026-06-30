# CEO-Bench Framework Copy

> Princeton arXiv:2606.18543 | zlab-princeton/ceobench-src

Agent 以 34 個工具透過 Python interface 經營一家 SaaS 公司 NovaMind，500 天模擬，起始 $1M。成績 = 期末現金。

---

## 📊 19 張資料庫表

| 表 | 用途 | 關鍵欄位 |
|:---|:-----|:---------|
| **customers** | 所有客戶(個人/企業) | type, group_id, persona×5, company×4, **hidden**: c_max/q_max/q_min, steepness_L/R, sensitivities, usage_demand |
| **subscriptions** | 訂閱(當前+歷史) | plan(A/B/C), listed_price, promotion→effective_price, status(lead/subscribed/cancelled/lost), seat_count, contract_months |
| **daily_usage** | 逐日用量 | usage_units |
| **ledger** | 總帳流水 | category(14種), amount(+/−), note |
| **service_day** | 服務指標 | p95_ms, error_rate, downtime_minutes, capacity_tier |
| **config_history** | 每日配置快照 | prices, tiers, spends, capacity, quotas, 5-channel ads |
| **social_media_posts** | 客戶社群反饋 | content; **hidden**: sentiment, reputation_impact |
| **agent_social_media_posts** | CEO 發文 | content≤280, reply_to_post_id, views; **hidden**: effect_by_group |
| **predictions** | 現金預測記錄 | horizon(7/28/84/182d), predicted_value±95%CI |
| **enterprise_turns** | 企業協商對話 | thread_type(new_lead/plan_change/churn_prevention/renegotiation/renewal), offer_json, closed |
| **notifications** | 收件匣 | type, message |
| **research_projects** | R&D 專案 | tier(1-20), status, expected_completion_day, quality_boost |
| **macroeconomic_conditions** | 總經(ISM PMI) | pmi_value(30-70), cycle_phase, 約30天延遲 |
| **ad_channel_leads** | 廣告成效歷史 | channel_id, group_id, leads_generated, spend |
| **group_info_levels** | 群組研究等級 | info_level(0-5), is_discoverable |
| **segment_discovery** | 市場研究歷史 | cost, success, discovered_group_id |
| **issues** | 支援案件 | customer_id, days_open, status, resolution_type |
| **ads_revenue** | 廣告收入明細 | ads_strength, revenue; **hidden**: seat_count, sensitivity |
| **config_overrides** | 進階配置變更 | tool_name, setting_type, JSON全量快照 |

## 🔧 34 個工具

### 商業配置 (4)
| 工具 | 參數 |
|:-----|:-----|
| `set_prices` | A/B/C = $float |
| `set_model_tiers` | A/B/C = int(1-5) [品質乘數 0.60/0.75/0.90/1.00/1.10] |
| `set_capacity_tier` | tier = int(0-7) [50K/$85 ~ 300M/$75K daily] |
| `set_usage_quotas` | A/B/C = int (units/day/customer) |

### 行銷支出 (7)
| `set_daily_spend` | operations=$float, development=$float |
| `set_targeted_ad_spend` | {channel_id: {group_id: $/day}} | 5渠道×26群組，唯一廣告管道 |
| `set_targeted_ops_spend` | by_group/plan/group_plan/customer | 4種scope獨立Poisson池 |
| `set_targeted_dev_spend` | {group_id: $/day} | 累積性群組品質加成(5×效率) |
| `set_ads_strength` | global=0-1, by_group{}, by_customer{} | 品質懲罰↔營收回報 |
| `set_lead_promotion` | global, by_group, by_channel, by_channel_group | 新客折扣(疊加) |
| `set_promotion` | global, by_group, by_customer, by_group_plan | 既有客折扣(下計費週期生效) |

### 企業銷售 (2)
| `send_enterprise_deal` | [{customer_id, plan, price, seat_count, contract_months}] | 多輪議價 |
| `reject_enterprise_deal` | [{customer_id}] | 拒絕提案 |

### 資訊分析 (7)
| `get_social_posts` | days=N, limit=N | 社群貼文 |
| `get_cost_info` | — | 成本/層級資訊 |
| `get_market_overview` | — | 市場+總經 |
| `get_group_insights` | group_id | 群組洞察(參數估計/網絡/聲譽) |
| `describe_tables` | [table_names] | SQL schema |
| `list_all_tables` | — | 19張表名 |
| `get_tool_documentation` | [tool_names] | 工具文檔 |

### 市場研究 (2)
| `research_market` | — | 發現新群組(20個可發現) |
| `research_group` | group_id, target_level(1-5) | 提升群組資訊層級 |

### R&D (2)
| `start_research_project` | tier=int(1-20) | 1-10標準/11-20前沿 |
| `list_research_projects` | — | 專案清單 |

### 社群媒體 (1)
| `post_social_media` | content≤280, reply_to_post_id | 發文/回覆→影響獲客 |

### 執行腳本 (7)
| `python_exec` | code | 執行Python+SQL |
| `register/remove/list_daily_calculation` | name, code | 每日自動計算 |
| `register/run/list/delete_script` | name | 命名腳本系統 |

### 模擬控制 (1)
| `next_week` | rationale, dashboard_notes, 4組預測值 | 前進一週+7/28/84/182d現金預測 |

## 👥 26 客戶群

| 群 | 類型 | 描述 | 初始? |
|:---|:-----|:------|:-----|
| S1 | 個人 | 價格敏感自由業 | ✅ |
| S2 | 個人 | 品質導向專業人士 | ✅ |
| S3 | 個人 | 重度使用者 | ✅ |
| E1 | 企業 | 成本削減型 | ✅ |
| E2 | 企業 | 品質優先型 | ✅ |
| E3 | 企業 | 戰略夥伴型 | ✅ |
| D_S01-10 | 個人 | 創作者/學者/非營利/小代理/獨立開發/自由作家/資料分析/社群經理/UX/音樂 | 🔍 |
| D_E01-10 | 企業 | 政府/教育/醫療/銀行/保險/營造/電信/能源/房地產/航運 | 🔍 |

## 📡 5 廣告渠道

每渠道對每群組有 hidden `leads_per_1000_dollars` 參數：
- **social_media** → S群最強(E群接近0)
- **search_ads** → S2/S3強
- **linkedin** → E群唯一有效管道
- **content_marketing** → S2/S1強, E2/E3略有效
- **referral_program** → 最便宜,S1/S3強

## ⚙️ 世界機制核心公式

**每日現金變化**: 訂閱收入 + 廣告收入 − 容量 − 算力 − 運維 − 研發 − 廣告 − 線索獲取 − 市場研究 − 群組研究 − 研發專案

**新客戶期望**: 聲譽 × 市場飽和 × 日曆週期 × 總經週期 × 社群反應 × 需求暴衝 × (各渠道線索 + 網絡效應)

**客戶感知品質**: model_tier乘數 × (初始+全域dev+群組dev) − 超載懲罰 − 停機懲罰 + 關係加成 + 黏性加成 − 未解決問題懲罰 − 配額飽和懲罰 − 廣告強度懲罰

**Stochastic 機制**:
- Normal → R&D品質增益
- Poisson → 每日獲客
- Bernoulli → 非自願退訂
- Uniform → 聲譽損害噪音
- Log-normal → 競爭者品質跳升

## 🏆 Rule-Based 最佳策略 (終值 $15.76M)

固定價格(~$9.99/$29.99/$99.99) + 固定廣告(~$20K/週/2-3群組) + 固定研發(~$30K/週) + 運維~15%營收 + 中階容量 → 不做市場研究/不調價/不升model tier/不用promotion/不enterprise協商。Claude Opus 4.8 和 GPT-5.5 勉強超過它。

---

*Source: github.com/zlab-princeton/ceobench-src/tree/main*
