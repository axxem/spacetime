I'll convert your file to a Swarm-ready version.

Your file needed a few real changes beyond the usual cleanup. The new version is in **docker-stack.yml**.

**What changed and why**

- **Traefik uses the Swarm provider.** In Traefik v3, `--providers.docker` doesn't see Swarm services, so it's now `--providers.swarm`. I also set `--providers.swarm.network=spacetime` so Traefik knows which network to reach containers on.
- **Labels moved under `deploy.labels`.** Traefik reads service-level labels in Swarm, not container labels, so labels in the old spot would be silently ignored and every route would 404.
- **The dashboard has a dummy port label.** The Swarm provider requires a port on every enabled service, even for `api@internal`.
- **A named overlay network (`spacetime`)** is attached to all services, so the name doesn't depend on what you call the stack.
- **Traefik's ports 80 and 443 use `mode: host`,** which keeps real client IPs in your logs instead of the routing mesh's internal IP.
- **Mongo, Postgres, Server, and Traefik are pinned to the manager node** because they use local bind mounts (and Traefik needs a manager's Docker socket). The web frontends and agent can run anywhere.
- **Removed:** `container_name`, `restart`, `pull_policy`, and `depends_on`. Restart behavior moved to `deploy.restart_policy`.
- `./letsencrypt` is now `${DATA}/letsencrypt`, for consistency with your other volumes.

**Deploying**

```bash
docker swarm init

# Swarm won't create bind-mount folders for you
mkdir -p "${DATA:-.}"/{letsencrypt,mongo,postgres,server}

# stack deploy doesn't read .env, so export it into the shell first
set -a; . ./.env; set +a

docker stack deploy -c docker-stack.yml spacetime
```

Use an absolute path for `DATA` in `.env`, such as `/srv/spacetime`. To update to a new `latest` image later, run the same `docker stack deploy` command again. Swarm checks the registry and rolls out any changed images.

**Things to know**

- **`server` must retry its database connection.** Swarm ignores `depends_on`, so on first boot `server` may start before Postgres is ready. If your app exits instead of retrying, the restart policy will keep restarting it until the database is up. That works, but it's noisy in the logs.
- **The direct ports (8010, 3000, 3002, 3005, 3006, 8030) bypass Traefik and HTTPS.** In Swarm they're also published on every node. If you only need them for debugging, consider removing them.
- **Your API keys could become Swarm secrets** instead of environment variables, but your agent would then need to read them from `/run/secrets/...`. That's worth doing if this server is shared.