# Troubleshooting

Work through these roughly in order — each step isolates one layer
(container health → internal networking → nginx config → external
DNS/firewall) so you don't waste time guessing.

## "I deployed the stack but can't reach it via the domain at all"

### 1. Are both containers actually healthy?

```bash
docker stack services metabase
docker service ps --no-trunc metabase_metabase
docker service ps --no-trunc metabase_metabase-db
```

Both should read `1/1` and `Running`. If a task keeps restarting, check its
logs (see next step) — common causes are a bad `MB_DB_PASS`/`POSTGRES_PASSWORD`
mismatch between the two services, or a bind-mount path that doesn't exist
on the constrained node yet.

### 2. Check the logs of each service

```bash
docker service logs metabase_metabase --tail 100
docker service logs metabase_metabase-db --tail 50
```

What you want to see:
- `metabase-db`: `database system is ready to accept connections`
- `metabase`: eventually, `Metabase Initialization COMPLETE`

A one-line `ERROR ... Failed to load sample database: Not something with an
ID: nil` right after startup is **benign** — it's Metabase's bundled demo
dataset failing to load, unrelated to your actual setup. Everything else
proceeding normally after that line means the app started fine.

On first run, Metabase also prints the setup URL directly in the logs:
```
Please use the following URL to setup your Metabase installation:
http://0.0.0.0:3000/setup/
```
That confirms the app is serving on port 3000 internally, which is useful to
know before even touching nginx.

### 3. Test container-to-container reachability, bypassing nginx entirely

```bash
docker exec -it $(docker ps -q -f name=<your_nginx_service>) sh
# inside the nginx container:
curl -v http://metabase_metabase:3000/api/health
```

- **DNS resolution fails** → nginx and Metabase aren't actually on the same
  overlay network, despite both compose files listing it. Double check with:
  ```bash
  docker network inspect <network_name> | grep -A 3 metabase
  docker network inspect <network_name> | grep -A 3 <nginx_container_name>
  ```
  Both should show up as members of the same network with IPs in the same
  subnet.
- **Resolves but connection refused/timeout** → Metabase isn't listening
  where expected; re-check the container logs from step 2.
- **Succeeds** → the problem is downstream of here — nginx config or the
  external path (DNS/firewall), not container networking. Move to step 4.

### 4. Did nginx actually pick up the new config?

Editing `nginx.conf` on disk does nothing until nginx reloads. This is the
single most common cause of "I added the server block but nothing changed."

```bash
docker exec -it $(docker ps -q -f name=<your_nginx_service>) nginx -t
docker service update --force <your_nginx_service>
```

`nginx -t` catches syntax errors before you reload (a typo anywhere in the
file will stop the *entire* nginx config from loading, breaking every other
proxied service too — always run `-t` before forcing an update).

After forcing the update, check that access/error log files for the new
vhost actually exist:
```bash
ls -la /var/log/nginx/ | grep metabase
```
If they don't exist yet, nginx never served a request matching that
`server_name` — which usually means either the reload didn't happen, or the
`server_name`/DNS don't actually match what your browser is requesting.

### 5. DNS

```bash
dig +short <your-metabase-domain>
```

Compare the result against one of your other, already-working
`*.example.com` entries pointed at the same server. If it doesn't resolve at
all, or resolves to the wrong IP, that's outside of Docker/nginx entirely —
check your DNS provider/zone file. DNS propagation can also take a few
minutes depending on the record's TTL.

### 6. Host firewall / cloud security group

If you were testing via a directly published port earlier (e.g. `3000:3000`)
and it worked from `localhost` on the server but not from your own machine,
that's a firewall/security-group issue rather than anything Docker-related:

```bash
# on the server itself:
curl -v http://localhost:3000        # should work if the app is healthy
curl -v http://<server-public-ip>:3000   # from an external machine

iptables -L -n | grep 3000
ufw status | grep 3000
```

This class of issue goes away entirely once you drop the published port and
go through nginx's existing 80/443, since those are presumably already open.

### 7. Sanity-check node placement

```bash
docker node ls
```

Confirm the hostname you used in `deploy.placement.constraints` actually
matches a node in `docker node ls`'s `HOSTNAME` column. If Swarm can't
satisfy a constraint, the task sits in `Pending` forever rather than failing
loudly — so if a service *is* running, its constraint was satisfiable, but
it's still worth confirming it landed on the node you actually intended
(e.g., watch out for internal DNS aliases or short-vs-long hostnames causing
confusion between what you typed and what `docker service ps` reports back).

## "It loads, but the data source connection fails"

- Confirm you're connecting to a coordinator (or the haproxy in front of
  them), never a worker node directly — see
  [`connection-strings.md`](connection-strings.md).
- If the error mentions SSL/TLS, toggle Metabase's "Use a secure connection
  (SSL)" option — Postgres servers configured with `sslmode: allow` will
  usually negotiate either way, but some drivers default differently than
  you'd expect.
- Confirm the specific coordinator/proxy you're pointing at is currently
  healthy — in a Patroni cluster, roles can change after failover events.

## "The site loads over HTTP but not HTTPS" (or vice versa)

Check that the hostname was added to **both**:
1. The HTTP→HTTPS redirect block's `server_name` list.
2. Its own dedicated `listen 443 ssl` block with a valid `server_name`
   matching exactly what's in DNS (typos or trailing differences here are a
   common miss).
