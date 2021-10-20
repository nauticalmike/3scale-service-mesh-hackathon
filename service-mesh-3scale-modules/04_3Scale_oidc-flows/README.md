# Use Case: 3Scale OIDC flows using Apicast Gateway

The `bookinfo` APIs you deployed and configured until now on the 3scale API Management platform were secured with an API Key (user key), or with a combination of an App ID and an App Key.

In this lab you'll secure an API with Open ID Connect (OIDC), using Red Hat Single Sign-On (RH-SSO) as identity provider.

## Goals
* Synchronize between 3scale and RH-SSO
* Secure APIs using OpenID Connect and _Authorization Code_ flow protocol

## Prerequisites

1. Have an OCP v4.x running cluster
2. Have 3scale installed
3. Have installed the bookinfo example app on the `bookinfo` ns
4. Have installed the RHSSO operator with a `keycloak` instance named `sso`
5. You don't need OSSM for this lab so make sure there are no conflicting resources assigned to the bookinfo service (Virtual Service/Gateway)

#
## Introduction
### Environment Variables

The instructions in this lab refer to several environment variables. As part of the instructions, you need to replace these environment variables with their value. This can be done automatically by setting the environment variables in the current shell. Beware that every time you close or change the terminal window the environment variables need to be set again. 
## Setup the Realm clients

In the same ns where you installed the SSO operator (rhsso) and your keycloak instance (sso), find the route to access your keycloak instance:
```
https://keycloak-rhsso.<WILDCARD_DOMAIN>.com/
```
Now lets save this URL on a variable we can use later:
```
$ export RHSSO_URL=<URL of the RH-SSO server>
```

In the same ns find the `credential-sso` secret with the credentials needed to login into the keycloak admin console using the route in `RHSSO_URL`.

These credentials should have you logged in as an `admin` into the `master` realm. On the left bar menu click on the `master` realm name and then on `add realm` and upload the file in this folder named: `rhsso-realm.json`. This would create the `threescale-realm` realm and the user `user1` to be used on our lab. 

Now lets save into a variable the realm name:
```
$ export AMP_SSO_REALM=threescale-realm
```

Log out and try to log into the `threescale-realm` using the username `user1` and the password `openshift`.

Now save the credentials by exporting them to variables on your terminal:
```
$ export RHSSO_REALM_USERID=user1
$ export RHSSO_REALM_PASSWD=openshift
```
After login in you should be able to see the keycloak admin console UI on the realm called `threescale-realm`. We are going to use this realm to create a client for our bookinfo app.
#
## Realm User Registration: Only if you can't login with user1

This section of the lab assumes your instructor has already provisioned an SSO realm for you. Only use this section if you can't login using the provided user1.

The purpose of this section is to provide details regarding how to register as an administrative user of that previously provisioned SSO realm.

Your realm is configured such that the first user that registers becomes the realm administrator. If when you login a *403 Forbiden* message shows up, just follow the instructions bellow to add the needed role mappings to your user.

Obtain the URL to the console of your target realm:

```
$ echo -en "\nhttps://$RHSSO_URL/auth/admin/$AMP_SSO_REALM/console\n\n"
```

Open a web browser and navigate to the URL. Expect to see the RH-SSO login screen:

![](../images/rhsso_realm_login.png)

