
```sh
# manual test
gh api repos/simonangel-fong/Demo_Argocd_Notification/dispatches \
  -f event_type=argocd-deployed \
  -f client_payload[app]=manual-test \
  -f client_payload[status]=Synced \
  -f client_payload[health]=Healthy \
  -f client_payload[revision]=abc1234 \
  -f client_payload[path]=manual
```