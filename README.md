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

3. Install [kubectl](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html)

4. Install [Helm](https://docs.aws.amazon.com/eks/latest/userguide/helm.html)

    ```
    curl https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 > get_helm.sh
    chmod 700 get_helm.sh
    ./get_helm.sh
    ```

### AppMesh integration with EKS

1. Execute `pre_upgrade_check.sh` script

    ```
    curl -o pre_upgrade_check.sh https://raw.githubusercontent.com/aws/eks-charts/master/stable/appmesh-controller/upgrade/pre_upgrade_check.sh

    ./pre_upgrade_check.sh
    ```

2. Deploy AppMesh Custom Resource Definitions (CRD's)

    ```
    kubectl apply -k "https://github.com/aws/eks-charts/stable/appmesh-controller/crds?ref=master"
    ```

3. Deploy AppMesh Controller

    ```
    helm upgrade -i appmesh-controller eks/appmesh-controller \
    --namespace appmesh-system \
    --set region=$AWS_REGION \
    --set serviceAccount.create=false \
    --set serviceAccount.name=appmesh-controller
    ```
- Confirm Controller version

    ```
    kubectl get deployment appmesh-controller \
    -n appmesh-system \
    -o json  | jq -r ".spec.template.spec.containers[].image" | cut -f2 -d ':'
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

2. Create Mesh in AWS AppMesh using below manifest

```
cat <<"EOF" > /tmp/eks-scripts/yelb-mesh.yml
apiVersion: appmesh.k8s.aws/v1beta2
kind: Mesh
metadata:
  name: yelb
spec:
  namespaceSelector:
    matchLabels:
      mesh: yelb
EOF
```

3. Create AppMesh resources in AWS by deploying YAML manifests in Kubernetes

    ```
    kubectl apply -f https://github.com/aws/aws-app-mesh-examples/tree/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates
    ```

4. Application pods should be restarted with Envoy side car containers.

### Rolling update deployment for Yelb application

1. Create a new Virtual Node in AppMesh

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-yelb-appserver-v2.yaml
```

2. Create a new Deployment in EKS

```
kubectl apply -f yelb_appserver_v2_deployment.yml
```

3. Update Virtual Route in AppMesh to point with new Deployment

```
kubectl apply -f https://raw.githubusercontent.com/aws/aws-app-mesh-examples/master/walkthroughs/eks-getting-started/infrastructure/appmesh_templates/appmesh-virtual-router-appserver-v1-v2.yaml
```