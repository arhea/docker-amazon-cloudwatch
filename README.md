# Deploy the Amazon Cloudwatch Agent as a DaemonSet

This guide walks you through deploying the Amazon Cloudwatch Agent to collect host level metrics via a Kubernetes DaemonSet. These configurations leverage the [official image published by AWS on Docker Hub](https://hub.docker.com/r/amazon/cloudwatch-agent). I have also included a [Dockerfile](./Dockerfile) in this repository if you want to build your own image. If you decide to build your own image, swap out the name of the image in the Kubernetes Manifests inside the `deploy` directory.

## Getting Started with EKSCTL

Before getting started we need an IAM role for our agent to use, so that it can publish metrics to the Cloudwatch API. In this example, we will use the [eksctl](https://eksctl.io/) utility to wire up the IAM role with a Service Account.

```bash
eksctl create iamserviceaccount \
    --cluster=basic-demo \
    --name=cloudwatch-agent \
    --namespace=amazon-cloudwatch \
    --attach-policy-arn=arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy \
    --approve
```

Next we need to deploy the agent to the cluster. This repository contains sample configurations for deployment. I strongly recommend reviewing these configurations and editing where needed.

```bash
kubectl apply -f ./deploy/
```

Metrics should start publishing to CloudWatch within a few minutes.

It is that simple!

## Getting Started with kube2iam

Before deploying you will need to create a specific role to attach to the pod.

```bash

# specify the role ARN of the IAM role assigned to worker nodes in the cluster
KUBE_WORKER_ROLE="<ROLE ARN ASSIGNED TO WORKER NODE>"

# create the IAM Role for kube2iam that we will attach the DaemonSet
aws iam create-role \
  --role-name KubernetesCloudwatchAgentRole \
  --description "Kubernetes administrator role (for AWS IAM Authenticator for Kubernetes)." \
  --assume-role-policy-document "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Sid\":\"\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"ec2.amazonaws.com\"},\"Action\":\"sts:AssumeRole\"},{\"Sid\":\"\",\"Effect\":\"Allow\",\"Principal\":{\"AWS\":\"${KUBE_WORKER_ROLE}\"},\"Action\":\"sts:AssumeRole\"}]}" \
  --output text \
  --query 'Role.Arn'

# attach the CloudWatch Agent Policy to the new Role
aws iam attach-role-policy \
    --role-name KubernetesCloudwatchAgentRole
    --policy-arn arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy

```

Next we will need to adjust the DaemonSet to reference the ARN of our new IAM role.

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
    name: cloudwatch-agent
    namespace: amazon-cloudwatch
spec:
    selector:
        matchLabels:
            name: cloudwatch-agent
    template:
        metadata:
            labels:
                name: cloudwatch-agent
            annotations:
                iam.amazonaws.com/role: role-arn
...
```

Once this file is updated, deploy the configuration.

```bash
kubectl apply -f ./deploy/
```

Metrics should start publishing to CloudWatch within a few minutes.
