# sidecar-patches

Patches to add sidecars to Cloud Foundry.

`capi-release` 1.31.0 with these submodules:
- `src/cloud_controller_ng` commit `c11841b5675aaa47ccd2a70b742eb25601d26d2c` with [this patch](cloud_controller_ng.patch)

`diego-release` 1.19.0 with [this patch](diego-release.patch), a blob for envoy (see below) and these submodules:
- `src/code.cloudfoundry.org/buildpackapplifecycle` commit `554a15cdf273fa9781b0e4d5379e72886ca280f6` with [this patch](buildpackapplifecycle.patch)
- `src/code.cloudfoundry.org/executor` commit `fc18a46aff90d3047361e6f9d2237c4f362cf4df` with [this patch](executor.patch)

### Envoy blob
We extracted the envoy binary like this:
```
docker run --rm -v $PWD:/envoy-binary/ lyft/envoy /bin/bash -c 'cp `which envoy` /envoy-binary/'
```
