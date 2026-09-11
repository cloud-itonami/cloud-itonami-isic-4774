(ns resale.phase
  "Phase 0→3 staged rollout — this actor's analog of robotaxi's ODD phases
  and `cloud-itonami-isic-6311`'s rollout phases: start narrow (read-only),
  widen as trust grows. Where the ResaleGovernor answers 'is this
  allowed?', the phase answers 'how much autonomy does the actor have
  *yet*?'. It can only ever make the actor MORE conservative than the
  governor: it downgrades a governor-clean commit to approval or hold,
  never the reverse.

    Phase 0  read-only          — no writes at all. `:disclosure/query`
                                  only (still governor-gated).
    Phase 1  assisted-intake    — `:item/intake` allowed, every intake
                                  needs human approval.
    Phase 2  + authentication   — adds `:item/authenticate` and
                                  `:correction/request` (still
                                  approval-only).
    Phase 3  supervised auto    — governor-clean, high-confidence
                                  `:item/intake`/`:item/authenticate`/
                                  `:sale/confirm` may auto-commit (a
                                  high-value sale still always escalates
                                  via the governor's own
                                  high-value-holding-period gate,
                                  independent of phase).

  `:correction/request` is deliberately NEVER a member of any phase's
  `:auto` set, at any phase — a dispute always reaches a human.")

(def read-ops  #{:disclosure/query})
(def write-ops #{:item/intake :item/authenticate :sale/confirm :correction/request})

(def phases
  "phase → {:label .. :writes <ops allowed to write> :auto <ops allowed to
  auto-commit when governor-clean>}. `:correction/request` is intentionally
  absent from every phase's `:auto` set."
  {0 {:label "read-only"             :writes #{}
                                      :auto #{}}
   1 {:label "assisted-intake"       :writes #{:item/intake}
                                      :auto #{}}
   2 {:label "assisted-authenticate" :writes #{:item/intake :item/authenticate :correction/request}
                                      :auto #{}}
   3 {:label "supervised-auto"       :writes #{:item/intake :item/authenticate :sale/confirm :correction/request}
                                      :auto #{:item/intake :item/authenticate :sale/confirm}}})

(def default-phase
  "The phase used when `context` carries no :phase at all
  (`resale.operation`: `(:phase context phase/default-phase)`), AND the
  fallback `gate` itself uses for an unrecognized phase NUMBER. This is
  directly reachable by any ordinary caller that simply omits :phase — not
  just malformed/malicious input — so it must be the MOST CONSERVATIVE
  phase, never the most permissive. Matches the fix applied this session to
  `cloud-itonami-isic-6311`'s `marketdata.phase` and `cloud-itonami-isic-
  7820`'s `staffing.phase` (both originally shipped with a fail-open
  default of 3, later corrected to 1) — this actor is implemented with the
  conservative default from the start."
  1)

(defn gate
  "Adjust a governor disposition for the rollout phase. Returns
  {:disposition kw :reason kw|nil}."
  [phase {:keys [op]} governor-disposition]
  (let [{:keys [writes auto]} (get phases phase (get phases default-phase))]
    (cond
      (= :hold governor-disposition)      {:disposition :hold :reason nil}
      (contains? read-ops op)             {:disposition governor-disposition :reason nil}
      (not (contains? writes op))         {:disposition :hold :reason :phase-disabled}
      (and (= :commit governor-disposition)
           (not (contains? auto op)))     {:disposition :escalate :reason :phase-approval}
      :else                               {:disposition governor-disposition :reason nil})))

(defn verdict->disposition
  "Map a ResaleGovernor verdict to a base disposition before the phase
  gate."
  [verdict]
  (cond (:hard? verdict) :hold
        (:escalate? verdict) :escalate
        :else :commit))
