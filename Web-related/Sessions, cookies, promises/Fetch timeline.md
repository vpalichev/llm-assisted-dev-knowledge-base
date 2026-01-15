TIME →

[js/fetch url called]
        ↓
    Promise #1 (pending...)
        ↓
    [network packets arriving...]
        ↓
    Headers received! ✓
        ↓
    Promise #1 resolves
    Value: Response object {status: 200, headers: {...}, body: ReadableStream}
        ↓
    First .then runs: #(.json %)
        ↓
    (.json %) called on Response
        ↓
    Promise #2 (pending...)
        ↓
    [reading body from stream...]
        ↓
    Body fully received! ✓
    JSON parsed! ✓
        ↓
    Promise #2 resolves
    Value: JavaScript object {users: [{name: "Alice"}, {name: "Bob"}]}
        ↓
    Second .then runs: #(js->clj % :keywordize-keys true)
        ↓
    Conversion happens (synchronous)
        ↓
    Final value: {:users [{:name "Alice"} {:name "Bob"}]}