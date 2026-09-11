(ns resale.llm
  "ResaleAdvisor-LLM client — the *contained intelligence node*.

  It normalizes seller-submitted item intake, drafts authentication
  verdicts (from an authenticator's structured input), proposes sale
  confirmations, proposes subscriber disclosure column sets, and drafts
  dispute resolutions. CRITICAL: it is a smart-but-untrusted advisor. It
  returns a *proposal*, never a committed or disclosed record. Every output
  is censored downstream by `resale.policy` (the ResaleGovernor) before
  anything touches the SSoT or is disclosed.

  Like `cloud-itonami-isic-6311`'s MarketData-LLM, this is a deterministic
  mock so the actor graph runs offline and the governor contract is
  exercised end-to-end. In production this calls a real LLM (kotoba-llm)
  with the same proposal shape.

  Proposal shape (all kinds):
    {:summary    str
     :rationale  str
     :cites      [kw|str ..]
     :source     {:class kw :ref str :license-id str?}|nil
     :effect     kw
     :value      map|nil
     :columns    [kw ..]|nil
     :confidence 0..1}"
  (:require #?(:clj  [clojure.edn :as edn]
               :cljs [cljs.reader :as edn])
            [kotoba.lang.text :as str]
            [langchain.model :as model]
            [resale.store :as store]))

(defn- propose-intake
  "Item intake — the LLM only normalizes the seller-submitted patch (adds
  no new facts about the seller's eligibility; that is the governor's
  stolen-goods-reporting-gate's job, evaluated against the store's own
  seller record)."
  [_db {:keys [item-id category brand condition-claimed seller-id price]}]
  {:summary   (str "item intake: " item-id " (" (name category) ")")
   :rationale "seller提出情報の正規化のみ。seller の適格性判定は行わない。"
   :cites     [:item-id :category :condition-claimed :seller-id :price]
   :source    nil
   :effect    :item-upsert
   :value     {:id item-id :category category :brand brand :condition-claimed condition-claimed
               :condition-verified nil :status :intake :seller-id seller-id :price price
               :value-tier :standard}
   :confidence 0.95})

(defn- propose-authenticate
  "Authentication verdict draft. `:unsourced?` injects the failure mode we
  must defend against: a verdict arriving with no source citation at all —
  the ResaleGovernor's source-provenance-gate must reject this outright,
  regardless of how confident the LLM is."
  [_db {:keys [item-id verdict confidence source condition-verified unsourced?]}]
  (let [src (when-not unsourced? source)]
    {:summary   (str "authentication: " item-id " → " (name verdict))
     :rationale "出典引用済みの認証結果の記録のみ。"
     :cites     [:item-id :verdict]
     :source    src
     :effect    :authentication-upsert
     :value     {:item-id item-id :verdict verdict :confidence confidence :source src}
     :confidence (or confidence 0.9)
     :condition-verified condition-verified}))

(defn- propose-sale
  "Sale confirmation draft. The LLM does not evaluate authenticity or
  condition risk itself — those are read directly from the store by
  `resale.policy`'s counterfeit-flag-gate / condition-misrepresentation-gate,
  independent of whatever the LLM proposes here."
  [db {:keys [item-id buyer-id]}]
  (let [it (store/item db item-id)]
    {:summary   (str "sale confirm: " item-id " → buyer " buyer-id)
     :rationale "既存 item レコードに基づく確定のみ。"
     :cites     [:item-id]
     :source    nil
     :effect    :sale-confirm
     :value     {:id item-id :status :sold :buyer-id buyer-id
                 :category (:category it) :condition-claimed (:condition-claimed it)
                 :condition-verified (:condition-verified it) :value-tier (:value-tier it)}
     :confidence 0.95}))

(defn- propose-disclosure
  "Disclosure column-set proposal for a licensed subscriber query.
  `:greedy?` injects over-disclosure (pulls seller-id/raw-source columns
  beyond a basic-tier contract) — the ResaleGovernor's licensed-disclosure
  gate must reject the excess columns."
  [_db {:keys [item-id greedy?]}]
  (let [base [:id :category :brand :condition-claimed :status :price]
        greedy-extra [:seller-id :raw-source]]
    {:summary   (str "開示列提案: " item-id)
     :rationale (if greedy? "分析に有用そうな列を広めに含めた。" "契約 tier に必要な最小列のみ。")
     :cites     base
     :source    nil
     :effect    :disclosure-serve
     :columns   (if greedy? (into base greedy-extra) base)
     :confidence 0.9}))

