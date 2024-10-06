# Add all workload resources

```shell
kubectl apply -f workloads/simple-web-server-deploy-a.yml
kubectl apply -f workloads/simple-web-server-deploy-b.yml
kubectl apply -f workloads/curl-deploy.md
```

## Test

Test that everything works:

Get a shell on curl pod

```shell
curl simple-web-server-a
curl simple-web-server-b
```

# Install Istio

Make sure you're pointing to the correct kubernetes cluster

```shell
istioctl install --set meshConfig.accessLogFile=/dev/stdout --set meshConfig.accessLogEncoding=JSON
```

## Enable istio on all workloads on default namespace

```shell
kubectl label namespace default istio-injection=enabled
```

## Test

Test that everything works:

Get a shell on curl pod

```shell
curl simple-web-server-a
curl simple-web-server-b
```

Now it has Envoy headers!

# Basic routing

```shell
curl simple-web-server-a
```

```shell
curl simple-web-server-b
```

We don't rely on the hostname and the ip

```shell
curl 100.100.100.100 -H "Host: simple-web-server-a"
curl nubank.com.br -H "Host: simple-web-server-a"
```

I can use any HTTP Client (and server on the other end)

## Envoy config

You can see the routing defined here for a given pod:

```shell
istioctl proxy-config all deploy/curl | less
```

You can look for the config dump

```shell
kubectl port-forward deploy/curl 15000:15000
curl localhost:15000/config_dump | jless
```

# Virtual Service

Apply the virtual service configuration

```shell
kubectl apply virtual-service/simple-web-server.yml
```

## Test the weighting

```shell
rm /tmp/output.txt
for i in $(seq 100); do curl -s simple-web-server | jq .meta.hostname >> /tmp/output.txt ; done
```

Validate the 90% to A and 10% to B

```shell
cat /tmp/output.txt | cut -d\- -f4 | sort | uniq -c
```
## Test the header

App A:

```shell
rm /tmp/output.txt
for i in $(seq 100); do curl -s simple-web-server -H "app: simple-web-server-a" | jq .meta.hostname >> /tmp/output.txt ; done

cat /tmp/output.txt | cut -d\- -f4 | sort | uniq -c
```

App B:

```shell
rm /tmp/output.txt
for i in $(seq 100); do curl -s simple-web-server -H "app: simple-web-server-a" | jq .meta.hostname >> /tmp/output.txt ; done
cat /tmp/output.txt | cut -d\- -f4 | sort | uniq -c
```

## Injecting delay and abort

### Delay

We can inject a delay to a route

```shell
kubectl apply -f virtual-service/simple-web-server-with-delay.yml
```

Every request takes an extra 5s

```shell
curl -s simple-web-server
```

### Abort

We can force to return a status code to a route

```shell
for i in $(seq 100); do curl  simple-web-server -H "iteration: $i" -w "%{http_code}" -o /dev/null -s; echo ;done | sort | uniq -c
```

```shell
for i in $(seq 100); do curl -s simple-web-server -H "iteration: $i" -w "%{http_code}"; done
```

### Retries

```shell
for i in $(seq 100); do curl  simple-web-server/random-failure/10 -H "iteration: $i" -w "%{http_code}" -o /dev/null -s; echo ;done | sort | uniq -c
```

```shell
kubectl apply -f virtual-service/simple-web-server-with-abort-and-retry.yml
```

```shell
for i in $(seq 100); do curl  simple-web-server/random-failure/10 -H "iteration: $i" -w "%{http_code}" -o /dev/null -s; echo ;done | sort | uniq -c
```

# Metrics

Install Prometheus and Kiali

```shell
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/addons/prometheus.yaml
kubectl apply -f https://raw.githubusercontent.com/istio/istio/refs/heads/master/samples/addons/kiali.yaml
```

Add a port-forwarding to kiali:

```shell
kubectl port-forward svc/kiali 9090:20001 -n istio-system
```

http://localhost:9090

```shell
while true; do curl simple-web-server; done
```

We can apply the same patterns and see what happens.

# Destination rule

Apply the neutral Virtual Service

```shell
kubectl apply -f virtual-service/simple-web-server.yml
```

Similar to Finagle's - we can set to round-robin

```shell
kubectl apply -f destination-rule/simple-web-server.yml
```

Test it

```shell
for i in $(seq 100); do curl  simple-web-server-a -H "customer-id: e66919cd-6ca1-4319-8f5b-038249dcf005" -s | jq .meta.hostname; done  | sort | uniq -c
```

## Consistent Hashing

```shell
kubectl apply -f destination-rule/simple-web-server-consistent-hashing.yml
```

```shell
for i in $(seq 100); do curl  simple-web-server-a -H "customer-id: e66919cd-6ca1-4319-8f5b-038249dcf005" -s | jq .meta.hostname ;done  | sort | uniq -c
```