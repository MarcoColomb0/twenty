# docker

Compose stacks deployed by [doco-cd](https://doco.cd) to hosts outside the
Kubernetes cluster. Flux does not look here — it reconciles `./kubernetes` only,
and `.doco-cd.yaml` at the repo root is what doco-cd reads. The two tools share
the repository and ignore each other's paths.

| Stack | Host | Managed by |
|---|---|---|
| `oracle-ai` | Oracle Ampere A1 | doco-cd, from git |
| `doco-cd` | Oracle Ampere A1 | **bootstrap only**, brought up by hand |

`doco-cd` cannot deploy itself, so its own compose file is started manually once
and then left alone.

## The age key is a separate one, deliberately

`docker/oracle-ai/.env` is SOPS-encrypted like everything else in this
repository, but it must **not** be encrypted to the cluster's age key.

That key decrypts every secret here — `cluster-secrets`, every app secret, the
Talos machine secrets. Putting it on a public-facing VM so it can read one env
file would mean that compromising that VM hands over the whole cluster. A
dedicated key limits the blast radius to this one stack.

Generate it on the Oracle box, keep the private half there and nowhere else:

```sh
sudo mkdir -p /etc/doco-cd
age-keygen | sudo tee /etc/doco-cd/age.key >/dev/null
sudo chmod 600 /etc/doco-cd/age.key
sudo grep 'public key' /etc/doco-cd/age.key      # the recipient, safe to share
```

Add that public key to `.sops.yaml` as a creation rule for this path:

```yaml
- path_regex: docker/.*\.env$
  age: "age1<the new public key>"
```

Then create the encrypted env file from the repo root:

```sh
cat > docker/oracle-ai/.env <<'ENV'
AI_TOKEN=<openssl rand -hex 32>
AI_HOSTNAME=ai.<your-domain>
ACME_EMAIL=you@example.com
CF_API_TOKEN=<Cloudflare token, Zone:DNS:Edit on this zone only>
ENV
sops --config .sops.yaml -e -i docker/oracle-ai/.env
grep -c 'ENC\[' docker/oracle-ai/.env      # must be > 0 before committing
```

`AI_TOKEN` is the same value the cluster uses as `OPENAI_API_KEY` in the
karakeep secret.

## Bootstrap on the Oracle box

```sh
sudo mkdir -p /opt/doco-cd && cd /opt/doco-cd
curl -O https://raw.githubusercontent.com/MarcoColomb0/twenty/main/docker/doco-cd/compose.yaml
docker compose up -d
docker compose logs -f
```

From then on, a commit to `main` touching `docker/oracle-ai` is what deploys.
Polling is outbound only, every 5 minutes; no webhook and no inbound port, which
matters because the OCI security list admits Cloudflare and nothing else.

See `docker/oracle-ai/README.md` for the firewall, DNS and certificate setup
that has to exist before the stack can serve anything.
