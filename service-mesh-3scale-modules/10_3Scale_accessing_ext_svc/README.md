# Use Case: Accessing external service whose authorization policy is managed by 3scale apicast gateway

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0
3. 3scale

## Deploy sleep sample app
We are going to use the sleep sample app as our app inside the mesh whose going to access a service outside the mesh with authorization policy managed by the 3scale apicast gateway.

The sleep service does nothing but sleep. It's a ubuntu container with curl installed that can be used as a request source for invoking other services to experiment with Istio networking.

Lets deploy the sleep example app in the `sleep` namespace: 
```
oc new-project sleep
```
Now lets create the SMMR including the sleep ns:
```
oc apply -f ServiceMeshMemberRoll.yaml -n istio-system
```
Now lets deploy the sleep service:
```
oc apply -f https://raw.githubusercontent.com/istio/istio/master/samples/sleep/sleep.yaml -n sleep
```
Notice that the sleep's deployment resource is not annotated for sidecar injection, so lets edit the deployment and add it:
```
oc patch deployment sleep --type merge -n sleep -p '{"spec":{"template":{"metadata":{"annotations":{"sidecar.istio.io/inject":"true"}}}}}'
``` 
At this point the app is only accessible within the mesh:
```
oc run nc-rhel --image=registry.redhat.io/rhel7/rhel-tools -i -t --restart=Never --rm -- /bin/sh -c 'curl -v sleep.sleep.svc.cluster.local'
```
Now lets validate the out of the box 3scale product called "API" which uses the echo service as a backend named "API backend". Go to the 3scale API manager 
```
oc run nc-rhel --image=registry.redhat.io/rhel7/rhel-tools -i -t --restart=Never --rm -- /bin/sh -c 'curl -v https://echo-api.3scale.net:443'
```
You should have received a 200 http response code from the service proving that you can access it from within the cluster, now lets access the API manager console, you can find the route hostname in the 3scale ns leading with "3scale-admin". Log in using the admin credentials you can find in the same ns under the `system-seed` secret, then click under the product named `API` and in the left menu under `integration` click on `configuration` and promote to `staging`, this will expose the product configuration to the staging APIcast gateway, in the staging page you can find the curl command including the pre-defined user-key:
```
curl "https://api-3scale-apicast-staging.<MY-WILDCARD-DOMAIN>443/?user_key=<MY-APP-USER-KEY>"
```
Test from your command line and you should receive a similar response as when we tested above from the rhel-tools container. 
Now test again by replacing an invalid user key or without key and you should be getting "authentication failed" and/or "Authentication parameters missing" responses from the staging APIcast gateway in-front of the echo service.

Now lets test if our sleep curl service can access the echo service through the stating gateway:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -c sleep -- curl -s "https://api-3scale-apicast-staging.<MY-WILDCARD-DOMAIN>:443/?user_key=<MY-APP-USER-KEY>"
```
You should be able to see that our sleep curl service within the mesh was able to access the echo service protected by 3scale's APIcast gateway if a valid user key is provided. 

Now lets block all outbound communication from the mesh and explore what happens. Patch your ServiceMeshControlPlane instance to so the outbound traffic policy allows egress traffic only to registered services:
```
oc patch smcp basic \
--type merge -n istio-system \
-p '{"spec":{"proxy":{"networking":{"trafficControl":{"outbound":
{"policy":"REGISTRY_ONLY"}}}}}}'
```
And now lets test again if we can access the echo service from our sleep curl service within the mesh:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -c sleep -- curl -s -v "https://api-3scale-apicast-staging.<MY-WILDCARD-DOMAIN>:443/?user_key=<MY-APP-USER-KEY>"
```
You should receive an error not being able to go outbound along the lines "OpenSSL SSL_connect: SSL_ERROR_SYSCALL". You can even try from the same sleep container to try to access the echo service and receive the same error:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -c sleep -- curl -v https://echo-api.3scale.net:443
```
You can visualize this in Kiali as a `BlackHoleCluster`. 

Now lets register the sleep service by creating a ServiceEntry resource so further request are allowed. Update the `sleep-service-entry.yaml` file with your `https://api-3scale-apicast-staging.<MY-WILDCARD-DOMAIN>` then apply the resource as follows:
```
oc apply -f Sleep-service-entry.yaml -n sleep
```
Now lets try again:
```
oc exec $(oc get pod -o custom-columns=POD:.metadata.name --no-headers) -c sleep -- curl -s -v "https://api-3scale-apicast-staging.<MY-WILDCARD-DOMAIN>:443/?user_key=<MY-APP-USER-KEY>"
```
You should be able to get outbound access from the mesh again.

