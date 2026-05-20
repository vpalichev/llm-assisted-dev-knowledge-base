The audit log solved it in one shot. Everything before that was guessing.

  The progression was:
  1. Symptom: writes fail ~5% of the time
  2. Hypothesis: pod IPC contention → added serialization → still fails
  3. Hypothesis: pod flakiness → added agent queue → still fails
  4. Actual fix: logged the full error message → CHECK constraint failed → two-line fix

  The lesson: read the actual error before building workarounds. The catch Exception e in the original code logged (.getMessage e) to the log file, but nobody was reading it. If we'd looked at the log on day one, we'd have seen CHECK constraint failed and been done in 5 minutes.

  General rules:

  - Log the error, not just the fact that an error happened. "write failed" is useless. "write failed: CHECK constraint failed: status IN (...)" is the answer.
  - Don't theorize past one failed attempt. Serialization didn't help → the theory was wrong. Should have stopped and gathered data instead of trying the next theory (agent queue).
  - Structured audit beats speculation. The write-audit.edn with full context (form, status, error, payload size, timing) made the pattern obvious: every failure had eval-status: "exception" + the same constraint error.
  - Test your schema against all real data paths. The nREPL client produced statuses (exception, class-not-found) that the schema never accounted for. The dashboard CSS already handled them — the DB didn't.

  Want me to save this as a feedback memory?
