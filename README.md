# k8s_cheatsheet
Kubernetes cheatsheet

## Kubernetes Cluster namespace and info
```bash
bucket_name=kops-state-store-qa
export KOPS_CLUSTER_NAME=k8s-cluster.example.com
export KOPS_STATE_STORE=s3://${bucket_name}
```

## Kubernetes Cluster state store location
```bash
aws s3api create-bucket --bucket ${bucket_name} --region cn-north-1 --create-bucket-configuration LocationConstraint=cn-north-1

aws s3api put-bucket-versioning \
    --bucket ${bucket_name} \
    --versioning-configuration \
    Status=Enabled
```

## Kubernetes public key for nodes SSH
```bash
kops create secret sshpublickey admin -i k8s.pub.pem --name ${KOPS_CLUSTER_NAME} --state ${KOPS_STATE_STORE}
```

## Create AWS Kubernetes cluster
`NOTE: dns-zone's domain 'example.com' must exist in Route53`
```bash
kops create cluster --name=${KOPS_CLUSTER_NAME} \
  --state=s3://${bucket_name} --zones=us-east-2a \
  --node-count=1 --node-size=t2.small --master-size=t2.micro \
  --dns-zone=example.com \
  --ssh-public-key k8s.pub.pem
```

## Validate cluster
```bash
kops validate cluster --state=${KOPS_STATE_STORE}
```

## Edit AWS cluster
```bash
kops edit cluster --name ${KOPS_CLUSTER_NAME}

kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```

## Edit AWS cluster instance nodes
If node instance type is edited, below commands will apply to new instances only.
```bash
kops edit instancegroup nodes

kops update cluster --name ${KOPS_CLUSTER_NAME} --yes
```

If want to make changes to existing nodes, then continue to execute the following:
```bash
kops rolling-update cluster
kops rolling-update cluster --yes
kubectl rolling-update cluster -f cluster-v2.yml
```

## Install Kubernetes Dashboard in newly created cluster
Dashboard URL: Domain of the master node
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/master/src/deploy/recommended/kubernetes-dashboard.yaml
```

## Create a new deployment from file
```bash
kubectl create -f FILE.yml
```

## Update existing deployment from file
```bash
kubectl apply -f FILE.yml
```

## Update existing deployment with command
```bash
kubectl set image deployment/nginx-deployment nginx=nginx:1.9.1 --record
# OR
kubectl edit deployment/nginx-deployment --record
```

## View rollout status
```bash
kubectl rollout status deployment/nginx-deployment
```

## Rollback deployment to previous stable revision
```bash
# View previous deployment revisions
kubectl rollout history deployment/nginx-deployment
kubectl rollout history deployment/nginx-deployment --revision=2

# Rollback to previous revision
kubectl rollout undo deployment/nginx-deployment

# Rollback to specific revision
kubectl rollout undo deployment/nginx-deployment --to-revision=2
```

## Rolling update deployment
```bash
kubectl rolling-update DEPLOYMENT_NAME --update-period=10s -f DEPLOY.yml
```

## Get cluster info
```bash
kubectl cluster-info
```

## Get kubernetes cluster secret

```bash
kops get secrets kube --type secret -oplaintext
```

## Get kubernetes cluster token
Also use this secret to login dashboard

```bash
kops get secrets admin --type secret -oplaintext
```

## Create Nexus registry pull secret
Used as authentication for Kubernetes to pull images from Nexus.

Generate base64 authentication token first.
```bash
cat ~/.docker/config.json | base64
```

### registry.yml
```yaml
apiVersion: v1
kind: Secret
metadata:
 name: registrypullsecret
data:
 .dockerconfigjson: <base-64-encoded-json-here>
type: kubernetes.io/dockerconfigjson
```

### Create secret
```bash
kubectl create -f registry.yaml && kubectl get secrets
```

### Used the secret from above in deployment file
```bash
# Add below code into deployment yml ######
imagePullSecrets:
 — name: registrypullsecret
```

## Create ConfigMap from env file
Used as environment variables for containers
```bash
# Create
kubectl create configmap YOUR_CONFIG_MAP_NAME --from-env-file=YOUR_ENV_FILE

# Get all configmaps
kubectl get configmap

# See configmap details
kubectl describe configmap YOUR_CONFIG_MAP_NAME
```

## Switch kubectl to use between different clusters
```bash
kubectl config get-contexts
kubectl config use-context CONTEXT_NAME
```

## SSH into pods/EC2 instances
```bash
ssh -i "k8s.pem" admin@ec2-18-191-202-67.us-east-2.compute.amazonaws.com
```

## Describe Kubernetes Service
`NOTE: External IP is used to access application via Load Balancer`
```bash
kubectl describe service SERVICE_NAME
```

## Logs - all pods labels with app=api
```bash
kubectl logs -lapp=api --all-containers=true
```

## Restart deployment pods
```bash
kubectl replace --force -f <yml_file_describing_pod>
```

## Grep Docker container logs
```bash
docker logs CONTAINER_ID 2>&1 | grep -i -A 10 -B 10 "Critical"
```
