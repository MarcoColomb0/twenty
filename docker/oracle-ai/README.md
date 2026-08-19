# oracle-ai — host prerequisites

Ampere A1, 2 vCPU / 12GB, Ubuntu, no GPU.

The stack itself is deployed by doco-cd from this repository — see
`../README.md`. What follows is the part that cannot be expressed in git:
firewall rules, DNS, and the Cloudflare API token.

```
Karakeep ──HTTPS──► Cloudflare proxy ──HTTPS──► Oracle :443
                                                  Caddy (Let's Encrypt via DNS-01, checks Bearer)
                                                    └─► Ollama (never exposed)
```

## 1. Docker

```sh
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER" && newgrp docker
```

Caddy comes from `ghcr.io/serfriz/caddy-cloudflare`: stock Caddy plus the
Cloudflare DNS provider and `caddy-cloudflare-ip`. The published Caddy image has
no DNS modules compiled in at all, so DNS-01 is impossible with it and something
custom is unavoidable. The tag is pinned -- nothing on this host watches for
updates the way Renovate does for the cluster.

## 2. Cloudflare API token

My Profile → API Tokens → Create Token → **Edit zone DNS**, restricted to this
zone. Create a fresh one rather than reusing the cluster's cert-manager token —
this is a different host with a different blast radius, and revoking one should
not take TLS off the other.

## 3. DNS

An `A` record `ai` → the instance's public IP, **proxied** (orange cloud).

DNS-01 does not care whether the record is proxied: Let's Encrypt validates by
reading a TXT record Caddy publishes through the API, never by connecting to
this host. That is what makes it work behind a firewall that only admits
Cloudflare — HTTP-01 could not, because LE's validators would be rejected along
with everyone else.

## 4. Firewall — both layers

Oracle needs the port opened twice. The OCI security list is the one people
remember; Ubuntu images also ship a local iptables chain with a REJECT that
silently drops what the security list just allowed.

**OCI console** → VCN → Security List → Ingress: TCP 443, source restricted to
Cloudflare's ranges (https://www.cloudflare.com/ips/).

**On the box** — `-I`, not `-A`. Appending puts the rule after Ubuntu's REJECT,
where it never matches:

```sh
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  sudo iptables -I INPUT -p tcp -s "$ip" --dport 443 -j ACCEPT
done
sudo apt-get install -y iptables-persistent   # answer yes to save
```

No rule for :80 — nothing listens there and ACME does not need it.

## 5. Verify, in this order

Locally, before involving Cloudflare — note no `-k`, the cert is real:

```sh
curl -s --resolve "$AI_HOSTNAME:443:127.0.0.1" "https://$AI_HOSTNAME/v1/models" \
  -H "Authorization: Bearer $(grep ^AI_TOKEN .env | cut -d= -f2)" | head
curl -s -o /dev/null -w '%{http_code}\n' --resolve "$AI_HOSTNAME:443:127.0.0.1" \
  "https://$AI_HOSTNAME/v1/models" -H "Authorization: Bearer wrong"   # expect 401
```

Then end to end:

```sh
curl -s https://ai.<your-domain>/v1/models -H "Authorization: Bearer <AI_TOKEN>" | head
```

## 6. Measure this — it decides whether the design works

```sh
time curl -s https://ai.<your-domain>/v1/chat/completions \
  -H "Authorization: Bearer <AI_TOKEN>" -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5:3b-instruct-q4_K_M","messages":[{"role":"user","content":"Give three tags for an article about kubernetes storage."}]}'
```

**Cloudflare's proxy times out a request at 100 seconds** and returns 524. Not
configurable below Enterprise. If a realistic tagging call approaches it:

- drop `INFERENCE_CONTEXT_LENGTH` in Karakeep to 1024 — prefill is the cost, and
  halving the context roughly halves it
- or move to a smaller model (`qwen2.5:1.5b-instruct-q4_K_M`)

Karakeep's `INFERENCE_JOB_TIMEOUT_SEC` cannot rescue this: Cloudflare cuts the
connection first and Karakeep only sees a failed request.

## Notes

- Ollama has no authentication. It is never published here — only Caddy is, and
  only with the bearer token. Never add a `ports:` mapping to the ollama service.
- Keep the `caddy_data` volume. It holds the certificate and the ACME account
  key; losing it forces re-issuance, and Let's Encrypt rate-limits that.
- Renewal is automatic and needs no inbound access, same as issuance.
- Oracle reclaims idle Always Free compute. Confirm the instance is
  Always-Free-shaped before depending on it.
