## learning-appmesh

### Overview

1. pre-requisites
2. AppMesh integration with EKS
3. Deploy a sample application

### pre-requisites

1. Clone this repository

    ```
    git clone https://github.com/saiteja313/learning-appmesh.git
    ```

2. Create EKS cluster

- [Install ekctl](https://docs.aws.amazon.com/eks/latest/userguide/getting-started-eksctl.html)

    ```
    export AWS_REGION="us-east-2"
    export AWS_DEFAULT_REGION="us-east-2"
    export CLUSTER_NAME="appmesh-l3"
    eksctl create cluster --name appmesh-l3 --managed --region us-east-2
    ```

3. List worker node IAM role ARN

    ```
    eksctl get iamidentitymapping --cluster appmesh-l3
    ```

4. Add AppMesh policy to worker node role ARN

    ```
    aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AWSAppMeshFullAccess --role-name <ROLE_NAME_FROM_PREVIOUS_STEP>
    ```

5. Install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

6. Install [Helm](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

    ```
    export KUBBECONFIG="~/.kube/config"
    curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
    helm version --short
    ```

### AppMesh integration with EKS

1. Deploy AppMesh Custom Resource Definitions (CRD's)

    ```
    kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
    ```
2. Create `appmesh-system` namespace

    ```
    kubectl create namespace appmesh-system
    ```

3. Enable IAM OIDC provider on EKS Cluster

    ```
    eksctl utils associate-iam-oidc-provider --region=$AWS_REGION \
    --cluster=$CLUSTER_NAME \
    --approve
    ```

4. Download IAM Policy and create policy

    ```
    curl -o controller-iam-policy.json https://raw.githubusercontent.com/aws/aws-app-mesh-controller-for-k8s/master/config/iam/controller-iam-policy.json
    
    aws iam create-policy \
        --policy-name AWSAppMeshK8sControllerIAMPolicy \
        --policy-document file://controller-iam-policy.json
    ```
    
5. Create Service Account

    ```
    eksctl create iamserviceaccount --cluster $CLUSTER_NAME \
        --namespace appmesh-system \
        --name appmesh-controller \
        --attach-policy-arn arn:aws:iam::$AWS_ACCOUNT_ID:policy/AWSAppMeshK8sControllerIAMPolicy  \
        --override-existing-serviceaccounts \
        --approve
    ```

6. Deploy AppMesh Controller

    ```
    export AWS_REGION=us-east-2
    
    helm repo add eks https://aws.github.io/eks-charts

    helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=true \
    --set serviceAccount.name=appmesh-controller \
    --set sidecar.logLevel=error
    ```

7. Validate AppMesh Controller pod is in 'Running' status

    ```
    kubectl get pods -n appmesh-system
    ```


### Deploy a Sample application
1. Deploy Sample application manifests in EKS

    - [Yelb application](https://github.com/mreferre/yelb)

        ```
        kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/yelb_initial_deployment.yaml
        ```

2. Validate that application is running.

    ```
    kubectl get svc -n yelb
    ```
    - Copy Load Balancer URL

### Deploy sample application with AppMesh resources


1. Label namespace

    ```

    kubectl label namespace yelb mesh=yelb 
    
    kubectl label namespace yelb appmesh.k8s.aws/sidecarInjectorWebhook=enabled
    ```

2. Create AppMesh resources in AWS using YAML manifests

    ```
    kubectl apply -f https://raw.githubusercontent.com/saiteja313/learning-appmesh/master/yelb-mesh.yml

    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-redis.yaml

    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-db.yaml

    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-appserver.yaml

    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-ui.yaml
    ```

3. Re-deploy Yelb application

    ```
    kubectl delete -f delete_yelb_initial_deployment.yaml

    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/yelb_initial_deployment.yaml
    ```

### Rolling update deployment for Yelb application

1. Create a new Virtual Node in AppMesh

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-appserver-v2.yaml
```

2. Create a new Deployment in EKS

```
kubectl apply -f https://raw.githubusercontent.com/saiteja313/learning-appmesh/master/yelb_appserver_v2_deployment.yml
```

1. Update Virtual Route in AppMesh to send 50% traffic to new Deployment

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-virtual-router-appserver-v1-v2.yaml
```

2. Update Virtual Route in AppMesh to send 100% traffic to new Deployment

    ```
    kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-virtual-router-appserver-v2.yaml
    ```
