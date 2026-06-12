# skynet — CUDA/GPU image builder for GHCR

This is a fork of [**jitsi/skynet**](https://github.com/jitsi/skynet) repurposed to do one
thing: automatically build the **default CUDA/GPU** image of skynet and publish it to this
repo's **GitHub Container Registry (GHCR)** namespace, so you can `docker pull` a ready-made
GPU image instead of building the multi-GB CUDA + PyTorch image on your own server.

> Looking for the skynet application docs themselves? See upstream:
> <https://github.com/jitsi/skynet>.

## What this image is, and why

`skynet` is the [Faster-Whisper](https://github.com/SYSTRAN/faster-whisper) streaming
speech-to-text server used by **Jitsi Meet** for live transcriptions (the
`streaming_whisper` module).

Upstream's `Dockerfile` defaults its build args to CUDA bases:

```dockerfile
ARG BASE_IMAGE_BUILD=nvidia/cuda:12.2.2-cudnn8-devel-ubuntu22.04
ARG BASE_IMAGE_RUN=nvidia/cuda:12.2.2-cudnn8-runtime-ubuntu22.04
```

So a plain build of that `Dockerfile` **with no build-arg overrides** produces the GPU image.
The images jitsi publishes to Docker Hub are CPU-only; this repo specifically builds the
**default CUDA** variant for NVIDIA GPU hosts. The build itself does not need a GPU (only the
runtime does), so it runs on a standard GitHub-hosted runner.

The workflow ([`.github/workflows/build.yml`](.github/workflows/build.yml)) always checks out
**upstream `jitsi/skynet`** (never this fork's copy) at a ref you control, so the published
image tracks real upstream source and not fork drift.

## Pull reference

> Replace **`OWNER`** below with your GitHub account/org that owns this repo
> (lowercase — GHCR image names must be lowercase). For this repo that is your GitHub login.

```bash
docker pull ghcr.io/OWNER/skynet:cuda
```

Tags published by the workflow:

| Tag                          | Meaning                                                        |
| ---------------------------- | ------------------------------------------------------------- |
| `cuda`                       | Latest CUDA/GPU build — **use this for your homelab**.        |
| `latest`                     | Alias of the most recent build (also CUDA).                   |
| `cuda-YYYYMMDD-<shortsha>`   | Immutable version tag: build date + upstream skynet short SHA. |

The version tag lets you pin/rollback to an exact upstream commit, e.g.
`ghcr.io/OWNER/skynet:cuda-20260612-a1b2c3d`.

## GHCR packages are PRIVATE by default

A package pushed to GHCR starts out **private**, so a homelab host cannot pull it without
credentials. Pick one of the two options:

### Option A — make the package public (no auth needed to pull)

1. After the first successful run, open your repo's package:
   `https://github.com/users/OWNER/packages/container/skynet` (or via the repo's right-hand
   **Packages** sidebar).
2. **Package settings** → **Danger Zone** → **Change visibility** → **Public** → confirm.

Now any host can pull `ghcr.io/OWNER/skynet:cuda` with no login.

### Option B — keep it private and log in on the pulling host

Create a Personal Access Token (classic) with the **`read:packages`** scope, then on the host
that pulls the image:

```bash
echo "<YOUR_PAT>" | docker login ghcr.io -u OWNER --password-stdin
```

After that, `docker pull ghcr.io/OWNER/skynet:cuda` works on that host. (For an unattended
server, store the token via `docker login` once; Docker persists it in `~/.docker/config.json`.)

## docker-compose snippet for your Jitsi stack

Ready to paste into your Jitsi `docker-compose.yml`. Requires the
[NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
on the host.

```yaml
services:
  skynet:
    image: ghcr.io/OWNER/skynet:cuda   # <-- replace OWNER
    restart: unless-stopped
    ports:
      - "8000:8000"
    environment:
      ENABLED_MODULES: streaming_whisper
      BYPASS_AUTHORIZATION: "1"
      WHISPER_MODEL_NAME: large-v3
      WHISPER_MODEL_PATH: /models/streaming-whisper
      BEAM_SIZE: "1"
    volumes:
      - ./models:/models
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: 1
              capabilities: [gpu]
```

The `/models` volume persists the downloaded Whisper weights (`large-v3` is several GB) so they
aren't re-downloaded on every container start. The model files land in
`/models/streaming-whisper` to match `WHISPER_MODEL_PATH`.

## One-time setup

This fork ships the workflow, but you have to enable and seed it once:

1. **Enable Actions on the fork.** Repo → **Settings → Actions → General** → allow workflows
   (forks have Actions disabled until you opt in). Under **Workflow permissions**, confirm
   **Read and write permissions** is allowed (the workflow also declares `packages: write`
   itself, so this is just a belt-and-suspenders check).
2. **Run the build once.** Repo → **Actions** → *Build & publish skynet CUDA image* → **Run
   workflow** (leave `skynet_ref` = `master`). First run takes a while (multi-GB CUDA build).
3. **Set the package visibility.** After it finishes, the `skynet` package exists under your
   account — make it **Public** (Option A) or set up `docker login` on your host (Option B),
   per the section above.

## Triggering a rebuild when skynet updates

You normally don't have to do anything — the workflow keeps the image current:

- **Weekly schedule** (Mondays 05:25 UTC): re-checks upstream and rebuilds **only if
  something changed**.
- **Manual:** Actions → *Build & publish skynet CUDA image* → **Run workflow**. Set
  `skynet_ref` to any branch, tag, or commit SHA to build a specific upstream version
  (e.g. a release tag). A manual run always builds.
- **On workflow edits:** pushing changes to `build.yml` / `renovate.json` triggers a build so
  edits are validated.

### How "rebuild on external changes" works

Adapted from the pattern in
[`v3DJG6GL/nextcloud-fpm-nginx`](https://github.com/v3DJG6GL/nextcloud-fpm-nginx/tree/master/.github),
the scheduled run is gated by a `check` job that avoids pointless multi-GB rebuilds:

1. Resolves `jitsi/skynet@<ref>` to a commit SHA (`git ls-remote`, no clone).
2. Reads the labels off the currently-published `:cuda` image with `crane` (config only — no
   layer download).
3. **Builds only if** the upstream **source SHA changed**, **or** the **CUDA base-image digest
   changed**, **or** the published image is **older than 14 days** (so base-image security
   rebuilds still get picked up). The gate is **fail-open**: any lookup error builds rather
   than silently skipping.

Each build records what it built against as image labels
(`skynet.source.revision`, `skynet.base.digest`, `org.opencontainers.image.created`), which is
exactly what the next run compares against.

### Keeping the workflow itself up to date

[`.github/renovate.json`](.github/renovate.json) configures
[Renovate](https://docs.renovatebot.com/) to keep the workflow's pinned GitHub Actions and
tool versions current (grouped PRs, minor/patch auto-merged, majors gated on the Dependency
Dashboard). Enable it by installing the **Renovate GitHub App** on this repo
(<https://github.com/apps/renovate>) — no extra workflow needed. It's optional; the scheduled
build above is what keeps the *image* current regardless.

> Note: because the build always pulls upstream `jitsi/skynet` directly, you do **not** need to
> keep this fork's git contents in sync with upstream for the image to stay current — syncing
> the fork is purely cosmetic here.

## License

skynet is distributed under the Apache 2.0 License (see [`LICENSE`](LICENSE)). This fork only
adds CI to build and publish the upstream image.
