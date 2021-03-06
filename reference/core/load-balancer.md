# Load Balancing in Ambassador

Load balancing configuration can be set for all Ambassador mappings in the [ambassador](/reference/core/ambassador) module, or set per [mapping](https://www.getambassador.io/reference/mappings#configuring-mappings). If nothing is set, simple round robin balancing is used via Kubernetes services.

To use advanced load balancing, you must first configure a [resolver](/reference/core/resolvers) that supports advanced load balancing (e.g., the Kubernetes Endpoint Resolver or Consul Resolver). Once a resolver is configured, you can use the `load_balancer` attribute. The following fields are supported:

```yaml
load_balancer:
  policy: <load balancing policy to use>
```

Supported load balancer policies:
- `round_robin`
- `least_request`
- `ring_hash`
- `maglev`

For more information on the different policies and the implications, see [load balancing strategies in Kubernetes](https://blog.getambassador.io/load-balancing-strategies-in-kubernetes-l4-round-robin-l7-round-robin-ring-hash-and-more-6a5b81595d6c?source=collection_home---4------0---------------------).

## Round Robin
When policy is set to `round_robin`, Ambassador discovers healthy endpoints for the given mapping, and load balances the incoming L7 requests in a round robin fashion. For example:

```yaml
apiVersion: getambassador.io/v1
kind:  Module
metadata:
  name:  ambassador
spec:
  config:
    resolver: my-resolver
    load_balancer:
      policy: round_robin
```

or, per mapping:

```yaml
---
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-ui
spec:
  prefix: /
  service: tour:5000
  resolver: my-resolver
  load_balancer:
    policy: round_robin
```

Note that load balancing may not appear to be "even" due to Envoy's threading model. For more details, see the [Envoy documentation](https://www.envoyproxy.io/docs/envoy/latest/faq/concurrency_lb).

## Least Request
When policy is set to `least_request`, Ambassador discovers healthy endpoints for the given mapping, and load balances the incoming L7 requests to the endpoint with the fewest active requests. For example:

```yaml
apiVersion: getambassador.io/v1
kind:  Module
metadata:
  name:  ambassador
spec:
  config:
    resolver: my-resolver
    load_balancer:
      policy: least_request
```

or, per mapping:

```yaml
---
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-ui
spec:
  prefix: /
  service: tour:5000
  resolver: my-resolver
  load_balancer:
    policy: least_request
```

## Sticky Sessions / Session Affinity
Configuring sticky sessions makes Ambassador route requests to the same backend service in a given session. In other words, requests in a session are served by the same Kubernetes pod. Ambassador lets you configure session affinity based on the following parameters in an incoming request:

- Cookie
- Header
- Source IP

**NOTE:** Ambassador supports sticky sessions using two load balancing policies, `ring_hash` and `maglev`.


### Cookie
```yaml
load_balancer:
  policy: ring_hash
  cookie:
    name: <name of the cookie, required>
    ttl: <TTL to set in the generated cookie>
    path: <name of the path for the cookie>
```

If the cookie you wish to set affinity on is already present in incoming requests, then you only need the `cookie.name` field. However, if you want Ambassador to generate and set a cookie in response to the first request, then you need to specify a value for the `cookie.ttl` field which generates a cookie with the given expiration time.

For example, the following configuration asks the client to set a cookie named `sticky-cookie` with expiration of 60 seconds in response to the first request if the cookie is not already present.

```yaml
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-ui
spec:
  prefix: /
service: tour:5000
resolver: my-resolver
load_balancer:
  policy: ring_hash
  cookie:
    name: sticky-cookie
    ttl: 60s
```

### Header
```yaml
load_balancer:
  policy: ring_hash
  header: <header name>
```

Ambassador allows header based session affinity if the given header is present on incoming requests.

Example:
```yaml
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-ui
spec:
  prefix: /
  service: tour:5000
  resolver: my-resolver
  load_balancer:
    policy: ring_hash
    header: STICKY_HEADER
```

##### Source IP
```yaml
load_balancer:
  policy: ring_hash
  source_ip: <boolean>
```

Ambassador allows session affinity based on the source IP of incoming requests. For example:

```yaml
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-ui
spec:
  prefix: /
  service: tour:5000
  resolver: my-resolver
  load_balancer:
    policy: ring_hash
    source_ip: true
```

Load balancing can be configured both globally, and overridden on a per mapping basis. The following example configures the default load balancing policy to be round robin, while using header-based session affinity for requests to the `/backend/` endpoint of the tour application:

```yaml
apiVersion: getambassador.io/v1
kind:  Module
metadata:
  name:  ambassador
spec:
  config:
    resolver: my-resolver
    load_balancer:
      policy: round_robin
```
```yaml
apiVersion: getambassador.io/v1
kind:  Mapping
metadata:
  name:  tour-backend
spec:
  prefix: /backend/
  service: tour:8080
  resolver: my-resolver
  load_balancer:
    policy: ring_hash
    header: STICKY_HEADER
```

## Disabling advanced load balancing

In Ambassador 0.60, you can disable advanced load balancing features by setting the environment variable `AMBASSADOR_DISABLE_ENDPOINTS` to any value. If you find that this is necessary, please reach out to us on [Slack](https://d6e.co/slack) so we can fix whatever is wrong!

