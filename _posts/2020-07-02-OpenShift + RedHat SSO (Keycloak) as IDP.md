---
layout:  post
title:  OpenShift 4.4 + RedHat SSO (Keycloak) as IdP
---

Even though it is a straightforward setup, I struggled to get the recipe right. So sharing my findings.

The official recipe is provided in this [documentation](https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.4/) or this [documentation](https://docs.openshift.com/container-platform/4.4/authentication/identity_providers/configuring-oidc-identity-provider.html) but I found it difficult to follow..

 In order to configure RedHat SSO as IdP in OpenShift, you'll need to do the following:

- Red Hat Single Sign-On Operator (at the time of writing, version is 7.4.1 provided by Red Hat)
- Create an instance of Keycloak
- Define the OAuth configuration
- Define the OAuthClient to be added to the built-in oauth provider of OpenShift  

You will also need to have CLI access to the cluster, although eveything can be done through the portal.

## Pre-requisites

- OpenShift cluster 4.4 (this recipe is probably compatible from 4.x onwards)

## Deploy a Keycloak instance

1. From the `OperatorHub`, search for, then install, Red Hat Single Sign-On Operator. You can either install in all namesapces, or in a specific namespace. In this example, we installed the operator in the `keycloak` namespace, previsously created.
2. From the operator, create a Keycloak instance.
3. From the namespace where you've deployed the Keycloak instance, get the admin credentials - you can also get this from the admin Dashboard.
`kubectl get secret -n keycloak credential-keycloak
    -o=jsonpath='{.data.ADMIN_PASSWORD}' | base64 --decode`
4. Log into the Keycloak Administrator console, using the password previously retrieved, with the username being `admin`. In this example, it's FQDN is `https://keycloak-keycloak.apps.adt.openshift.openstack216.nso.ltec.int.bell.ca/auth/`

## Configure Keycloak

For that, you can simply follow step [### 5.3.1. Configuring Red Hat Single Sign-On Credentials](https://access.redhat.com/documentation/en-us/red_hat_single_sign-on/7.4/html/red_hat_single_sign-on_for_openshift_on_openjdk/tutorials#OSE-SSO-AUTH-TUTE) 

For the client configuration element `Valid Redirect URIs`, you need to specify as follow:
`https://oauth-openshift.apps.<cluster_name>.<cluster_domain>/oauth2callback/<idp_provider_name>`
The `idp_provider_name` **must** be the same name as the OAuth name that we will configure in next section.
In my case, it looks like this: `https://oauth-openshift.apps.adt.openshift.openstack216.nso.ltec.int.bell.ca/oauth2callback/keycloak`

## Configure OpenShift OAuth

#### (Optional) Create a ConfigMap holding the certificate

If you have a self signed certificate, you will need to provide it in the configuration to avoid `# x509: certificate signed by unknown authority`
Retrieve the certificate associated with the apps domain. In my case, `apps-adt-openshift-openstack216-nso-ltec-int-bell-ca`
Then create a configmap holding the certificate:
`oc create configmap ca-config-map --from-file=ca.crt=apps-adt-openshift-openstack216-nso-ltec-int-bell-ca.pem -n openshift-config`
In my case, the configmap is called `ca-config-map`.

#### Create a secret holding the `clientSecret` as identified during the configuration of Keycloak

`oc create secret generic <secret_name> --from-literal=clientSecret=<secret> -n openshift-config`
In my case, the secret is called `openid-client-secret-vv5dn`

#### Get the issuer URL - which must be HTTPS

The easiest way to get it, is true Keycloak Administrator dashboard, under the `Realm Settings`. Click on the `OpenID Endpoint Configuration` next to `Endpoints`, and you should find the issuer URL there.
In my case, it is the following: `https://keycloak-keycloak.apps.adt.openshift.openstack216.nso.ltec.int.bell.ca/auth/realms/OpenShift`

#### Build the OAuth configuration template

By assembling the various element defined or created above, build the configuration as per as bellow example.
Make sure the **name** is the same as the one configured in previous section, refered as `idp_provider_name`
~~~
apiVersion: config.openshift.io/v1
kind: OAuth
metadata:
  name: cluster
spec:
  identityProviders:
  - mappingMethod: claim
    name: keycloak
    openID:
      ca:
        name: ca-config-map
      claims:
        email:
        - email
        name:
        - name
        preferredUsername:
        - preferred_username
        - username
      clientID: openshift
      clientSecret:
        name: openid-client-secret-vv5dn
      extraScopes: []
      issuer: https://keycloak-keycloak.apps.adt.openshift.openstack216.nso.ltec.int.bell.ca/auth/realms/OpenShift
    type: OpenID
~~~

Apply this config `kubectl apply -f ...yaml`

## Configure the OAuth Client

In order to tell OpenShift you wish to use a new authentication method, we need to register it. 
To do so, we need to apply the following config, that will reference the `clientSecret` secret previously created, as well as the redirect URL configured in the Keycloak configuration

~~~
apiVersion: oauth.openshift.io/v1
grantMethod: prompt
kind: OAuthClient
metadata:
  name: keycloak
redirectURIs:
- https://oauth-openshift.apps.adt.openshift.openstack216.nso.ltec.int.bell.ca/oauth2callback/keycloak
secret: openid-client-secret-vv5dn
~~~

Apply this config `kubectl apply -f ...yaml`

## Test the added OAuth provider

Either through CLI using `oc login` or through the portal.
