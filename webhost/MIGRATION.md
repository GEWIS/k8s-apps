# Migrating a webhost to `static-webhost`

How to move a site off the old Virtualmin server (`gewiswebhost`) onto the
`static-webhost` Helm chart running on the k3s cluster. `abc` (done 2026-06-24)
is the worked example throughout.

- Chart: `static-webhost` (repo `GEWIS/webhost-helm-chart`), deployed from this
  repo via Flux. See `webhost/` for the releases and the chart's `README.md`/`values.yaml`
  for the full schema.
- Cluster access: `KUBECONFIG=../fleet-infra/kubeconfig.yaml` (via netbird). No `flux`
  CLI — reconcile with `kubectl annotate --overwrite <kind>/<name> -n flux-system reconcile.fluxcd.io/requestedAt="$(date +%s)"`.

## Does the site even fit?

`static-webhost` serves a **read-only** file tree with FrankenPHP (Caddy + PHP),
**multi-replica**, no `.htaccess`. It suits **static sites and stateless-ish PHP**:
reads its own folder, talks to *external* DBs/APIs, and at most writes to a few
explicitly-declared `writablePaths` (caches/uploads).

It does **not** suit apps that need a broadly writable tree, `.htaccess` rewrites,
or shared login sessions across replicas — e.g. **MediaWiki, osTicket**. Those need
their own proper deployment (or stay on the old box). Decide this first.

### Assess (read-only, on the old server)

```sh
ssh cbclocal@gewiswebhost            # cbclocal has passwordless `sudo /usr/bin/rsync` ONLY
```
- **Domains → docroots:** Apache vhosts in `/etc/apache2/sites-available/<domain>.conf`
  are world-readable — read `ServerName` / `ServerAlias` / `DocumentRoot`. Primary
  domain docroot = `/home/<user>/public_html`; sub-domains = `/home/<user>/domains/<d>/public_html`.
- **Explore the tree** (home dirs are `0750`, so use the root rsync):
  `sudo -n rsync --list-only -r /home/<user>/public_html/`
- **Characterize each app:** count `.php`, look for DB use (`mysqli`/`PDO`), disk
  writes (`file_put_contents`/`fopen`/`mkdir`), `session_start`, outbound calls
  (`curl`/`file_get_contents http`), `.htaccess`, cron. That tells you the
  `writablePaths` it needs and whether it fits at all.

## 1. Add the release / group (this repo)

Either add a group to an existing release's `domainGroups`, or create a new release
under `webhost/<env>/` (mirror `webhost/prod/`: `release.yaml` + `kustomization.yaml`
+ `rbac/` with the `app-rbac` component and an `ns.yaml` NamespaceTransformer; add the
dir to `webhost/kustomization.yaml`). Group shape:

```yaml
domainGroups:
  - name: <group>
    domains:
      - name: <domain>
        writablePaths:            # relative dirs the app writes to (caches/uploads)
          - <dir>                 # served read-only otherwise; blocked from HTTP (404)
    adminDomain: <group>-admin.gewis.nl   # optional; editor host (see step 4)
    oidc:
      groups:
        - "<OIDC group>"          # "CBC - Application Hosting Team (ADM)" is always allowed
```

Pin the chart `version:`. Commit + push to `main` (Flux deploys). Watch:
`kubectl get helmrelease -n flux-system <release>` and `kubectl get pods -n <ns>`.
The caddy pods being `Running` confirms the `seed-writable` init created any
`writablePaths` mountpoints.

## 2. Copy the files (once the release is running)

Pull from the old server with the root rsync, **excluding runtime cache dirs**
(they regenerate into the writable mounts; copying them is dead weight):

```sh
rsync -a --rsync-path='sudo -n /usr/bin/rsync' \
  --exclude='<cachedir>' \
  cbclocal@gewiswebhost:/home/<user>/public_html/ /tmp/stage/
```

Adapt as needed before transfer:
- **Writes:** an app's relative writes (e.g. `./cache/...`) only work if that dir is a
  declared `writablePath`. If the app writes to a subdir it doesn't create, add
  `@mkdir(dirname($f), 0775, true)`. For multi-replica safety, write atomically
  (temp file + `rename()`), since `file_put_contents` isn't atomic on NFS.
- **`.htaccess`:** drop them (Caddy ignores them). Re-implement any *live* redirects
  as a tiny `path/index.php` doing `header('Location: …', true, 301)`.

Transfer into the PVC via a throwaway pod that mounts the data PVC:

