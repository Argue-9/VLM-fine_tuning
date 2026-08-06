# 风险内容分类与 Evidence 对应关系

---

## 一、gambling

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `gamble_lottery` | 网络博彩/私彩/非法彩票投注/报码杀号/图谜预测 | `online_lottery_activity` |
| `gamble_casino` | 线上赌场/博彩平台（新葡京、WG娱乐等品牌平台，含真人视讯、老虎机、充值提现） | `online_gambling_platform` / `online_casino_or_live_dealer` |
| `gamble_sports` | 体育赛事博彩（球队/比分/赛程/盘口） | `sports_content` |
| `gamble_mark_six` | 港澳六合彩（香港彩/澳门彩，特码/生肖/杀号/玄机图） | `mark_six_lottery` |
| `gamble_chess_card` | 棋牌游戏（线上线下棋牌室、棋牌App，牛牛/斗地主/百家乐图标） | `chess_card_game` / `fishing_game` / `electronic_game` |
| `gamble_ad` | 博彩广告/引流（视频/动漫/直播中插播的博彩弹窗、横幅广告） | `gambling_ad_in_video` |
| `gamble_entry` | 线路入口/线路检测/安全检测/倒计时读秒/备用网址 | `route_indicator_text` |
| `gamble_other` | 以上都不匹配 | — |

### 边界说明

- `gamble_casino` 和 `gamble_sports` / `gamble_chess_card` 的区别：casino 有平台品牌 + 促销 + 充值提现组合，是完整赌场入口；sports/chess_card 是内容类型。同一页面同时有 platform + chess_card 时，evidence 都保留。
- `gamble_ad` 和 `gamble_entry` 的区别：ad 是内容页里插播的广告（页面主体是视频/动漫），entry 是页面本身就是博彩网站的入口/检测页。

---

## 二、finance

finance 下的 `risk_type` 本身就是小类，不需要额外拆分：

| 小类（=risk_type） | 场景描述 | 对应 evidence |
|---|---|---|
| `crypto` | 虚拟币交易/USDT/BTC/交易所/钱包/充币提币 | `virtual_currency_trading`, `primary_language_chinese`, `language_switch_entry` |
| `investment` | 投资理财/股票/基金/证券开户/K线行情/入金出金 | `customer_service_entry`, `investment_task_entry` |
| `rebate` | 刷单返佣/做任务/抢单/佣金提现/垫付充值/保证金 | `advance_payment_or_recharge`, `task_or_order_rebate`, `commission_or_cashout`, `agent_recruitment`, `supply_or_substation_entry`, `registered_company_customer` |
| `digital_goods` | 账号/卡密/会员/兑换码/虚拟商品/供货/分销/代理后台 | `virtual_goods_or_account`, `virtual_goods_trading`, `order_query_entry`, `vpn_or_proxy_service`, `agent_recruitment`, `supply_or_substation_entry` |
| `mall` | 商品/店铺/订单/异常商城交易/虚假销量/诱导下单 | `mall_brand`, `transaction_entry`, `mall_transaction`, `exact_regulatory_match`, `high_risk_login_location` |
| `payment` | 支付/转账/充值/提现/余额/红包/收款/退款/异常资金操作 | `payment_or_fund_entry`, `suspected_unsafe_payment` |

---

## 三、vpn / game

各自只对应一个违规场景，自身即小类：

| 小类（=risk_type） | 场景描述 | 对应 evidence |
|---|---|---|
| `vpn` | 翻墙/绕过网络审查/科学上网/节点/机场 | `vpn_circumvention_purpose`, `vpn_proxy_usage_state`, `vpn_proxy_service_provider`, `legitimate_vpn_service` |
| `game` | 游戏私服/外挂/作弊工具/传奇游戏 | `private_server_text`, `cheat_tool`, `legend_game_keyword`, `suspicious_game_domain`, `game_content` |

---

## 四、porn

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `porn_crime` | 性犯罪意图（迷药/听话水/催情药）+ 未成年色情 | `minor_sexual_content`, `sexual_assault_or_crime_intention` |
| `porn_adult_app` | 成人App/品牌/图标/名称，含色情黑话暗语（蜜穴、抖阴、茄子视频等） | `adult_app_or_brand` |
| `porn_nudity` | 直接裸露（乳头/生殖器可见） | `female_nipple_visible`, `genital_visible` |
| `porn_sex_service` | 卖淫招嫖/约炮裸聊/同城服务/上门服务 | `commercial_sex_service_text` |
| `porn_explicit_act` | 明确性行为（视觉动作或文字描述） | `explicit_sexual_act`, `explicit_sexual_act_text` |
| `porn_suggestive` | 擦边/性感/暧昧姿势，无露点。单独不构成 porn，FP 重点区 | `non_explicit_suggestive_visual` |
| `porn_other` | 以上都不匹配 | — |

---

