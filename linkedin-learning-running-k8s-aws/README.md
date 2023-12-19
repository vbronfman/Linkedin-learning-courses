# LinkedIn Learning course Running Kubernetes on AWS (EKS)

Course addresses objectives: 
- Installing of command-line tools
   aws-cli, kubectl, eksctl
to provision and manage k8s cluster on AWS.

- Use AWS IAM to create policy and user groups to control which users and service accounts can able to access AWS account.

- Spin up EKS cluster, tag VPC subnets,
 installed cert-manager and aws-load-balancer-controllerto spin up AWS Elastic Load Balancers

- Deployment of applications and make them available on the internet. Services and ingresses created to communicate with  AWS Load Balancers, so that traffic from the internet could access the applications. 


## Prepare env and create cluster.

eksctl create cluster -f cluster.yaml [ --profile lucy-il-central-1 ] 


## Install application:
kubectl apply -f deploy.yaml
kubectl apply -f service.yaml


## Expose application to internet

In AWS console ppen VPC -> subnets and add tags.  Look into Chapter-03/subnet-tags.txt

Add tags for private subnets :  pay attention to set proper cluster name
kubernetes.io/cluster/lil-eks

Add tags for public subnets: 
kubernetes.io/cluster/lil-eks


Create IAM policy bound to a kubernetes service account
with permision to create ELB ; look into ./Chapter_03/3.2/iam_policy.json

Apply policy via CLI - look into  commands.txt 
  aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy.json

Add OIDC credentials provider:
eksctl utils associate-iam-oidc-provider --cluster linkedin-learning-cluster --approve --profile lucy-il-central-1

Create k8s service account and link with policy "AWSLoadBalancerControllerIAMPolicy". In our case SA should be able create,communicate with and delete AWS ELB.
Use eksctl to create AWS iamserviceaccount:
eksctl create iamserviceaccount \
--cluster=linkedin-learning-cluster  \
    --name=aws-load-balancer-controller \
    --namespace=kube-system \
    --attach-policy-arn=arn:aws:iam::512587241100:policy/AWSLoadBalancerControllerIAMPolicy \
    --approve --profile lucy-il-central-1


## Troubleshooting  Service Account issues
eksctl is effectively runs Cloudformation 
so to check status of service account the one can look into logs of CloudFormation stack.

## Create load balancer controller
Now, we're going to create AWS Load Balancer Controller within EKS cluster
Look https://kubernetes-sigs.github.io/aws-load-balancer-controller/v2.6/

AWS Load Balancer Controller is a controller to help manage Elastic Load Balancers for a Kubernetes cluster. Once it sees ingress resources in cluster it kicks on creation of Elastic Load Balancer.
To work properly requires installing of cert-manager https://cert-manager.io/ thish is kubernets add-on that simplify creation and management of TSL certificates.
AWS Load Balancer Controller needs cert-manager to encript data in and out of the cluster EKS.
Get command to install cert-manager 3.2/commands.txt:
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.3/cert-manager.yaml --validate=false

Once done, update  load-balancer-controller.yaml wth actual values - see TODO tag in manifest

Apply manifest load-balancer-controler.yaml

kubectl get deployments.apps -n kube-system

> [!NOTE]
> Pay attention, there is no ELB in EC2 section yet. This is because there is no ingress object in the cluster.

Deploy custom resources of ingress class by applying ingress-class.yaml
It creates resource definitons for custom ingress.
Apply ingress resource by pod-info-ingress.yaml. 

It triggers creation of ALB in EC2

## CHALLENGE
Go to /Chapter_03/03_07 and try to deploy resources of quote-app. 
Test if it works



## DELETE resources

### Manually delete ELB via console. For my comprehension it effectively can be done by deleting of ingresses.

### Run eksctl delete
eksctl delete cluster --name linkedin-learning-cluster --profile <admin-profile>

Look into VPC, EC2, CloudFormation and EKS for leftovers and remove them if any.

> [!CAUTION]
> Do double check to avoid unnecessary spendings.



git subtree add --prefix ./linkedin-learning-running-k8s-aws https://github.com/vbronfman/Linkedin-learning-courses.git main --squash

