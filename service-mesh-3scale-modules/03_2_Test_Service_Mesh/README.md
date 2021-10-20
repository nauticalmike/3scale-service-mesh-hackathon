# Test your Service Mesh Control Plane (SMCP)

## Prerequisites

1. Have an OpenShift (OCP) v4.x running cluster
2. Have SMCP working instance

## Provision BookInfo demo

Create a new project named `bookinfo`:
```
oc new-project bookinfo
```
Add the project to the SMMR so it could be discovered by your SMCP:
```
oc apply -f ServiceMeshMemberRoll.yaml -n istio-system
```
Deploy the `bookinfo` sample app:
```
oc apply -f https://raw.githubusercontent.com/maistra/istio/maistra-2.1/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
```
Now lets create a `VirtualService` and a `Gateway` in order to access the service through the mesh:
```
oc apply -f https://raw.githubusercontent.com/maistra/istio/maistra-2.1/samples/bookinfo/networking/bookinfo-gateway.yaml -n bookinfo
```
***NOTE***
***
Find your ingress gateway route using:
```
ISTIO_GW=$(oc get route istio-ingressgateway -n istio-system -o jsonpath="{.spec.host}{.spec.path}")
```
```
echo $ISTIO_GW
```
***

You can now verify that the bookinfo service is responding:
```
curl -v $ISTIO_GW/productpage
```

Congratulations, you successfully tested your SMCP.