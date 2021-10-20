# Use Case: OIDC Authorization Code Flow using 3scale and OSSM

## Prerequisites

1. Have an OCP v4.x running cluster
2. OSSM v2.0 with a SMCP instance
3. Have installed the bookinfo example app on the `bookinfo` ns, part of SMMR and exposed gateway and virtual service, including adding the istio labels to the service
4. If you have created the `instance`, `handler` and `rule` as explained on the main README.md, please delete them
5. Have installed the RHSSO operator with a `keycloak` instance named `sso`
6. Have the bookinfo app setup as a 3scale product and protected using an API key
7. Have Postman installed

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
You should see an HTTP 401 Unauthorized response if you have the bookinfo app setup as a product and protected using an API key. Now lets try with the user-key token from 3scale product app:
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
oc apply -f Request-auth.yaml -n bookinfo
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
Where client ID would be `zync-sso`, client secret would be the secret saved from the credentials tab on the client's page, then host would be your keycloak host URL and port and the realm name would be `threescale-realm`. 

Finally make sure the checkbox that reads `Authorization Code Flow` is checked and hit the blue button that states "Update Product".

After this you can go to the bookinfo's product Applications -> Listing then click on the `Developer` account or the account you used for your previous app and in the account page click on the `N Applications` link at the top. Then click on the green link on the right of the application list page named "Create Application" and create a new application named `bookinfo-app-auth-flow`, select the bookinfo-plan and hit "Create Application".

The application page should state the client ID and secret under the API Credentials section. In the redirect URL field add:
```
https://www.getpostman.com/oauth2/callback
```

If everything worked as expected, you can double check this same client ID under the list of clients in your realms client list on the keycloak's console. 

Save the client ID and secret under the API Credentials by exporting them to terminal variables, e.g:
```
export SSO_CLIENT_ID=4cedb1d3
export SSO_CLIENT_SECRET=184d6e7dc3415c73081b087e1f11e930
export SSO_URL=keycloak-rhsso.apps.cluster-c957.c957.sandbox749.opentlc.com
```

## Test the API with Authorization Code flow.

Check you have Postman installed:

```
$ which Postman
```
Open the Postman application. If this is the first time you used Postman, expect to be greeted with a sign-up page. Feel free to skip this stage and go directly to the application:

![](../images/postman_signup_page.png)

Expect to see the landing page of the Postman application:
![](../images/postman_empty_home_page.png)

Click *Create a request*. 

Enter the URL to the production APIcast of the _Products API OIDC_ application in the *Enter request URL* text box. 

The URL can be obtained from the following command:

```
$ echo $ISTIO_GW/productpage
```

![](../images/postman_request_url.png)

Click *Send*

Expect a `401 Unauthorized` return code.

![](../images/postman_response_unauthenticated.png)

### Configure Postman to obtain an access token from the RH-SSO server.

Click the *Authorization* tab.

From the *Type* field, select _OAuth 2.0_.

Enter the following values into the *Configure New Token* dialog box:
* *Token Name*: `productpage Access Token`
* *Grant Type*: `Authorization Code`
* *Callback URL*: `https://www.getpostman.com/oauth2/callback`
* *Auth URL*: Use the following command to get the URL:
```
$ echo -en "\nhttps://$SSO_URL/auth/realms/$AMP_SSO_REALM/protocol/openid-connect/auth\n\n"
```
* *Access Token URL*: Use the following command to get the URL:
```
$ echo -en "\nhttps://$SSO_URL/auth/realms/$AMP_SSO_REALM/protocol/openid-connect/token\n\n"
```
* *ClientID*: The value of `$SSO_CLIENT_ID`
* *Client Secret*: The value of `$SSO_CLIENT_SECRET`
* *Scope* : openid
* *Client Authentication* : `Send as Basic Auth header`

![](../images/postman_configure_new_token.png)

Click *Get New Access Token*.

A new dialog box appears that shows the login screen for your realm on the RH-SSO server

![](../images/postman_request_token_login.png)

Enter the username and password of a realm user.  For now, you can just use the values of `$RHSSO_REALM_USERID` and `$RHSSO_REALM_PASSWD`. Click *Log in*.

A new pop-up appears that shows the details of the Access token that was obtained from the RH-SSO server.

Click *Use Token*

![](../images/postman_manage_access_token.png)

Back on the request page, click *Send*. 

This time expect a successful response.

![](../images/postman_response_ok2.png)

You have successfully secured your the `bookinfo` `productpage` using OpenID Connect Authorization code flow.