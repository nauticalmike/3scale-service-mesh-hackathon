# Use Case: Accessing External Services 

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0
3. Have installed the bookinfo example app on the `bookinfo` ns.

## Test bookinfo access
Retrieve the URL for the ingress gateway:
```
ISTIO_GW=$(oc get route istio-ingressgateway -n istio-system -o jsonpath="{.spec.host}{.spec.path}")
echo $ISTIO_GW
```
You can now verify that the bookinfo service is responding:
```
curl -v $ISTIO_GW/productpage
```
You should be able to get an HTTP 200 response code along with the `Simple Bookstore App` tittle.

## Deploy sleep sample app

We are going to deploy the sleep example app in the `sleep` namespace: 
```
oc new-project sleep
oc apply -f ServiceMeshMemberRoll.yaml -n istio-system
```
Notice here that the SMMR doesn't include the sleep ns, so lets patch the SMMR to add the sleep ns:
```
oc patch smmr default --type merge -n istio-system -p '{"spec":{"members":["bookinfo", "sleep"]}}'
```
Now lets deploy the sleep service:
```
oc apply -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml -n sleep
```
Notice that the sleep's deployment resource is not annotated for sidecar injection as we have no intention to include this service in the mesh. The sleep service does nothing but sleep. It's a ubuntu container with curl installed that can be used as a request source for invoking other services to experiment with Istio networking.
Now lets execute the sleep service by using it to access the bookinfo's productpage:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -- curl -s productpage.bookinfo.svc.cluster.local:9080/productpage
```
You should get a similar response as in the test bookinfo access section, but this is using the internal OCP networking within the same cluster to access the service. 
Now lets use the sleep's service to access the bookinfo app through the ingressgateway:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -- curl -s $ISTIO_GW/productpage
```
If you got a similar response this proves an external service not part of the mesh can access a service within the mesh from a different namespace.

Now what if you want the opposite? lets block the bookinfo productpage so only services within the mesh can access:
```
oc create -f Bookinfo-authpolicy.yaml
```
The authorization policy targets all pods with the label app=productpage and requests from the istio-ingressgateway-service-account identity are
permitted but restricted to method GET on port 9080.
Now lets use the sleep curl service again targeting the OCP internal networking:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -- curl -s productpage.bookinfo.svc.cluster.local:9080/productpage
```
You should see a response along the lines `RBAC: access denied`.
You should still be able to access the service from the ingress gateway:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -- curl -s $ISTIO_GW/productpage
```