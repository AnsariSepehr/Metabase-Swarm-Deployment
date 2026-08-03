# Metabase on Docker Swarm (behind nginx, single-node service)

This repo documents how we deployed [Metabase](https://www.metabase.com/) as a
single-replica service on an existing Docker Swarm cluster, with its own
dedicated Postgres metadata database, reachable only through our existing
nginx reverse proxy (no published ports on the host).

It was built and verified on a **stage** environment first, then replicated
to **production**. Both environments follow the same pattern; only network
names, node hostnames, and domains differ.

> All hostnames, domains, IPs, and credentials in this repo are placeholders.
> Replace anything in `<ANGLE_BRACKETS>` or marked `REPLACE_WITH_*` before use.

## Contents

```
.
├── README.md                                  <- you are here
├── compose/
│   ├── docker-compose.metabase.stage.yml
│   └── docker-compose.metabase.prod.yml
├── nginx/
│   ├── metabase-server-block.stage.conf
│   └── metabase-server-block.prod.conf
└── docs/
    ├── connection-strings.md
    └── troubleshooting.md
```

## Architecture

```
                         ┌────────────────────────┐
   Browser  ── HTTPS ──▶ │  nginx (swarm, global)  │
                         │  listens :80 / :443     │
                         └───────────┬─────────────┘
                                     │ overlay network
                                     │ (service-name DNS,
                                     │  no published port)
                         ┌───────────▼─────────────┐
                         │   metabase (container)  │
                         │   listens :3000 (internal)
                         └───────────┬─────────────┘
                                     │ same overlay network
                         ┌───────────▼─────────────┐
                         │  metabase-db (Postgres) │
                         │  Metabase's own app DB  │
                         └──────────────────────────┘

                         ┌──────────────────────────┐
                         │  Citus / Patroni cluster │  ◀── added as a
                         │  (coordinators + workers)│      *data source*
                         │  same overlay network    │      inside Metabase,
                         └──────────────────────────┘      not at deploy time
```

Two databases are involved and it's important to keep them mentally separate:

1. **Metabase's own app database** (`metabase-db` in the compose file) — stores
   dashboards, users, saved questions, permissions, etc. This is created and
   wired up automatically by the compose file below.
2. **Your actual data** (the Citus/Patroni cluster) — this is what you *point
   Metabase at* after first login, as a data source, via the Admin UI. It is
   **not** part of the compose file and is never modified by Metabase itself
   (read/query access only, assuming you follow the least-privilege advice in
   [`docs/connection-strings.md`](docs/connection-strings.md)).

## Prerequisites

- An existing Docker Swarm cluster with at least one worker/manager node.
- An existing overlay network shared with your database stack (`etcd-net` on
  stage, `etcd` on production in this setup) — created externally, not by
  this compose file.
- An existing nginx service that is *already attached to that same overlay
  network* (this is what allows nginx to reach `metabase_metabase:3000` by
  Swarm service-name DNS, with zero published ports).
- A DNS record for the Metabase hostname you choose, pointed at the same IP
  as your other reverse-proxied services.
- A valid TLS cert/key already used by your other nginx server blocks (we
  reused the existing wildcard/cert pair — no new certificate needed).

## Deployment steps

1. **Pick the compose file for your environment** (`compose/docker-compose.metabase.stage.yml`
   or `compose/docker-compose.metabase.prod.yml`) and fill in the placeholders:
   - `<STAGE_NODE_1>` / `<PROD_NODE_1>` — the swarm node hostname you want
     Metabase pinned to (see [Why pin to a single node](#why-pin-to-a-single-node) below).
   - `REPLACE_WITH_STRONG_SECRET` — set a real password for the `metabase`
     Postgres role (used twice: once for `metabase-db`'s `POSTGRES_PASSWORD`,
     once for `metabase`'s `MB_DB_PASS` — **they must match**).

2. **Deploy the stack:**
   ```bash
   docker stack deploy -c docker-compose.metabase.<env>.yml metabase
   ```

3. **Confirm both services are healthy:**
   ```bash
   docker stack services metabase
   docker service ps --no-trunc metabase_metabase
   docker service ps --no-trunc metabase_metabase-db
   ```
   Both should show `1/1` replicas and `Running`.

4. **Add the nginx server block** from `nginx/metabase-server-block.<env>.conf`
   into your existing `nginx.conf`, filling in your real domain and cert
   paths. Also add the hostname to your existing HTTP→HTTPS redirect
   `server_name` list (see the comment at the top of each `.conf` file).

5. **Test and reload nginx:**
   ```bash
   docker exec -it $(docker ps -q -f name=<your_nginx_service>) nginx -t
   docker service update --force <your_nginx_service>
   ```

6. **Verify end-to-end** — see the [Verification checklist](#verification-checklist) below.

7. **Finish first-run setup** by visiting `https://<your-metabase-domain>` in
   a browser, creating the admin account, and either connecting your real
   data source now or clicking **Skip** and adding it later from
   **Admin Settings → Databases** (recommended — gives you the fuller
   connection UI and lets you use a scoped-down DB role, see
   [`docs/connection-strings.md`](docs/connection-strings.md)).

### Why pin to a single node

Metabase's own app database (`metabase-db`) stores its data on a host bind
mount (`/ssd-store/.../metabase/pgdata`). Docker Swarm doesn't move bind-mount
data between nodes, so both `metabase` and `metabase-db` are pinned to the
same node via `deploy.placement.constraints`. If that node goes down, the
stack won't reschedule elsewhere with its data intact — for a single-node
analytics tool this was judged an acceptable tradeoff, but if you need HA,
put `pgdata` on shared/networked storage (NFS, Ceph, etc.) instead of a local
path, and drop the node constraint.

### Why no published port

Early in this deployment we briefly published `3000:3000` directly on the
host to test reachability before nginx was wired up. Once the nginx server
block was confirmed working, the `ports:` block was removed entirely. Since
nginx and Metabase share the same Swarm overlay network, nginx reaches
Metabase via Swarm's internal service DNS (`metabase_metabase:3000`) — this
works whether or not the port is published to the host. Removing the
publish closes off direct, unauthenticated access to Metabase from anything
that can reach the host's IP, leaving nginx (with TLS, and any auth you layer
on top later) as the only path in.

## Verification checklist

Run through these in order — each one narrows down where a problem is if
something doesn't work.

- [ ] `docker stack services metabase` shows both services at `1/1`.
- [ ] `docker service logs metabase_metabase --tail 100` shows
      `Metabase Initialization COMPLETE` with no fatal errors. (An `ERROR`
      about the bundled sample database failing to load is benign and can be
      ignored.)
- [ ] `docker service logs metabase_metabase-db --tail 50` shows
      `database system is ready to accept connections`.
- [ ] From inside the nginx container, `curl -v http://metabase_metabase:3000/api/health`
      succeeds — confirms container-to-container reachability over the
      overlay network, independent of nginx config.
- [ ] `nginx -t` inside the nginx container reports syntax OK, and the
      service was reloaded (`docker service update --force <nginx_service>`)
      after editing `nginx.conf`.
- [ ] `dig +short <your-metabase-domain>` resolves to the expected IP.
- [ ] `https://<your-metabase-domain>` loads the Metabase setup wizard (first
      run) or login page (subsequent runs).
- [ ] After adding your real data source: a test query against it in
      Metabase's query builder returns rows.

See [`docs/troubleshooting.md`](docs/troubleshooting.md) for what to do when
any of the above fails.

## Moving from stage to production

The two compose files and nginx blocks differ only in:

| | Stage | Production |
|---|---|---|
| Overlay network(s) | `etcd-net` only | `etcd` **and** `nginx-net` |
| Node constraint | single stage node | single prod node |
| Domain | `metabase-<stage>.example.com` | `metabase.example.com` |

The extra `nginx-net` attachment in production reflects how that
environment's nginx service is wired (it sits on both `etcd` and `nginx-net`,
whereas stage's nginx only needed `etcd-net`). Attach your Metabase services
to whichever network(s) your own nginx service is actually on — that's the
only thing that determines reachability, not the network's name.
