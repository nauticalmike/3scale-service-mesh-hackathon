# Service Mesh Observability

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0 with a SMCP instance
3. Have installed the bookinfo example app on the `bookinfo` ns, part of SMMR and exposed gateway and virtual service

## Service Mesh Observability
Get your Kiali URL:

```
KIALI_URL=$(oc get route kiali \ -n istio-system -o jsonpath='{.spec.host}')
```

Open the URL on a browser and then click on the `bookinfo` app:

```
firefox ${KIALI_URL} &
```
![](../images/kiali-app.png)

Make sure all services are healthy:
![](../images/kiali-app2.png)

Call the `productinfo` page as done on the other labs:
```
ISTIO_GW=$(oc get route istio-ingressgateway -n istio-system -o jsonpath="{.spec.host}{.spec.path}")
```
```
curl $ISTIO_GW/productpage | grep '<title>Simple Bookstore App</title>'
```

Setup Kiali for traffic visualization by going to the left menu `Graph` then click on the drop down `Display` and select the `Request Percentage` and `Traffic Animation` options:
![](../images/kiali-trafficanimation.png)

Now lets do a loop to generate requests to be able to watch the animation:
```
while true; \
do curl $ISTIO_GW/productpage; \
sleep 3;done
```

Switch to the graph and watch the animation, observe how the requests are mostly balanced between the three versions of the reviews service:
![](../images/kiali-traffic-animation.png)

Select the reviews service by clicking on the node and observe on the right the incoming traffic:
![](../images/kiali-incoming.png)

Now we are going to redirect all traffic to the v1 services by creating the `VirtualService` and `DestinationRules`, let start with the former which looks like:
```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```
Now apply them all:
```
oc apply -f https://raw.githubusercontent.com/maistra/istio/maistra-2.1/samples/bookinfo/networking/destination-rule-all.yaml -n bookinfo
```

Now lets review the corresponding `VirtualService`s:
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```
Now Apply:
```
oc apply -f https://raw.githubusercontent.com/maistra/istio/maistra-2.1/samples/bookinfo/networking/virtual-service-all-v1.yaml -n bookinfo
```

Run the loop again and switch to the traffic animation:
```
while true; \
do curl $ISTIO_GW/productpage; \
sleep 3;done
```
You should see now something like this:
![](../images/kiali-route-v1.png)