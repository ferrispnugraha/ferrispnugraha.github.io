# ferrispnugraha.github.io
This is the repository of my [personal website](https://ferrispnugraha.github.io).

Inspired from [al-folio](https://github.com/alshedivat/al-folio)

## Local development (Docker)

Two compose files are available, both serving the site at http://localhost:8080 with live reload:

| File | Image | When to use |
|------|-------|-------------|
| `docker-compose.yml` | Prebuilt from Docker Hub (`ferrispnugraha/personal`) | Fast start, no build step |
| `docker-local.yml` | Built from the local `Dockerfile` (`al-folio-local-docker`) | After changing `Gemfile`/`Dockerfile`, or when the prebuilt image is out of date |

### Start (build & run)

```bash
# Fast path: prebuilt image
docker compose up -d

# When deps changed (Gemfile, plugins) or image is stale: rebuild from source
docker compose -f docker-local.yml up --build -d
```

The server only starts after the initial build completes (~20–30s). Verify it is ready:

```bash
docker ps -a --filter name=jekyll                      # container should be "Up" with 0.0.0.0:8080->8080
docker logs ferrispnugrahagithubio-jekyll-1 --tail 30  # prebuilt image; use ...-jekyll_custom-1 for docker-local.yml
# ready when you see: Server address: http://0.0.0.0:8080
```

### Stop

The stop command must match the file used to start — each creates its own container
(`...-jekyll-1` vs `...-jekyll_custom-1`), and using the wrong one leaves the other
running and holding port 8080:

```bash
docker compose down                        # stops the prebuilt-image container (started via docker-compose.yml)
docker compose -f docker-local.yml down    # stops the locally-built container (started via docker-local.yml)
```

### Deploy

Automated via GitHub Actions (`.github/workflows/deploy.yml`): pushing to `main`
triggers `bundle exec jekyll build` (Ruby 3.2.2) and deploys `_site/` to GitHub
Pages via `JamesIves/github-pages-deploy-action`. No manual docker step is needed
for deployment — docker is for local testing only.

In case of deployment or workflow issue, please refer to al-folio repository and
[this issue](https://github.com/alshedivat/al-folio/issues/1398)
