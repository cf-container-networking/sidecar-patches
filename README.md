# sidecar-patches

## Overview
Every app instance gets an [Envoy](https://github.com/lyft/envoy) sidecar that
listens on port 10000, terminates TLS using the [app instance certificate](https://github.com/cloudfoundry/diego-release/blob/develop/docs/instance-identity.md)
and forwards the TCP connection to the CF application listening on port 8080.

The GoRouter is then modified to TLS handshake with the app instance backend (proxy).

Finally, as a hack, we currently have to modify the app ports and route mappings in Cloud Controller
in order to point the GoRouter at the correct app ports.  Further work can remove the need for this.

## Deployment
Extract the Envoy binary like so:
```
docker run --rm -v $PWD:/envoy-binary/ lyft/envoy /bin/bash -c 'cp `which envoy` /envoy-binary/'
```

Add it as a blob to Diego Release.  Then build the following customized BOSH releases:

`diego-release` 1.19.0 with [this patch](diego-release.patch), the blob for envoy, and modified submodules:
- `src/code.cloudfoundry.org/buildpackapplifecycle` commit `554a15cdf273fa9781b0e4d5379e72886ca280f6` with [this patch](buildpackapplifecycle.patch)
- `src/code.cloudfoundry.org/executor` commit `fc18a46aff90d3047361e6f9d2237c4f362cf4df` with [this patch](executor.patch)

`capi-release` 1.31.0 with modified submodules:
- `src/cloud_controller_ng` commit `c11841b5675aaa47ccd2a70b742eb25601d26d2c` with [this patch](cloud_controller_ng.patch)

`routing-release` develop with patches to these submodules:
- `src/code.cloudfoundry.org/gorouter` commit `dc2d9491dbda135929f9ddb608bd9425a0e832b1` with [this patch](gorouter.patch)

Deploy using a recent [cf-deployment](https://github.com/cloudfoundry/cf-deployment) and
[this opsfile to enable Diego Instance Identity features](https://github.com/cloudfoundry-incubator/cf-networking-ci/blob/master/opsfiles/diego-instance-identity.yml).

## Modifying ports on your app
To get our test app to work, we had to make two changes:

First, we needed to expose port 10000, where the Envoy is listening:

```
cf curl /v2/apps/$(cf app --guid myApp) -X PUT -d '{ "ports": [8080, 10000] }'
```

Its important right now that port 8080 remain on there, since CC uses `ports[0]`
as the value of the `$PORT` environment variable.

Then, since the app's routes all point to the 8080 backend, we need to update that as well:
so that the router can reach the proxy on port 10000.
We did this for an example app by creating a new `route_mapping` for the app and deleting the old one:
```
cf curl /v2/route_mappings -X POST -d '{ "app_port": 10000, "app_guid": "$APP_GUID", "route_guid": "$ROUTE_GUID" }'
cf curl /v2/route_mappings/$OLD_ROUTE_GUID -X DELETE
```

## Improvements
To make this experiment more real, we'll need to distinguish
between two different entities that we currently assume to be identical:

- the port inside the container where the app listens
- the port inside the container where ingress connections should land

The separation exists already in the Diego LRP (environment variables vs ports).
But they are currenty assumed to be equal in Cloud Controller, [see here](https://github.com/cloudfoundry/cloud_controller_ng/blob/7a152fe9b8423fd4707ab83524f79626917db928/lib/cloud_controller/diego/buildpack/desired_lrp_builder.rb#L54-L59).

Once we do that, then the modifications to ports and route mappings above won't be necessary.
