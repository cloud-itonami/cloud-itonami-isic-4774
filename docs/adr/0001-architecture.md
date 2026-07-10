# ADR-0001: cloud-itonami-isic-4774 — ResaleAdvisor-LLM を封じ込めた知能ノードとする secondhand-resale actor 設計

- Status: Accepted (2026-07-10)
- 関連: `cloud-itonami-isic-6311`(MarketData-LLM を MarketDataGovernor で
  封じ込める構図の直接の手本)、`cloud-itonami-isic-8291`(収集・保持・契約者
  限定開示パターン)、langgraph ADR-0001(Pregel superstep + interrupt +
  Datomic checkpoint)
- 文脈: com-junkawasaki/root superproject ADR(本 ADR の対)

## 課題

`kotoba-lang/industry` registry の未着手 `:spec` スロットから ISIC Rev.4
4774「Retail sale of second-hand goods」を選定した。secondhand-resale は
既存の実装済み actor とは異なる、業態固有の2つのリスク面を持つ:
**fenced goods(盗品)の intake** と **counterfeit(偽造品)の販売**。両方
とも、確信度に依存しない構造的なチェックを要する(高確信のまま盗品/偽造品
が流通してしまうことこそが実害)。

## 決定

### 1. ResaleAdvisor-LLM は最下層の1ノードに封じ込め、直接取込/認証確定/
   販売確定/開示させない

**単一の不変条件**:

> **ResaleAdvisor-LLM は、ResaleGovernor が拒否する item の取込・認証・
> 販売確定・紛争解決を決して行わない。**

### 2. ResaleGovernor は6 HARD + 3 SOFT

stolen-goods-reporting-gate と counterfeit-flag-gate は、この業態に固有
かつ他の cloud-itonami actor に存在しないチェックである。米国の secondhand
dealer 規制(California Business and Professions Code §21625 et seq.;
New York General Business Law Article 5 §§60-70)が、高額品/宝飾品等の
intake に seller ID 確認・保有期間・警察報告を義務付けていることの構造的
反映。

### 3. R0 の正直なスコープ(捏造禁止)

出典カタログ(`src/resale/facts.cljc`)は実在する1つの自由・公式参照ソース
(USPTO TESS、商標登録確認のみ)+ 2つの構造的ライセンスクラス(認証サービス・
盗品報告フィード)。無料の公式真贋判定ソースは存在しないため、これを偽装
しない。

### 4. Robotics premise: false

出品/真贋判定/販売確定は書面/システム上の意思決定であり、実際の配送・
保管は actor の境界の外にある。

## Consequences

- (+) `kotoba-lang/industry` registry の 4774 スロットが実装へ昇格。
- (+) stolen-goods-reporting-gate・counterfeit-flag-gate という、他の
  cloud-itonami actor に存在しない secondhand-resale 固有の HARD チェック
  を新設した。
- (-) R0 の自由公式ソースは1種のみ(商標登録確認)。真贋判定・盗品報告は
  operator の licensed integration が必須。
- (-) Datomic/kotoba-server backend は次のシーム(未接続)。

## 代替案と不採用理由

- **stolen-goods-reporting-gate を SOFT にとどめる**: 盗品の流通は法的
  報告義務違反であり、人間承認で事後的に許容できる性質のものではない。
  HARD が必須と判断した。
- **counterfeit-flag-gate を confidence floor と統合**: 偽造品販売は
  IP法違反であり、確信度とは独立したカテゴリカルな禁止事項である。
  混同すると高確信の誤判定が偽造品販売を許してしまう。
