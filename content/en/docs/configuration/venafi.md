---
title: "Venafi"
linkTitle: "Venafi"
weight: 40
type: "docs"
---

The Venafi `Issuer` types allows you to obtain certificates from [Venafi
Cloud](https://www.venafi.com/cloud) and [Venafi Trust Protection
Platform](https://venafi.com) instances.

Create your Venafi Cloud account on this [page](https://www.venafi.com/cloud)
and get an API key from your dashboard.

You can have multiple different Venafi `Issuer` types installed within the same
cluster, including mixtures of Cloud and TPP issuer types. This allows you to be
flexible with the types of Venafi account you use.

Automated certificate renewal and management are provided for `Certificates`
using the Venafi `Issuer`.

## Creating an Issuer resource

A single Venafi `Issuer` represents a single 'zone' within the Venafi API,
therefore you must create an `Issuer` resource for each Venafi Zone you want to
obtain certificates from.

You can configure your `Issuer` resource to either issue certificates only
within a single namespace, or cluster-wide (using a `ClusterIssuer` resource).
For more information on the distinction between `Issuer` and `ClusterIssuer`
resources, read the [Namespaces](../../concepts/issuer/#namespaces) section.

### Creating a Venafi Cloud Issuer

In order to set up a Venafi Cloud `Issuer`, you must first create a Kubernetes
`Secret` resource containing your Venafi Cloud API credentials:

```bash
$ kubectl create secret generic \
       cloud-secret \
       --namespace='NAMESPACE OF YOUR ISSUER RESOURCE' \
       --from-literal=apikey='YOUR_CLOUD_API_KEY_HERE'
```

> **Note**: If you are configuring your issuer as a `ClusterIssuer` resource in
> order to serve `Certificates` across your whole cluster, you must set the
> `--namespace` parameter to `cert-manager`, which is the default `Cluster
> Resource Namespace`. The `Cluster Resource Namespace` can be configured
> through the `--cluster-resource-namespace` flag on the cert-manager controller
> component.

This API key will be used by cert-manager to interact with the Venafi Cloud
service on your behalf.

Once the API key `Secret` has been created, you can create your `Issuer` or
`ClusterIssuer` resource. If you are creating a `ClusterIssuer` resource, you
must change the `kind` field to `ClusterIssuer` and remove the
`metadata.namespace` field.

Save the below content after making your amendments to a file named
`venafi-cloud-issuer.yaml`.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: cloud-venafi-issuer
  namespace: <NAMESPACE YOU WANT TO ISSUE CERTIFICATES IN>
spec:
  venafi:
    zone: "801bdbd0-8587-11ea-b487-4d978b4efe3d" # Set this to the GUID of the Venafi policy zone you want to use
    cloud:
      apiTokenSecretRef:
        name: cloud-secret
        key: apikey
```

You can then create the Issuer using `kubectl create`.

```bash
$ kubectl create -f venafi-cloud-issuer.yaml
```

Verify the `Issuer` has been initialized correctly using `kubectl describe`.

```bash
$ kubectl get issuer cloud-venafi-issuer --namespace='NAMESPACE OF YOUR ISSUER RESOURCE' -o wide
NAME           READY   STATUS                 AGE
venafi-issuer  True    Venafi issuer started  2m
```

You are now ready to issue certificates using the newly provisioned Venafi
`Issuer`.

Read the [Issuing Certificates](../../usage/certificate/) document for
more information on how to create Certificate resources.


### Creating a Venafi Trust Protection Platform Issuer

The Venafi Trust Protection integration allows you to obtain certificates from
a properly configured Venafi TPP instance.

The setup is similar to the Venafi Cloud configuration above, however some of
the connection parameters are slightly different.

> Note: You *must* allow "User Provided CSRs" as part of your TPP policy, as
> this is the only type supported by cert-manager at this time.

In order to set up a Venafi Trust Protection Platform `Issuer`, you must first
create a Kubernetes `Secret` resource containing your Venafi TPP API
credentials.

NOTE: For TPP >= 19.2 use Token Authentication
and for older versions of TPP, use username / password authentication.

#### Refresh Token Authentication

Use refresh-token authentication if you are connecting to TPP >= 19.2 and you want to regularly and automatically rotate the access-token.

TPP supports a refresh token mechanism which allows an application to regularly and automatically rotate its API access-tokens
and the refresh-token its self.

If you supply a refresh-token and client-id in the Secret for the Venafi Issuer,
cert-manager will use that to request a short-lived access-token from the TPP authorization server,
and a new refresh-token for use next time the access-token needs to be refreshed.
cert-manager will update the Venafi Issuer Secret with that new refresh-token
so that it is available, even if cert-manager is restarted.

The advantage of this mechanism is that
it reduces the risk of an attacker using a stolen token to interact with the TPP API because:

* the access-tokens used by cert-manager to interact with the TPP resource API will be short lived.
* the refresh-tokens used by cert-manager to interact with the TPP authorization API are single-use.

The disadvantage of this mechanism is that
there is a risk that cert-manager fails to store the new refesh-token when it is rotated,
either because of a network failure during the HTTP transaction with the TPP authorization server,
or because the cert-manager process is killed after starting the refresh-token transaction
but before persisting the refresh-token to a Kubernetes Secret.

This disadvantage is not unique to TPP.
It is noted in [Section 5.2.2.3.  Refresh Token Rotation](https://tools.ietf.org/html/rfc6819#section-5.2.2.3)
of the RFC [OAuth 2.0 Threat Model and Security Considerations](https://tools.ietf.org/html/rfc6819):

 Note: This measure may cause problems in clustered environments,
 since usage of the currently valid refresh token must be ensured.  In
 such an environment, other measures might be more appropriate.

NOTE: This problem may be addressed in a future version of Venafi TPP by adding the ability to create a long-lived / re-usable refresh-token.

With those caveats in mind, here are the steps required to set up TPP and cert-manager for refresh-token based authorization:

1. [Set up token authentication](https://docs.venafi.com/Docs/19.2/TopNav/Content/SDK/WebSDK/t-sdk-Setup-OAuth.php).

   NOTE: Select "Refresh Token Enabled" and set a *short* "Token Validity (days)".

2. Create a new user with sufficient privileges to manage and revoke certificates in a particular policy zone.

   E.g. k8s-xyz-automation

3. [Create a new application integration](https://docs.venafi.com/Docs/19.2/TopNav/Content/API-ApplicationIntegration/t-APIAppIntegrations-creatingNew-Aperture.htm)

   Create an application integration with name and ID `cert-manager`.
   Set the "API Access Settings" to `Certificates: Read,Manage,Revoke`.

   "Edit Access" to the new application integration, and allow it to be used by the user you created earlier.

4. [Generate an access token](https://github.com/Venafi/vcert/blob/v4.11.0/README-CLI-PLATFORM.md#obtaining-an-authorization-token)

   ```
   vcert getcred \
     --username k8s-xyz-automation \
     --password somepassword \
     -u https://tpp.example.com/vedsdk \
     --client-id cert-manager \
     --scope "certificate:manage,revoke"
   ```

   This will print an access-token and a refresh-token to `stdout`. E.g.

   ```
   vCert: 2020/10/07 16:34:27 Getting credentials
   access_token:  I69n.............y1VjNJT3o9U0Wko19g==
   access_token_expires:  2021-01-05T15:34:30Z
   refresh_token:  Zd48............RJ1sAK6MTnebXlm7+Q==
   ```

5. Save the client-id and refresh-token to a Secret in the Kubernetes cluster

```bash
$ kubectl create secret generic \
       tpp-secret \
       --namespace=<NAMESPACE OF YOUR ISSUER RESOURCE> \
       --from-literal=client-id=cert-manager \
       --from-literal=refresh-token='YOUR_TPP_REFRESH_TOKEN'
```


#### Access Token Authentication

Use access-token authentication if you are connecting to TPP >= 19.2 and if do not want to use a refresh-token mechanism

NOTE: The refresh token mechanism mechanism may not be enabled on your TPP
and using the refresh-token mechanism introduces a point of failure (see above).


1. [Set up token authentication](https://docs.venafi.com/Docs/19.2/TopNav/Content/SDK/WebSDK/t-sdk-Setup-OAuth.php).

   NOTE: Do not select "Refresh Token Enabled" and set a *long* "Token Validity (days)".

2. Create a new user with sufficient privileges to manage and revoke certificates in a particular policy zone.

   E.g. k8s-xyz-automation

3. [Create a new application integration](https://docs.venafi.com/Docs/19.2/TopNav/Content/API-ApplicationIntegration/t-APIAppIntegrations-creatingNew-Aperture.htm)

   Create an application integration with name and ID `cert-manager`.
   Set the "API Access Settings" to `Certificates: Read,Manage,Revoke`.

   "Edit Access" to the new application integration, and allow it to be used by the user you created earlier.

4. [Generate an access token](https://github.com/Venafi/vcert/blob/v4.11.0/README-CLI-PLATFORM.md#obtaining-an-authorization-token)

   ```
   vcert getcred \
     --username k8s-xyz-automation \
     --password somepassword \
     -u https://tpp.example.com/vedsdk \
     --client-id cert-manager \
     --scope "certificate:manage,revoke"
   ```

   This will print an access-token to `stdout`. E.g.

   ```
   vCert: 2020/10/07 16:34:27 Getting credentials
   access_token:  I69n.............y1VjNJT3o9U0Wko19g==
   access_token_expires:  2021-01-05T15:34:30Z
   ```

5. Save the access-token to a Secret in the Kubernetes cluster

```bash
$ kubectl create secret generic \
       tpp-secret \
       --namespace=<NAMESPACE OF YOUR ISSUER RESOURCE> \
       --from-literal=access-token='YOUR_TPP_ACCESS_TOKEN'
```

#### Username / Password Authentication

NOTE: username / password authentication is deprecated and should only be used when connecting to TPP < 19.2.
It requires the username and password of a TPP user to be stored in the Kubernetes cluster
and it does not allow scoped access to the API.
This means that if these credentials are leaked an attacker may gain long term access to the TPP API and web UI.

```bash
$ kubectl create secret generic \
       tpp-secret \
       --namespace=<NAMESPACE OF YOUR ISSUER RESOURCE> \
       --from-literal=username='YOUR_TPP_USERNAME_HERE' \
       --from-literal=password='YOUR_TPP_PASSWORD_HERE'
```

> Note: If you are configuring your issuer as a `ClusterIssuer` resource in
> order to issue `Certificates` across your whole cluster, you must set the
> `--namespace` parameter to `cert-manager`, which is the default `Cluster
> Resource Namespace`. The `Cluster Resource Namespace` can be configured
> through the `--cluster-resource-namespace` flag on the cert-manager controller
> component.

These credentials will be used by cert-manager to interact with your Venafi TPP
instance. Username attribute must be adhere to the `<identity
provider>:<username>` format.  For example: `local:admin`.

Once the Secret containing credentials has been created, you can create your
`Issuer` or `ClusterIssuer` resource. If you are creating a `ClusterIssuer`
resource, you must change the `kind` field to `ClusterIssuer` and remove the
`metadata.namespace` field.

Save the below content after making your amendments to a file named
`venafi-tpp-issuer.yaml`.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: tpp-venafi-issuer
  namespace: <NAMESPACE YOU WANT TO ISSUE CERTIFICATES IN>
spec:
  venafi:
    zone: devops\cert-manager # Set this to the Venafi policy zone you want to use
    tpp:
      url: https://tpp.venafi.example/vedsdk # Change this to the URL of your TPP instance
      caBundle: <base64 encoded string of caBundle PEM file, or empty to use system root CAs>
      credentialsRef:
        name: tpp-secret
-```

You can then create the `Issuer` using `kubectl create -f`.
```bash
$ kubectl create -f venafi-tpp-issuer.yaml
```

Verify the `Issuer` has been initialized correctly using `kubectl describe`.
```bash
$ kubectl describe issuer tpp-venafi-issuer --namespace='NAMESPACE OF YOUR ISSUER RESOURCE'
```

You are now ready to issue certificates using the newly provisioned Venafi
`Issuer`.

Read the [Issuing Certificates](../../usage/certificate/) document for
more information on how to create Certificate resources.

# Issuer specific annotations

## Custom Fields
Starting `v0.14` you can pass custom fields to Venafi (TPP version `v19.2` and higher) using the `venafi.cert-manager.io/custom-fields` annotation on Certificate resources.
The value is a JSON encoded array of custom field objects having a `name` and `value` key.
For example:
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com-certificate
  annotations:
    venafi.cert-manager.io/custom-fields: |-
      [
        {"name": "field-name", "value": "vield value"},
        {"name": "field-name-2", "value": "vield value 2"}
      ]
...
```
