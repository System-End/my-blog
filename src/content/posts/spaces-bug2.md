---
title: "OAuth Token Leak in Hack Club Spaces"
description: "Your auth token is in your browser history"
pubDate: 15 Jan 2026
heroImage: /src/assets/oauth-url.png
---

# OAuth Token Leak in Hack Club Spaces

Second bug I found while poking around Spaces. This one's simpler but arguably worse.

## The Problem

After you login with Hack Club OAuth, the app redirects you to:

```
https://spaces/#oauth_success=true&user_data={"authorization":"your_secret_token","username":"you"}
```

Your full auth token. Right there in the URL.

## Why That's Bad

URLs stick around:

1. **Browser history** - anyone who opens your history sees it
2. **Analytics** - if they run Google Analytics or whatever, full URLs get logged
3. **Referer header** - click a link right after login, the next site might see it
4. **Extensions** - browser extensions can read URLs
5. **Shoulder surfing** - it's literally on screen

The token gives full account access. Your spaces, your data, everything.

## The Code

```js
// src/api/oauth/oauth.route.js lines 184-186
const encodedData = encodeURIComponent(JSON.stringify(userData));
res.redirect(`/#oauth_success=true&user_data=${encodedData}`);
```

They're already setting an httpOnly cookie with the token on line 177. So why also put it in the URL? No idea.

## The Fix

Just... don't:

```diff
- const encodedData = encodeURIComponent(JSON.stringify(userData));
- res.redirect(`/#oauth_success=true&user_data=${encodedData}`);
+ res.redirect('/#oauth_success=true');
```

Frontend can fetch user data from a `/me` endpoint using the cookie. Problem solved.

## Proof

Logged in, checked browser history:

![history showing token](/src/assets/oauth-history.png)

There it is. Permanently saved until you clear history.

# Status

Reported on 15-01-26

On 15-01-26 Ivie (the maintainer of Space) contacted me stating

"Hey End!

Re: the two security bounties you submitted:

Neither are these are eligible for a payout, due to spaces being in a private beta state.

However, even if it was in production, they would not likely get a payout as:

The account token stored in URL: This is an issue, but due to it requiring access to the users computer to actually exploit, it is likely out of scope.

Both will be fixed before spaces goes to production though, thanks for the report!"

---