```sh
kubectl apply --validate=false -f - <<'EOF'
apiVersion: v1
kind: Pod
metadata: { name: xfer, namespace: <ns> }
spec:
  restartPolicy: Never
  securityContext: { runAsUser: 0 }
  containers:
    - name: t
      image: dunglas/frankenphp:1-php8.5-alpine    # has php for `php -l`
      command: ["sh","-c","sleep 1800"]
      volumeMounts: [{ name: data, mountPath: /data }]
  volumes: [{ name: data, persistentVolumeClaim: { claimName: <release>-static-webhost-data } }]
EOF

tar -C /tmp/stage -cf - . | kubectl exec -i -n <ns> xfer -- tar -C /data/site/<group>/<domain> -xf -
kubectl exec -n <ns> xfer -- php -l /data/site/<group>/<domain>/<entry>.php   # sanity
kubectl exec -n <ns> xfer -- chown -R 1000:1000 /data/site/<group>           # code-server uid
kubectl delete pod -n <ns> xfer
```

Clean up `/tmp/stage` locally afterwards (it may hold secrets — tokens, cookies).

## 3. Verify before touching DNS

From a caddy pod (or any pod that can reach Traefik's ClusterIP), curl Traefik with
the real Host/SNI — this exercises the whole Traefik → caddy path with the real cert:

```sh
curl --resolve <domain>:443:<traefik-clusterIP> https://<domain>/      # 200 + ssl_verify=0
```
Check the app's pages return 200, that PHP executes, and that `writablePaths` are
**blocked from HTTP** (`/<writablepath>/...` → 404).

## 4. Go-live: WAF, DNS, OIDC

GEWIS ingress is: **client → FortiWeb WAF (`waf.gewis.nl`) → Traefik (`websecure`) → app**.
Traefik holds the default `*.gewis.nl` cert, so TLS for any `*.gewis.nl` host just works.

- **Site DNS:** `<domain>` resolves (via `waf.gewis.nl`) to the WAF. If the domain was
  already a webhost, the DNS is unchanged — you only re-point its **WAF backend** to the
  cluster. New domains also need a **FortiWeb server-policy** (WAF is university-managed).
- **The editor must bypass the WAF.** The WAF mangles code-server's service-worker /
  web-worker traffic, so `/admin` (served under a stripped subpath through the WAF) loses
  syntax highlighting and errors out. Fix: set the group's **`adminDomain`** and point its
  DNS at a **campus-direct server** (the non-WAF ingress). code-server is then served at the
  **root** of that host, which both bypasses the WAF and avoids the subpath service-worker
  problem. That host is campus-only by virtue of the campus-direct path; OIDC gates it on top.
- **OIDC:** the `traefik-auth` Keycloak client must allow the editor's redirect URI
  `https://<adminDomain>/admin/oidc/callback`.

## Gotchas

- **Read-only tree:** PHP can only write under `writablePaths`. `/tmp` is writable but
  per-pod and ephemeral.
- **Multi-replica:** PHP sessions default to `/tmp` and aren't shared across replicas.
  Login-based apps need sticky sessions or shared session storage.
- **Real client IP:** PHP sees the visitor in `$_SERVER['REMOTE_ADDR']` only because of
  `caddy.trustedProxies` (`private_ranges`) + the chart's `php_server { env REMOTE_ADDR
  {client_ip} }`. (Traefik's `trusted_proxies` alone does NOT fix FrankenPHP's REMOTE_ADDR.)
- **Mail stays on the old box** — `Maildir`, `mail.*` aliases, and `ssl.*` are not migrated.
- **Secrets in site files** (API tokens, cookies) land on the editor-writable volume
  (OIDC-gated). Note them; they're as exposed as they were before, just now browser-editable.

## Worked example — `abc` (done 2026-06-24)

- One domain `abc.gewis.nl`; tools `aoc_poster` (AoC leaderboard) + `pr_poster` (GitHub PRs),
  both PHP that write a cache → `writablePaths: [aoc_poster/cache, pr_poster/cache]`.
  `aoc_poster/data.php` patched: `@mkdir` + atomic temp-rename write.
- `charts.php` is campus-only via `REMOTE_ADDR` — works thanks to `trustedProxies`.
- `.htaccess` files dropped (the 3 redirects on the *old* box were already dead — 404).
- Editor at `abc-admin.gewis.nl` (campus-direct, WAF-bypassing).
- Known: the AoC `session` cookie in `aoc_poster/data.php` expires periodically and must be
  refreshed (log into adventofcode.com → copy the `session` cookie).
