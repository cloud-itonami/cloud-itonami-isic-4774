(ns resale.report
  "Disclosure rendering — output as a GOVERNED read. The column set is not
  chosen here; it is whatever the ResaleGovernor's licensed-disclosure gate
  approved for the caller's contract tier."
  (:require [resale.store :as store]))

(defn render-item
  "Render one item's listing over exactly `columns` (already governor-
  approved)."
  [db item-id columns]
  (let [it (store/item db item-id)
        auth (store/authentication db item-id)
        cell (fn [col]
               (case col
                 :authentication auth
                 :raw-source     (:source auth)
                 (get it col)))]
    (into {} (map (juxt identity cell)) columns)))
