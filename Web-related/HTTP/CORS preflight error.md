Why not 401 or something? Usually when server is not avaialble , it's the "server error", not "cors error"

Great question! The distinction is important:

**Why you see a CORS error instead of a connection error:**

When JavaScript in a browser makes a **cross-origin request** (frontend on localhost:80 calling backend on localhost:3000 - different ports = different origins), the browser **requires a CORS preflight check** before allowing the actual request.

Here's what happens:

1. **Browser**: "I need to make a POST to http://localhost:3000/auth/login from origin http://localhost"
2. **Browser**: "This is cross-origin, so I'll send an OPTIONS preflight request first"
3. **Browser tries to connect**: OPTIONS http://localhost:3000/auth/login
4. **Connection fails** (server not running)
5. **Browser**: "The CORS preflight didn't succeed - I'll report this as a CORS error"

The browser **never even attempts** the actual POST request because the CORS preflight failed at the network level.

---

**If the server WAS running but rejected you:**

- You'd see HTTP status codes: 401 Unauthorized, 403 Forbidden, 500 Server Error, etc.
- The connection succeeded, but the server responded with an error

**If the server is NOT running:**

- The browser can't complete the CORS preflight
- It reports "CORS request did not succeed" with status code (null)
- This is confusing because it sounds like a CORS configuration problem, but it's actually a "can't connect at all" problem

The key indicator is **status code: (null)** - that means no HTTP response was received at all, which means the server isn't listening.

So it's not "CORS request did not succeed" , it's "CORS preflight request did not succeed"

**Exactly!** You got it perfectly.

The browser's error message is technically misleading. It should say:

> "CORS **preflight** request did not succeed"

But instead it just says "CORS request did not succeed", which makes developers think it's a CORS configuration problem when it's actually a network connectivity problem.

**What's really happening:**

1. Browser tries: OPTIONS http://localhost:3000/auth/login (CORS preflight)
2. No server listening → Connection refused
3. Browser reports: "CORS request did not succeed" ❌ (misleading!)
4. Should say: "CORS preflight failed - cannot connect to server" ✓ (accurate!)

The actual POST /auth/login request **never even gets attempted** because the preflight failed.

This is why the **status code is (null)** - no HTTP response was ever received, because the TCP connection couldn't be established.

So yes, you've nailed the distinction:

- **CORS preflight** failed (couldn't connect)
- Not the actual request
- Not a CORS headers/configuration issue

Start the backend with clj -M:run and it'll work!