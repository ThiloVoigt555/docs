---
title: Authentication Tokens
---

Screeps doesn't have a documented public Web API. However, if you want to use undocumented HTTP endpoints which our server uses to communicate with the client, that's fine. We've developed an **authentication tokens** system to make this process easier for you.

The regular web browser client uses Google Invisible reCAPTCHA to validate some requests in the background, including the sign-in request. The Steam client uses an encrypted local Steam connection for similar purpose. If you're building some external tool that doesn't require human interaction, you can generate a persistent auth token to make requests without solving CAPTCHA. Such token is generated once and doesn't have expiration time. 

## Using Auth Tokens

You can generate an auth token in your [account settings](https://screeps.com/a/#!/account/auth-tokens):

![](img/auth_tokens.png) 

A token with **full access** will have the same access scope as your usual authentication credentials. You can also limit the access scope to **selected endpoints**, **websockets events** and **memory segments**.

There are two identically valid ways to use this token:

* Set `X-Token` header in your request:
   ```
   X-Token: 3bdd1da7-3002-4aaa-be91-330562f54093
   ```     
   
* Add `_token` query param to the URL:
   ```
   https://screeps.com/api/user/name?_token=3bdd1da7-3002-4aaa-be91-330562f54093
   ```
 
## Rate Limiting

Regular requests made by browser or Steam client are **NOT** rate limited.

However, all requests authenticated by auth tokens are subject to rate limiting rules. When rate capacity is exceeded, you will get `429` HTTP code in response:

```
HTTP/1.1 429 Too Many Requests

Rate limit exceeded, retry after 51243ms
```

Three HTTP header are set for informational purposes which you can use to handle rate limiting on your side:

| Header Name | Description |
|-|-|
| `X-RateLimit-Limit` | The maximum number of requests you're permitted to make per limit window. |
| <nobr>`X-RateLimit-Remaining`</nobr> | The number of requests remaining in the current rate limit window. |
| `X-RateLimit-Reset` | The time at which the current rate limit window resets in UTC epoch seconds. |

```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 35
X-RateLimit-Reset: 1514539728
```

There are two levels of rate limiting: global and per-endpoint, shown in the table below:

| Endpoint | Rate |
|----------|------|
| **Global**   | **120 / minute** |
| GET /api/game/room-terrain | 360 / hour |
| POST /api/game/map-stats | 60 / hour |
| GET /api/user/code | 60 / hour |
| POST /api/user/code | 240 / day
| POST /api/user/set-active-branch | 240 / day |
| GET /api/user/memory | 1440 / day |
| POST /api/user/memory | 240 / day |
| GET /api/user/memory-segment | 360 / hour |
| POST /api/user/memory-segment | 60 / hour |
| POST /api/user/console | 360 / hour | 
| GET /api/game/market/orders-index | 60 / hour |
| GET /api/game/market/orders | 60 / hour |
| GET /api/game/market/my-orders | 60 / hour |
| GET /api/game/market/stats | 60 / hour |
| GET /api/game/user/money-history | 60 / hour |

### Turning rate limiting off

If you develop a third-party tool that requires human interaction, you can integrate a special flow to turn off rate limiting on a specific token. In order to do so, you must provide the user with a link `https://screeps.com/a/#!/account/auth-tokens/noratelimit?token=XXX`, which they should navigate to. When the user clicks the "Proceed" button on the page, the token will be granted with a 2-hour period with no rate limits.

![](img/token-noratelimit.png) 

If your tool is web-based, you can embed this page in an `<iframe>` and listen to a `screepsTokenSuccess` message:

```javascript
window.addEventListener('message', (event) => {
    if(event.data === 'screepsTokenSuccess') {
        console.log("We are not rate limited now!");
    }   
}, false);
```

Please note that this page uses Google Invisible reCAPTCHA, so that it cannot be used automatically.

You can query info on a specific token (including its unlimited period timer) using the endpoint `https://screeps.com/api/auth/query-token?token=XXX`.