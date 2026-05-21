issue: argocd configure and process works but the slack show no message.

- Confirm argocd notification log

```sh
kubectl logs -n argocd deploy/argocd-notifications-controlle0 -f
# {"level":"info","msg":"Start processing","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
# {"level":"info","msg":"Trigger 'on-deployed' TRIGGERED | revision: 6e489943 | templates: [app-deployed]","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
# {"level":"info","msg":"Notification about condition 'on-deployed.[0].Yv81TUs0gmPTEyk-v8Lb6ZKEo0g' already sent to '{slack project-gitops-demo}' using the configuration in namespace argocd","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
# {"level":"info","msg":"Trigger 'on-health-degraded' FAILED | revision:  | templates: [app-health-degraded]","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
# {"level":"info","msg":"Trigger 'on-sync-failed' FAILED | revision:  | templates: [app-sync-failed]","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
# {"level":"info","msg":"Processing completed","resource":"argocd/sample-app","time":"2026-05-21T19:57:26Z"}
```

- root cause
  - channel integration is not disabled

```sh
# test channel with token
curl -s -X POST https://slack.com/api/chat.postMessage   -H "Authorization: Bearer $TOKEN"   -H "Content-Type: application/json; charset=utf-8"   -d '{"channel":"project-gitops-demo","text":"direct curl test"}'
# {"ok":true,"channel":"C0B2X513WG4","ts":"1779393408.521249","message":{"user":"U0B2VDWM049","type":"message","ts":"1779393408.521249","bot_id":"B0B2Q5V356Z","app_id":"A0B2X56MCAG","text":"direct curl test","team":"T0AM95WNPK4","bot_profile":{"id":"B0B2Q5V356Z","app_id":"A0B2X56MCAG","user_id":"U0B2VDWM049","name":"GitOps Demo","icons":{"image_36":"https:\/\/a.slack-edge.com\/80588\/img\/plugins\/app\/bot_36.png","image_48":"https:\/\/a.slack-edge.com\/80588\/img\/plugins\/app\/bot_48.png","image_72":"https:\/\/a.slack-edge.com\/80588\/img\/plugins\/app\/service_72.png"},"deleted":false,"updated":1778383054,"team_id":"T0AM95WNPK4"},"blocks":[{"type":"rich_text","block_id":"Hzos5","elements":[{"type":"rich_text_section","elements":[{"type":"text","text":"direct curl test"}]}]}]}}
```
