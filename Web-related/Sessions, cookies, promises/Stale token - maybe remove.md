## Why You Needed to Log In Again

### The Problem: Stale Token in LocalStorage

When the backend was initially running, it had **no sessions stored**. But your browser's localStorage still had an **old token from a previous session** (probably from before the backend was restarted or data was cleared).

 

Here's the sequence:

1. **Old session** existed and you had a token stored in localStorage
2. **Backend was restarted** (or sessions were cleared) → `sessions.edn` became empty `{}`
3. **Your browser still had the old token** in localStorage
4. When you clicked download, the frontend sent that **stale token**
5. **Backend couldn't find it** in its empty sessions file → 401 Unauthorized

### The Solution: Fresh Session

When you **logged in again**:

1. Backend created a **new session** with a new token
2. Backend **saved it to disk** → `sessions.edn` now had the token
3. Frontend **updated localStorage** with the new token
4. Download worked because frontend and backend tokens now **matched**

### The Root Issue: Session Persistence

The real underlying issue is that the backend's session file was cleared or lost. This can happen because:

- Backend process was killed/restarted
- Sessions file was manually deleted
- Data directory wasn't properly set up
- Session data wasn't being persisted correctly

### Why It's Now Working

You now have an **active session** that exists on both:

- ✅ Frontend (in localStorage)
- ✅ Backend (in sessions.edn file)

As long as you don't log out or the backend doesn't restart, downloads will continue to work. If the backend restarts again, you'll need to log in once more to re-establish the session.

---

**Note:** On the production server (`project-control.intranet.silleno.kz`), you'll need to do the same thing - log out and log back in - to create a fresh session on the production backend. The production backend likely has the same empty sessions file issue.