# Microservice DevOps Pipeline Project

✨ **Technologies**: AWS Elastic Kubernetes Service (EKS), Jenkins, Docker, Kubernetes

### Summary: Each branch of the repository is a microservice. The Jenkins file in each branch is responsible for each deployment.
1. Make each microservice a Docker image and push it to Docker Hub
2. Deploy the images in Docker Hub to Kubernetes

Inspired by jaiswaladi246
https://youtu.be/SO3XIJCtmNs?si=jnQylVoH-SoZS6Sp

# <span style="background-color: cyan;">1) Prepare AWS</span>
### 1. Create VPC

### 2. Create Security Group
| Type | Protocol | Port range | Source |
|---------|----------|----------|----------|
| SSH | TCP | 22|0.0.0.0/0|
|HTTP|TCP|80|0.0.0.0/0|
| HTTPS | TCP | 443|0.0.0.0/0| | SMTP |TCP| 25 |0.0.0.0/0|
| SMTPS| TCP | 465|0.0.0.0/0|
| Custom TCP | TCP | 3000-10000 |0.0.0.0/0|
| Custom TCP | TCP | 27017 |0.0.0.0/0|
| Custom TCP | TCP | 30000-32767|0.0.0.0/0|

### 3. Create an IAM User to associate to the cluster
- User: eks-user
- Policy list
1. AmazonEC2FullAccess
2. AmazonEKS_CNI_Policy
3. AmazonEKSClusterPolicy
4. AmazonEKSWorkerNodePolicy
5. AWSCloudFormationFullAccess
6. IAMFullAccess
7. Custom policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        }
    ]
}
```
### 4. Install AWS CLI, kubectl, eks-ctl
1. Install AWS CLI
```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" sudo apt install unzip unzip awscliv2.zip sudo ./aws/install aws configure ``` 2. Install kubectl ```bash curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws .com/1.19.6/2021-01-05/bin/linux/amd64/kubectl chmod +x ./kubectl sudo mv ./kubectl /usr/local/bin kubectl version --short --client ``` 3. eks-ctl ```bash curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp sudo mv /tmp/eksctl /usr/local/bin eksctl version ``` ### 5. Creating an EKS cluster 1. Creating an EKS cluster with eks-ctl ```bash eksctl create cluster --name=EKS-1 --region=us-east-1 --zones=us-east-1a,us-east-1b --without-nodegroup eksctl utils associate -iam-oidc-provider --region us-east-1 --cluster EKS-1 --approve eksctl create nodegroup --cluster=EKS-1 --region=us-east-1 --name=node2 --node-type=t2.medium --nodes=1 --nodes-min=1 --nodes-max=2 --node-volume-size=20 --ssh-access --ssh-public-key=devops-pipeline-project --managed --asg-access --external-dns-access --full-ecr-access --appmesh-access --alb-ingress-access
```

2. Setting up with the created EKS cluster
```bash
aws eks --region <region-code> update-kubeconfig --name <cluster-name>
```
# <span style="background-color: cyan;">2) Kubernetes</span>
### 1. Kubernetes
RBAC - Service Account & Role & Binding & Service Account Token - GitHub File
Get Service Account Token: kubectl describe secret mysecretname -n webapps

1. Create Service Account
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: webapps
```
```bash 
kubectl apply -f sa.yaml
```
2. Create Role
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: app-role
  namespace: webapps
rules:
  - apiGroups:
        - ""
        - apps
        - autoscaling
        - batch
        - extensions
        - policy
        - rbac.authorization.k8s.io
    resources:
      - pods
      - componentstatuses
      - configmaps
      - daemonsets
      - deployments
      - events
      - endpoints
      - horizontalpodautoscalers
      - ingress
      - jobs
      - limitranges
      - namespaces
      - nodes
      - pods
      - persistentvolumes
      - persistentvolumeclaims
      - resourcequotas
      - replicasets
      - replicationcontrollers
      - serviceaccounts
      - services
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```
```bash
kubectl apply -f role.yaml
```
3. Create Bind
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: app-rolebinding
  namespace: webapps 
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: app-role 
subjects:
- namespace: webapps 
  kind: ServiceAccount
  name: jenkins 
```
```bash
kubectl apply -f bind.yaml 
```
4. Creating a Token Secret
```yaml
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: mysecretname
  annotations:
    kubernetes.io/service-account.name: jenkins
```