In a new tab navigate to the same URL, log in the Realm and make sure that the *User registration* option is enabled (The user and password is in the secret *credential-sso*

![](../images/rhsso_realm_user_registration_enable.png)

If the 'User registration' field is OFF, change it to ON and click 'Save', then logout from this session and go back to the previous TAB

Click the *Register* link to register as a new user.

Fill in all fields of the registration form.  The email address has to be a valid, but not necessarily a genuine email address.

Click *Register*.

You are redirected to the realm overview page. The newly registered realm user has administrative access to all settings of the realm.

**NOTE**
If you see an 403 error when redirected to the realm overview page, than follow the 4 steps bellow to correct this:

- You will need to change your new user Role Mapping, using the RHSSO admin user. To do so, log in the rhsso as admin (the username and password can be retrieved in the secret credential-sso, inside the RHSSO project on openshift), after login go to the $AMP_SSO_REALM and select *Users*

![](../images/rhsso_realm_user_page.png)

- In the list of users that is presented, click on the id of the user you just created:

![](../images/rhsso_realm_user_list_page.png)

- Then go to the *Role Mappings* tab, in *Client Roles* write 'realm-management' and in the list of *Available Roles* select *manage-realms* and *realm-admin*, then click on the *Add selected* option.

![](../images/rhsso_realm_user_rolemapping_page.png)

- Now you can go back to the tab where you were redirected and refresh the page, you will now see the Realm configs.

![](../images/rhsso_realm_overview_page.png)

In a terminal window of your local machine, set environment variables specific to this new realm user:
```
$ export RHSSO_REALM_USERID=<change me>
$ export RHSSO_REALM_PASSWD=<change me>
```
You will reference these environment variables in later in this lab.
#
## 3scale and Red Hat Single Sign-On Integration
### Overview

Every OIDC secured _application_ in 3scale needs a corresponding SSO client in RH-SSO.
The details between the API managed _application_ should match the details of the corresponding SSO _client_.

Subsequently, it is important to automate the synchronization between 3scale and RH-SSO.

The _zync_ component in 3scale is the component responsible for this synchronization.
Its purpose is to synchronize API managed _applications_ to external systems such as RH-SSO.
It does so in an automated manner by pushing updates to external systems every time a state change occurs with an API managed _application_.

In this section, you create a SSO client in the realm of your RH-SSO server that will be leveraged by the _zync_ component of the 3scale API management platform.
Zync will use the new SSO client to synchronize between OIDC secured _applications_ and their corresponding SSO _clients_ in RH-SSO.

The name of this new SSO client will be called:  _zync-sso_.
The _zync-sso_ client will be configured to utilize the OIDC _client credentials_ flow.
Via the _zync-sso_ client, the zync component will retrieve an access token that then allows for the creation and update of new SSO clients through the RH-SSO API.

**NOTE**
Within OIDC and OAuth, the Client Credential Flow is used for server-to-server scenarios. In this scenario, the client application (in our case the 3scale zync component) is a confidential client that’s acting on its own, not on behalf of the user. It’s more of a service account type of scenario. It’s a back channel flow to obtain an access token using the client’s credentials.
A good explanation of the different OIDC flows can be found here: https://developer.okta.com/blog/2017/06/21/what-the-heck-is-oauth.

### Create _zync_sso_ client

In a browser window, navigate to the console of your realm on the RH-SSO server.

On the right menu, click *Clients* to open the realm clients overview page,

Click *Create* to create a new realm client.

![](../images/rhsso_realm_new_client.png)

In the _Add Client_ page, enter the following values:
* *Client ID*: `zync-sso`
* *Client Protocol*: `openid-connect`
* *Root URL*: leave blank

Click *Save* to create the realm client.

You are redirected to the client _Settings_ page.

![](../images/rhsso_realm_client_settings.png)

Make the following changes to the client settings:
* Set *Access Type* to `Confidential`.
* Set *Standard Flow Enabled* to `OFF`.
* Set *Direct Access Grants Enabled* to `OFF`.
* Set *Service Accounts Enabled* to `ON`.

![](../images/rhsso_realm_client_settings_client_credentials.png)

Scroll down and click *Save* to save the client configuration.

Navigate to the *Service Account Roles* tab of the client settings.

In the *Client Roles* dropdown, type `realm-management`.

Ensure that the `manage-clients` role is listed as one of the *Assigned Roles*.

![](../images/rhsso_realm_client_service_account_roles.png)

Take note of the _Client ID_ - which is `zync-sso` unless you changed the value.

Take note of the _Client Secret_. You can find the value of the Client 

Secret on the *Credentials* page of the client.

![](../images/rhsso_realm_client_credentials.png)

Set environment variables for the Client ID and Client Secret:
```
$ export ZYNC_SSO_CLIENT_ID=zync-sso
$ export ZYNC_SSO_CLIENT_SECRET=<zync_sso client secret>
```
#
## Create an OIDC Secured API
### Create API and Configure to Use OIDC

In this section of the lab you configure an API product on 3scale protected with OpenID Connect.

The backend for this new API is the `bookinfo` _productpage service_ which you leveraged in previous labs as well.

In a browser window, navigate to your tenant's 3scale Admin Portal.
Navigate to *Dashboard*, in the *Products* window, click *Create Product*.
On the _New Product_ page:
* Select *Define manually*
* *Name*: `Bookinfo API OIDC`
* *System Name*: `bookinfo_api_oidc`
Select *Create Product* to create the API.
In the *Overview* page that is presented after you create your Product , click on *Create Application Plan* link and fill with the follow information:
* *Name*: `Bookinfo OIDC Basic Plan`
* *System Name*: `bookinfo_oidc/basic`
* Click *Create Application Plan*
Publish the application plan.

![](../images/amp_admin_portal_application_plan_publish.png)

Before adding the `productpage` service backend to your `bookinfo` product, make sure is responsive by testing it on the terminal:
```
oc run nc-rhel --image=registry.redhat.io/rhel7/rhel-tools -i -t --restart=Never --rm -- /bin/sh -c 'curl -v productpage.bookinfo.svc.cluster.local:9080' | grep "<title>Simple Bookstore App</title>"
```
You should see a response like: `<title>Simple Bookstore App</title>`. 
Now add the backend to the API. Select the `Products Service Backend` backend, and set the path to `/`. To do this, go to *Products* -> *Products API OIDC* -> *Integration* -> *Backends* and click *Add Backend*

![](../images/amp_admin_portal_product_backend.png)

Use the local cluster service as `private base URL` with value `http://productpage.bookinfo.svc.cluster.local:9080`:
![](../images/backend.png)

The default mapping rules allow all `GET` operations, which is sufficient for this lab.

On the *Settings* page of the API:
* Select `APIcast managed`

The zync component should generate the routes but if not, then:
* Set the *Staging Public Base URL* to `https://bookinfo-api-oidc-${OCP_USER}-staging.${OCP_WILDCARD_DOMAIN}`
* Set the *Production Public Base URL* to `https://bookinfo-api-oidc-${OCP_USER}.${OCP_WILDCARD_DOMAIN}`

In the *Authentication* section, select `OpenID Connect`.

![](../images/amp_admin_portal_product_settings_authentication.png)

Scroll down and fill in the details for the authentication settings:
* *OpenID Connect Issuer Type*: `Red Hat Single Sign-On`
* Set the value of *OpenID Connect Issuer* to the URL of your realm on RH-SSO, to which you add the _zync_sso_ client ID and secret for authentication. 
You can obtain the URL from the output of the following command:
```
$ echo -en "\n\nhttps://$ZYNC_SSO_CLIENT_ID:$ZYNC_SSO_CLIENT_SECRET@$RHSSO_URL/auth/realms/$AMP_SSO_REALM\n\n"
```
**NOTE**
---
This URL serves the following purposes:

* Provides zync with the URL to add or update SSO clients in RH-SSO.
* Provides APIcast with the URL to RH-SSO to retrieve the public key of the RH-SSO realm in order to verify the JWT token.
---


* In the *OIDC AUTHORIZATION FLOW* section, ensure that the _Authorization Code Flow_ checkbox is checked.

![](../images/amp_admin_portal_product_settings_authentication_settings.png)
* Scroll down and change *Credentials Location* to _As HTTP Headers_.

![](../images/amp_admin_portal_product_settings_authentication_jwt.png)
Click *Update Product* to update the settings.

Go to Integration -> Configuration and check if the Apicast staging and Apicast production routes were generated. If not, then you can create the routes as follows:
* Create the OpenShift routes to the staging and production APIcast gateways. 
Using the `oc` CLI tool:
```
$ oc create route edge products-oidc-staging-route  --service=apicast-stage --hostname=bookinfo-api-oidc-${OCP_USER}-staging.${OCP_WILDCARD_DOMAIN} -n ${OCP_USER}-3scale-gw
$ oc create route edge products-oidc-production-route  --service=apicast-prod --hostname=bookinfo-api-oidc-${OCP_USER}.${OCP_WILDCARD_DOMAIN} -n ${OCP_USER}-3scale-gw
```
#
## Create Application

In this section of the lab you create an application for the `bookinfo` API OIDC_ API product.

In a browser window, navigate to your tenant's 3scale Admin Portal.
Navigate to *Audience -> Accounts -> Listing*, and either use the developer account or click to create a new account:
![](../images/amp_admin_portal_account_application.png)

Then under the account selected click on "Create Application":
![](../images/amp_admin_portal_account_application_create.png)

Create a new _Application_ for the TestOIDC account.
* *Application Plan*: `bookinfo Plan`
* *Name*: `bookinfo App`

On the overview page for the application, click edit in the _API Credentials_ section.
![](../images/amp_admin_portal_application_api_credentials.png)
Set the redirect URL to `https://www.getpostman.com/oauth2/callback`
![](../images/amp_admin_portal_application_api_credentials_edit_url.png)

* This redirect URL will become useful later in this lab when you test using the _Authorization Code_ OAuth2 flow.
* Notice that the API Credentials for this application do no consist of a user key, but rather a Client ID and a Client Secret.

In a terminal window of your local machine, set environment variables referring to the Client ID and Secret of the application:
```
$ export PRODUCTS_OIDC_APP_CLIENT_ID=<change me>
$ export PRODUCTS_OIDC_APP_CLIENT_SECRET=<change me>
```
You will reference these environment variables in later in this lab. 

### Verify Application Client in RH-SSO

As a result of creating the application, the 3scale zync component has created a new realm client for your realm in the RH-SSO server.

Log in to the realm console of your realm in RH-SSO, and navigate to the *Clients* section.
Notice that a new client has been created with a client ID corresponding to the client ID of the `bookinfo app` application in 3scale.
![](../images/rhsso_realm_client_zync_created.png)
Click the *Edit* button for the new client.
* Notice that the client name corresponds to the name of the application in 3scale.
Set *Direct Grant Enabled* to `ON`.
![](../images/rhsso_realm_client_settings_direct_access_grant.png)

This will allow to test the OIDC setup using a token obtained from the realm with a username and password.
Click *Save* to save the change.

## Test the OIDC Secured API

In this section of the lab you test the OIDC secured Products API using the OIDC _Resource Owner Password Credentials_ and the _Authorization Code_ flows.

### Promote to Staging and Production APIcast

In the 3scale Admin Portal, navigate to *Product: bookinfo -> Integration -> Configuration*.
Promote the API to the staging and production APIcast.

### Test the API with Resource Owner Password Credentials flow.
Obtain a token from the RH-SSO client for the `bookinfo` application. Use the username and the password you used to register with the your RH-SSO realm.
```
TKN=$(curl -X POST \
 -H "Content-Type: application/x-www-form-urlencoded" \
 -d "username=$RHSSO_REALM_USERID" \
 -d "password=$RHSSO_REALM_PASSWD" \
 -d "grant_type=password" \
 -d "client_id=$PRODUCTS_OIDC_APP_CLIENT_ID" \
 -d "client_secret=$PRODUCTS_OIDC_APP_CLIENT_SECRET" \
 http://$RHSSO_URL/auth/realms/$AMP_SSO_REALM/protocol/openid-connect/token \
| sed 's/.*access_token":"//g' | sed 's/".*//g')
```
Verify the result of the command. Expect something like:
```
$ echo $TKN
eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJZdk1xOTNMWE1hSWtHMmZKSldTUjJxSmRYd0NkRE9Xck9Ncm9XUHBWZFl3In0.eyJleHAiOjE2MTE4NTQ5NjQsImlhdCI6MTYxMTg1NDY2NCwianRpIjoiMjBjYmQwMjktNDg1OC00ZTQ0LTk5M2YtYzczODNhZmJlNjU2IiwiaXNzIjoiaHR0cHM6Ly9yaHNzby1yaHNzby5hcHBzLmNsdXN0ZXItZTE4OS5lMTg5LnNhbmRib3gxMjA4Lm9wZW50bGMuY29tL2F1dGgvcmVhbG1zL2FtcC1yZWFsbS11c2VyNTAiLCJhdWQiOlsicmVhbG0tbWFuYWdlbWVudCIsImFjY291bnQiXSwic3ViIjoiMDdjZDhhMzEtOWQ4ZS00YWMxLWFhOTAtYmNmODA2NWQ3ODUyIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiMjJjMmFkZGIiLCJzZXNzaW9uX3N0YXRlIjoiZmFjYTk1OTQtNjRjNy00NWJjLTkwNzYtYTg3YjM3OGU4MmM2IiwiYWNyIjoiMSIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsicmVhbG0tbWFuYWdlbWVudCI6eyJyb2xlcyI6WyJ2aWV3LWlkZW50aXR5LXByb3ZpZGVycyIsInZpZXctcmVhbG0iLCJtYW5hZ2UtaWRlbnRpdHktcHJvdmlkZXJzIiwiaW1wZXJzb25hdGlvbiIsImNyZWF0ZS1jbGllbnQiLCJtYW5hZ2UtdXNlcnMiLCJxdWVyeS1yZWFsbXMiLCJ2aWV3LWF1dGhvcml6YXRpb24iLCJxdWVyeS1jbGllbnRzIiwicXVlcnktdXNlcnMiLCJtYW5hZ2UtZXZlbnRzIiwibWFuYWdlLXJlYWxtIiwidmlldy1ldmVudHMiLCJ2aWV3LXVzZXJzIiwidmlldy1jbGllbnRzIiwibWFuYWdlLWF1dGhvcml6YXRpb24iLCJtYW5hZ2UtY2xpZW50cyIsInF1ZXJ5LWdyb3VwcyJdfSwiYWNjb3VudCI6eyJyb2xlcyI6WyJtYW5hZ2UtYWNjb3VudCIsIm1hbmFnZS1hY2NvdW50LWxpbmtzIiwidmlldy1wcm9maWxlIl19fSwic2NvcGUiOiJwcm9maWxlIGVtYWlsIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJuYW1lIjoiQmVybmFyZCBUaXNvbiIsInByZWZlcnJlZF91c2VybmFtZSI6ImJ0aXNvbiIsImdpdmVuX25hbWUiOiJCZXJuYXJkIiwiZmFtaWx5X25hbWUiOiJUaXNvbiIsImVtYWlsIjoiYnRpc29uK3Nzb0ByZWRoYXQuY29tIn0.BDo2tWXypdPrhP8Vlc8BajhbIQMefyFtvSDbFnlwwbfgePcIpjAJNN599jtOHJg3P4JBN2ljaDfi92c0WwaqM1XYXc5qGJ5IApPpGRQ3ahCHH25ZR5QUg2GmcW4ifBcUI0kHoe9Sd3Cr1i7ym5BqPeTIpziOAvMTC4C5nDdgL_DEpMOYnO8duzcOOVFIS_wdrA-9duRaMgIS5MOvnh2sFpfMIdLuuP5haOUnsrt8GDm2HekMRZmnmwiiyO7i-kypMFpygvAOg8CkUpBwAC-pChd9CAmv0mBCkfJbflrlk8urJkkITOMvznpmkrnjxh-a7RYedt3CGkLI88ZIewUmLw
```

Verify the token contents:
* In a browser window, navigate to https://jwt.io.
* Paste the contents of the token in the right pane of the _Debugger_ on the jwt.io homepage.
![](../images/jwt_io.png)
* Notice the contents of the `azp` field in the token, which corresponds to the application ID in 3scale. 
APIcast uses the contents of this field to identify the target application in order to apply the policies and limits defined for the application.
A token is only valid for 5 minutes. At any time you can obtain a new token using the _curl_ command above.
Test the API using the JWT token to authenticate. +
Using _curl_, issue the following command:

```
$ curl -v -X GET -H "Accept: application/json" -H "Authorization: Bearer $TKN" http://bookinfo-3scale-apicast-staging.apps.cluster-c957.c957.sandbox749.opentlc.com
```

Expected Output (truncated)
```
*   Trying 34.240.138.165:443...
* Connected to products-api-oidc-user50.apps.cluster-e189.e189.sandbox1208.opentlc.com (34.240.138.165) port 443 (#0)
[...]
> GET /api/products HTTP/1.1
> Host: products-api-oidc-user50.apps.cluster-e189.e189.sandbox1208.opentlc.com
> User-Agent: curl/7.71.1
> Accept: application/json
> Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cCIgOiAiSldUIiwia2lkIiA6ICJZdk1xOTNMWE1hSWtHMmZKSldTUjJxSmRYd0NkRE9Xck9Ncm9XUHBWZFl3In0.eyJleHAiOjE2MTE4NjQ2NTYsImlhdCI6MTYxMTg2NDM1NiwianRpIjoiM2FlYTJmZmUtZGFhNy00YTQ3LTk5NDktZmM1MmU1ZDVhZmFhIiwiaXNzIjoiaHR0cHM6Ly9yaHNzby1yaHNzby5hcHBzLmNsdXN0ZXItZTE4OS5lMTg5LnNhbmRib3gxMjA4Lm9wZW50bGMuY29tL2F1dGgvcmVhbG1zL2FtcC1yZWFsbS11c2VyNTAiLCJhdWQiOlsicmVhbG0tbWFuYWdlbWVudCIsImFjY291bnQiXSwic3ViIjoiMDdjZDhhMzEtOWQ4ZS00YWMxLWFhOTAtYmNmODA2NWQ3ODUyIiwidHlwIjoiQmVhcmVyIiwiYXpwIjoiMjJjMmFkZGIiLCJzZXNzaW9uX3N0YXRlIjoiYmE1OWI1OTAtZjBhNi00NDEzLWFlNDktOWQ3ZGI4ZWE1Mjc5IiwiYWNyIjoiMSIsInJlYWxtX2FjY2VzcyI6eyJyb2xlcyI6WyJvZmZsaW5lX2FjY2VzcyIsInVtYV9hdXRob3JpemF0aW9uIl19LCJyZXNvdXJjZV9hY2Nlc3MiOnsicmVhbG0tbWFuYWdlbWVudCI6eyJyb2xlcyI6WyJ2aWV3LWlkZW50aXR5LXByb3ZpZGVycyIsInZpZXctcmVhbG0iLCJtYW5hZ2UtaWRlbnRpdHktcHJvdmlkZXJzIiwiaW1wZXJzb25hdGlvbiIsImNyZWF0ZS1jbGllbnQiLCJtYW5hZ2UtdXNlcnMiLCJxdWVyeS1yZWFsbXMiLCJ2aWV3LWF1dGhvcml6YXRpb24iLCJxdWVyeS1jbGllbnRzIiwicXVlcnktdXNlcnMiLCJtYW5hZ2UtZXZlbnRzIiwibWFuYWdlLXJlYWxtIiwidmlldy1ldmVudHMiLCJ2aWV3LXVzZXJzIiwidmlldy1jbGllbnRzIiwibWFuYWdlLWF1dGhvcml6YXRpb24iLCJtYW5hZ2UtY2xpZW50cyIsInF1ZXJ5LWdyb3VwcyJdfSwiYWNjb3VudCI6eyJyb2xlcyI6WyJtYW5hZ2UtYWNjb3VudCIsIm1hbmFnZS1hY2NvdW50LWxpbmtzIiwidmlldy1wcm9maWxlIl19fSwic2NvcGUiOiJwcm9maWxlIGVtYWlsIiwiZW1haWxfdmVyaWZpZWQiOmZhbHNlLCJuYW1lIjoiQmVybmFyZCBUaXNvbiIsInByZWZlcnJlZF91c2VybmFtZSI6ImJ0aXNvbiIsImdpdmVuX25hbWUiOiJCZXJuYXJkIiwiZmFtaWx5X25hbWUiOiJUaXNvbiIsImVtYWlsIjoiYnRpc29uK3Nzb0ByZWRoYXQuY29tIn0.G7p3f4z3a14lSV_EstlRCffwUNbFDippU99ePhFNZ6wK9XvbczCTpm8xDDDvx0zHuXScYYjAyP_7f5zKzYIuLfT6FzzoCDuoqkJaVEgm8_M68pt6LnOZgFkskUj3RhnRfB5ZOXg8oNKE4uj4haCCM_f0R9mqum0utIGzE4IgL2yGgYTDVFJXPrGnoExaOQg1_fpuuDOblOwoliV97Ls5LYIGbW7rVO5CBQheJJZhfm5RrpeD9HEvFQZG2KIRmP4ivBFQckfZR-8t14YsGvH5_dtJuvlE_-3xVnB8lEUX3l4klAXvtPdBh3vceUXqRx3H9dIPLv7SwNSlFvcateVkwQ
> 
< HTTP/1.1 200 OK
< server: openresty
< date: Wed, 13 Oct 2021 16:11:22 GMT
< content-type: text/html; charset=utf-8
< content-length: 1683
< set-cookie: d19dafa8e1c06de5381ba97b6bb892e9=b06a6d046f8df3963609446ef21d6393; path=/; HttpOnly
< cache-control: private
< 
<!DOCTYPE html>
<html>
  <head>
    <title>Simple Bookstore App</title>
```

* Expect a HTTP `200 OK` response code.
* If the token is expired or otherwise invalid (e.g. tampered with), expect an HTTP error return code `403 Forbidden`. 

#
### Test the API with Authorization Code flow.

In the previous section of this lab, you used the _Resource Owner Password Credentials_ flow of the OAuth2 specification to obtain an access token.
That flow was helpful as a simple mechanism to test initial SSO configurations between your API gateway, the API Manager, Red Hat SSO and your backend service.

In this exercise, you use the more real-world OAuth2 flow called _Authorization Code_ to obtain your SSO access token.

### Install and Configure Postman
Install Postman and check if its working:
```
$ which Postman
```

Open the Postman application. If this is the first time you used Postman, expect to be greeted with a sign-up page. Feel free to skip this stage and go directly to the application.

![](../images/postman_signup_page.png)

Expect to see the landing page of the Postman application:
![](../images/postman_empty_home_page.png)

Click *Create a request*. 
Enter the URL to the production APIcast of the `bookinfo` application in the *Enter request URL* text box. 
The URL can be obtained from the following command:
```
$ echo -en "\nhttp://bookinfo-3scale-apicast-staging.apps.cluster-c957.c957.sandbox749.opentlc.com\n\n"
```

![](../images/postman_request_url.png)

Click *Send*
Expect a `403 Forbidden` return code, and a response `Authentication parameters missing`.

![](../images/postman_response_forbidden.png)

Configure Postman to obtain an access token from the RH-SSO server.
Click the *Authorization* tab.
From the *Type* field, select _OAuth 2.0_.
Enter the following values into the *Configure New Token* dialog box:
* *Token Name*: `bookinfo Access Token`
* *Grant Type*: `Authorization Code`
* *Callback URL*: `https://www.getpostman.com/oauth2/callback`
* *Auth URL*: Use the following command to get the URL:

```
$ echo -en "\nhttps://$RHSSO_URL/auth/realms/$AMP_SSO_REALM/protocol/openid-connect/auth\n\n"
```

* *Access Token URL*: Use the following command to get the URL:
```
$ echo -en "\nhttps://$RHSSO_URL/auth/realms/$AMP_SSO_REALM/protocol/openid-connect/token\n\n"
```

* *ClientID*: The value of `$PRODUCTS_OIDC_APP_CLIENT_ID`
* *Client Secret*: The value of `$PRODUCTS_OIDC_APP_CLIENT_SECRET`
* *Scope* : openid
* *Client Authentication* : `Send as Basic Auth header`
![](../images/postman_configure_new_token.png)

Click *Get New Access Token*.
* A new dialog box appears that shows the login screen for your realm on the RH-SSO server
![](../images/postman_request_token_login.png)

Enter the username and password of a realm user.  For now, you can just use the values of `$RHSSO_REALM_USERID` and `$RHSSO_REALM_PASSWD`. Click *Log in*.

A new pop-up appears that shows the details of the Access token that was obtained from the RH-SSO server.

Click *Use Token*
![](../images/postman_manage_access_token.png)

Back on the request page, click *Send*. 
This time expect a successful response.
![](../images/postman_response_ok2.png)

You have successfully secured your API using OpenID Connect.
