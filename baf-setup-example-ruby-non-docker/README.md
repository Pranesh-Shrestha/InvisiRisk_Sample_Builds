# Ruby Project - InvisiRisk BAF Example Setup

This guide explains how to integrate the InvisiRisk BAF into your AWS CodeBuild pipeline. This setup assumes an Ubuntu runner where Ruby gems are installed **directly on the build machine** (no Docker).

## Prerequisites

Ensure the `API_URL` and `APP_TOKEN` environment variables are set in your CodeBuild project before applying these changes.

---

## Step 1: Modify `buildspec.yml`

Add the BAF startup and cleanup steps to your `buildspec.yml`, and run your `bundle install` command in the `build` phase.

### `pre_build` phase - start the BAF:

```yaml
pre_build:
  commands:
    - echo "InvisiRisk startup script..."
    - curl $API_URL/pse/bitbucket-setup/pse_startup | bash # Download and execute the BAF setup script.
    - . /etc/profile.d/pse-proxy.sh # Source the environment variables set by the setup script.
```

### `build` phase - install your Ruby gems ([see what changed](#appendix-what-changed-in-the-bundle-install-command)):

```yaml
build:
  commands:
    - echo "Installing Ruby dependencies with InvisiRisk BAF..."
    - gem install bundler -v 2.4.22
    - bundle config set --local without "development test private"
    - bundle install
```

### `post_build` phase - run the cleanup script:

```yaml
post_build:
  commands:
    - echo "Build complete!"
    - bash /tmp/pse_cleanup/cleanup.sh # Sends build data to the InvisiRisk portal.
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

| Variable     | Description                               |
|--------------|-------------------------------------------|
| `API_URL`    | https://app.invisirisk.com                |
| `APP_TOKEN`  | APP token received from InvisiRisk portal |

---

## Notes

- The BAF startup must complete before the `build` phase so that all network traffic during gem installation is routed correctly.
- `PSE_PROXY_IP` is set automatically by the startup script and sourced via `/etc/profile.d/pse-proxy.sh`.
- The cleanup script in `post_build` should always run, even if the build fails.

---

## Appendix: What Changed in the `buildspec.yml`


The key change is the addition of the BAF startup and cleanup around your existing build commands:

| Addition | What it does |
|---|---|
| `curl $API_URL/pse/bitbucket-setup/pse_startup \| bash` | Downloads and starts the InvisiRisk PSE proxy on the build machine, and installs the PSE CA certificate so that HTTPS traffic can be inspected. |
| `. /etc/profile.d/pse-proxy.sh` | Sources the proxy environment variables set by the startup script, so that `gem` and `bundler` automatically route all traffic through the BAF. |
| `bash /tmp/pse_cleanup/cleanup.sh` | Runs after the build to send the collected dependency data to the InvisiRisk portal. |
