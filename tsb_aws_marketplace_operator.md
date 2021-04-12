# **Install Tetrate Service Bridge from the AWS Container Marketplace**

Use the Tetrate Operator from the AWS Container Marketplace to deploy Tetrate Service Bridge (TSB) in your Amazon Kubernetes (EKS) cluster.


#### Note

This document is intended for users who have purchased Tetrate’s AWS Container Marketplace offering. It will not work if you have not subscribed to the Tetrate Container Marketplace offering.

Please contact Tetrate if you’re interested in an AWS Marketplace Private Offer.


## Overview of the Tetrate Operator

The Tetrate Operator is a Kubernetes Operator from Tetrate that makes it easier to install, deploy, and upgrade TSB. The AWS Container Marketplace offering for Tetrate Service Bridge installs a version of the Tetrate Operator in an EKS cluster. After that, TSB can be installed in any namespace in your EKS cluster; this document assumes that Tetrate will be installed in the `tsb` namespace.


## AWS Resources

Before you install Tetrate on AWS, it is essential that you familiarize yourself with relevant AWS services.


## Prerequisites for using the Tetrate Operator

To use the Marketplace’s Tetrate offering, make sure you meet the following requirements:



*   You have access to an EKS cluster (Kubernetes 1.16 or above) configured with [IAM roles for service accounts](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts.html).
*   You have `cluster-admin` access on the EKS cluster.


## Installation summary

This document covers the following high-level steps:



1. Creating and configuring the necessary AWS IAM roles for your Kubernetes cluster
2. Installing the Tetrate Operator Custom Resource Definitions (CRDs) for TSB into your Kubernetes cluster
3. Installing the Tetrate Operator
4. Creating a Tetrate Service Custom Resource
5. Exposing your Tetrate instance


## Create and configure the AWS IAM roles for your Kubernetes cluster

AWS IAM permissions are granted to Tetrate through the use of AWS’s IAM roles for Kubernetes Service Accounts. This feature must be enabled at a cluster level:



*   An IAM role for the Tetrate Operator (`tsb-operator` ServiceAccount in `tsb` namespace) that has these permissions `aws-marketplace:RegisterUsage`
*   _AWS permissions are **not** needed to deploy to the EKS cluster where TSB is installed._

Upon completion of this section, you should have the following  IAM role:



*   `arn:aws:iam::AWS_ACCOUNT_ID:role/eks-tsb-operator` granted to the Kubernetes Service Account `system:serviceaccount:tsb:tsb-operator-management-plane`


### IAM role for Tetrate Operator Pod

Create an IAM role for the Tetrate Operator pod (call it `eks-tsb-operator`) and configure it for use by EC2. You will replace the trust relationship later.

Grant the role the AWS managed policy `AWSMarketplaceMeteringRegisterUsage`.

Create this trust relationship on the IAM role, with these fields replaced:



