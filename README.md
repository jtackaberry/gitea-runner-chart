# Helm Chart for Gitea Actions Runner

This Helm chart deploys Gitea's [act_runner](https://gitea.com/gitea/act_runner).

It's currently considered **experimental** as it hasn't been very widely tested.


## Quick Start

Because of its experimental status, the chart isn't published in a way that it can be
added as a Helm repo. Instead, currently you'll need to clone this repo and deploy from
local filesystem.

First, create a minimal `values.yaml`:

```yaml
runner:
  # This is the base URL of whatever your Gitea instance is
  gitea_url: https://gitea.example.com
  # Get the registration token from your Gitea instance
  token: whateveritis
```

You'll need to [retrieve the registration
token](https://docs.gitea.com/usage/actions/act-runner#obtain-a-registration-token) from your Gitea instance.

Then, install the chart:

```bash
git clone https://github.com/jtackaberry/gitea-runner-chart/ chart
helm upgrade --install -n gitea-actions gitea-runner chart -f values.yaml
```

Note that by default persistence is enabled, which ensures the runner has a stable
identity between restarts, and improves performance by maintaining a cache of pulled
docker images and downloaded actions.

See [values.yaml](values.yaml) for more information on how the chart can be configured.



## Securing the Token

The token is sensitive, so it's obviously not ideal to include it cleartext as in the
above example.  You have two options.

1. Precreate your own Kubernetes secret (with a key `token` containing the token value),
   and specify `runner.tokenSecretName` instead of `runner.token` as in the example; or
2. Use a Helm secrets plugin (I use and recommend [this
   one](https://github.com/jkroepke/helm-secrets))



## Deployment Security

Multiple different kinds of deployments are possible, each with different security tradeoffs.

| Deployment | Pros | Cons | Method |
|-|-|-|-|
| Root DinD with Docker exposed to jobs | Multi-platform Docker builds and related actions (e.g. [setup-qemu-action](https://github.com/docker/setup-qemu-action), [setup-buildx-action](https://github.com/docker/setup-buildx-action), etc.) Just Work | Privileged actions are directly accessible to jobs; full Kubernetes node takeover possible; must not host untrusted workloads! | Default configuration |
| Rootless DinD with Docker exposed to jobs | More secure: jobs can't spawn privileged containers, but docker builds still possible | Qemu-based cross-platform docker builds not possible; docker actions like setup-{qemu,buildx}-action will fail | Set `docker.rootless` to true
| Rootless DinD without Docker in jobs | Kubernetes node is fairly well insulated from attacks by jobs | Jobs can't build docker images | Set `docker.rootless` to true and `runner.config.container` to `null` to prevent use of the runner's Docker daemon by jobs
| Kata container with Root DinD and Docker exposed to jobs | Everything works as the first option and the Kubernetes node is protected from malicious jobs (although the runner is not) | Requires kata as a runtime class preconfigured on the nodes; Kata has some jank (which [prevents execing](https://github.com/kata-containers/kata-containers/issues/9701) into the runner's docker container, and [broken on arm64](https://github.com/kata-containers/kata-containers/issues/2291) when definining cpu limits to use more than 1 core) | Set `runtimeClassName` to `kata` (or whatever the kata runtime class name is in your cluster); set `docker.loopMountedDisk.enabled` to true (to enable [this workaround](https://github.com/kata-containers/kata-containers/blob/main/docs/how-to/how-to-run-docker-with-kata.md))
