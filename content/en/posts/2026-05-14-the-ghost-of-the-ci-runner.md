---
title: "The Ghost of the CI Runner"
date: 2026-05-14T10:00:00+00:00
series: ["symfony-to-the-cloud"]
part: 1
categories: [development]
tags: [symfony, cloud, storage, flysystem, minio, kubernetes, 12factor, ci]
description: "How a CI runner's home directory in production config revealed a storage architecture problem — and how Flysystem lazy adapters fixed it."
---

```dotenv
APP__COLD_STORAGE__FILESYSTEM_PATH="/home/jenkins-slave/share_media/media"
APP__COLD_STORAGE__FILESYSTEM_PATH_CACHE="/home/jenkins-slave/share_media/media/cache"
APP__COLD_STORAGE__RAW_IMAGE_PATH="/home/jenkins-slave/share_media/media_raw"
APP__SHARE_STORAGE__FILESYSTEM_PATH="/home/jenkins-slave/share_storage"
```

These lines were in the production `.env` of the media service. Not staging. Not a local override. Production, committed to the repository, read on every startup.

The paths end where you'd expect: `/media`, `/share_storage`. They start somewhere more surprising: `/home/jenkins-slave`, the home directory of a CI runner from an old Jenkins setup.

## How a runner's home directory ends up in production config

The platform had grown from a single machine. One server ran everything — the application, the CI runner, the database, the file storage. Files moved between the app and the CI system via NFS: a directory mounted on the same host, accessible to both the containers and the runner.

The path `/home/jenkins-slave/share_media` was where the NFS share landed on that machine. When the team migrated to Docker Compose, the containers inherited the NFS mount. The path made it into the `.env` because the application needed to know where to find files. Nobody changed it because it worked. The mount was still there. The path was valid. The application started. Files appeared where they should.

Three years later, nobody thought about it at all. It was just how the media path was configured.

## What kubectl apply found

The first `kubectl apply` for the media service ended with a pod stuck in CrashLoopBackOff. The container started. The entrypoint ran. The application tried to access `/home/jenkins-slave/share_media/media`. No such file or directory. No NFS mount. No runner.

The path didn't document a design decision. It documented the machine that happened to be running at the time the `.env` was written.

