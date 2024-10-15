# Docker Registry

The [Docker Registry](https://distribution.github.io/distribution/about/)
program can be self-hosted, and one of the storage backends it supports is S3.
You can use Tigris instead of S3 to pull images as fast as possible.

## Usecase

Many times you can get away with the limitations of a public docker registry or
the one that your cloud provider of choice offers. When using Docker images for
AI workloads, you can frequently run into fundamental limits of your registry.
This means you need to ship models separately from your inference code, which
can make cold start times a lot longer.

Hosting your own registry on top of Tigris means you can benefit from Tigris'
[globally distributed pull-through caching architecture](/docs/overview/),
enabling your workloads to start quickly no matter where they are in the world.

## Getting started

At a high level, here's what you need to do:

1. Write your registry configuration (.env or YAML)
2. Start your registry somewhere
3. Push/pull images to test

For a full end-to-end example, check out the
[Registry example in tigrisdata-community](https://github.com/tigrisdata-community/docker-registry).

You can create this in any cloud you want (including self-hosting it on top of a
Kubernetes cluster), but this blueprint will be using [fly.io](https://fly.io)
as an example. The important parts are all in the environment variable
configuration.

Create a new configuration folder using `fly launch`:

```shell
mkdir -p registry
cd registry
fly launch --no-deploy --from=https://github.com/tigrisdata-community/docker-registry
```

When asked, copy the configuration to the new app. You won't need to tweak the
settings.

### Registry configuration

The Docker registry has multiple storage driver implementations. Tigris is
compatible with the
[s3 backend](https://distribution.github.io/distribution/storage-drivers/s3/).

[Create a new bucket](/docs/buckets/create-bucket):

```shell
fly storage create
```

Let Tigris pick a bucket name and then copy the secrets to your notes. You
should get output like this:

```text
Your Tigris project (adjective-noun-1234) is ready. See details and next steps with: https://fly.io/docs/reference/tigris/

Setting the following secrets on tigris-registry:
AWS_ACCESS_KEY_ID: tid_AzureDiamond
AWS_ENDPOINT_URL_S3: https://fly.storage.tigris.dev
AWS_REGION: auto
AWS_SECRET_ACCESS_KEY: tsec_hunter2hunter2hunter2
BUCKET_NAME: delicate-sea-639
```

Copy the variables over according to this table:

| Fly storage create secret |  →  | Environment variable in `.env`  |
| ------------------------: | :-: | :------------------------------ |
|             `BUCKET_NAME` |  →  | `REGISTRY_STORAGE_S3_BUCKET`    |
|       `AWS_ACCESS_KEY_ID` |  →  | `REGISTRY_STORAGE_S3_ACCESSKEY` |
|   `AWS_SECRET_ACCESS_KEY` |  →  | `REGISTRY_STORAGE_S3_SECRETKEY` |

```shell
# Change these settings
REGISTRY_STORAGE_S3_BUCKET=bucketName
REGISTRY_STORAGE_S3_ACCESSKEY=tid_accessKey
REGISTRY_STORAGE_S3_SECRETKEY=tsec_secretAccessKey

# Don't change these settings
REGISTRY_STORAGE=s3
REGISTRY_STORAGE_S3_REGION=auto
REGISTRY_STORAGE_S3_REGIONENDPOINT=https://fly.storage.tigris.dev
REGISTRY_STORAGE_S3_FORCEPATHSTYLE=false
REGISTRY_STORAGE_S3_ENCRYPT=false
REGISTRY_STORAGE_S3_SECURE=true
REGISTRY_STORAGE_S3_V4AUTH=true
REGISTRY_STORAGE_S3_CHUNKSIZE=5242880
REGISTRY_STORAGE_S3_ROOTDIRECTORY=/
```

Write this to `.env` in your current working directory.

### Starting your registry

Then import the secrets into your app:

```shell
fly secrets import < .env
```

And add a randomly generated HTTP secret:

```shell
fly secret set REGISTRY_HTTP_SECRET=$(uuidgen || uuid)
```

Now you can deploy it:

```shell
fly deploy --no-ha
```

Once it’s up, you can push and pull like normal.

### Push/pull images to test

Now that we have a private Docker registry, let's give it a whirl with smollm.
Clone the example models repo and build smollm:

```shell
docker build -t your-registry.fly.dev/models/smollm/135m:q4 https://raw.githubusercontent.com/tigrisdata-community/models-in-docker/refs/heads/main/smollm/Dockerfile
```

Then push it to your registry:

```shell
docker push your-registry.fly.dev/models/smollm/135m:q4
```

## [SUGGESTED] Securing the registry

Right now this registry is open for anyone in the world to pull from and push to
it. This is not ideal. In lack of a better option, we're going to use htpasswd
authentication for the registry. Depending on the needs of your threat model,
you may not need to do this (eg: if your registry is deployed to a private
network and you don't have security compliance requiring segmentation of
workloads).

In order to get this set up, we need to shut down the registry for a moment:

```shell
fly scale count 0
```

And then start an ephemeral Alpine Machine with the htpasswd volume mounted:

```shell
fly machine run --shell --command /bin/sh alpine
```

Once you're in, install the apache2-utils package to get the htpasswd command:

```shell
apk -U add apache2-utils
```

Then create a password for your administrator user:

```shell
htpasswd -B -c /data/htpasswd admin
```

It'll ask you for your password and write the result to `/data/htpasswd`. Repeat
this for every person or bot that needs access to the registry. Make sure to
save these passwords to a password manager as you will not be able to recover
them later.

Once you're done, exit out of the Alpine Machine and configure these secrets so
that the Docker registry server will use that shiny new htpasswd file:

```shell
fly secrets set REGISTRY_AUTH=htpasswd REGISTRY_AUTH_HTPASSWD_REALM=basic-realm REGISTRY_AUTH_HTPASSWD_PATH=/data/htpasswd
```

Then turn the registry back on:

```shell
fly scale count 1
```

Finally, log into your registry with docker login:

```shell
docker login tigris-registry.fly.dev -u admin
```

Paste your password, hit enter, and you're in\!

Here's what people without authentication will see:

```text
$ docker run --rm -it tigris-registry.fly.dev/alpine
Unable to find image 'tigris-registry.fly.dev/alpine:latest' locally
docker: Error response from daemon: failed to resolve reference "tigris-registry.fly.dev/alpine:latest": pull access denied, repository does not exist or may require authorization: authorization failed: no basic auth credentials.
See 'docker run --help'.
```
