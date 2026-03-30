---
name: runpod-ssh
description: SSH into a running RunPod pod for live code editing and iteration without rebuilding Docker. Use when making quick code changes on a running pod, debugging live, or iterating rapidly on a pod without a full rebuild cycle.
---

# RunPod SSH

Pods use dynamic TCP ports — look up the current port before every connection:

```bash
RP_KEY=$(security find-generic-password -s runpod-api-key-dev -w 2>/dev/null)
curl -s -X POST https://api.runpod.dev/graphql \
  -H "Authorization: Bearer $RP_KEY" \
  -H "Content-Type: application/json" \
  -d '{"query":"query{pod(input:{podId:\"<POD_ID>\"}){runtime{ports{ip publicPort type}}}}"}' \
  | jq -r '.data.pod.runtime.ports[] | select(.type=="tcp") | "\(.ip):\(.publicPort)"'
```

SSH key: `~/.ssh/id_ed25519` or `~/.ssh/runpod`

```bash
ssh -i ~/.ssh/id_ed25519 -o StrictHostKeyChecking=no -p $PORT root@$HOST
```

## Gotchas

- `*.proxy.runpod.net` hostnames are HTTP only — use the IP:port from the API for SSH/scp/rsync
- Port changes on every pod restart — always re-query before connecting
- `/app/` is ephemeral; `/runpod-volume/` persists across restarts
- New dependencies need installing on the pod but won't survive a restart without a Docker rebuild