```bash
kubectl apply -f secret.yaml -n webapps
```
5. Get Token Secret
```bash
kubectl describe secret mysecretname -n webapps
```
Token example 
```bash 
eyJhbGciOiJSUzI1NiIsImtpZCI6IkE5MlNLeG9iVGRhcnJleGIwNW42XzVIOTY4R2JOTGdTdGxSRHcwMWtVWFkifQ.eyJpc3MiOiJrdWJlcm5ldGVzL3NlcnZpY2VhY2NvdW50 Iiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9uYW1lc3BhY2UiOiJ3ZWJhcHBzIiwia3ViZXJu ZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZWNyZXQubmFtZSI6Im15c2VjcmV0bmFtZSIsImt1YmVybmV0ZXMuaW8vc2VydmljZWFjY291bnQvc2VydmljZS1hY2NvdW50Lm5hbWU iOiJqZW5raW5zIiwia3ViZXJuZXRlcy5pby9zZXJ2aWNlYWNjb3VudC9zZXJ2aWNlLWFjY291bnQudWlkIjoiN2E3Mz dkODUtNjIwZi00NjExLTlmYmMtYTNmOGRkMmMyNDIwIiwic3ViIjoic3lzdGVtOnNlcnZpY2VhY2NvdW50OndlYmFwcHM6amVua2lucyJ9.ECg_jTdmi1uOaL0kgK3ZJ7h9pPc0XCAZVbQ5h CP0mBTHf9iJ4jj5cr8zBhTkxpGtuUZXKiMTLlHVNVKILgOXO31zdjCt8XO_p-dWAPCNxsjG42O9o6_3UDy nsArqF0QE38xwANloBFUyVKFOSV7Rn Irh6nBuapZ_hhF4y_nUnuFNtMwVf7x4_uSSOL57f7FdI6_q9CVcW6gd2xtWa5zngtP-JDSAYNV4hiKjKA
```

# <span style="background-color: cyan;">3) Jenkins settings</span> 
### 1. Install Jenkins (8080) 
```bash
sudo apt install openjdk-17-jre-headless -y sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \ https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key echo "deb [signed-by=/usr/share/keyrings /jenkins-keyring.asc]" \ https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
/etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins
```
### 2. Install Docker
```bash
sudo apt install docker.io -y
sudo chmod 666 /var/run/docker.sock
```
### 3. Jenkins Plugin
<table>
<tr>
<th colspan="5" style="background-color: lightgray;"> Jenkins Plugins</th>
</tr>
<tr>
<td>Docker</td>
<td>docker, docker pipeline</td>
</tr>
<tr>
<td>Kubernetes</td>
<td>kubernetes, kubernetes cli</td>
</tr>
<tr>
<td>Multibranch</td>
<td>multibranch scan webhook trigger</td>
</table>

### 4. Jenkins Tools
Docker name: docker / Install automatically: latest

### 5. Jenkins Credential
<table> <tr>
<th colspan="5" style="background-color: lightgray;">Jenkins Credentials</th>
</tr>
<tr>
<td>Docker Hub ID and Password</td>
<td>Username with password</td>
</tr>
<tr>
<td>GitHub Token</td>
<td>Username with password</td>
</tr>
<tr>
<td>Kubernetes Token</td> <td>Secret text</td>
</table>

### 6. Create Jenkins CI Pipeline
- Pipeline: Microservice-E-Commerse
- Type: Multibranch Pipeline
  
Branch Source: Git
Scan Multibranch Pipeline Triggers: Scan by webhook -> Microservices 깃헙 저장소 Settings - Webhooks 웹훅 추가
Payload URL: JENKINS_URL/multibranch-webhook-trigger/invoke?token=[Trigger token]
Content type: application/json

### 7. Jenkins CD Pipeline Dummy (Pipeline)
1. Deploy to Kubernetes
2. Verify Deployment
