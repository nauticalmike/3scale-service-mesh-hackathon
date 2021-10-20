# Use Case: OIDC Client Credentials Flow using 3scale adapter and Istio

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0 with a SMCP instance
3. Have installed the bookinfo example app on the `bookinfo` ns, part of SMMR and exposed gateway and virtual service, including adding the istio labels to the service
4. If you have created the `instance`, `handler` and `rule` as explained on the main README.md, please delete them
5. Have installed the RHSSO operator with a `keycloak` instance named `sso`
6. Have the bookinfo app setup as a 3scale product 

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
You should see an HTTP 401 Unauthorized response. Now lets try with the user-key token from 3scale product app:
```
curl -v $ISTIO_GW/productpage?user_key={user key}
```
You should be able to get an HTTP 200 response code along with the `Simple Bookstore App` tittle.

## Setup Service Mesh resources for bookinfo

As explained in the main lab (README.md), create the handler, the instance and the rule for the bookinfo service on the `istio-system` ns. Take a look at the instance and rule they look pretty similar and update the `handler` to include your 3scale access token and tenant URL: 
```
oc apply -f use-cases/client-credentials-flow/Handler.yaml -f use-cases/client-credentials-flow/Instance.yaml -f use-cases/client-credentials-flow/Rule.yaml -n istio-system
```
Now create the `RequestAuthentication` resource which "tells" the Service Mesh control plane the JWT rules (issuer and URI) for the matched app label.
Review the `request-auth.yaml` file and replace the `issuer` and `jwksUri` with the corresponding URLs from your SSO realm instance, e.g:
```
spec:
  jwtRules:
    - issuer: >-
        http://keycloak-insecure-rhsso.apps.cluster-c957.c957.sandbox749.opentlc.com/auth/realms/threescale-realm
      jwksUri: >-
        http://keycloak-insecure-rhsso.apps.cluster-c957.c957.sandbox749.opentlc.com/auth/realms/threescale-realm/protocol/openid-connect/certs
```
then create the resource:
```
oc apply -f use-cases/restrict-access-jwt/request-auth.yaml -n bookinfo
```

## Setup the Realm clients

In the same ns you installed the SSO operator (rhsso) and your keycloak instance (sso), find the `credential-sso` secret with the credentials needed to login into the keycloak admin console using the route in the same ns, e.g:
```
https://keycloak-rhsso.<WILDCARD_DOMAIN>.com/
```
These credentials should have you login as an `admin` into the `master` realm. On the left bar menu click on the `master` realm name and then on `add realm` and upload the file in this folder named: `rhsso-realm.json`. This would create the `threescale-realm` realm and the user `user1` to be used on our lab. Log out and try to log into the `threescale-realm` using the username `user1` and the password `openshift`.

Now save the credentials by exporting them to variables on your terminal:
```
export RHSSO_REALM_USERID=user1
export RHSSO_REALM_PASSWD=openshift
```
After login in you should be able to see the keycloak admin console UI on the realm called `threescale-realm`. We are going to use this realm to create a client for our bookinfo app.

In the left bar menu under clients create a new client named `zync-sso` and hit `save`. Then make sure the `Standard Flow Enabled` is `off` and the `Service Accounts Enabled` flow is `on`, then finally check the `Access Type` is confidential then save.
This configures the `zync` client on the realm to be used by 3scale's OIDC Zync component using the Service Account flow to automatically create realm clients when 3scale product applications are created. Take note of the zync client name (`zync-sso` if not changed) and secret to be used as follows.

In 3scale admin's console go to the bookinfo product Integration -> Settings -> Deployment
and change it to: `Istio`. Then go below to the Authentication section and change it from `user_key` to `OpenID Connect`. This would present the field to specify the OIDC issuer URL, e.g:
```
https://<CLIENT_ID>:<CLIENT_SECRET>@<HOST>:<PORT>/auth/realms/<REALM_NAME>
```
Where client ID would be `zync-sso`, client secret would be the secret saved from the credentials tab on the client's page, then host would be your keycloak host URL and port and the realm name would be `threescale-realm`. Finally make sure the checkbox that reads `Service Accounts Flow` is checked and hit the blue button that states "Update Product".

After this you can go to the bookinfo's product Applications -> Listing then click on the `Developer` account or the account you used for your previous app and in the account page click on the `N Applications` link at the top. Then click on the green link on the right of the application list page named "Create Application" and create a new application named `bookinfo-app-oidc`, select the bookinfo-plan and hit "Create Application".
The application page should state the client ID and secret under the API Credentials section. If everything worked as expected, you can double check this same client ID under the list of clients in your realms client list on the keycloak's console. 

Save the client ID and secret under the API Credentials by exporting them to terminal variables, e.g:
```
export SSO_CLIENT_ID=4cedb1d3
export SSO_CLIENT_SECRET=184d6e7dc3415c73081b087e1f11e930
export SSO_URL=keycloak-rhsso.apps.cluster-c957.c957.sandbox749.opentlc.com
```
Now you can test getting a token from your keycloak instance, using the user credentials, client ID and secret as follows:
```
TKN=$(curl -k -X POST \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=$RHSSO_REALM_USERID" \
 -d "password=$RHSSO_REALM_PASSWD" \
 -d "grant_type=client_credentials&client_id=$SSO_CLIENT_ID&client_secret=$SSO_CLIENT_SECRET" \
 http://$SSO_URL/auth/realms/threescale-realm/protocol/openid-connect/token \
| sed 's/.*access_token":"//g' | sed 's/".*//g')
```
Validate you got a proper token:
```
echo $TKN
```
Verify the contents by going to a jwt validator online like https://jwt.io and paste the content in the debugger. You should be able to validate in the payload the realm name and the client Id.

## Testing the OIDC flow 

Now try again to access the bookinfo productpage:
```
curl -v $ISTIO_GW/productpage
```
You should get a 401 response like: `Unauthorized`. Now try again by getting a token as explain before and sending it in the header:
```
curl -v -H "Accept: application/json" -H "Authorization: Bearer $TKN" $ISTIO_GW/productpage
```
You should receive a 200 response code along with the header `Authorization` and value `Bearer $TKN` including the productpage payload.

Keep in mind the JWT token is setup to expire after 5 minutes so you may receive a 401 response code stating the JWT is `expired` meaning you need to get a new token.