This is what [Factor IV](https://12factor.net/backing-services) of the twelve-factor app is warning against. Backing services — storage, queues, databases — should be attached resources, configured via URL or connection string, interchangeable between environments without code changes. A filesystem path on a shared host is not a backing service. It's a physical assumption about the machine. When the machine changes, the assumption fails.

## The path was the symptom

The obvious first step was removing the runner reference:

```dotenv
APP__COLD_STORAGE__FILESYSTEM_PATH="/share_media/media"
APP__SHARE_STORAGE__FILESYSTEM_PATH="/share_storage"
```

Cleaner. No more CI references in a production config. Still not right. The application still assumed a POSIX filesystem — either a volume mount or a directory on the node. In Kubernetes, a volume shared between multiple pods requires a `ReadWriteMany` PersistentVolumeClaim. Most storage providers don't support it. Those that do tend to be slow and expensive. And even where it works, you've replaced one shared filesystem assumption with another.

Renaming the path bought time. It didn't fix the problem.

The problem was that roughly twelve terabytes of images — originals and pre-generated derivatives in multiple formats — from multiple editorial brands — were treated as a directory. A directory can't be mounted cleanly across pods. A backing service can.

## Flysystem as the shape of the solution

The media service already had a Flysystem dependency. Three concrete adapters — local filesystem, AWS S3, Azure Blob — and one lazy adapter sitting on top:

```yaml
# config/packages/flysystem.yaml
flysystem:
    storages:
        media.storage.local:
            adapter: 'local'
            options:
                directory: "/"

        media.storage.aws:
            adapter: 'aws'
            options:
                client: 'aws_client_service'
                bucket: 'media'
                streamReads: true

        media.storage:
            adapter: 'lazy'
            options:
                source: '%env(APP__FLYSYSTEM_MEDIA_STORAGE)%'
```

All application code depends on `media.storage`. It doesn't know whether files live on the filesystem or in a cloud bucket. One environment variable determines which backend is active:

```dotenv
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.aws   # production
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.local  # local fallback still available
```

The path is gone. The filesystem assumption is gone. What remains is a service name — an attached resource in the twelve-factor sense, configurable without rebuilding the image.

The same pattern extends to the thumbnail cache. [LiipImagine](https://github.com/liip/LiipImagineBundle) generates resized images on demand; both the source originals and the generated cache go through separate Flysystem adapters:

```yaml
liip_imagine:
    loaders:
        default:
            flysystem:
                filesystem_service: 'media.storage'
        default_cache:
            flysystem:
                filesystem_service: 'media.cache.storage'
```

Two environment variables, two buckets. The full pipeline — receive upload, store original, generate thumbnail, cache it — is cloud-portable without touching a line of PHP.

What this doesn't cover is moving the data. The lazy adapter changes one environment variable. Getting twelve terabytes from an NFS mount into an S3 bucket is a different project — a migration window, double-write during cutover, verification that nothing was missed.

## What Minio makes possible in CI

Production uses S3. Local development uses [Minio](https://min.io/), an S3-compatible object store that runs in a Docker container. The AWS adapter talks to Minio locally and to S3 in production. The application doesn't notice the difference:

```dotenv
# local/CI
APP__FLYSYSTEM_MEDIA_STORAGE=media.storage.aws
APP__MINIO_ENDPOINT=http://minio:9000
APP__MINIO_ACCESS_KEY=minioadmin
APP__MINIO_SECRET_KEY=minioadmin
```

The same code, the same adapter, a different endpoint. No mocking, no special test paths, no environment-specific branches.

But the CI configuration goes one step further. The Minio image used in the pipeline isn't the standard upstream one — it's a custom image built with test fixtures preloaded:

```dockerfile
FROM minio/minio:latest
COPY tests/fixtures/ /fixtures_media/
```

Every CI run starts with a Minio instance that already contains the data the test suite expects. No setup script, no seed command, no "wait for fixtures to load" step before tests begin. The initial state of the test environment is part of the build artifact.

[Factor V](https://12factor.net/build-release-run) applied to test infrastructure: the environment state is built, versioned, and immutable. The CI pipeline builds the Minio image from the same source and at the same commit as the application image. The test fixtures and the code that exercises them are always in sync.

## The S3 tradeoff, honestly

S3 introduces a latency cost that local storage doesn't have. The first bytes of a file take 10 to 30 milliseconds to arrive from S3 — that's the documented first-byte latency for the service, not a measurement on this specific workload.

At 300 requests per second, the reasoning for accepting it was this: most reads hit already-generated thumbnails in the S3-backed cache, not the original files. A freshly uploaded image pays the cold-miss penalty once, on the first thumbnail request. Everything after that is a cache hit. Whether the actual tail latency under load bore that reasoning out required performance testing that was tracked separately — the architecture decision and the validation were decoupled.

The tradeoff was accepted: predictable behavior across multiple pods, no shared-state problems, a storage layer that scales without coordination. The full measurement story belongs in the load test report, not here.

## The ghost leaves

The path `/home/jenkins-slave` no longer appears in the configuration. But what it pointed to was a coupling that predated Docker, predated microservices, predated any conversation about cloud migration. The CI runner and the production application shared a filesystem because they lived on the same machine. Nobody designed it that way. It accumulated.

A `kubectl apply` error on a path that shouldn't have existed forced the question: why does this application assume a specific CI runner is present on the host? The answer was "because it always has." That's not a reason. It's a history.

Renaming the path was a paper fix. Flysystem's lazy adapter was the actual answer — not because it's more elegant, but because it makes the storage backend a decision that belongs to the environment, not to the application. The container starts, reads one variable, connects to whatever is at the other end. It doesn't know whether that's a bucket in a data center or a container on a laptop.

The runner's home directory is gone from the config. What replaced it is a service name. That's the difference.