## 五、politic

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `politic_leader` | 中国领导人负面信息/高级红低级黑/省级以上干部负面 | `leader_negative_info`, `senior_cadre_negative_info` |
| `politic_government` | 涉华政治敏感/政府机构/党代会/政策歪曲/反华宣传 | `china_political_sensitive`, `government_or_public_service` |
| `politic_military` | 军警敏感/军事机密/对军队警察攻击丑化 | `military_or_police_sensitive` |
| `politic_separatism` | 分裂主义（新疆独立/西藏独立/东突/台湾独立） | `separatism_symbol`, `separatist_movement` |
| `politic_history` | 敏感历史事件（六四/64/720反压迫日） | `sensitive_historical_event` |
| `politic_terrorism` | 暴恐/极端主义（恐怖组织标识/暴恐宣传） | `terrorism_or_extremism` |
| `politic_other` | 以上都不匹配 | — |

---

## 六、fake

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `fake_brand` | 仿冒知名品牌（澳门新葡京、金年会等） | `suspected_brand_impersonation` + `known_entity_branding` |
| `fake_official` | 仿冒政府/机构/官方实体（反诈中心、教育部门等） | `suspected_brand_impersonation` + `official_entity_text_claim` |
| `fake_apple` | 仿冒苹果/App Store/iOS描述文件安装/证书信任 | `app_download_or_install_entry` + `official_app_store_or_apple_claim` + `ios_profile_or_trust_instruction` |
| `fake_financial_lure` | 仿冒官方 + 利益诱导（政府发钱、教育补贴、免费领红包等） | `suspected_brand_impersonation` + (`known_entity_branding` OR `official_entity_text_claim`) + `reward_benefit_fraud_text` |

---

## 七、other_fraud

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `fraud_multi_weak` | 多弱信号组合（2个以上弱可疑信号同时出现） | `abnormal_contact_or_redirect` + `suspicious_service_or_claim` / `suspicious_fraud_content` / `high_risk_private_transaction` / `unknown_paid_service_or_unlock` |
| `fraud_pure_cs` | 纯客服聊天界面（只有聊天窗口/客服头像/消息输入框，无业务内容） | `pure_customer_service_interface` |
| `fraud_single_weak` | 单一弱信号（不构成 other_fraud，降级为 good） | 任一弱信号单独出现 |
| `fraud_qr_only` | 仅含二维码（需扫码后才能判断内容） | `qr_code_only` |
| `fraud_other` | 以上都不匹配 | — |

---

## 八、login

login 本身不构成违规，但需验证 VLM 表现：

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `login_normal` | 普通登录/注册页（有品牌标识或公开注册入口） | `public_registration_entry` / `known_entity_branding` |
| `login_invitation` | 邀请码/上级码/推荐码准入（可疑准入场景） | `invitation_code_required` |
| `login_customer_service` | 客服协助登录（无品牌佐证） | `customer_service_entry` |
| `login_pure_framework` | 纯登录框架（如 webmail，只含账户/密码/验证码） | `pure_login_framework` |
| `login_route_switch` | 登录页含切换线路文字（诈骗平台多入口策略） | `route_switch_text` |

---

## 九、good

good 的类型无法枚举，只区分有无风险信号，目的是识别潜在 FN：

| 小类 | 场景描述 | 对应 evidence |
|---|---|---|
| `good_clean` | 无任何违规证据，纯安全页面 | `no_risk_evidence` |
| `good_risk_signal` | evidence 含弱风险信号但根据字段边界不构成违规，潜在 FN 需核查 | `government_or_public_service` / `video_platform_sexual_title` / `official_public_safety_content` / `known_entity_branding` / `payment_or_fund_entry` / `mall_brand` / `national_lottery` / `genital_visible` / `sex_toy` / `suspected_brand_impersonation` / `pure_login_framework` / `pure_customer_service_interface` / `qr_code_only` / `cash_transaction_evidence` / `route_indicator_text` 等 |

---

## 十、小类汇总表

| advice | 小类数 | 小类列表 |
|---|---:|---|
| `gamble` | 8 | `gamble_lottery` / `gamble_casino` / `gamble_sports` / `gamble_mark_six` / `gamble_chess_card` / `gamble_ad` / `gamble_entry` / `gamble_other` |
| `porn` | 7 | `porn_crime` / `porn_adult_app` / `porn_nudity` / `porn_sex_service` / `porn_explicit_act` / `porn_suggestive` / `porn_other` |
| `politic` | 7 | `politic_leader` / `politic_government` / `politic_military` / `politic_separatism` / `politic_history` / `politic_terrorism` / `politic_other` |
| `fake` | 4 | `fake_brand` / `fake_official` / `fake_apple` / `fake_financial_lure` |
| `other_fraud` | 5 | `fraud_multi_weak` / `fraud_pure_cs` / `fraud_single_weak` / `fraud_qr_only` / `fraud_other` |
| `finance` | 6 | `crypto` / `investment` / `rebate` / `digital_goods` / `mall` / `payment` |
| `vpn` | 1 | `vpn` |
| `game` | 1 | `game` |
| `login` | 5 | `login_normal` / `login_invitation` / `login_customer_service` / `login_pure_framework` / `login_route_switch` |
| `good` | 2 | `good_clean` / `good_risk_signal` |