*   replace `AWS_ACCOUNT_ID` with your AWS account ID
*   replace `OIDC_PROVIDER` with the “OpenID Connect provider URL” for your Kubernetes cluster (_with the <code>https://</code> removed</em>) for more details:

    [https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html)


{

  "Version": "2012-10-17",

  "Statement": [

    {

      "Effect": "Allow",

      "Principal": {

        "Federated": "arn:aws:iam::AWS_ACCOUNT_ID:oidc-provider/OIDC_PROVIDER"

      },

      "Action": "sts:AssumeRoleWithWebIdentity",

      "Condition": {

        "StringEquals": {

          "OIDC_PROVIDER:sub": "system:serviceaccount::tsb:tsb-operator-management-plane"

        }

      }

    }

  ]

}

For example:

{

  "Version": "2012-10-17",

  "Statement": [

    {

      "Effect": "Allow",

      "Principal": {

        "Federated": "arn:aws:iam::111222333444:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/AAAABBBBCCCCDDDDEEEEFFFF00001111"

      },

      "Action": "sts:AssumeRoleWithWebIdentity",

      "Condition": {

        "StringEquals": {

          "oidc.eks.us-east-1.amazonaws.com/id/AAAABBBBCCCCDDDDEEEEFFFF00001111:sub": "system:serviceaccount::tsb:tsb-operator-management-plane"

        }

      }

    }

  ]

}


## Install the Tetrate Operator Custom Resource Definitions (CRDs)

Using tetrate CLI (tctl) Generate the Kubernetes manifest for Tetrate Operator and install it into your Kubernetes cluster:

Folow instruction at [https://docs.tetrate.io/service-bridge/en-us/setup/requirements-and-download#download](https://docs.tetrate.io/service-bridge/en-us/setup/requirements-and-download#download) to download ‘tctl` utility

Generate CRD for TSB Management plane per [https://docs.tetrate.io/service-bridge/en-us/setup/management-plane-installation#operator-installation](https://docs.tetrate.io/service-bridge/en-us/setup/management-plane-installation#operator-installation)

tctl install manifest management-plane-operator \

 --registry 709825985650.dkr.ecr.us-east-1.amazonaws.com/tetrate-io > managementplaneoperator.yaml


## Install the Tetrate Operator

Update the manifest for the Tetrate Operator with your AWS Account ID:


*   You must update `AWS_ACCOUNT_ID` (in the ServiceAccount annotation) with your account ID, so the ServiceAccount can access your AWS IAM roles

    Edit generated in previous step managementplaneoperator.yaml file, Sevice account definition should have added annotation part and replace AWS_ACCOUNT_ID with yours:
    ```
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    platform.tsb.tetrate.io/application: tsb-operator-managementplane
    platform.tsb.tetrate.io/component: tsb-operator
    platform.tsb.tetrate.io/plane: management
  name: tsb-operator-management-plane
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::AWS_ACCOUNT_ID:role/eks-tsb-operator
  namespace: 'tsb'
```


# Deploy the Tetrate Operator

kubectl apply -f managementplaneoperator.yaml

Deploying the Tetrate Operator may take a little bit of time. You can monitor its status by running this command:

kubectl -n tsb get pod -owide

You’re looking for the deployment to be completely up (READY of `1/1` and STATUS of `Running`):


```
$ kubectl -n tsb get pod -owide
NAME                                             READY   STATUS    RESTARTS   AGE   IP               NODE                                              NOMINATED NODE   READINESS GATES
tsb-operator-management-plane-68c98756d5-n44d7   1/1     Running   0          71s   192.168.17.234   ip-192-168-24-207.ca-central-1.compute.internal   <none>          <none>
```


Follow Tetrate documentation on generating custom resources [https://docs.tetrate.io/service-bridge/en-us/setup/management-plane-installation#management-plane-installation](https://docs.tetrate.io/service-bridge/en-us/setup/management-plane-installation#management-plane-installation) - If everything is configured properly, the TSB Operator should see the TSB custom resources, and start creating Kubernetes Deployments, ServiceAccounts, and Secrets in the `tsb` Namespace. You can monitor this with the following:

kubectl -n tsb get all -owide


```
$ kubectl -n tsb get pods -owide
NAME                                             READY   STATUS    RESTARTS   AGE   IP               NODE                                              NOMINATED NODE   READINESS GATES
central-84dd68dc5d-x6xkm                         1/1     Running   0          59s   192.168.2.127    ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
elasticsearch-0                                  1/1     Running   0          86s   192.168.12.9     ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
envoy-788b759bb5-cmlwn                           1/1     Running   0          85s   192.168.37.215   ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
envoy-788b759bb5-jnp6z                           1/1     Running   0          85s   192.168.11.32    ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
iam-f985ffcf6-8strf                              1/1     Running   0          86s   192.168.7.143    ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
ldap-786b66cf97-w779d                            1/1     Running   0          86s   192.168.25.87    ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
mpc-7d4749bf4d-v9b8g                             1/1     Running   0          86s   192.168.17.234   ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
oap-6f78fc94c8-pdxpl                             1/1     Running   0          85s   192.168.17.56    ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
otel-collector-8d995b9bc-rztkw                   1/1     Running   0          85s   192.168.47.14    ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
postgres-85688cd778-pjg6t                        1/1     Running   0          86s   192.168.41.194   ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
tsb-5f5bd86877-zxpvm                             1/1     Running   0          86s   192.168.1.93     ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
tsb-data-reconcile-6cb65fb658-w5fn7              1/1     Running   0          86s   192.168.57.57    ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
tsb-operator-management-plane-6d9585bbfc-tlq8b   1/1     Running   0          24m   192.168.17.200   ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
web-7c4fbd767f-v6nsd                             1/1     Running   0          86s   192.168.55.16    ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
xcp-operator-central-5f85598c9-zwbl4             1/1     Running   0          85s   192.168.50.0     ip-192-168-51-71.ca-central-1.compute.internal    <none>           <none>
zipkin-5d5f8659b9-fsh44                          1/1     Running   0          85s   192.168.12.136   ip-192-168-24-207.ca-central-1.compute.internal   <none>           <none>
```

## Accessing TSB UI:

Obtain ELB Address by executing:


```
$ kubectl -n tsb get svc -l=app=envoy
NAME    TYPE           CLUSTER-IP       EXTERNAL-IP                                                                 PORT(S)                                         AGE
envoy   LoadBalancer   10.100.157.254   a72dd70af1bf64e7d86a7352a9568ea1-952780637.ca-central-1.elb.amazonaws.com   8443:32457/TCP,9443:30475/TCP,42422:32238/TCP   10m
```


Assign DNS record pointing to your ELB.

Access UI using https://&lt;DNS Name>:8443
