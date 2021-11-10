# CNAP

Somethings up with metrics. Promtheus is un able to scrape a lot of endpoints so its preventing istio metrics from being collected.

https://prometheus.bigbang.dev/targets

# Ingress through certain points

- Istio Ingress Gateway allows for a single ingress IP address

-

# Egress through certain points

1. Turn on egress gateway

2. When things are in the mesh, all traffic would egress through the istio-egress gateway, so we should use network policies to prevent talking to external services without going through the mesh: https://istio.io/latest/docs/tasks/traffic-management/egress/egress-gateway/#apply-kubernetes-network-policies

# Only allow egress to certain endpoints

1. Deploy istio with:

```yaml
spec:
  components:
    egressGateways:
      - name: istio-egressgateway
    enabled: true
  meshConfig:
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
```

2. Ensure no egress traffic occurs by setting NetworkPolicies to prevent egress traffic:

# require auth at the gateway

# Require

Thoughts:

- we apply authorization policies to gateways as part of the re-work for authservice at the gateway. Perhaps we can apply authpolices for an egress gateway per namespace!

# Whitelist ingress vi authorization plicy on the IngressGateway:

https://istio.io/latest/docs/tasks/security/authorization/authz-ingress/

```bash
$ kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: ingress-policy
  namespace: istio-system
spec:
  selector:
    matchLabels:
      app: istio-ingressgateway
  action: ALLOW
  rules:
  - from:
    - source:
        ipBlocks: ["1.2.3.4", "5.6.7.0/24"]
EOF
```

AUTHZ ON EGRESS GATEWAWY:

Its almost in here somewhere...

https://gist.github.com/yangminzhu/2123c6614825aeadba38279110450b62
