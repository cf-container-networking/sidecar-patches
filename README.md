# sidecar-patches

## Overview
This patch set provides route integrity for Cloud Foundry, via TLS from router to app instance.
By using a sidecar in the app container, there is no additional work for the app developer.

Every app container runs an [Envoy](https://github.com/lyft/envoy) sidecar that
listens on port 8443, terminates TLS using the [app instance certificate](https://github.com/cloudfoundry/diego-release/blob/develop/docs/instance-identity.md)
and forwards the TCP connection to the CF application listening on port 8080.

Cloud Controller is modified to broadcast the 8443 port to the routing tier instead of the unsecured port 8080.
Finally, GoRouter is modified to require a TLS handshake with application backends.

## Deployment
Extract the Envoy binary like so:
```
docker run --rm -v $PWD:/envoy-binary/ lyft/envoy /bin/bash -c 'cp `which envoy` /envoy-binary/'
```

Add the binary as a blob to Diego Release.  Then build the following customized BOSH releases:

`diego-release` 1.19.0 with [this patch](diego-release.patch), a blob for envoy, and modified submodules:
- `src/code.cloudfoundry.org/buildpackapplifecycle` commit `554a15cdf273fa9781b0e4d5379e72886ca280f6` with [this patch](buildpackapplifecycle.patch)
- `src/code.cloudfoundry.org/executor` commit `fc18a46aff90d3047361e6f9d2237c4f362cf4df` with [this patch](executor.patch)

`capi-release` 1.31.0 with modified submodule:
- `src/cloud_controller_ng` commit `c11841b5675aaa47ccd2a70b742eb25601d26d2c` with [this patch](cloud_controller_ng.patch)

`routing-release` develop with modified submodule:
- `src/code.cloudfoundry.org/gorouter` commit `dc2d9491dbda135929f9ddb608bd9425a0e832b1` with [this patch](gorouter.patch)
  This patch is based on (and mostly the same as) the work that the Routing team did in [this exploration branch](https://github.com/cloudfoundry/gorouter/commits/explore-tls-gorouter-backend-145318557-tmp).

Deploy using a recent [cf-deployment](https://github.com/cloudfoundry/cf-deployment) and
[this opsfile to enable Diego Instance Identity features](https://github.com/cloudfoundry-incubator/cf-networking-ci/blob/master/opsfiles/diego-instance-identity.yml).

## Notes
These two ports used to be the same (port 8080), but now they are different:
- the port inside the container where the app listens (still 8080)
- the port inside the container where ingress connections arrive from the routing tier (now 8443)

We're working on [an Envoy SDS implementation here](https://github.com/cf-container-networking/envoy-test).