(defn- propose-correction
  "Dispute resolution draft. This NEVER auto-applies — `resale.policy` and
  `resale.phase` both structurally force every `:correction/request` to
  human review, independent of confidence."
  [_db {:keys [disputed-field claim]}]
  {:summary   (str "item の " disputed-field " について紛争解決案ドラフト")
   :rationale (str "申立て内容: " claim "。裏取りは人間レビューで行う。")
   :cites     [disputed-field]
   :source    nil
   :effect    :correction-apply
   :value     {:kind :items :patch {disputed-field claim}}
   :confidence 0.5})

(defn infer
  "Route a request to the right proposal generator.
  request: {:op kw :subject id ...op-specific...}"
  [db {:keys [op] :as request}]
  (case op
    :item/intake         (propose-intake db request)
    :item/authenticate   (propose-authenticate db request)
    :sale/confirm        (propose-sale db request)
    :disclosure/query    (propose-disclosure db request)
    :correction/request  (propose-correction db request)
    {:summary "未対応の操作" :rationale (str op) :cites [] :source nil
     :effect :noop :confidence 0.0}))

;; ───────────────────────── Advisor protocol ─────────────────────────

(defprotocol Advisor
  (-advise [advisor store request] "store + request → proposal map"))

(defn mock-advisor
  "The deterministic advisor (the `infer` logic above). Default everywhere."
  [] (reify Advisor (-advise [_ st req] (infer st req))))

(def ^:private system-prompt
  (str "あなたは中古品リセールマーケットプレイスの出品/真贋/販売アドバイザー"
       "です。与えられた事実のみに基づき、提案を1つだけ EDN マップで返します。"
       "説明や前置きは一切書かず、EDN だけを出力します。\n"
       "キー: :summary(人向けドラフト) :rationale(根拠) :cites(使った事実キー) "
       ":source({:class .. :ref .. :license-id? ..}か nil) "
       ":effect(:item-upsert|:authentication-upsert|:sale-confirm|"
       ":disclosure-serve|:correction-apply) :value(該当マップ) :confidence(0..1)。\n"
       "重要: 出典を伴わない認証結果は絶対に提案してはいけません。"
       "盗品報告義務の判定・真贋の最終可否・条件不一致の可否はあなたの責務では"
       "ありません(governor が判定します)。"))

(defn- facts-for [st {:keys [op subject item-id]}]
  (case op
    :sale/confirm {:item (store/item st (or item-id subject))
                   :authentication (store/authentication st (or item-id subject))}
    {:item (store/item st (or item-id subject))}))

(defn- parse-proposal
  "Parse the model's EDN proposal defensively. Any parse/shape failure yields
  a safe low-confidence noop so the ResaleGovernor escalates/holds."
  [content]
  (let [p (try (edn/read-string (str/trim (str content)))
               (catch #?(:clj Exception :cljs :default) _ nil))]
    (if (map? p)
      (-> p
          (update :cites #(vec (or % [])))
          (update :confidence #(if (number? %) (double %) 0.0))
          (update :effect #(or % :noop)))
      {:summary "LLM応答を解釈できませんでした" :rationale (str content)
       :cites [] :source nil :effect :noop :confidence 0.0})))

(defn llm-advisor
  "An advisor backed by a `langchain.model/ChatModel` (real inference)."
  ([chat-model] (llm-advisor chat-model {}))
  ([chat-model gen-opts]
   (reify Advisor
     (-advise [_ st req]
       (let [msgs [{:role :system :content system-prompt}
                   {:role :user :content (str "操作: " (:op req)
                                              "\n対象: " (:subject req)
                                              "\n事実: " (pr-str (facts-for st req)))}]
             resp (model/-generate chat-model msgs gen-opts)]
         (parse-proposal (:content resp)))))))

(defn trace
  "Decision-grounded audit record."
  [request proposal]
  {:t          :resalellm-proposal
   :op         (:op request)
   :subject    (:subject request)
   :summary    (:summary proposal)
   :rationale  (:rationale proposal)
   :cites      (:cites proposal)
   :source     (:source proposal)
   :confidence (:confidence proposal)})
