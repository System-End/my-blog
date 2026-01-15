---
title: "Finding a CSRF Vuln in Hack Club Spaces"
description: "How I found a bug by reading source code"
pubDate: 15 Jan 2026
---

Was planning on using spaces for my APCSA class and show it to my peers, teacher asked me if the code was secure and asked me to check it.... i ended up finding a CSRF bug in the admin panel. Here's how.

## The Bug

Admin endpoints read auth tokens from the request body instead of headers:

```js
// src/middlewares/admin.middleware.js line 6
const { authorization } = req.body;  // bad

// vs everywhere else
const authorization = req.headers.authorization;  // good
```

Why does this matter? HTML forms can set body data, but they can't set custom headers. So I can make a form on any website that submits to their admin API.

## The Attack

```html
<form method="POST" action="https://spaces-api.com/api/v1/admin/users/1/update">
  <input type="hidden" name="authorization" value="STOLEN_TOKEN">
  <input type="hidden" name="is_admin" value="true">
  <button>free robux</button>
</form>
```

If an admin clicks that button (or I auto-submit it with JS), their browser sends the request. No CORS issues, no preflight - just a normal form submission.

## Proving It

Saved the HTML locally, opened it, clicked the button:

- `Origin: null` - request came from a file, not their site
- `Sec-Fetch-Site: cross-site` - browser confirmed it's cross-origin
- Response: `401 Invalid authorization token` - server processed it, just rejected the fake token

With a real token, this executes. Game over.

## How Would You Get The Token?

The OAuth flow leaks it in the URL fragment after login (`/#oauth_success=true&user_data={"authorization":"xxx"}`). This gets saved in browser history, can be logged by analytics, or leak via Referer header. Any XSS would also grab it.

## The Fix

One line change:

```diff
- const { authorization } = req.body;
+ const authorization = req.headers.authorization;
```

# Status

Reported on 15-01-26

On 15-01-26 Ivie (the maintainer of Space) contacted me stating:

"Hey End!

Re: the two security bounties you submitted

Neither are these are eligible for a payout, due to spaces being in a private beta state.

However, even if it was in production, they would not likely get a payout as:

The admin csrf requires you to already have an admin token, and if you had an admin token you could just use the admin endpoints, no csrf needed

Both will be fixed before spaces goes to production though, thanks for the report!"

---
