# Server Restart Runbook (EC2 + minikube + ArgoCD)

How to bring the Online Boutique app **and** ArgoCD back online after the EC2
instance is restarted, and how to stop the public-IP churn from breaking
routing every time.

---

## Why things break on restart

The stack is: `internet → EC2 (public IP) → iptables DNAT → minikube
NodePorts → ingress-nginx → services`. Three things depend on the EC2 state:

| Thing | Survives stop/start? | Notes |
|---|---|---|
| **Public IP** | ❌ changes (unless Elastic IP) | New IP each stop/start. Private IP `172.31.34.46` is kept. |
| **minikube** | ✅ auto-starts | Node IP `192.168.49.2` and bridge `br-bb8cc26c6086` are stable across restarts. |
| **PREROUTING DNAT** (80→30240, 443→30308) | ⚠️ currently survives | Re-assert anyway (idempotent) in case it doesn't. |
| **DOCKER-USER forward rules** | ❌ wiped | The fix that lets external traffic reach the bridge. Must be re-applied. |
| **Ingress hostnames** (`*.nip.io`) | n/a | Encode the public IP, so they break when the IP changes. |

`nip.io` resolves `boutique.<IP>.nip.io` → `<IP>`. When the public IP changes,
the hostname must change too — which is the painful part, because
`frontend-ingress` is **managed by ArgoCD** (in `kubernetes-manifests/`), so a
live `kubectl edit` is reverted by self-heal. It can only change via a Git
commit — and `main` is protected, so that means a PR.

`argocd-ingress` lives in the `argocd` namespace and is **not** GitOps-managed,
so it can be patched live with `kubectl`.

---

## Permanent fix (do this once): Elastic IP

An Elastic IP (EIP) stays attached to the instance across stop/start, so the
public IP never changes again. This eliminates ingress-host edits and PRs on
every restart.

**Console:** EC2 → Network & Security → Elastic IPs → *Allocate* →
*Associate* to this instance.

**CLI (run on the box, the instance role can be granted these perms):**
```bash
ALLOC=$(aws ec2 allocate-address --domain vpc --region ap-south-1 --query AllocationId --output text)
IID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)   # IMDSv1; use token form if disabled
aws ec2 associate-address --region ap-south-1 --instance-id "$IID" --allocation-id "$ALLOC"
aws ec2 describe-addresses --region ap-south-1 --allocation-ids "$ALLOC" --query 'Addresses[0].PublicIp' --output text
```

After allocating the EIP (call it `<EIP>`), set the hostnames **once**:

