---
title: Route53
description: 'cert-manager configuration: ACME DNS-01 challenges using Amazon AWS Route53 DNS'
---

This guide explains how to set up an `Issuer`, or `ClusterIssuer`, to use Amazon
Route53 to solve DNS01 ACME challenges. It's advised you read the [DNS01
Challenge Provider](./README.md) page first for a more general understanding of
how cert-manager handles DNS01 challenges.

> ℹ️ This guide assumes that your cluster is hosted on Amazon Web Services
> (AWS) and that you already have a hosted zone in Route53.
>
> 📖 Read
> [Deploy cert-manager on AWS Elastic Kubernetes Service (EKS) and use Let's Encrypt to sign a certificate for an HTTPS website](../../../tutorials/getting-started-aws-letsencrypt/README.md),
> a tutorial which explains how to create an EKS cluster, install cert-manager, and configuring ACME ClusterIssuer using Route53 for DNS-01 challenges.

## Set up an IAM Policy

To solve a DNS01 challenge, cert-manager needs permission to add TXT records to your Route53 zone.
It needs permission to delete the TXT records, after the challenge succeeds or fails.
It also needs permission to list the hosted zones, unless you use the `hostedZoneID` field.
To enable this, create an IAM policy with the following permissions in the AWS account of the Route53 zone:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "route53:GetChange",
      "Resource": "arn:aws:route53:::change/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets",
        "route53:ListResourceRecordSets"
      ],
      "Resource": "arn:aws:route53:::hostedzone/*"
    },
    {
      "Effect": "Allow",
      "Action": "route53:ListHostedZonesByName",
      "Resource": "*"
    }
  ]
}
```

> ℹ️ The `route53:ListHostedZonesByName` statement can be removed if you
> specify the (optional) `hostedZoneID`. You can further tighten the policy by
> limiting the hosted zone that cert-manager has access to (e.g.
> `arn:aws:route53:::hostedzone/DIKER8JEXAMPLE`).
>
> 📖 Read about [actions supported by Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/APIReference/API_Operations_Amazon_Route_53.html),
> in the [Amazon Route 53 API Reference](https://docs.aws.amazon.com/Route53/latest/APIReference/Welcome.html).
>
> 📖 Learn how [`eksctl` can automatically create the cert-manager IAM policy](https://eksctl.io/usage/iam-policies/#cert-manager-policy), if you use EKS.

## Credentials

cert-manager uses the Route53 API to create, update and then delete TXT records, solve a DNS01 challenge.
The [requests to the Route53 API must be signed](https://docs.aws.amazon.com/Route53/latest/APIReference/requests-authentication.html)
using an access key ID and a secret access key.
There are several ways for cert-manager to get the access key ID and secret access key.
These mechanisms are categorized as either "ambient" or "non-ambient".
Here's what that means:

**Ambient credentials** are credentials which are made available in the cert-manager controller Pod via:
- A mounted Kubernetes ServiceAccount token (`AWS_WEB_IDENTITY_TOKEN_FILE`), with IAM Roles for Service Accounts (IRSA).
- A Kubernetes HTTPS endpoint () for Pod Identity.
- An EC2 HTTPS endpoint (`https://169.254.169.254/latest/meta-data/iam/security-credentials`), for IMDS instance credentials.
- Environment variables (`AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`), if cert-manager has been deployed with static environment variable credentials.
- A mounted AWS configuration file (`~/.aws/credentials`), if cert-manager has been deployed with static mounted credentials.

The advantage of ambient credentials is that they are easier to set up, well
documented, and AWS provide cluster helpers to automate the configuration.
The disadvantage of ambient credentials is that they are globally available to
all ClusterIssuer and all Issuer resources, which means that in a multi-tenant
environment, any tenant who has permission to create Issuer or ClusterIssuer may
use the ambient credentials and gain the permissions granted to that account.

**Non-ambient credentials** are configured on the Issuer or ClusterIssuer.
For example:
- Access key Secret reference: `{cluster,}issuer.spec.acme.solvers.dns01.route53.{accessKeyIDSecretRef,secretAccessKeySecretRef}`
- ServiceAccount reference`{cluster,}issuer.spec.acme.solvers.dns01.route53.auth.kubernetes.serviceAccountRef`

The advantage of non-ambient credentials is that cert-manager can perform Route53 operations in a multi-tenant environment.
Each tenant can be granted permission to create and update Issuer resources in their namespace and they can provide their own AWS credentials in their namespace.

