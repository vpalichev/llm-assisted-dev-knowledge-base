
  ---                                                                                                                                                                                                                                                                                                                    
  
  Run all 30 one after another. Mix of fast, slow, large, errors, exceptions, edge cases. Watch the health bar fill up.
                                                                                                                                                                                                                                                                                                                       
  (+ 1 1)
  (/ 1 0)
  (throw (ex-info "boom" {:x 1}))
  (+ 2 2)
  (Integer/parseInt "not-a-number")
  (apply str (repeat 20000 "z"))
  (vec (range 500))
  (/ 1 0)
  (do (println "out1") (println "out2") 42)
  (throw (ex-info "nested" {:inner (ex-info "deep" {:level 3})}))
  (frequencies (repeatedly 10000 #(rand-int 50)))
  (mapv #(hash-map :i % :sq (* % %) :s (str %)) (range 300))
  (Long/MAX_VALUE)
  (NullPointerException. "fake npe")
  (reduce + (range 100000))
  (with-out-str (dotimes [i 100] (println (str "line-" i))))
  (/ 1 0)
  (let [m (into {} (for [i (range 200)] [(keyword (str "k" i)) (vec (range i 0 -1))]))] (count m))
  (Thread/sleep 2000) :waited
  (str/join "\n" (map #(str "row " % ": " (apply str (repeat 80 (char (+ 65 (mod % 26)))))) (range 100)))
  (throw (ex-info "with-data" {:keys (vec (range 100)) :vals (mapv str (range 100))}))
  (Class/forName "com.fake.DoesNotExist")
  (bean (Runtime/getRuntime))
  (sort (repeatedly 1000 #(rand)))
  (/ 1 0)
  (with-out-str (clojure.pprint/pprint (into (sorted-map) (System/getProperties))))
  (Math/pow 2 1000)
  (throw (ex-info "final-boom" {}))
  (mapv char (range 32 127))
  (str "ALL DONE — " (.toString (java.time.Instant/now)))

  After all 30: bridge_control {action: "status"} — expect 30/30 in the health bar, 0 missed writes. Every exception and error should persist cleanly now.

