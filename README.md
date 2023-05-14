# Ship EKS logs with Promtail, Loki and Amazon Managed Grafana

## Prerequisites

- EKS Cluster
- AWS CLI
- EKSCTL

## 1) Install the ALB Ingress controller in the EKS Cluster

Follow the guidance in the documentation below [1] to install the AWS ALB ingress controller in your EKS cluster (using eksctl tool [2] [3] is the easiest way, if you already have this installed)

## 2) Setup S3 buckets for loki backend storage

Create 3 x S3 buckets for Loki backend storage 

    $ aws s3api create-bucket --bucket amg-loki-datastore --create-bucket-configuration "LocationConstraint=ap-southeast-2"

## 3) Create IAM role and service account for Loki backend storage [4]

First, create the Kubernetes namespace

    $ kubectl create namespace loki

Create an IAM policy allowing access to the S3 buckets

iam-loki-s3-policy.json

    {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Action": [
                    "s3:ListObjects",
                    "s3:ListBucket",
                    "s3:PutObject",
                    "s3:GetObject",
                    "s3:DeleteObject"
                ],
                "Resource": [
                    "arn:aws:s3:::amg-loki-datastore"
                ]
            }
        ]
    }

    $ aws iam create-policy --policy-name amg-loki-s3 --policy-document file://iam-loki-s3-policy.json

Create the Kubernetes service account with eksctl [4]

    $ eksctl create iamserviceaccount --name loki-s3 --namespace loki --cluster cluster --role-name "loki-s3" \
    --attach-policy-arn arn:aws:iam::<ACCOUNT-ID>:policy/amg-loki-s3 --approve

## 4) Deploy Loki

Create the helm chart values. This will setup the backend S3 storage for loki using the service account, the ALB Ingress and the basic auth for the Loki gateway

loki_values.yaml

    loki:
    auth_enabled: false
    storage:
        type: s3
        bucketNames: 
        chunks: amg-loki-datastore
        ruler: amg-loki-datastore
        admin: amg-loki-datastore
        s3:
        endpoint: https://s3.ap-southeast-2.amazonaws.com
        region: ap-southeast-2
        #s3forcepathstyle: false
        sse_encryption: true
        insecure: false
        http_config:
            insecure_skip_verify: true
    serviceAccount:
    create: false
    name: loki-s3
    gateway:
    enabled: true
    service:
        type: NodePort
    ingress:
        enabled: true
        ingressClassName: "alb"
        annotations:
        alb.ingress.kubernetes.io/scheme  : internet-facing
        alb.ingress.kubernetes.io/ip-address-type  : ipv4
        hosts:
        - paths:
            - path: /
                pathType: Prefix
    basicAuth:
        enabled: true
        username: test
        password: test

--- 

    $ helm repo add grafana https://grafana.github.io/helm-charts
    $ helm repo update
    $ helm upgrade --install --values loki_values.yaml loki grafana/loki -n loki

## 4) Deploy Promtail

promtail_values.yaml

    config:
    # publish data to loki
    clients:
        - url: http://loki-gateway/loki/api/v1/push
        basic_auth:
            username: test
            password: test
        tenant_id: 1

---

    $ helm upgrade --install --values promtail_values.yaml promtail grafana/promtail -n loki

---

## 5) Configure the Amazon Managed Grafana - Loki data source

Find the load balancer DNS endpoint

    $ kubectl get ingress -n loki

    NAME           CLASS   HOSTS   ADDRESS                                                                    PORTS     AGE
    loki-gateway   alb     *       <ALB-DNS>.ap-southeast-2.elb.amazonaws.com   80, 443   27m

In the Grafana console, Administration --> Data sources --> Loki and configure the following settings:

URL: http://<Load Balancer DNS>/
Basic Auth: On
Basic Auth Details:
  Username: <username>
  Password: <password>

Custom HTTP Headers:
  Header: X-Scope-OrgID
  Value: 1


---

[1] Installing the AWS Load Balancer Controller add-on - https://docs.aws.amazon.com/eks/latest/userguide/aws-load-balancer-controller.html 
[2] https://github.com/weaveworks/eksctl/blob/main/README.md 
[3] https://eksctl.io/introduction/ 
[4] https://eksctl.io/usage/iamserviceaccounts/
[4] Configuring a Kubernetes service account to assume an IAM role - https://docs.aws.amazon.com/eks/latest/userguide/associate-service-account-role.html
