# Securing Pods for EKS

## Goal of the Workshop
This project will take you through the process of securing Kubernete Pods running on Amazon Elastic Kubernetes Service (EKS). We will setup a VPC with proper tagging, configure kubectl, create the required trust, roles, and IAM policies that are required to apply pod level security using [kube2iam](https://github.com/jtblin/kube2iam) for EKS. We will also apply IAM roles to a namespace to show how you can restrict pods to only using specific IAM roles. 

![Overview of Architeture](https://github.com/meyjames/Kubernetes/blob/master/podlevel.png)

```
Things I would copy down as you see them.
1) EKS Cluster Name
        You will need this when you create your nodes
2) NodeInstanceRole
        This will be available in Outputs after you launch the nodes
3) Arn of the Role you create to access S3
        You will need this for the applicatioin testing
```

# Environment Setup:
## Step 1:
Create a VPC for Kubernetes. When you create your Amazon EKS cluster, Amazon EKS tags the VPC containing the subnets you specify in the appropriate way so Kubernetes can discover them. You can read about the subnet and VPC tagging performed [here](https://docs.aws.amazon.com/eks/latest/userguide/network_reqs.html#vpc-tagging). For this example the default IP range will work.

Launch Stack: For now please launch amazon-eks-vpc2.yaml, I will update with an automated link from my bucket later. [stack](https://us-east-2.console.aws.amazon.com/cloudformation/home?region=us-east-2#/stacks/create/template?stackName=stack_name&templateURL=http://dy9fuoxcazx2i.cloudfront.net/amazon-eks-vpc.yaml)

## Step 2: Deploy EKS Cluster:
1) Check the region in the Management Console. We are deploying to US East (Ohio) us-east-2. The format for you subnets will be: stackname-Subnet#. Please select the three subnets that were created for you based on name.

## Step 3: Update AWS ClI to latest version:
Update or install the Latest [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-install.html) for your operating system.

## Step 4: Install Kubectl based on your OS:
EKS uses a command line utility called kubectl for communicating with the cluster API server. The instructions for installing for your specific operating system or package mananager are [here](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html).

## Step 5: Launch Instances:
You need the current optimized AMI for the [Amazon EKS worker nodes](https://docs.aws.amazon.com/eks/latest/userguide/eks-optimized-ami.html)
Use the CloudFormation Stack called nodesWorkshop2.yaml

## Step 6: Configure Instances to Join Cluster:
        
You must enable the worker nodes to join the cluster created. 

* Download, edit, and apply the AWS IAM Authenticator configuration map. Use the following command to download the configuration map:
```
curl -o aws-auth-cm.yaml https://amazon-eks.s3-us-west-2.amazonaws.com/cloudformation/2019-02-11/aws-auth-cm.yaml
```
* Open the file with your favorite text editor. Replace the <ARN of instance role (not instance profile)> snippet with the NodeInstanceRole value that you recorded in the previous procedure, and save the file.

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: <ARN of instance role (not instance profile)>
      username: system:node:{{EC2PrivateDNSName}}
      groups:
        - system:bootstrappers
        - system:nodes
```

## Step 6: 
Use the AWS CLI to create or update your kubeconfig for your cluster. This will combine other contexts. You can read more about this process including troubleshooting tips here.
https://docs.aws.amazon.com/eks/latest/userguide/create-kubeconfig.html

```
aws eks --region <region> update-kubeconfig --name <cluster_name>
```
```
kubectl get svc
```
When that command is successful, add a namespace for testing POD level IAM control.
```
kubectl create namespace test
```
## Step 7: Create IAM Policy for Nodes:
IAM roles

You need to create a policy and role for the Pod to use. Create a policy like the one below for EC2 and then create a role with the trust below.   
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": [
        "sts:AssumeRole"
      ],
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
```

The roles that will be assumed must have a Trust Relationship which allows them to be assumed by the kubernetes worker role. 
```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    },
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/kubernetes-worker-role"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

Step 8: Deploy kube2iam 


```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: kube2iam
  namespace: kube-system
---
apiVersion: v1
items:
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRole
    metadata:
      name: kube2iam
    rules:
      - apiGroups: [""]
        resources: ["namespaces","pods"]
        verbs: ["get","watch","list"]
  - apiVersion: rbac.authorization.k8s.io/v1
    kind: ClusterRoleBinding
    metadata:
      name: kube2iam
    subjects:
    - kind: ServiceAccount
      name: kube2iam
      namespace: kube-system
    roleRef:
      kind: ClusterRole
      name: kube2iam
      apiGroup: rbac.authorization.k8s.io
kind: List
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube2iam
  namespace: kube-system
  labels:
    app: kube2iam
spec:
  selector:
    matchLabels:
      name: kube2iam
  updateStrategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        name: kube2iam
    spec:
      serviceAccountName: kube2iam
      hostNetwork: true
      containers:
        - image: jtblin/kube2iam:latest
          imagePullPolicy: Always
          name: kube2iam
          args:
            - "--auto-discover-base-arn"
            - "--auto-discover-default-role=true"
            - "--iptables=true"
            - "--host-ip=$(HOST_IP)"
            - "--node=$(NODE_NAME)"
            - "--host-interface=eni+"
            - "--verbose"
          env:
            - name: HOST_IP
              valueFrom:
                fieldRef:
                  fieldPath: status.podIP
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
          ports:
            - containerPort: 8181
              hostPort: 8181
              name: http
          securityContext:
              privileged: true
```
Step 9: Deploy Application to test namespace
```
apiVersion: v1
kind: Pod
metadata:
  name: s3
  namespace: test
  labels:
    name: s3
  annotations:
    iam.amazonaws.com/role: <this is the arn for the role you created above>
spec:
  containers:
  - image: fstab/aws-cli
    command:
      - "/home/aws/aws/env/bin/aws"
      - "s3"
      - "ls"
      - "<your bucket name here>"
    name: s3
```
## Namespace Restrictions
Let's apply a namespace restriction based on the current role. 
By using the flag --namespace-restrictions you can enable a mode in which the roles that pods can assume is restricted by an annotation on the pod's namespace. This annotation should be in the form of a json array.

To allow the aws-cli pod specified above to run in the default namespace your namespace would look like the following.
```
apiVersion: v1
kind: Namespace
metadata:
  annotations:
    iam.amazonaws.com/allowed-roles: |
      ["put s3 arn here"]
  name: test
```
We need to test if the namespace control is working. Create a role with Administrative privileges and copy the are into the application above and redeploy. The steps to redeploy are below. 
```
kubectl delete -f <path to Application deployed above>
```
```
kubectl apply -f <path to updated application deployed above>
```
```
kubectl logs <pod> --namespace=test
```

You can read about path-based and glob-based matching for additional namespace restriction approaches on the [kube2iam site](https://github.com/jtblin/kube2iam#namespace-restrictions)