> ⚠️ By default, cert-manager will only use ambient credentials for
> `ClusterIssuer` resources, not `Issuer` resources.
>
> This is to prevent unprivileged users, who have permission to create Issuer
> resources, from issuing certificates using credentials that cert-manager
> incidentally has access to.
> ClusterIssuer resources are cluster scoped (not namespaced) and only platform
> administrators should be granted permission to create them.
>
> ⚠️ It is possible (but not recommended) to enable this authentication mechanism
> for `Issuer` resources, by setting the `--issuer-ambient-credentials` flag on
> the cert-manager controller to true.

### (Non-ambient) OIDC with dedicated Kubernetes ServiceAccount resources

In this scenario, cert-manager authenticates to Route53 using temporary credentials obtained from STS,
by exchanging a temporary Kubernetes ServiceAccount token for a temporary AWS access key.
Some key characteristics of this mechanism are:
- Uses short-lived credentials.
- Does not require key rotation.
- Does not require Kubernetes Secret resources.
- Works with any Kubernetes cluster.
- Good isolation of Issuer permissions.
- Complicated to configure.

> 📖 This is the mechanism used in the tutorial:
> [Deploy cert-manager on AWS Elastic Kubernetes Service (EKS) and use Let's Encrypt to sign a certificate for an HTTPS website](../../../tutorials/getting-started-aws-letsencrypt/README.md),
>.

#### Mechanism

1. cert-manager gets a [Kubernetes ServiceAccount token](https://kubernetes.io/docs/concepts/security/service-accounts/#authenticating-credentials) (a signed JWT), from the Kubernetes API server.
1. cert-manager sends the signed JWT to [AWS Security Token Service (STS)](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)
   invoking the [AssumeRoleWithWebIdentity Action](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html),
   also supplying the Role ARN specified in the ClusterIssuer (or Issuer).
1. STS returns a [Temporary security credential](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
   (a short lived AWS access key).
1. cert-manager uses the temporary credential to authenticate to [AWS Route53](https://docs.aws.amazon.com/Route53/latest/APIReference/API_Operations.html)
   invoking various actions to create TXT records to satisfy the ACME challenge.
   > ℹ️ It [signs the requests](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html) using the short lived access key.

#### Configuration

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    ...
    solvers:
    - dns01:
        route53:
          role: arn:aws:iam::YYYYYYYYYYYY:role/dns-manager
          auth:
            kubernetes:
              serviceAccountRef:
                name: <service-account-name> # The name of the service account created
```


### (Ambient) IAM Roles for Service Accounts (IRSA) with mounted ServiceAccount token

In this scenario, cert-manager authenticates to Route53 using temporary credentials obtained from STS,
by exchanging a temporary Kubernetes ServiceAccount token for a temporary AWS access key.
The Kubernetes ServiceAccount token is mounted into the Pod of the cert-manager controller.

In AWS this mechanism is branded as [IAM Roles for Service Accounts (IRSA)](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html).
IRSA comprises:
1. The [Amazon EKS Pod Identity Webhook](https://github.com/aws/amazon-eks-pod-identity-webhook)
   which runs in the cluster,
   watches for Pods that use a service account with the following annotation: `eks.amazonaws.com/role-arn`, and
   adds a [ServiceAccount token volume projection](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#serviceaccount-token-volume-projection) to the Pod and adds an environment variable to the Pod to allow the AWS SDK to discover and load the token.
2. A client using an [AWS SDK with default credential loader](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-minimum-sdk.html),
   which is able to discover the projected token created by the webhook, and
   automatically load it and send it to STS invoking the AssumeRoleWithWebIdentity action.

Some key characteristics of this mechanism are:
- Uses short-lived credentials.
- Does not require key rotation.
- Does not require Kubernetes Secret resources.
- Easy to configure on EKS clusters
- Poor isolation of Issuer permissions

#### Mechanism

1. Kubernetes API server regularly generates a signed JWT which is mounted into the cert-manager controller Pod,
   at file path which is stored in the environment variable: `AWS_WEB_IDENTITY_TOKEN_FILE`.
1. cert-manager reads the signed JWT from the file system.
1. cert-manager connects to [AWS Security Token Service (STS)](https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html)
   invoking the [AssumeRoleWithWebIdentity Action](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html),
   supplying the Kubernetes signed JWT and the Role ARN specified in the ClusterIssuer (or Issuer).
1. STS returns a [Temporary security credential](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_temp.html)
   (a short lived AWS access key).
1. cert-manager connects to [AWS Route53](https://docs.aws.amazon.com/Route53/latest/APIReference/API_Operations.html)
   invoking various actions to create TXT records to satisfy the ACME challenge.
   It [signs the requests](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_aws-signing.html) using the short lived access key.


> 📖 Read [Configure the AWS Security Token Service endpoint for a service account](https://docs.aws.amazon.com/eks/latest/userguide/configure-sts-endpoint.html) if you need to change the

#### Configuration

TODO

### (Ambient) Load credentials from EC2 Instance Metadata Service (IMDS)

In this scenario, cert-manager authenticates to Route53 using temporary credentials [obtained from the EC2 Instance Metadata Service (IMDS)](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html).
The temporary credentials correspond to the role specified in the [instance profile](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use_switch-role-ec2_instance-profiles.html).

| ✔️ Advantages                     | ✖️ Disadvantages                                     |
|----------------------------------|-----------------------------------------------------|
| Uses short-lived credentials     |                                                     |
| No key rotation necessary        | **Poor isolation of Issuer permissions**            |
| No Kubernetes Secrets            | **(Issuers can assume the Roles of other Issuers)** |
| **Minimal configuration needed** | **Works ONLY on EKS clusters**                      |

#### Mechanism

If you enable debug logging you will see the following series of requests to the EC2 metadata service at `Host: 169.254.169.254`:

1. `PUT /latest/api/token`:
   Get an authentication token.
1. `GET /latest/meta-data/iam/security-credentials/`:
   Discover the instance profile Role name.
1. `GET /latest/meta-data/iam/security-credentials/<ROLE_NAME>`:
   Get temporary access key for the instance profile Role.

Then you will see a series of requests to Route53:
1. `GET /2013-04-01/hostedzonesbyname?dnsname=<DNS_NAME>`: Discover the Hosted Zone for the DNS name.
1. `POST /2013-04-01/hostedzone/<HOSTED_ZONE_ID>/rrset`: Create a change set with the ACME challenge TXT record.
1. `GET /2013-04-01/change/<CHANGE_ID`: Check the status of the change set.

> ℹ️ If you **also** specify an Issuer `role`, cert-manager will additionally connect to STS and invoke the `AssumeRole` action,
> to get temporary credentials for the assumed role.
> It will then use those temporary credentials to authenticate with Route53.
>
> 📖 Read [Cross Account Access](#cross-account-access) (below) to learn about the required IAM configuration.

### Load User access key from dedicated Secret

TODO

### Load User access key from mounted Secret

TODO

## Cross Account Access

Example: Account Y manages Route53 DNS Zones. Now you want cert-manager running in Account X (or many other accounts) to be able to manage records in Route53 zones hosted in Account Y.

First, create a role with the permissions policy above (let's call the role `dns-manager`)
in Account Y, and attach a trust relationship like the one below.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::XXXXXXXXXXX:role/cert-manager"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Bear in mind, that you won't be able to define this policy until `cert-manager` role on account Y is created. If you are setting this up using a configuration language, you may want to define principal as:

```json
"Principal": {
        "AWS": "XXXXXXXXXXX"
      }
```
And restrict it, in a future step, after all the roles are created.

This allows the role `cert-manager` in Account X to assume the `dns-manager` role in Account Y to manage the Route53 DNS zones in Account Y. For more information visit the [official
documentation](https://docs.aws.amazon.com/IAM/latest/UserGuide/tutorial_cross-account-with-roles.html).

Second, create the cert-manager role in Account X; this will be used as a credentials source for the cert-manager pods running in Account X. Attach to the role the following **permissions** policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Resource": "arn:aws:iam::YYYYYYYYYYYY:role/dns-manager",
      "Action": "sts:AssumeRole"
    }
  ]
}
```

And the following trust relationship (Add AWS `Service`s as needed):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## Creating an Issuer (or `ClusterIssuer`)

Here is an example configuration for a `ClusterIssuer`:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    ...
    solvers:

    # example: cross-account zone management for example.com
    # this solver uses ambient credentials (i.e. inferred from the environment or EC2 Metadata Service)
    # to assume a role in a different account
    - selector:
        dnsZones:
          - "example.com"
      dns01:
        route53:
          region: us-east-1
          hostedZoneID: DIKER8JEXAMPLE # optional, see policy above
          role: arn:aws:iam::YYYYYYYYYYYY:role/dns-manager

    # this solver handles example.org challenges
    # and uses explicit credentials
    - selector:
        dnsZones:
          - "example.org"
      dns01:
        route53:
          region: eu-central-1
          # The AWS access key ID can be specified using the literal accessKeyID parameter
          # or retrieved from a secret using the accessKeyIDSecretRef
          # If using accessKeyID, omit the accessKeyIDSecretRef parameter and vice-versa
          accessKeyID: AKIAIOSFODNN7EXAMPLE
          accessKeyIDSecretRef:
            name: prod-route53-credentials-secret
            key: access-key-id
          secretAccessKeySecretRef:
            name: prod-route53-credentials-secret
            key: secret-access-key
          # you can also assume a role with these credentials
          role: arn:aws:iam::YYYYYYYYYYYY:role/dns-manager
```

Note that, as mentioned above, the pod is using `arn:aws:iam::XXXXXXXXXXX:role/cert-manager` as a credentials source in Account X, but the `ClusterIssuer` ultimately assumes the `arn:aws:iam::YYYYYYYYYYYY:role/dns-manager` role to actually make changes in Route53 zones located in Account Y.

## EKS IAM Role for Service Accounts (IRSA)

While [`kiam`](https://github.com/uswitch/kiam) / [`kube2iam`](https://github.com/jtblin/kube2iam) work directly with cert-manager, some special attention is needed for using the [IAM Roles for Service Accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html) feature available on EKS.

This feature uses Kubernetes `ServiceAccount` tokens to authenticate with AWS using the [API_AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html).

> **Note**: For using IRSA with cert-manager you must first enable the feature for your cluster. You can do this by
> following the [official documentation(https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

Because `ServiceAccount` tokens are used to authenticate there are two modes of operation, you can either use cert-manager's own `ServiceAccount` to authenticate or you can reference your own `ServiceAccount` within your `Issuer`/`ClusterIssuer` config. Each option is described below.

### Using the cert-manager ServiceAccount

In this configuration an IAM role is mapped to the cert-manager `ServiceAccount` allowing it to authenticate with AWS. The IAM role you map to the `ServiceAccount` will need permissions on any and all Route53 zones cert-manager will be using.

#### IAM role trust policy

The cert-manager role needs the following trust relationship attached to the role in order to use the IRSA method. Replace the following:

- `<aws-account-id>` with the AWS account ID of the EKS cluster.
- `<aws-region>` with the region where the EKS cluster is located.
- `<eks-hash>` with the hash in the EKS API URL; this will be a random 32 character hex string (example: `45DABD88EEE3A227AF0FA468BE4EF0B5`)
- `<namespace>` with the namespace where cert-manager is running.
- `<service-account-name>` with the name of the `ServiceAccount` object created by cert-manager.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>"
      },
      "Condition": {
        "StringEquals": {
          "oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

**Note:** If you're following the Cross Account example above, this trust policy is attached to the cert-manager role in Account X with ARN `arn:aws:iam::XXXXXXXXXXX:role/cert-manager`. The permissions policy is the same as above.

#### Service annotation

Annotate the `ServiceAccount` created by cert-manager:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXXXX:role/cert-manager
```

You will also need to modify the cert-manager `Deployment` with the correct file system permissions, so the `ServiceAccount` token can be read.

```yaml
spec:
  template:
    spec:
      securityContext:
        fsGroup: 1001
```

The cert-manager Helm chart provides a variable for injecting annotations into cert-manager's `ServiceAccount` and `Deployment` object like so:

```yaml
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::XXXXXXXXXXX:role/cert-manager
securityContext:
  fsGroup: 1001
```

**Note:** If you're following the Cross Account example above, modify the `ClusterIssuer` in the same way as above with the role from Account Y.

### Referencing your own ServiceAccount within Issuer/ClusterIssuer config

In this configuration you can reference your own `ServiceAccounts` within your `Issuer`/`ClusterIssuer` and cert-manager will issue itself temporary credentials using these `ServiceAccounts`. Because each issuer can reference a different `ServiceAccount` you can lock down permissions much more, with each `ServiceAccount` mapped to an IAM role that only has permission on the zones it needs for that particular issuer.


#### Creating a ServiceAccount

In order to reference a `ServiceAccount` it must first exist. Unlike normal IRSA the `eks.amazonaws.com/role-arn` annotation is not required, however you may wish to set it as a reference.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: <service-account-name>
  annotation:
    eks.amazonaws.com/role-arn: <iam-role-arn>
```

#### IAM role trust policy

For every `ServiceAccount` you want to use for AWS authentication you must first set up a trust policy. Replace the following:

- `<aws-account-id>` with the AWS account ID of the EKS cluster.
- `<aws-region>` with the region where the EKS cluster is located.
- `<eks-hash>` with the hash in the EKS API URL; this will be a random 32 character hex string (example: `45DABD88EEE3A227AF0FA468BE4EF0B5`)
- `<namespace>` with the namespace of the `ServiceAccount` object.
- `<service-account-name>` with the name of the `ServiceAccount` object.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Principal": {
        "Federated": "arn:aws:iam::<aws-account-id>:oidc-provider/oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>"
      },
      "Condition": {
        "StringEquals": {
          "oidc.eks.<aws-region>.amazonaws.com/id/<eks-hash>:sub": "system:serviceaccount:<namespace>:<service-account-name>"
        }
      }
    }
  ]
}
```

**Note:** If you're following the Cross Account example above, this trust policy is attached to the cert-manager role in Account X with ARN `arn:aws:iam::XXXXXXXXXXX:role/cert-manager`. The permissions policy is the same as above.

#### RBAC

In order to allow cert-manager to issue a token using your `ServiceAccount` you must deploy some RBAC to the cluster. Replace the following:

- `<service-account-name>` name of the `ServiceAccount` object.
- `<service-account-namespace>` namespace of the `ServiceAccount` object.
- `<cert-manager-service-account-name>` name of cert-managers `ServiceAccount` object, as created during cert-manager installation.
- `<cert-manager-namespace>` namespace that cert-manager is deployed into.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: <service-account-name>-tokenrequest
  namespace: <service-account-namespace>
rules:
  - apiGroups: ['']
    resources: ['serviceaccounts/token']
    resourceNames: ['<service-account-name>']
    verbs: ['create']
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: cert-manager-<service-account-name>-tokenrequest
  namespace: <service-account-namespace>
subjects:
  - kind: ServiceAccount
    name: <cert-manager-service-account-name>
    namespace: <cert-manager-namespace>
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: <service-account-name>-tokenrequest
```

#### Issuer/ClusterIssuer config

Once you have completed the above you should have:
- An IAM role with permissions required to on the Route53 zone.
- A Kubernetes `ServiceAccount`.
- A trust policy to allow the Kubernetes `ServiceAccount` access to your IAM role.
- RBAC to allow cert-manager to issue a token using the Kubernetes `ServiceAccount`.

You should be ready at this point to configure an Issuer to use the new `ServiceAccount`. You can see example config for this below:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example
spec:
  acme:
    ...
    solvers:
    - selector:
        dnsZones:
          - "example.com"
      dns01:
        route53:
          region: us-east-1
          role: <iam-role-arn> # This must be set so cert-manager what role to attempt to authenticate with
          auth:
            kubernetes:
              serviceAccountRef:
                name: <service-account-name> # The name of the service account created
```


### Troubleshooting

#### Why is the Issuer `region` field required?

The Issuer `region` field should not be required.
cert-manager uses [AWS SDK for Go V2](https://aws.github.io/aws-sdk-go-v2/),
which will discover the region from the environment when running on AWS EKS or EC2.

[Route53 API is a global service with a single global endpoint](https://docs.aws.amazon.com/general/latest/gr/rande.html#global-endpoints), so the region should not be necessary.
[When you use the AWS SDK to submit requests, you can leave the Region and endpoint unspecified](https://docs.aws.amazon.com/general/latest/gr/r53.html#r53_region).

This is a legacy of the original Route53 integration, which was developed at a time when cert-manager was expected to be running on non AWS managed clusters where neither the ambient `AWS_REGION` environment variable nor the EC2 instance metadata service were expected to be available.
The maintainer are aware of this inconsistency and the validation will be relaxed in a future version of cert-manager.

> 📖 Read [Route53: document use of "region" field](https://github.com/cert-manager/website/issues/56).

#### Turn on Debug Logging

If you set the cert-manager controller log level to `>=4` you will see AWS requests and responses among the log messages.

> ℹ️ Available since cert-manager >=1.16
>
> 📖 Read about [`ClientLogMode` in AWS SDK for Go V2](https://aws.github.io/aws-sdk-go-v2/docs/configuring-sdk/logging/#clientlogmode).

#### Beware of Route53 quotas

[Amazon Route 53 API requests and entities are subject to the following quotas](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/DNSLimitations.html)

#### Use `AWS_` environment variables to override defaults

If necessary you can set any of the [Environment variables supported by the AWS SDK](https://docs.aws.amazon.com/sdkref/latest/guide/settings-reference.html#EVarSettings).
You can set these using the [`extraEnv` value in the Helm chart](https://artifacthub.io/packages/helm/cert-manager/cert-manager/#extraenv-~-array).
