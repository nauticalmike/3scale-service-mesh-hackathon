# Use Case: Restrict Access to the Mesh using JSON Web Tokens 

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0 with a SMCP instance
3. Have installed the bookinfo example app on the `bookinfo` ns, part of SMMR and exposed gateway and virtual service
4. Have installed the RHSSO operator with a `keycloak` instance named `sso`

## Test bookinfo access
Retrieve the URL for the ingress gateway:
```
ISTIO_GW=$(oc get route istio-ingressgateway -n istio-system -o jsonpath="{.spec.host}{.spec.path}")
```
```
echo $ISTIO_GW
```
You can now verify that the bookinfo service is responding:
```
curl -v $ISTIO_GW/productpage
```
You should be able to get an HTTP 200 response code along with the `Simple Bookstore App` tittle.

## Setup a RHSSO Realm

In the same ns you installed the SSO operator (rhsso) and your keycloak instance (sso), find the `credential-sso` secret with the credentials needed to login into the sso admin console using the route in the same ns, e.g:
```
https://keycloak-rhsso.<WILDCARD_DOMAIN>.com/
```
Now save the credentials by exporting them to variables:
```
export RHSSO_REALM_USERID=<MY-USER-ID>
export RHSSO_REALM_PASSWD=<MY-USER-PASSWORD>
```
After login in you should be able to see the SSO admin console UI on the default realm out of the box called `master`. We are going to use this realm to create a client for our bookinfo app.

In the left bar menu under clients create a new client named `my-client` and hit `save`. Then make sure the `Standard Flow Enabled` is `off` and the `Service Accounts Enabled` flow is `on`, then finally check the `Access Type` is confidential then save.
After saving, go to the `credentials` tab on your client and copy the secret used to acquire the token, then export on your terminal the secret, the client ID and the SSO instance URL:
```
export SSO_CLIENT_ID=my-client
export SSO_CLIENT_SECRET=xxxxx-xxx-xxxx-xxxx-xxxxxxxxxx
export SSO_URL=keycloak-rhsso.<WILDCARD_DOMAIN>.com/
```
Now test you can get a token using the user credentials you got from the secret to access the SSO admin console:
```
TKN=$(curl -k -X POST \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=$RHSSO_REALM_USERID" \
 -d "password=$RHSSO_REALM_PASSWD" \
 -d "grant_type=client_credentials&client_id=$SSO_CLIENT_ID&client_secret=$SSO_CLIENT_SECRET" \
 https://$SSO_URL/auth/realms/master/protocol/openid-connect/token \
| sed 's/.*access_token":"//g' | sed 's/".*//g')
```
```
echo $TKN
```
Verify the contents by going to a jwt validator online like https://jwt.io and paste the content in the debugger. You should be able to validate in the payload the realm name and the client Id.

## Create RequestAuthentication and AuthorizationPolicy resources for bookinfo

The `RequestAuthentication` resource "tells" the Service Mesh control plane the JWT rules (issuer and URI) for the matched app label. The `AuthorizationPolicy` "tells" the SMCP to require an `Authorization` header for requests matching the app label `productpage`.

Review the `request-auth.yaml` file and replace the `issuer` and `jwksUri` with the corresponding URLs from your SSO realm instance, then create the resource:
```
oc apply -f Request-auth.yaml -n bookinfo
```
and the AuthorizationPolicy resource:
```
oc apply -f Auth-policy.yaml -n bookinfo
```
Now try again to access the bookinfo productpage:
```
curl -v $ISTIO_GW/productpage
```
You should get a 403 response like: `RBAC: access denied`. Now try again by getting a token as explain before and sending it in the header:
```
curl -v -H "Accept: application/json" -H "Authorization: Bearer $TKN" $ISTIO_GW/productpage
```
You should receive a 200 response code along with the header `Authorization` and value `Bearer $TKN` including the productpage payload.

Keep in mind the JWT token is setup to expire after 5 minutes so you may receive a 401 response code stating the JWT is `expired` meaning you need to get a new token.