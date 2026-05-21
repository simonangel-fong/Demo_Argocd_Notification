
```sh
# manual test
gh api repos/simonangel-fong/Demo_Argocd_Notification/dispatches -f event_type=argocd-deployed -f client_payload[app]=manual-test -f client_payload[status]=Synced -f client_payload[health]=Healthy -f client_payload[revision]=abc1234 -f client_payload[path]=manual


kubectl logs -n argocd deploy/argocd-notifications-controller --tail=30
```

```sh
# debug
TOKEN=$(kubectl get secret argocd-notifications-secret -n argocd -o jsonpath='{.data.github-token}' | base64 -d)
echo "Token starts: ${TOKEN:0:15}..."

curl -sS -i -X POST https://api.github.com/repos/simonangel-fong/Demo_Argocd_Notification/dispatches \
  -H "Authorization: token $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -d '{"event_type":"argocd-deployed","client_payload":{"app":"diag","status":"Synced","health":"Healthy","revision":"diag1","path":"diag"}}'

# if works:  
# HTTP/2 204
# date: Thu, 21 May 2026 20:37:31 GMT
# github-authentication-token-expiration: 2026-08-14 22:35:28 UTC
# x-github-media-type: github.v3; format=json
# x-accepted-github-permissions: contents=write
# x-github-api-version-selected: 2022-11-28
# access-control-expose-headers: ETag, Link, Location, Retry-After, X-GitHub-OTP, X-RateLimit-Limit, X-RateLimit-Remaining, X-RateLimit-Used, X-RateLimit-Resource, X-RateLimit-Reset, X-OAuth-Scopes, X-Accepted-OAuth-Scopes, X-Poll-Interval, X-GitHub-Media-Type, X-GitHub-SSO, X-GitHub-Request-Id, Deprecation, Sunset, Warning
# access-control-allow-origin: *
# strict-transport-security: max-age=31536000; includeSubdomains; preload
# x-frame-options: deny
# x-content-type-options: nosniff
# x-xss-protection: 0
# referrer-policy: origin-when-cross-origin, strict-origin-when-cross-origin
# content-security-policy: default-src 'none'
# vary: Accept-Encoding, Accept, X-Requested-With
# x-github-request-id: D139:2FAB9E:11C74B2:43DB60F:6A0F6D0B
# server: github.com
# x-ratelimit-limit: 5000
# x-ratelimit-remaining: 4979
# x-ratelimit-reset: 1779396844
# x-ratelimit-used: 21
# x-ratelimit-resource: core

```

common issue: pat access

Open https://github.com/settings/personal-access-tokens
Click your token → Edit
Scroll to Repository permissions → find Contents → change Access: Read-only → Access: Read and write
Click Save