1. **App (via PR — it's GitOps-managed):** edit
   `kubernetes-manifests/frontend-ingress.yaml`, set
   `host: boutique.<EIP>.nip.io`, open a PR to `main`, merge. ArgoCD syncs.
2. **ArgoCD (live — not GitOps-managed):**
   ```bash
   kubectl -n argocd patch ingress argocd-ingress --type=json \
     -p='[{"op":"replace","path":"/spec/rules/0/host","value":"argocd.<EIP>.nip.io"}]'
   ```

Once the IP is stable, **restarts require no hostname changes at all** — only
the iptables rules need re-asserting (and the systemd unit below does that
automatically).

### Optional: make the app ingress IP-agnostic (alternative to / on top of EIP)

If you'd rather not depend on the hostname matching the IP, remove the `host:`
field from `frontend-ingress.yaml` so it becomes a catch-all. Then the shop is
reachable at `http://<any-current-IP>/` with no further edits. This still needs
one PR (it's GitOps-managed), but never again after that. ArgoCD keeps its own
host rule, so the two ingresses don't collide.

---

## Make the routing rules persistent (do this once)

So you don't re-run iptables by hand after every restart, install a systemd
oneshot unit that re-applies them on boot (after Docker/minikube).

Create `/usr/local/sbin/k8s-reroute.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail
BR=br-bb8cc26c6086      # minikube docker bridge
NODE=192.168.49.2       # minikube node IP
HTTP_NP=30240           # ingress-nginx HTTP NodePort
HTTPS_NP=30308          # ingress-nginx HTTPS NodePort

# Wait for the bridge to exist (minikube up)
for i in $(seq 1 30); do ip link show "$BR" &>/dev/null && break; sleep 2; done

# PREROUTING DNAT: public 80/443 -> minikube ingress NodePorts (idempotent)
iptables -t nat -C PREROUTING -i ens5 -p tcp --dport 80  -j DNAT --to-destination $NODE:$HTTP_NP 2>/dev/null \
  || iptables -t nat -A PREROUTING -i ens5 -p tcp --dport 80  -j DNAT --to-destination $NODE:$HTTP_NP
iptables -t nat -C PREROUTING -i ens5 -p tcp --dport 443 -j DNAT --to-destination $NODE:$HTTPS_NP 2>/dev/null \
  || iptables -t nat -A PREROUTING -i ens5 -p tcp --dport 443 -j DNAT --to-destination $NODE:$HTTPS_NP

# FORWARD: allow the DNAT'd traffic into the bridge + established return (idempotent)
iptables -C DOCKER-USER -i ens5 -o $BR -p tcp -d $NODE -m multiport --dports $HTTP_NP,$HTTPS_NP -j ACCEPT 2>/dev/null \
  || iptables -I DOCKER-USER -i ens5 -o $BR -p tcp -d $NODE -m multiport --dports $HTTP_NP,$HTTPS_NP -j ACCEPT
iptables -C DOCKER-USER -i $BR -o ens5 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT 2>/dev/null \
  || iptables -I DOCKER-USER -i $BR -o ens5 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
echo "k8s-reroute applied."
```

Install + enable:
```bash
sudo chmod +x /usr/local/sbin/k8s-reroute.sh
sudo tee /etc/systemd/system/k8s-reroute.service >/dev/null <<'UNIT'
[Unit]
Description=Re-apply iptables routing for minikube ingress
After=docker.service
Requires=docker.service

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/k8s-reroute.sh
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
UNIT
sudo systemctl daemon-reload
sudo systemctl enable --now k8s-reroute.service
```

> Note: the NodePorts (`30240`/`30308`) and bridge name (`br-bb8cc26c6086`) are
> stable as long as the minikube cluster isn't deleted/recreated. If you ever
> `minikube delete`, re-check them with
> `kubectl -n ingress-nginx get svc ingress-nginx-controller` and `ip -o addr | grep br-`.

---

## Per-restart runbook

### A) With Elastic IP + systemd unit (target state — ~0 manual steps)
1. Start the instance. minikube auto-starts; `k8s-reroute.service` re-applies rules on boot.
2. Verify (from your laptop):
   ```bash
   curl -I http://boutique.<EIP>.nip.io        # app
   curl -kI https://argocd.<EIP>.nip.io         # argocd
   ```
   Both should respond. Done.

### B) Without Elastic IP (IP changed to `<NEWIP>`)
1. SSH in (your source IP must be in the security group's port-22 rule).
2. Ensure minikube is up: `minikube status` (else `minikube start`).
3. Re-apply routing: `sudo /usr/local/sbin/k8s-reroute.sh`
   (or run the idempotent iptables block above).
4. Update **ArgoCD** host (live):
   ```bash
   kubectl -n argocd patch ingress argocd-ingress --type=json \
     -p='[{"op":"replace","path":"/spec/rules/0/host","value":"argocd.<NEWIP>.nip.io"}]'
   ```
5. Update **app** host — this needs a **PR to `main`** (GitOps + branch
   protection): set `host: boutique.<NEWIP>.nip.io` in
   `kubernetes-manifests/frontend-ingress.yaml`, open PR, merge; ArgoCD syncs.
   *(This step is why an Elastic IP — or a host-less app ingress — is strongly
   recommended.)*
6. Verify with the `curl` commands above using `<NEWIP>`.

---

## Quick verification commands

```bash
# On the box — confirm forwarding rules present:
sudo iptables -t nat -L PREROUTING -n | grep DNAT
sudo iptables -L DOCKER-USER -n -v

# Through the ingress (Host header), bypassing public IP:
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: boutique.<IP>.nip.io" http://192.168.49.2:30240
curl -s -o /dev/null -w "%{http_code}\n" -H "Host: argocd.<IP>.nip.io"   http://192.168.49.2:30240

# Live ingress hosts:
kubectl get ingress -A
```
