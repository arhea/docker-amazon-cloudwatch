# Deploy the Amazon Cloudwatch Agent as a DaemonSet

This guide walks you through deploying the Amazon Cloudwatch Agent to collect host level metrics via a Kubernetes DaemonSet. These configurations leverage the [official image published by AWS on Docker Hub](https://hub.docker.com/r/amazon/cloudwatch-agent). I have also included a [Dockerfile](./Dockerfile) in this repository if you want to build your own image. If you decide to build your own image, swap out the name of the image in the Kubernetes Manifests inside the `deploy` directory.

## Getting Started

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
