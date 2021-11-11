# CNAP

## What is the CNAP

Reference document

## Requriements

- Traffic egresses from single point
- Traffic ingresses from single point
- Whitelist egress traffic
- Whitelist ingress traffic
- RBAC on workload traffic entering the cluster
- RBAC on workload traffic leaving the cluster

Things to look at

- Geolocation of incoming IP addresses
- SPI
-

## Setup BigBang cluster

The file `bigbang.yaml` in the repo shows how istio was patched to support an egress gateway that prevents pods in the mesh from talking to endpoints explicitly added to the mesh. For example deploy BigBang and this sleep pod for testing

## Egress Traffic through Egress Gateway

Let's deploy a pod in the default namespace that we can use for testing, and ensure it has an istio sidecar

```bash
kubectl label ns default istio-injection=enabled
kubectl apply -f sleep.yaml
```

### Block Unapproved Endpoints

External (non-mesh) traffic can be blocked by default by creating this sidecar object in the namespace:

```bash
kubectl apply -f register.yaml
```

And now access to google is denied:

```bash
export SOURCE_POD=$(kubectl get pod -l app=sleep -o jsonpath='{.items..metadata.name}')
kubectl exec -it ${SOURCE_POD} -c sleep -- curl -svvvv https://google.com | grep "HTTP"
command terminated with exit code 35
```

We also need to add a network policy to the default namespace, because this control only applies to pods in the mesh that go through a sidecar. By making this call in the side car, the pod is still able to get to the endpoint:

```bash
kubectl exec -it ${SOURCE_POD} -c istio-proxy -- curl -svvvv https://google.com | grep "HTTP"
```

If a process was able to escape the application (sleep) pod and get into the istio sidecar, they would be able to egress anywhere in the traffic and bypass the controls being put in place. As a result, we put a network policy that blocks all egress traffic leaving the cluster, by just allowing traffic to all namespaces in the cluster:

```bash
kubectl apply -f namespace-egress-networkpolicy.yaml
```

//TODO @runyontr show this working. NetworkPolicies not doing aynthing in my k3d cluster right now.

### Direct Traffic to egress gateway

