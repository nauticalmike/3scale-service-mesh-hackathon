# Secure the Bookinfo App Using OSSM

### Configure BookInfo 3scale Product

Log into 3scale using the route whose hostname begins with `3scale-admin` in the `3scale` namespace.

NOTE: You will find the admin username and password in a secret called `system-seed` in the `3scale` namespace.

1. Create a new Product and give it any name
2. If using the bookinfo app make sure you add the mapping rule `/productpage`
3. Go to Integration->Settings and choose `Istio` as the deployment
4. Go to Integration->Configuration and Promote the config
5. Go to Overview and create an application plan
6. Go to Applications->Application Plans and publish the application plan
7. Go to the Product Overview and take note of the ID given to the API (This will be used in the next step)

### Label Pods in BookInfo

Adjust the pod spec template for your target service. You will need two values:
- The 3scale API ID (from the previous step)
- The name of your handler

The pod labels will look something like this:

```
spec:
  template:
    metadata:
      labels:
        app: productpage
        service-mesh.3scale.net/credentials: basic
        service-mesh.3scale.net/service-id: '3'
```

### Authorize an Application to Consume the API

1. In 3scale, go to Audience
2. Choose an account to authorize (you can use the default Developer account)
3. Follow the Link at the top of the page that says `N Applications` (N being the number of applications the account has)
4. Click Create Application
5. Select the target application plan and provide a name, then create

You should now have an API key that you can copy and use for authorization

### Verify the policy enforcement

Access without credentials:

`curl -v http://istio-ingressgateway-istio-system.{cluster wildercard url}/productpage`

You should see an HTTP 401 response.

Access with credentials (from the previous step):

`curl -v http://istio-ingressgateway-istio-system.{cluster wildcard url}/productpage?user_key={user key}`

You should see an HTTP 200 response.
