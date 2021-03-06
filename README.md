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
    eksctl create cluster --name appmesh-l3 --managed --region us-east-2
    ```

3. List worker node IAM role ARN

    ```
    eksctl get iamidentitymapping --cluster appmesh-l3
    ```

4. Add AppMesh policy to worker node role ARN

    ```
    aws iam attach-role-policy --policy-arn arn:aws:iam::aws:policy/AWSAppMeshFullAccess --role-name <POLICY_ARN_FROM_STEP_NOT_THE_ARN>
    ```

5. Install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

6. Install [Helm](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

    ```
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

### AppMesh integration with EKS

1. Execute `pre_upgrade_check.sh` script

    ```
    curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh

    chmod +x ./pre_upgrade_check.sh
    ./pre_upgrade_check.sh
    ```

2. Deploy AppMesh Custom Resource Definitions (CRD's)

    ```
    kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
    ```
3. Create `appmesh-system` namespace

    ```
    kubectl create namespace appmesh-system
    ```

4. Deploy AppMesh Controller

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

5. Validate AppMesh Controller pod is in 'Running' status

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
