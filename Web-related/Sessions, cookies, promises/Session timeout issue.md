Yes! This is a **session timeout issue**. Here's what's happening:

## The Problem

1. **User logs in** → Token saved to localStorage → app-state shows :logged-in true
2. **User leaves tab open for 30+ minutes** → Backend session expires (30-minute timeout)
3. **User comes back** → Frontend still thinks it's logged in (localStorage has the token)
4. **Frontend tries to fetch reports** → Backend rejects with 401 (session expired)
5. **Frontend keeps retrying** → Endless 401 errors

## Root Cause

The frontend has **no logic to handle 401 responses** by logging the user out. It just keeps retrying with the expired token.

Let me check the current error handler:

Grephandle-api-error

Perfect! I can see the issue. The handle-api-error function doesn't check for 401 status and log the user out. Let me fix this:

ReadD:\projects\reporting-planner-cljc-v03\src\cljs\reporting_planner\core.cljs

Read 10 lines

EditD:\projects\reporting-planner-cljc-v03\src\cljs\reporting_planner\core.cljs

`(defn handle-api-error [response]`

`(let [error-msg (or (get-in response [:response :error])`

`(:status-text response)`

`"An error occurred")]`

`(swap! app-state assoc :error error-msg :loading false)))`

`"Handles API errors. On 401 (session expired), logs user out automatically."`

`(let [status (:status response)`

`error-msg (or (get-in response [:response :error])`

`(:status-text response)`

`"An error occurred")]`

`;; If session expired (401), clear session and force re-login`

`(if (= status 401)`

`(do`

`(println "Session expired (401). Logging out...")`

`(remove-from-local-storage! "token")`

`(remove-from-local-storage! "username")`

`(swap! app-state assoc`

`:logged-in false`

`:username nil`

`:token nil`

Show full diff (4 more lines)

Now I need to add the translation key for the session expired message:

ReadD:\projects\reporting-planner-cljc-v03\src\cljs\reporting_planner\i18n.cljs

Read 208 lines

EditD:\projects\reporting-planner-cljc-v03\src\cljs\reporting_planner\i18n.cljs

`:cert-certified "Certified"`

`:cert-verified "Verified"`

`:cert-community "Community"}`

`:cert-community "Community"`

`:session-expired "Your session has expired. Please log in again."}`

`:ru`

EditD:\projects\reporting-planner-cljc-v03\src\cljs\reporting_planner\i18n.cljs

`:cert-certified "Сертифицирован"`

`:cert-verified "Проверен"`

`:cert-community "Сообщество"}})`

`:cert-community "Сообщество"`

`:session-expired "Ваша сессия истекла. Пожалуйста, войдите снова."}})`

`;; ============================================`

Perfect! Now let me verify the changes with a git diff:

Bashcd "D:\projects\reporting-planner-cljc-v03" && git diff src/cljs/reporting_planner/

;; ============================================ ;; Current Locale State... +53 lines

Excellent! The fix is complete. Here's what I've done:

## ✅ Fixed Session Timeout Handling

### The Problem

- Frontend kept expired token in localStorage
- When session expired (30-min timeout), backend returned 401
- Frontend kept retrying with expired token → endless 401 errors
- User had to manually refresh page to log in again

### The Solution

**1. Updated handle-api-error function** (core.cljs:70-89)

- Now checks for status 401 (Unauthorized)
- On 401:
    - Clears localStorage (removes token and username)
    - Resets app-state to logged-out state
    - Shows localized "session expired" message
    - User sees login form again
- On other errors:
    - Shows normal error message (existing behavior)

**2. Added translations** (i18n.cljs)

- :session-expired key added to both English and Russian
- **English**: "Your session has expired. Please log in again."
- **Russian**: "Ваша сессия истекла. Пожалуйста, войдите снова."

### What Now Happens

1. **User leaves tab open for 30+ minutes**
2. **Backend session expires** (cleaned up by daemon thread)
3. **User comes back, frontend tries to fetch reports**
4. **Backend returns 401**
5. **Frontend automatically:**
    - Logs user out
    - Clears localStorage
    - Shows login form with friendly message
    - **Stops retrying** (no more endless 401 barrage)

### Testing

To test this fix:

1. Log in to the app
2. Wait 30+ minutes (or manually delete session from data/sessions.edn for faster testing)
3. Try to interact with the app (it will fetch reports)
4. You should see:
    - Login form appears automatically
    - Error message: "Your session has expired. Please log in again."
    - No console spam with 401 errors

The fix is clean, user-friendly, and respects the i18n system!