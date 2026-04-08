# Ruby Project - InvisiRisk BAF Example Setup (Non-Dockerized)

This guide explains how to integrate the InvisiRisk BAF into an AWS CodeBuild pipeline that builds Ruby apps **directly on the build machine** — without Docker.

---

## How This Differs from the Dockerized Setup

| | Dockerized Build | Non-Dockerized Build |
|---|---|---|
| **What runs the app build** | `docker build` inside a container | `bundle install` directly on the CodeBuild machine |
| **How BAF intercepts traffic** | Custom BuildKit frontend (`--build-arg BUILDKIT_SYNTAX=...`) swaps into the Docker build process | BAF proxy is set up on the host; `bundle install` automatically routes through it |
| **Dockerfile needed?** | Yes | **No** |
| **`docker build` command needed?** | Yes | **No** |
| **BAF startup & cleanup** | Same | Same |

**In short:** The `pre_build` and `post_build` steps are identical in both setups.
The only difference is in the `build` phase — instead of running `docker build` with special BAF flags, you run `bundle install` directly, and the BAF proxy (already started in `pre_build`) automatically intercepts all gem download traffic.

---

## Prerequisites

Ensure the `API_URL` and `APP_TOKEN` environment variables are set in your CodeBuild project before applying these changes.

The CodeBuild environment image must have Ruby installed, or you can use `mise` (see `mise.toml`).

---

## Step 1: Modify `buildspec.yml`

### `pre_build` phase - start the BAF (same as dockerized):

```yaml
pre_build:
  commands:
    - echo "InvisiRisk startup script..."
    - curl $API_URL/pse/bitbucket-setup/pse_startup | bash  # Downloads and runs the BAF setup script.
    - . /etc/profile.d/pse-proxy.sh                          # Loads the proxy environment variables (including PSE_PROXY_IP).
```

### `build` phase - install gems directly on the host (NO docker build):

```yaml
build:
  commands:
    - echo "Installing Ruby dependencies with InvisiRisk BAF..."
    - gem install bundler -v 2.4.22
    - bundle config set --local without "development test private"
    - bundle install   # All gem downloads are automatically routed through the BAF proxy.
```

> **Why does this work without any extra flags?**
> The startup script in `pre_build` sets standard HTTP/HTTPS proxy environment variables on the machine.
> Tools like `bundler`, `gem`, `pip`, `npm`, etc. all automatically respect these standard proxy variables.
> So no special flags are needed — just run your normal build commands.

### `post_build` phase - run the cleanup script (same as dockerized):

```yaml
post_build:
  commands:
    - echo "Build complete!"
    - bash /tmp/pse_cleanup/cleanup.sh  # Sends build data to the InvisiRisk portal.
```

### Full `buildspec.yml` example:

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo "InvisiRisk startup script..."
      - curl $API_URL/pse/bitbucket-setup/pse_startup | bash
      - . /etc/profile.d/pse-proxy.sh

  build:
    commands:
      - echo "Installing Ruby dependencies with InvisiRisk BAF..."
      - gem install bundler -v 2.4.22
      - bundle config set --local without "development test private"
      - bundle install

  post_build:
    commands:
      - echo "Build complete!"
      - bash /tmp/pse_cleanup/cleanup.sh
```

---

## Required Environment Variables

The following environment variables must be set in the CodeBuild project:

| Variable    | Description                               |
|-------------|-------------------------------------------|
| `API_URL`   | https://app.invisirisk.com                |
| `APP_TOKEN` | APP token received from InvisiRisk portal |

> `PSE_PROXY_IP` is **not** set manually — it is automatically set by the startup script and loaded via `. /etc/profile.d/pse-proxy.sh`.

---

## Notes

- The BAF startup (`pre_build`) **must complete before** the `build` phase so that all gem download traffic is routed through the proxy.
- The cleanup script (`post_build`) should **always run**, even if the build fails.
- There is **no Dockerfile** in this setup — Ruby gems are installed directly onto the CodeBuild machine.
- The `Gemfile` is the same as in the dockerized setup — BAF works the same way regardless of how you install gems.

---

## Appendix: What the BAF Startup Script Does

When you run `curl $API_URL/pse/bitbucket-setup/pse_startup | bash`, it:

1. Downloads and starts the InvisiRisk PSE (Proxy Security Engine) on the build machine.
2. Installs the PSE CA certificate at `/etc/ssl/certs/pse.pem` so TLS traffic can be inspected.
3. Writes the proxy address to `/etc/profile.d/pse-proxy.sh` (which is why you need `. /etc/profile.d/pse-proxy.sh` immediately after).

After sourcing `pse-proxy.sh`, the following standard proxy environment variables are set on the machine:
- `http_proxy` / `HTTP_PROXY`
- `https_proxy` / `HTTPS_PROXY`

`bundler`, `gem`, and most other package managers automatically use these variables — no extra flags needed.

---

## Appendix: Dockerized vs Non-Dockerized — What Changed in `buildspec.yml`

**Dockerized `build` phase:**
```yaml
build:
  commands:
    - DOCKER_BUILDKIT=1 docker build --no-cache --platform linux/amd64 \
        -t $IMAGE_NAME:$CODEBUILD_BUILD_NUMBER \
        --build-arg BUILDKIT_SYNTAX=kkisalaya/pse-frontend:1.4.3 \
        --secret id=pse-ca,src=/etc/ssl/certs/pse.pem \
        --build-arg PSE_PROXY=http://${PSE_PROXY_IP}:3128 .
```

**Non-Dockerized `build` phase:**
```yaml
build:
  commands:
    - gem install bundler -v 2.4.22
    - bundle config set --local without "development test private"
    - bundle install
```

The special Docker flags (`BUILDKIT_SYNTAX`, `--secret`, `PSE_PROXY`) are **only needed for Docker builds** because Docker runs in an isolated environment that does not inherit the host proxy settings. Without Docker, `bundle install` runs directly on the host and automatically picks up the proxy that was configured in `pre_build`.
