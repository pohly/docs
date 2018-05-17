# Testing
This document describes to developers how they can test their CSI clients or drivers.

## Kubernetes

### Logging

All sidecar contains log the gRPC calls they send or receive, so
``kubectl logs`` can provide some insight into what is going on.

### Distributed Tracing

Correlating the log files from all involved components (kubelet,
external attacher, external provisioner, driver registrar, the CSI
driver itself) can get tricky.

This is where distributed tracing, i.e. the collection of events from
different processes, becomes easier to use, because different events
from different processes related to the same operation are shown as
one sequence of events.

[OpenTracing](http://opentracing.io/) is a cross-product standard for
tracing such events. However, processes still need to activate tracing
using specific implementations like
[Jaeger](https://www.jaegertracing.io/docs/). At the moment, this is
only supported with non-upstream patches:

* TODO: collect links

To use container images with support for tracing, run a local Docker
registry, check out the sidecard code in the same directory, and build
it:

```
# https://docs.docker.com/registry/deploying/
docker run -d -p 5000:5000 --restart=always --name registry registry:2

for i in driver-registrar drivers external-attacher external-provisioner; do
    make -C $i REGISTRY_NAME=localhost:5000 push
done
```

Each process sends trace data to a Jaeger agent. The default is to
send via UDP to localhost, which typically does not work on a
Kubernetes cluster because each pod is self-contained and has no
Jaeger agent. Traffic can be sent to the right host by setting
`JAEGER_AGENT_HOST` in the environment of the containers.

For a local cluster, running the all-in-one Jaeger image under Docker
as explained in <https://www.jaegertracing.io/docs/getting-started/>
works fine. In this case, use `JAEGER_AGENT_HOST=<IP address of the
local host>` for the commands below.

For a remote cluster, one can run Jaeger on Kubernetes itself, for
example the
[development setup|https://github.com/jaegertracing/jaeger-kubernetes#development-setup]. In
this case, use `JAEGER_AGENT_HOST=jaeger-agent`.

In both cases the `JAEGER_AGENT_HOST` env variable must be injected
into the CSI containers. It is also useful to enable
`JAEGER_SAMPLER_TYPE=const` and `JAEGER_SAMPLER_PARAM=1` to capture
all traces. When using the [example .yaml file](./Example.html), this
can be done by manipulating the .yaml with `sed`. The same sed command can
also switch from quay.io images to locally built ones:

```
JAEGER_AGENT_HOST=...
sed -e 's;image: quay.io/k8scsi/\(.*\):v.*;image: localhost:5000/\1:canary;' \
    -e '/name: external-provisioner/s/.*/&\n    env:/' \
    -e "/env:/s/.*/&\n    - name: JAEGER_AGENT_HOST\n      value: ${JAEGER_AGENT_HOST}\n    - name: JAEGER_SAMPLER_TYPE\n      value: const\n    - name: JAEGER_SAMPLER_PARAM\n      value: '1'/" \
    docs/book/src/example/csi-setup.yaml | \
    kubectl create -f -
```

`external-provisioner` needs some special treatment here because it
does not already have an `env` list.

Alternatively,
[PodPreset](https://kubernetes.io/docs/tasks/inject-data-application/podpreset/)
could be used to inject the env variables, but `PodPreset` is
currently alpha and might not be available.

The resulting traces contain the following information:

* gRPC error codes and messages when a call fails including the `error` tag,
  so Jaeger will highlight those failed calls
* gRPC request and response as tags, when using `csicommon.LogGRPC`
* log messages generated while processing a gRPC call if those messages get
  logged with `csicommon.Infof` or `csicommon.Errorf`