In order to now connect to an external endpoint we need to create a [ServiceEntry](https://istio.io/latest/docs/reference/config/networking/service-entry/) to add the endpoint to the mesh.

Now we can talk to `www.google.com` but not `google.com` since the former is in the mesh:

```bash
❯ kubectl exec -it ${SOURCE_POD} -c sleep -- curl -svvvv https://google.com | grep "HTTP"
command terminated with exit code 35

❯ kubectl exec -it ${SOURCE_POD} -c sleep -- curl -sI https://www.google.com | grep "HTTP"
HTTP/2 200

```

The addition of the `ServiceEntry` into the mesh now allows the sleep pod to have an endpoint:

```bash
❯ istioctl pc endpoint ${SOURCE_POD} | grep google
172.217.6.68:80                  HEALTHY     OK                outbound|80||www.google.com
172.217.6.68:443                 HEALTHY     OK                outbound|443||www.google.com
```

Now that the endpoint is in the mesh and the pod can use it, we need to route the traffic to the egressgateway. The first step is to take requests to `www.google.com` and send them to a dedicated port on the egress gateway. Apply the `DestinationRule` and the `VirtualService` as follows and then we'll explore what each does

```bash
kubectl apply -f google-desitinationrule.yaml
kubectl apply -f google-virtualservice.yaml
kubectl apply -f gateway.yaml
```

Now we can see that there are new entries for google in the routing:

```bash
❯ istioctl pc endpoint ${SOURCE_POD} | grep google
10.42.1.47:8080                  HEALTHY     OK                outbound|80|google|istio-egressgateway.istio-system.svc.cluster.local
10.42.1.47:8443                  HEALTHY     OK                outbound|443|google|istio-egressgateway.istio-system.svc.cluster.local
10.42.1.47:8444                  HEALTHY     OK                outbound|8444|google|istio-egressgateway.istio-system.svc.cluster.local
172.217.6.68:80                  HEALTHY     OK                outbound|80||www.google.com
172.217.6.68:443                 HEALTHY     OK                outbound|443||www.google.com
```

TODO make it more clear the vs -> isito-egressgateway mapping inside the pod

#### DestinationRule

A [DestinationRule](https://istio.io/latest/docs/reference/config/networking/destination-rule/) defines policies that should be applied to traffic indended for a service after routing as occurred.

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: egressgateway-for-google
spec:
  host: istio-egressgateway.istio-system.svc.cluster.local <1>
  subsets:
    - name: google <2>
      trafficPolicy:
        loadBalancer:
          simple: ROUND_ROBIN
        tls:
          mode: ISTIO_MUTUAL <3>
          sni: www.google.com <4>
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: direct-google-through-egress-gateway
spec:
  hosts:
    - "www.google.com" <5>
  gateways:
  - mesh <6>
  - istio-system/egress-gateway <7>
  tls:
  - match:
    - gateways:
      - mesh <6>
      port: 443
      sniHosts:
      - www.google.com
    route:
    - destination:
        host: istio-egressgateway.istio-system.svc.cluster.local <8>
        subset: google <2>
        port:
          number: 8444 <9>
  tcp:
  - match:
    - gateways:
      - istio-system/egress-gateway <7>
      port: 8444 <9>
    route:
    - destination:
        host: www.google.com <11>
        port:
          number: 443 <12>
      weight: 100
```

- `<1>` This `DestinationRule` will get applied to traffic heading to this host.
- `<2>` This defines a subset that we'll use explicitly in the virtual service later
- `<3>` This forces traffic to use `ISTIO_MUTUAL`, i.e. mTLS, which will allow us to use `AuthorizationPolicies` later for Authn/Authz for which workloads can talk to what endpoints
- `<4>` This SNI value identifies the SNI string to present to the server during the TLS handshake. Since we're doing a tls passthrough to www.google.com
- `<5>` This `VirtualService` applies for requests going to `www.google.com`
- `<6>` This part of the `VirtualService` applies to requests in the mesh
- `<7>` This part of the `VirtualService` applies to requests in the egress gateway.
- `<8>` When a request for wwww.google.com origionates in the mesh, the request should be routed to `istio-egressgateway.istio-system.svc.cluster.local` on port `<9>` and use the `DestinationRule` defined in `<2>`
- `<9>` When requests come into the egressgateway `<7>` on port `<9>` the request should get routed as defined in this section and should be sent to www.google.com on port 443 <12>

Now that we have routing setup through the istio-egressgateway that was created in `gateway.yaml`, we can talk to www.google.com:

```bash
❯ kubectl exec -it ${SOURCE_POD} -c sleep -- curl -sI https://www.google.com | grep "HTTP"
HTTP/2 200
```

and see traffic getting routed through the egressgateway:

```bash
❯ k get pods ${SOURCE_POD} -o wide
NAME                     READY   STATUS    RESTARTS   AGE   IP           NODE                   NOMINATED NODE   READINESS GATES
sleep-857bd4d6d7-dfmn2   2/2     Running   3          27h   10.42.1.41   k3d-dev-env-server-0   <none>           <none>
❯ k get pods -n istio-system istio-egressgateway-679fd98779-rwnkl -o wide
NAME                                   READY   STATUS    RESTARTS   AGE    IP           NODE                   NOMINATED NODE   READINESS GATES
istio-egressgateway-679fd98779-rwnkl   1/1     Running   0          106m   10.42.1.47   k3d-dev-env-server-0   <none>           <none>

❯ kubectl logs -n istio-system -l app=istio-egressgateway | grep www.google.com
[2021-11-10T14:49:06.792Z] "- - -" 0 - - - "-" 892 5620 135 - "-" "-" "-" "-" "172.217.6.68:443" outbound|443||www.google.com 10.42.1.47:45712 10.42.1.47:8444 10.42.1.41:33992 www.google.com -
```

Looking at the log message we can see the pod IPs for the `${SOURCE_POD}` requesting the endpoint, and the egress pod (while its only one pod now, mutliple egresspods would be important for HA).

## EGress RBAC by Endpoint

Suppose we had multiple work loads in a cluster that needed egress traffic to different endpoints. In this scenario we don't want to allow one workload to piggyback approvals to another endpiont. Lets deploy another endpoint for `edition.cnn.com` via

```bash
❯ kubectl exec -it ${SOURCE_POD} -c sleep -- curl -sI https://edition.cnn.com | grep "HTTP"
command terminated with exit code 35

cnap on  main [!?] on ☁️  (us-gov-west-1) on ☁️ tom@leapfrog.ai
❯ k apply -f cnn.yaml
serviceentry.networking.istio.io/cnn created
destinationrule.networking.istio.io/egressgateway-for-cnn created
virtualservice.networking.istio.io/direct-cnn-through-egress-gateway created

cnap on  main [!?] on ☁️  (us-gov-west-1) on ☁️ tom@leapfrog.ai
❯ kubectl exec -it ${SOURCE_POD} -c sleep -- curl -sI https://edition.cnn.com | grep "HTTP"
HTTP/2 200
```

Because the destination rules specify tls mode as `ISTIO_MUTUAL`, we are able to use `AuthorizationPolicies` to make determinations about approvals for interpod communication.

```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: allow-default-sleep-cnn
  namespace: istio-system
spec:
  selector:
    matchLabels:
      istio: egressgateway <1>
  action: ALLOW <2>
  rules:
    - from:
        - source:
            principals: ["cluster.local/ns/default/sa/sleep"] <3>
      when: <4>
        - key: connection.sni
          values:
            - "edition.cnn.com"
```

- `<1>` This `AuthorizationPolicy` applies to pods in the istio-system namespace with this label
- `<2>` This explicitly allows the communication described in this policy.
- `<3>` The `sleep` serviceaccount in the `default` namespace is making the request
- `<4>` The SNI value matches `edition.cnn.com`

### Egress from selected Nodes

By egressing from a single node, or known subset of your cluster, the cluster admin is better able to restrict the permissions of the nodes running workloads. This could allow an external node group to be used for ingress and egress pods, and force the workloads to be executed on internal node groups.

# require auth at the gateway

# Require

Thoughts:

- we apply authorization policies to gateways as part of the re-work for authservice at the gateway. Perhaps we can apply authpolices for an egress gateway per namespace!

# Whitelist ingress vi authorizationplicy on the IngressGateway:

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

## Tailscale

If we deploy tailscale in the pod and expose it via a virtual service, we can let people get into tailscale and then expose objects t
