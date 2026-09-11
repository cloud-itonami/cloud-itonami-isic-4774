(ns resale.store
  "SSoT for the secondhand-resale actor, behind a `Store` protocol so the
  backend is a swap, not a rewrite:

    - `MemStore`     — atom of Datomic-shaped EDN. The deterministic default
                       for dev/tests/demo (no deps).
    - `DatomicStore` — backed by `langchain.db`, a Datomic-API-compatible EAV
                       store. Pure `.cljc`, so it runs offline AND can be
                       pointed at a real Datomic Local or a kotoba-server pod
                       by swapping `langchain.db`'s `:db-api`.

  Entity shapes: an item (intake→listed→sold→held), its seller (KYC status
  + prior stolen-goods flag), an authentication verdict (for branded/
  high-value items), a verification-license (provenance for licensed
  authentication/reporting classes), and a subscriber contract. There is NO
  field anywhere for order-routing custody, buyer payment credentials, or
  seller bank details — this actor mediates listing/authentication/sale
  DECISIONS only, never handles money or credentials directly (that is
  explicitly the payment processor's job, out of scope, ADR-2607113000 §1).

  The ledger stays append-only on every backend."
  (:require [kotoba.lang.text :as str]
            [langchain.db :as d]
            [langchain-store.core :as ls]))

(defprotocol Store
  (item [s id])
  (all-items [s])
  (seller [s id])
  (authentication [s item-id])
  (verification-license [s license-id])
  (contract [s tenant])
  (ledger [s])
  (commit-record! [s record] "apply a committed op's record to the SSoT")
  (append-ledger! [s fact]   "append one immutable decision/disclosure fact")
  (with-items [s items]                   "replace/seed items (map id→item)")
  (with-sellers [s sellers]               "replace/seed sellers (map id→seller)")
  (with-authentications [s auths]         "replace/seed authentications (map item-id→auth)")
  (with-verification-licenses [s licenses] "replace/seed verification licenses (map license-id→license)")
  (with-contracts [s contracts]           "replace/seed subscriber contracts (map tenant→contract)"))

;; ───────────────────────── demo data (fictitious, non-real listings) ─────

(defn demo-data
  "A small, entirely fictitious dataset. `it-300` carries a demo
  `:value-tier :high-value` + unverified seller purely to exercise the
  stolen-goods-reporting gate; `it-400` carries a demo counterfeit
  authentication verdict purely to exercise the counterfeit-flag gate."
  []
  {:items
   {"it-100" {:id "it-100" :category :apparel :brand "DemoWear" :condition-claimed :good
              :condition-verified nil :status :intake :seller-id "sl-1" :price 45.00M
              :value-tier :standard}
    "it-200" {:id "it-200" :category :luxury-handbags :brand "DemoLux" :condition-claimed :excellent
              :condition-verified :excellent :status :listed :seller-id "sl-2" :price 1200.00M
              :value-tier :high-value}
    "it-300" {:id "it-300" :category :jewelry :brand nil :condition-claimed :good
              :condition-verified nil :status :intake :seller-id "sl-3" :price 3500.00M
              :value-tier :high-value}
    "it-400" {:id "it-400" :category :watches :brand "DemoTime" :condition-claimed :good
              :condition-verified nil :status :intake :seller-id "sl-2" :price 2200.00M
              :value-tier :high-value}
    "it-600" {:id "it-600" :category :apparel :brand "DemoWear" :condition-claimed :excellent
              :condition-verified :poor :status :listed :seller-id "sl-1" :price 80.00M
              :value-tier :standard}}
   :sellers
   {"sl-1" {:id "sl-1" :kyc-verified? true :reported-stolen-flag? false :jurisdiction :usa-ca}
    "sl-2" {:id "sl-2" :kyc-verified? true :reported-stolen-flag? false :jurisdiction :usa-ny}
    "sl-3" {:id "sl-3" :kyc-verified? false :reported-stolen-flag? false :jurisdiction :usa-ca}
    "sl-4" {:id "sl-4" :kyc-verified? true :reported-stolen-flag? true :jurisdiction :usa-ca}}
   :authentications
   {"it-200" {:item-id "it-200" :verdict :authentic :confidence 0.95
              :source {:class :licensed-authentication-service :ref "lic-demo-auth:it-200" :license-id "lic-demo-auth"}}}
   :verification-licenses
   {"lic-demo-auth" {:license-id "lic-demo-auth" :provider "Demo Authentication Lab (fictitious)"
                      :classes #{:licensed-authentication-service} :active? true}
    "lic-lapsed" {:license-id "lic-lapsed" :provider "Lapsed Demo Authentication Lab (fictitious)"
                   :classes #{:licensed-authentication-service} :active? false}
    "lic-demo-leo" {:license-id "lic-demo-leo" :provider "Demo County Sheriff Secondhand-Dealer Reporting (fictitious)"
                     :classes #{:licensed-secondhand-reporting-feed} :active? true}}
   :contracts
   {"tenant-acme"  {:tenant "tenant-acme" :tier :tier/pro :active? true :purpose :marketplace-integration}
    "tenant-basic" {:tenant "tenant-basic" :tier :tier/basic :active? true :purpose :listing-widget}}})

;; ───────────────────────── MemStore (default) ─────────────────────────

(defrecord MemStore [a]
  Store
  (item [_ id] (get-in @a [:items id]))
  (all-items [_] (sort-by :id (vals (:items @a))))
  (seller [_ id] (get-in @a [:sellers id]))
  (authentication [_ item-id] (get-in @a [:authentications item-id]))
  (verification-license [_ license-id] (get-in @a [:verification-licenses license-id]))
  (contract [_ tenant] (get-in @a [:contracts tenant]))
  (ledger [_] (:ledger @a))
  (commit-record! [s {:keys [effect path value]}]
    (case effect
      :item-upsert           (swap! a update-in [:items (:id value)] merge value)
      :authentication-upsert (swap! a assoc-in [:authentications (:item-id value)] value)
      :sale-confirm          (swap! a update-in [:items (:id value)] merge value)
      :correction-apply       (case (:kind value)
                                 :items           (swap! a update-in [:items (first path)] merge (:patch value))
                                 :authentications (swap! a update-in [:authentications (first path)] merge (:patch value))
                                 nil)
      nil)
    s)
  (append-ledger! [_ fact] (swap! a update :ledger conj fact) fact)
  (with-items [s is]                   (when (seq is) (swap! a assoc :items is)) s)
  (with-sellers [s sls]                (when (seq sls) (swap! a assoc :sellers sls)) s)
  (with-authentications [s auths]      (when (seq auths) (swap! a assoc :authentications auths)) s)
  (with-verification-licenses [s fls]  (when (seq fls) (swap! a assoc :verification-licenses fls)) s)
  (with-contracts [s cts]              (when (seq cts) (swap! a assoc :contracts cts)) s))

(defn seed-db
  "A MemStore seeded with the demo data. The deterministic default."
  []
  (->MemStore (atom (assoc (demo-data) :ledger []))))

;; ───────────────────────── DatomicStore (langchain.db) ─────────────────

(def ^:private schema
  "DataScript/Datomic-style schema: only constraint attrs are declared.
  Map/compound values are stored as EDN strings so `langchain.db` doesn't
  expand them into sub-entities."
  {:item/id                 {:db/unique :db.unique/identity}
   :seller/id               {:db/unique :db.unique/identity}
   :authentication/item-id  {:db/unique :db.unique/identity}
   :verification-license/id {:db/unique :db.unique/identity}
   :contract/tenant         {:db/unique :db.unique/identity}
   :ledger/seq              {:db/unique :db.unique/identity}})

(defn- item->tx [{:keys [id category brand condition-claimed condition-verified status seller-id price value-tier]}]
  (cond-> {:item/id id}
    category           (assoc :item/category category)
    brand              (assoc :item/brand brand)
    condition-claimed  (assoc :item/condition-claimed condition-claimed)
    true               (assoc :item/condition-verified (ls/enc condition-verified))
    status             (assoc :item/status status)
    seller-id          (assoc :item/seller-id seller-id)
    price              (assoc :item/price (ls/enc price))
    value-tier         (assoc :item/value-tier value-tier)))

(defn- pull->item [m]
  (when (:item/id m)
    {:id (:item/id m) :category (:item/category m) :brand (:item/brand m)
     :condition-claimed (:item/condition-claimed m) :condition-verified (ls/dec* (:item/condition-verified m))
     :status (:item/status m) :seller-id (:item/seller-id m) :price (ls/dec* (:item/price m))
     :value-tier (:item/value-tier m)}))

(def ^:private item-pull
  [:item/id :item/category :item/brand :item/condition-claimed :item/condition-verified
   :item/status :item/seller-id :item/price :item/value-tier])

(defn- seller->tx [{:keys [id kyc-verified? reported-stolen-flag? jurisdiction]}]
  {:seller/id id :seller/kyc-verified kyc-verified? :seller/reported-stolen-flag reported-stolen-flag?
   :seller/jurisdiction jurisdiction})

(defn- pull->seller [m]
  (when (:seller/id m)
    {:id (:seller/id m) :kyc-verified? (:seller/kyc-verified m)
     :reported-stolen-flag? (:seller/reported-stolen-flag m) :jurisdiction (:seller/jurisdiction m)}))

(def ^:private seller-pull
  [:seller/id :seller/kyc-verified :seller/reported-stolen-flag :seller/jurisdiction])

(defn- authentication->tx [{:keys [item-id verdict confidence source]}]
  {:authentication/item-id item-id :authentication/verdict verdict
   :authentication/confidence (ls/enc confidence) :authentication/source (ls/enc source)})

(defn- pull->authentication [m]
  (when (:authentication/item-id m)
    {:item-id (:authentication/item-id m) :verdict (:authentication/verdict m)
     :confidence (ls/dec* (:authentication/confidence m)) :source (ls/dec* (:authentication/source m))}))

(def ^:private authentication-pull
  [:authentication/item-id :authentication/verdict :authentication/confidence :authentication/source])

(defn- verification-license->tx [{:keys [license-id provider classes active?]}]
  {:verification-license/id license-id :verification-license/provider provider
   :verification-license/classes (ls/enc classes) :verification-license/active active?})

(defn- pull->verification-license [m]
  (when (:verification-license/id m)
    {:license-id (:verification-license/id m) :provider (:verification-license/provider m)
     :classes (ls/dec* (:verification-license/classes m)) :active? (:verification-license/active m)}))

(def ^:private verification-license-pull
  [:verification-license/id :verification-license/provider :verification-license/classes :verification-license/active])

(defn- contract->tx [{:keys [tenant tier active? purpose]}]
  {:contract/tenant tenant :contract/tier tier :contract/active active? :contract/purpose purpose})

(defn- pull->contract [m]
  (when (:contract/tenant m)
    {:tenant (:contract/tenant m) :tier (:contract/tier m)
     :active? (:contract/active m) :purpose (:contract/purpose m)}))

(def ^:private contract-pull
  [:contract/tenant :contract/tier :contract/active :contract/purpose])

(defrecord DatomicStore [conn]
  Store
  (item [_ id] (pull->item (d/pull (d/db conn) item-pull [:item/id id])))
  (all-items [_]
    (->> (d/q '[:find [?id ...] :where [?e :item/id ?id]] (d/db conn))
         (map #(pull->item (d/pull (d/db conn) item-pull [:item/id %])))
         (sort-by :id)))
  (seller [_ id] (pull->seller (d/pull (d/db conn) seller-pull [:seller/id id])))
  (authentication [_ item-id]
    (pull->authentication (d/pull (d/db conn) authentication-pull [:authentication/item-id item-id])))
  (verification-license [_ license-id]
    (pull->verification-license (d/pull (d/db conn) verification-license-pull [:verification-license/id license-id])))
  (contract [_ tenant] (pull->contract (d/pull (d/db conn) contract-pull [:contract/tenant tenant])))
  (ledger [_]
    (->> (d/q '[:find ?s ?f :where [?e :ledger/seq ?s] [?e :ledger/fact ?f]] (d/db conn))
         (sort-by first)
         (mapv (comp ls/dec* second))))
  (commit-record! [s {:keys [effect path value]}]
    (case effect
      :item-upsert
      (d/transact! conn [(item->tx (merge (item s (:id value)) value))])
      :authentication-upsert
      (d/transact! conn [(authentication->tx value)])
      :sale-confirm
      (d/transact! conn [(item->tx (merge (item s (:id value)) value))])
      :correction-apply
      (case (:kind value)
        :items           (d/transact! conn [(item->tx (merge (item s (first path)) (:patch value)))])
        :authentications (d/transact! conn [(authentication->tx (merge (authentication s (first path)) (:patch value)))])
        nil)
      nil)
    s)
  (append-ledger! [s fact]
    (d/transact! conn [{:ledger/seq (count (ledger s)) :ledger/fact (ls/enc fact)}])
    fact)
  (with-items [s is]
    (when (seq is) (d/transact! conn (mapv item->tx (vals is)))) s)
  (with-sellers [s sls]
    (when (seq sls) (d/transact! conn (mapv seller->tx (vals sls)))) s)
  (with-authentications [s auths]
    (when (seq auths) (d/transact! conn (mapv authentication->tx (vals auths)))) s)
  (with-verification-licenses [s fls]
    (when (seq fls) (d/transact! conn (mapv verification-license->tx (vals fls)))) s)
  (with-contracts [s cts]
    (when (seq cts) (d/transact! conn (mapv contract->tx (vals cts)))) s))

(defn datomic-store
  "A DatomicStore (langchain.db backend) seeded from `data`; empty when
  omitted."
  ([] (datomic-store {}))
  ([{:keys [items sellers authentications verification-licenses contracts]}]
   (let [s (->DatomicStore (d/create-conn schema))]
     (-> s (with-items items) (with-sellers sellers)
         (with-authentications authentications) (with-verification-licenses verification-licenses)
         (with-contracts contracts)))))

(defn datomic-seed-db
  "A DatomicStore seeded with the demo data — the Datomic-backed analog of
  `seed-db`, used to prove protocol parity."
  []
  (datomic-store (demo-data)))

;; ───────────────────────── ledger formatting ─────────────────────────

(defn ledger-line
  "Human-readable one-liner for a ledger fact (used by the demo)."
  [{:keys [op actor subject disposition basis]}]
  (str/join " · "
            [(name disposition)
             (str "op=" op)
             (str "actor=" actor)
             (str "subject=" subject)
             (str "basis=" (pr-str basis))]))
