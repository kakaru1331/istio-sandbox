# External Authorization

## Prerequisite

```
kubectl create ns foo
kubectl label ns foo istio-injection=enabled
kubectl apply -f samples/httpbin/httpbin.yaml -n foo
kubectl apply -f samples/sleep/sleep.yaml -n foo
```

Verify the communication
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}')" -c sleep -n foo -- curl http://httpbin.foo:8000/ip -s -o /dev/null -w "%{http_code}\n"
```

## Deploy external authorizer
```
kubectl apply -n foo -f task/external-authorization/ext-authz.yaml
```

Verify workload is running
```
kubectl logs "$(kubectl get pod -l app=ext-authz -n foo -o jsonpath='{.items..metadata.name}')" -n foo -c ext-authz
```

## Define externak authorizer
open configmap
```
kubectl edit configmap istio -n istio-system
```

edit and save istio config
```
data:
  mesh: |-
    # Add the following content to define the external authorizers.
    extensionProviders:
    - name: "sample-ext-authz-grpc"
      envoyExtAuthzGrpc:
        service: "ext-authz.foo.svc.cluster.local"
        port: "9000"
    - name: "sample-ext-authz-http"
      envoyExtAuthzHttp:
        service: "ext-authz.foo.svc.cluster.local"
        port: "8000"
        includeRequestHeadersInCheck: ["x-ext-authz"]
```

Verify a request is denied
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}')" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -H "x-ext-authz: deny" -s
```

Verify a request is allowed
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}')" -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "%{http_code}\n"
```

Verify a request is allowed
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}')" -c sleep -n foo -- curl "http://httpbin.foo:8000/headers" -H "x-ext-authz: allow" -s
```

Verify a reques to path /ip does not trigger the external authorization
```
kubectl exec "$(kubectl get pod -l app=sleep -n foo -o jsonpath='{.items..metadata.name}')" -c sleep -n foo -- curl "http://httpbin.foo:8000/ip" -s -o /dev/null -w "%{http_code}\n"
```