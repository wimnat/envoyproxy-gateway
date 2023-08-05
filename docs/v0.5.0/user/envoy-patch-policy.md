# Envoy Patch Policy

This guide explains the usage of the [EnvoyPatchPolicy][] API.
__Note:__ This API is meant for users extremely familiar with Envoy [xDS][] semantics.
Also before considering this API for production use cases, please be aware that this API
is unstable and the outcome may change across versions. Use at your own risk.

## Introduction

The [EnvoyPatchPolicy][] API allows user to modify the output [xDS][]
configuration generated by Envoy Gateway intended for EnvoyProxy, 
using [JSON Patch][] semantics.

## Motivation

This API was introduced to allow advanced users to be able to leverage Envoy Proxy functionality
not exposed by Envoy Gateway APIs today.

## Quickstart

### Prerequistes

* Follow the steps from the [Quickstart](quickstart.md) guide to install Envoy Gateway and the example manifest.
Before proceeding, you should be able to query the example backend using HTTP.

### Enable EnvoyPatchPolicy

* By default EnvoyPatchPolicy][] is disabled. Lets enable it in the [EnvoyGateway][] startup configuration

* The default installation of Envoy Gateway installs a default [EnvoyGateway][] configuration and attaches it
using a `ConfigMap`. In the next step, we will update this resource to enable EnvoyPatchPolicy. 


```shell
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-gateway-config
  namespace: envoy-gateway-system
data:
  envoy-gateway.yaml: |
    apiVersion: config.gateway.envoyproxy.io/v1alpha1
    kind: EnvoyGateway
    provider:
      type: Kubernetes
    gateway:
      controllerName: gateway.envoyproxy.io/gatewayclass-controller
    extensionApis:
      enableEnvoyPatchPolicy: true
EOF
```

* After updating the `ConfigMap`, you will need to restart the `envoy-gateway` deployment so the configuration kicks in

```shell
kubectl rollout restart deployment envoy-gateway -n envoy-gateway-system
```

## Testing

### Customize Response

* Lets use EnvoyProxy's [Local Reply Modification][] feature to return a custom response back to the client when
the status code is `404`

* Lets apply the configuration

```shell
cat <<EOF | kubectl apply -f -
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata:
  name: custom-response-patch-policy
  namespace: default
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: eg
    namespace: default
  type: JSONPatch
  jsonPatches:
    - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
      # The listener name is of the form <GatewayNamespace>/<GatewayName>/<GatewayListenerName>
      name: default/eg/http
      operation:
        op: add
        path: "/default_filter_chain/filters/0/typed_config/local_reply_config"
        value:
          mappers:
          - filter:
              status_code_filter:
                comparison:
                 op: EQ
                 value:
                   default_value: 404
                   runtime_key: key_b
            status_code: 406
            body:
              inline_string: "could not find what you are looking for"
EOF
```

* Lets edit the HTTPRoute resource from the Quickstart to only match on paths with value `/get`

```
kubectl patch httproute backend --type=json --patch '[{
   "op": "add",
   "path": "/spec/rules/0/matches/0/path/value",
   "value": "/get",
}]'
```

* Lets test it out by specifying a path apart from `/get`

```
$ curl --header "Host: www.example.com" http://localhost:8888/find
Handling connection for 8888
could not find what you are looking for
```

## Debugging

### Runtime

* The `Status` subresource should have information about the status of the resource. Make sure
`Accepted=True` and `Programmed=True` conditions are set to ensure that the policy has been
applied to Envoy Proxy.

```
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"gateway.envoyproxy.io/v1alpha1","kind":"EnvoyPatchPolicy","metadata":{"annotations":{},"name":"custom-response-patch-policy","namespace":"default"},"spec":{"jsonPatches":[{"name":"default/eg/http","operation":{"op":"add","path":"/default_filter_chain/filters/0/typed_config/local_reply_config","value":{"mappers":[{"body":{"inline_string":"could not find what you are looking for"},"filter":{"status_code_filter":{"comparison":{"op":"EQ","value":{"default_value":404}}}}}]}},"type":"type.googleapis.com/envoy.config.listener.v3.Listener"}],"priority":0,"targetRef":{"group":"gateway.networking.k8s.io","kind":"Gateway","name":"eg","namespace":"default"},"type":"JSONPatch"}}
  creationTimestamp: "2023-07-31T21:47:53Z"
  generation: 1
  name: custom-response-patch-policy
  namespace: default
  resourceVersion: "10265"
  uid: a35bda6e-a0cc-46d7-a63a-cee765174bc3
spec:
  jsonPatches:
  - name: default/eg/http
    operation:
      op: add
      path: /default_filter_chain/filters/0/typed_config/local_reply_config
      value:
        mappers:
        - body:
            inline_string: could not find what you are looking for
          filter:
            status_code_filter:
              comparison:
                op: EQ
                value:
                  default_value: 404
    type: type.googleapis.com/envoy.config.listener.v3.Listener
  priority: 0
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: eg
    namespace: default
  type: JSONPatch
status:
  conditions:
  - lastTransitionTime: "2023-07-31T21:48:19Z"
    message: EnvoyPatchPolicy has been accepted.
    observedGeneration: 1
    reason: Accepted
    status: "True"
    type: Accepted
  - lastTransitionTime: "2023-07-31T21:48:19Z"
    message: successfully applied patches.
    reason: Programmed
    status: "True"
    type: Programmed
```

### Offline

* You can use [egctl x translate][] to validate the translated xds output.

## Caveats

This API will always be an unstable API and the same outcome cannot be garunteed
across versions for these reasons
* The Envoy Proxy API might deprecate and remove API fields
* Envoy Gateway might alter the xDS translation creating a different xDS output
such as changing the `name` field of resources.

[EnvoyPatchPolicy]: https://gateway.envoyproxy.io/v0.5.0/api/extension_types.html#envoypatchpolicy
[EnvoyGateway]: https://gateway.envoyproxy.io/v0.5.0/api/config_types.html#envoygateway
[JSON Patch]: https://datatracker.ietf.org/doc/html/rfc6902
[xDS]: https://www.envoyproxy.io/docs/envoy/v0.5.0/intro/arch_overview/operations/dynamic_configuration
[Local Reply Modification]: https://www.envoyproxy.io/docs/envoy/v0.5.0/configuration/http/http_conn_man/local_reply
[egctl x translate]: https://gateway.envoyproxy.io/v0.5.0/user/egctl.html#egctl-experimental-translate