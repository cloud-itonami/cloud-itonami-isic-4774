# Resale Actor Design — ResaleAdvisor-LLM as a contained intelligence node

eBay-secondhand / Vestiaire Collective / GameStop trade-in 級の secondhand-
resale marケットプレイスを、OSS の actor として自前運用するための設計。
`cloud-itonami-isic-6311`(MarketData-LLM を MarketDataGovernor で封じ込め
た構図)を、中古品の出品/真贋判定/販売のドメインへ写像している。

## 1. 前提: なぜ actor 層が要るのか

出品情報の正規化・真贋判定の記録・開示列の提案は LLM で加速できる。
しかし LLM は次の理由で**取込・真贋確定・販売確定の最終権限を持てない**:

| LLM が起こしうる失敗 | この業態での帰結 |
|---|---|
| seller の適格性(KYC)を確認せず高額品を取込 | 盗品(fenced goods)の流通 |
| 真贋不明のまま販売を許可 | 偽造品(counterfeit)の販売、IP侵害 |
| claimed/verified の条件差を無視 | 出品条件の不実表示 |
| 契約 tier を超えた列を開示 | 過剰開示・契約違反 |

したがって設計課題は「LLM でリセールを回す」ことではなく、**「LLM を
信頼境界の内側に封じ込め、盗品報告義務・真贋出典・条件整合性・開示ライ
センスの層をどう被せるか」**である。

## 2. OperationActor 内部(ResaleAdvisor-LLM ラッパー)

`src/resale/operation.cljk` の langgraph StateGraph として実装。
**1 run = 1 操作** — 有界で監査可能、無限内部ループを持たない。

```
intake → advise → govern → decide ─┬─ commit ───────────────────▶ commit → END
                                   ├─ escalate ─▶ request-approval ┐ [interrupt-before]
                                   │                               │ 承認/却下で resume
                                   │              approved ─▶ commit┘ / rejected ─▶ hold
                                   └─ hold ─────────────────────────────────────▶ hold → END
```

### 2.1 注入される3つの依存(すべて swap)

- **Store**(`resale.store/Store` プロトコル): `MemStore`(既定)/
  `DatomicStore`(`langchain.db`)。両者は同一契約テストで等価性を保証。
- **Advisor**(`resale.llm/Advisor` プロトコル): `mock-advisor`(既定)/
  `llm-advisor`(`langchain.model` の ChatModel)。
- **Phase**(`resale.phase`、context の `:phase 0..3`): 段階導入。
  **`default-phase` は最初から `1`**(fail-open バグを回避、CLAUDE.md 参照)。
  **`:correction/request` はどの phase の `:auto` にも入らない**。

## 3. ResaleGovernor(独立検閲層)

`src/resale/policy.cljk`。

判定の優先順位(上が強い、HARD は人間承認でも上書き不可):

1. **RBAC**
2. **stolen-goods-reporting-gate** — 高額/要注意カテゴリの `:item/intake`
   で seller が KYC 未完了、または過去の盗品報告フラグ付きなら拒否。
3. **source-provenance-gate** — `:item/authenticate` の出典クラスが許可
   リストに無ければ拒否。ライセンスクラスは加えてアクティブなライセンス
   を要求。
4. **counterfeit-flag-gate** — 要認証カテゴリの `:sale/confirm` で
   authentic verdict が無ければ拒否。
5. **condition-misrepresentation-gate** — claimed/verified の条件差が2段
   階以上なら `:sale/confirm` を拒否。
6. **licensed-disclosure** — 有効な契約(tenant×tier)が無い、または列が
   tier を超えたら拒否。
7. 確信度フロア(SOFT)
8. **high-value-holding-period gate**(SOFT) — 高額品の販売は governor
   clean でも常に人間承認。
9. dispute-request(SOFT・無条件)

## 4. SSoT と監査台帳

`src/resale/store.cljk`。entities: `items` `sellers` `authentications`
`verification-licenses` `contracts`。`append-ledger!` が全 commit/reject/
開示を不変台帳に積む。

## 5. デモ(`kbb -M:dev:run`)

`src/resale/sim.cljk` が8操作を actor に通す(§sim.cljc docstring 参照)。

## 6. テスト(`kbb -M:dev:test`)

`test/resale/policy_contract_test.cljk` がガバナンス契約を実行可能にする。
`test/resale/phase_test.cljk` が段階導入を保証。`test/resale/facts_test.cljk`
が出典カタログの正直さを保証。
