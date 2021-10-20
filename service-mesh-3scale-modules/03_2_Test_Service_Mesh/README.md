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
***NOTE***
***
Make sure you create the `SMMR` instance where you have your OSSM Control Plane instance installed
***

Deploy the `bookinfo` sample app:
```
oc apply -f https://raw.githubusercontent.com/maistra/istio/maistra-2.1/samples/bookinfo/platform/kube/bookinfo.yaml -n bookinfo
```

Wait for the pods to be ready and then check the bookinfo's productpage service is working before exposing the services through the gateway:
```
oc run nc-rhel --image=registry.redhat.io/rhel7/rhel-tools -i -t --restart=Never --rm -- /bin/sh -c 'curl http://productpage.bookinfo.svc.cluster.local:9080/productpage' | grep '<title>Simple Bookstore App</title>'
```
You should expect a response along the lines of:
```
<title>Simple Bookstore App</title>
```

Now lets create a `VirtualService` and a `Gateway` instances for bookinfo in order to access the service through the mesh:
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
curl $ISTIO_GW/productpage | grep '<title>Simple Bookstore App</title>'
```

You should expect a response along the lines of:
```
<title>Simple Bookstore App</title>
```

Congratulations, you successfully tested your SMCP.