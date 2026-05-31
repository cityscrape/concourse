# Concourse Runner Host Mounts (RFC 140) MVP

This document details the Cityscape MVP implementation of [Concourse RFC 140](https://github.com/concourse/rfcs/pull/140), which allows Concourse tasks to mount specific host paths/devices into task containers. 

This is primarily used for GPU rendering on external desktop workers at Cityscape (e.g., passing `/dev/dri` and `/dev/kfd` into tasks).

## Usage in Pipelines

To request a host mount, add the `host_mounts` array to a `task` step in your pipeline YAML:

```yaml
jobs:
- name: gpu-render-job
  plan:
  - task: render-scene
    host_mounts:
      - host: /dev/dri
        container: /dev/dri
      - host: /dev/kfd
        container: /dev/kfd
    config:
      platform: linux
      image_resource:
        type: registry-image
        source: { repository: my-gpu-image }
      run:
        path: /bin/sh
        args:
          - -c
          - |
            ls -la /dev/dri
            ls -la /dev/kfd
```

> [!NOTE]
> If `container` is omitted, it defaults to the `host` path. Both paths must be absolute.

## Worker Configuration

For security, the worker strictly validates requested mounts. 
You **must** configure the worker with the `--allowed-host-mounts` flag, providing a regular expression of allowed host paths. 

If this flag is missing, the worker operates in a **fail-closed** mode and will reject any task requesting host mounts.

Example for allowing AMD/Linux GPU devices:
```bash
concourse worker ... --allowed-host-mounts='^/dev/(dri|kfd)(/.*)?$'
```

> [!IMPORTANT]
> The worker automatically wraps the provided regex with `^(?:%s)$` to enforce strict full-matching. Do not rely on partial substring matches.

## Building and Pushing Custom Images

This implementation is a fork specifically for Cityscape. You must build and push the custom Concourse images to the local Harbor registry.

```bash
# 1. Build the Concourse image
docker build -t harbor.lunari.studio/cityscape/concourse:latest .

# 2. Log in to the Harbor registry
docker login harbor.lunari.studio

# 3. Push the image
docker push harbor.lunari.studio/cityscape/concourse:latest
```
