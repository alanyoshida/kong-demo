# Install Kind Cluster with Kong ingress gateway

This demo is for testing kong functionality locally within kind cluster.

### Create kind cluster

`kind create cluster --name kong --config=./kind-cluster-kong.yaml`

This will use 443 and 80 from your host machine, make sure that it's not being used

### install kong CRDs

`kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml`

or

`kubectl apply -f kong-install-crd.yaml`

### Configure Gateway

```bash
echo "
---
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: kong
  annotations:
    konghq.com/gatewayclass-unmanaged: 'true'

spec:
  controllerName: konghq.com/kic-gateway-controller
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: kong
spec:
  gatewayClassName: kong
  listeners:
  - name: proxy
    port: 80
    protocol: HTTP
" | kubectl apply -f -
```
### Add kong helm repo

```bash
helm repo add kong https://charts.konghq.com
helm repo update
```

### Install Kong

`helm install kong kong/ingress -n kong --create-namespace`

Change type: Loadbalancer to type: NodePort

`kubectl edit svc --namespace kong kong-gateway-proxy`

Add `hostPort: 443` and `hostPort: 80` to container named proxy

```yaml
name: proxy
ports:
    - containerPort: 8444
      hostPort: 443 # <-- ADD THIS LINE
      name: admin-tls
      protocol: TCP
    - containerPort: 8000
      hostPort: 80 # <-- ADD THIS LINE
      name: proxy
      protocol: TCP
```

To edit use this command:

`kubectl -n kong edit deploy kong-gateway`

## Install foo bar example

Now install the example from kind documentation:

`kubectl apply -f ./example.yaml`

Note that ingressClassName, like the following is indicating that this ingress will use kong as the ingress.

```yaml
spec:
  ingressClassName: kong
```

### Test

`curl localhost/foo/hostname`

`curl localhost/bar/hostname`

### How it works

Your kind cluster is port forwarding your host port 80 to a container port 80, and your host port 443 to a container port 443;

Them you configure the kong proxy to receive on hostPort 80 and 443.

Should be working like this:

http://localhost -> kind node port 80 -> container kong-proxy port 80

With the example:

http://localhost/foo port 80 -> kind node port 80 -> container kong-proxy port 80 -> foo-service port 8080 -> bar-app container port 8080

### Now configure kong plugin

This will create the kong response transformer configuration to add some headers to the response.

`kubectl apply -f ./kong-response-transformer-plugin.yaml`

Annotate the service you want to return the response headers:

`kubectl annotate service foo-service konghq.com/plugins=response-transformer-example`

Test it:

```bash
curl -v localhost/foo
*   Trying 127.0.0.1:80...
* Connected to localhost (127.0.0.1) port 80 (#0)
> GET /foo HTTP/1.1
> Host: localhost
> User-Agent: curl/7.81.0
> Accept: */*
>
* Mark bundle as not supporting multiuse
< HTTP/1.1 200 OK
< Content-Type: text/plain; charset=utf-8
< Content-Length: 62
< Connection: keep-alive
< Date: Fri, 29 Mar 2024 19:07:44 GMT
< x-new-header: value
< x-another-header: something
< X-Kong-Upstream-Latency: 0
< X-Kong-Proxy-Latency: 1
< Via: kong/3.6.1
< X-Kong-Request-Id: 022c3b07512b0948f391c920c4fb1c4d
<
* Connection #0 to host localhost left intact
NOW: 2024-03-29 19:07:44.113242258 +0000 UTC m=+2954.323630311⏎
```

Note that the headers was added like the kong response-transformer plugin configuration:

```
< x-new-header: value
< x-another-header: something
```
