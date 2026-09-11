(ns resale.facts
  "R0 source-basis catalog — the ONLY provenance/verification classes the
  ResaleGovernor will accept for a brand claim, authentication verdict, or
  stolen-goods-clearance decision (mirrors `cloud-itonami-isic-6311`'s
  `marketdata.facts` discipline: honesty over coverage). Two kinds of entry:

    1. A real, public, free, official reference source — genuinely citable
       today (USPTO trademark search, for verifying a claimed brand is a
       real registered mark).
    2. Structural licensed classes — this actor does not (and cannot) hold
       a free public authentication lab or a free public stolen-property
       database. Same boundary as `cloud-itonami-isic-6311`'s
       `:licensed-operator-feed` / `kotoba-lang/securities`'
       operator-supplies-own-feed philosophy: an operator registers a real
       `verification-license` (an authentication service contract or a
       local law-enforcement secondhand-dealer reporting integration)
       before ANY authentication or stolen-goods-clearance decision can
       cite it.

  Adding coverage means adding a real, citable catalog entry (kind 1) or a
  real registered verification-license (kind 2) — never fabricating
  either.")

(def catalog
  "Each entry: {:id :name :class :access :url}. `:class` is the value that
  must appear in a proposal's `:source :class` for the source-provenance
  gate to accept it as grounded (for the two `:licensed-*` classes,
  grounding also requires an ACTIVE `verification-license`, checked
  separately — this catalog only proves the CLASS itself is real)."
  [{:id :uspto-tess
    :name "USPTO Trademark Electronic Search System (TESS)"
    :class :public-trademark-registry
    :access :public-website
    :url "https://tmsearch.uspto.gov"}
   {:id :licensed-authentication-service
    :name "Operator-registered licensed authentication service (e.g. a third-party physical/digital authenticator contract)"
    :class :licensed-authentication-service
    :access :operator-licensed
    :url nil}
   {:id :licensed-secondhand-reporting-feed
    :name "Operator-registered licensed local law-enforcement secondhand-dealer reporting integration"
    :class :licensed-secondhand-reporting-feed
    :access :operator-licensed
    :url nil}])

(def allowed-source-classes
  "The set of `:source :class` values the source-provenance gate will accept
  anywhere. A closed set — a class not in `catalog` (e.g. :seller-assertion,
  :inference, :social-media) must be rejected, not silently accepted
  because it looks like a keyword."
  (into #{} (map :class catalog)))

(def high-value-categories
  "Categories where a stolen-goods/authenticity risk is elevated enough to
  require KYC-verified sellers and (for branded items) an authentication
  verdict before listing — this is the actor's honest, narrow R0 scope, not
  a claim to cover every secondhand-goods category. Mirrors real US
  secondhand-dealer statutes (California Business and Professions Code
  §21625 et seq.; New York General Business Law Article 5 §§60-70) which
  single out jewelry, precious metals, and serialized electronics for
  mandatory ID/reporting."
  #{:jewelry :precious-metals :serialized-electronics :luxury-handbags :watches})

(def authentication-required-categories
  "Categories where a `:counterfeit`-capable brand claim requires a real
  authentication verdict before sale — narrower than `high-value-
  categories` (a category can require KYC without requiring brand
  authentication, e.g. bulk unbranded jewelry by weight)."
  #{:luxury-handbags :watches})

(defn coverage
  "Honest, machine-checkable report of what R0 actually covers — never
  overstate ('全カテゴリの真贋判定' in prose, 1 free public trademark
  registry + 2 structural licensed classes in fact)."
  []
  {:source-count (count catalog)
   :free-public-sources (into #{} (map :id (filter #(= :public-website (:access %)) catalog)))
   :high-value-categories high-value-categories
   :authentication-required-categories authentication-required-categories
   :note (str "R0 scope: 1 free public reference (USPTO TESS, for brand-"
              "registration lookup only) + 2 structural licensed classes "
              "(authentication service, secondhand-dealer reporting feed) "
              "that require a real operator-registered license before use. "
              "Extend only by appending a real, citable catalog entry or a "
              "real registered verification-license — never fabricate "
              "either.")})

(defn class-allowed? [source-class]
  (contains? allowed-source-classes source-class))

(defn licensed-class? [source-class]
  (contains? #{:licensed-authentication-service :licensed-secondhand-reporting-feed} source-class))
