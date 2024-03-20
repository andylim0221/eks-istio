# EKS Istio

This serves as a playground to get to know better on using Istio in EKS Cluster.

In short, we are going to perform the following:
1. Install EKS
2. Install AWS ALB Controller in the EKS
3. Install Istio.
4. Deploy simple deployment.
5. Deploy Istio Gateway and Virtual Service.

## Prerequisite

1. [eksctl](https://eksctl.io/) installed.
2. [aws cli](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) installed and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.html).
3. [kubectl](https://kubernetes.io/docs/tasks/tools/) installed.
4. [istioctl](https://istio.io/latest/docs/ops/diagnostic-tools/istioctl/) installed.
5. [helm](https://helm.sh/docs/intro/install/) installed.

## Installation Guide

1. Install EKS cluster through `eksctl`.
   ```
   eks create cluster
   ```
   This will create a new VPC, a new EKS cluster and install all the necessary components in the EKS cluster, through CloudFormation.
2. Create OIDC provider.
    ```
    eksctl utils associate-iam-oidc-provider \
    --cluster <your-cluster-name> \
    --approve
    ```
3. Download IAM policy for the AWS Load Balancer Controller.
    ```
    curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.1/docs/install/iam_policy.json
    ```
4. Create an IAM policy called AWSLoadBalancerControllerIAMPolicy.
    ```
    aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam-policy.json
    ```
5. Create a IAM role and ServiceAccount for the AWS Load Balancer controller.
    ```
    eksctl create iamserviceaccount \
    --cluster=<cluster-name> \
    --namespace=kube-system \
    --name=aws-load-balancer-controller \
    --attach-policy-arn=arn:aws:iam::<AWS_ACCOUNT_ID>:policy/AWSLoadBalancerControllerIAMPolicy \
    --override-existing-serviceaccounts \
    --approve
    ```
6. Add EKS helm chart.
    ```
    helm repo add eks https://aws.github.io/eks-charts
    ```
7. Install AWS Load Balancer Controller helm chart.
    ```
    helm install aws-load-balancer-controller eks/aws-load-balancer-controller -n kube-system --set clusterName=<cluster-name> --set serviceAccount.create=false --set serviceAccount.name=aws-load-balancer-controller
    ```
8. Install istiod and istio-base.
    ```
    istioctl install
    ```
9. Enable Istio Ingress Gateway to be internet-facing and patch ExternalTrafficPolicy to **Local**.
    ```
    kubectl annotate svc istio-ingressgateway -n istio-system \
    service.beta.kubernetes.io/aws-load-balancer-scheme="internet-facing" \
    service.beta.kubernetes.io/aws-load-balancer-type="nlb"

    kubectl patch svc istio-ingressgateway -n istio-system \
    -p '{"spec":{"externalTrafficPolicy":"Local"}}'    
    ```
10. Get the Istio resources.
    ```
    kubectl -n istio-system get all
    NAME                                        READY   STATUS    RESTARTS   AGE
    pod/istio-ingressgateway-76ffd45584-rkgrf   1/1     Running   0          143m
    pod/istiod-85f56b5c65-vkjhc                 1/1     Running   0          143m

    NAME                           TYPE           CLUSTER-IP      EXTERNAL-IP                                                                          PORT(S)                                      AGE
    service/istio-ingressgateway   LoadBalancer   10.100.51.43    k8s-istiosys-istioing-<ID>.elb.ap-southeast-1.amazonaws.com   15021:30691/TCP,80:31301/TCP,443:31684/TCP   143m
    service/istiod                 ClusterIP      10.100.61.233   <none>                                                                               15010/TCP,15012/TCP,443/TCP,15014/TCP        143m

    NAME                                   READY   UP-TO-DATE   AVAILABLE   AGE
    deployment.apps/istio-ingressgateway   1/1     1            1           143m
    deployment.apps/istiod                 1/1     1            1           143m

    NAME                                              DESIRED   CURRENT   READY   AGE
    replicaset.apps/istio-ingressgateway-76ffd45584   1         1         1       143m
    replicaset.apps/istiod-85f56b5c65                 1         1         1       143m

    NAME                                                       REFERENCE                         TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
    horizontalpodautoscaler.autoscaling/istio-ingressgateway   Deployment/istio-ingressgateway   4%/80%    1         5         1          143m
    horizontalpodautoscaler.autoscaling/istiod                 Deployment/istiod                 0%/80%    1         5         1          143m
    ```
11. Apply manifest. This will deploy a simple NGINX deployment, service, Istio Gateway and Virtual Service.
    ```
    kubectl apply -f application.yaml
    ```
