# cb-eks - EKS infrastructure for Crypto Birds

Table of Contents:
- [cb-eks - EKS infrastructure for Crypto Birds](#cb-eks---eks-infrastructure-for-crypto-birds)
  - [Authentication to all resources](#authentication-to-all-resources)
    - [To authenticate in AWS](#to-authenticate-in-aws)
    - [To authenticate in EKS](#to-authenticate-in-eks)
    - [To authenticate in ECR](#to-authenticate-in-ecr)
    - [Change the context to a namespace](#change-the-context-to-a-namespace)
  - [Deploy an EKS cluster from scratch](#deploy-an-eks-cluster-from-scratch)
  - [Configure the cluster](#configure-the-cluster)
    - [Create an nginx ingress controller with an NLB](#create-an-nginx-ingress-controller-with-an-nlb)
      - [Usage](#usage)
    - [Configuring External DNS](#configuring-external-dns)
      - [References](#references)
    - [Configure TLS](#configure-tls)
      - [Install cert-manager](#install-cert-manager)
      - [Configure ACME](#configure-acme)
      - [Usage](#usage-1)
    - [Environments](#environments)
  - [Configure MongoDB Atlas](#configure-mongodb-atlas)
    - [(DO NOT WORK YET) IAM access to MongoDB](#do-not-work-yet-iam-access-to-mongodb)
  - [CryptoBirds workloads deployments](#cryptobirds-workloads-deployments)
    - [Create the ECR repositories](#create-the-ecr-repositories)
    - [Prepare IAM authentication for GitHub](#prepare-iam-authentication-for-github)
    - [Namespaces in Kubernetes](#namespaces-in-kubernetes)
    - [Crypto Birds Platform](#crypto-birds-platform)
      - [cb-platform-mysql](#cb-platform-mysql)
      - [cb-platfrom (all components)](#cb-platfrom-all-components)
    - [Secrets](#secrets)
      - [cb-bb - GitHub](#cb-bb---github)
      - [cb-bb - Twitter](#cb-bb---twitter)
      - [cb-bb - Launch certificator](#cb-bb---launch-certificator)
      - [cb-bb-python - BB Pro](#cb-bb-python---bb-pro)
  - [Automated build and deployment](#automated-build-and-deployment)
    - [The process should be:](#the-process-should-be)
  - [Monitoring and Operations](#monitoring-and-operations)
    - [Connect to the database using port forwarding](#connect-to-the-database-using-port-forwarding)
    - [Operations with TLS certificates](#operations-with-tls-certificates)
    - [Example: Observing a cronjob using kubectl](#example-observing-a-cronjob-using-kubectl)
    - [List of things we want to see when observing cronjobs](#list-of-things-we-want-to-see-when-observing-cronjobs)
    - [Running a cronjob manually](#running-a-cronjob-manually)
    - [Monitoring cronjobs using Kubernetes events](#monitoring-cronjobs-using-kubernetes-events)
    - [Installing the Kubernetes Metrics Server](#installing-the-kubernetes-metrics-server)
    - [Monitoring with Prometheus and Graphana](#monitoring-with-prometheus-and-graphana)
      - [Exporters:](#exporters)
    - [Alerting set-up and protocol](#alerting-set-up-and-protocol)
    - [Backups and restore](#backups-and-restore)
    - [Autoscaling](#autoscaling)


## Authentication to all resources

To operate the cloud resources we provision, like infrastruture, kubernetes or
applications, we need to authenticate our client tools to the resources APIs.

In this section, we summarise the authentication process for each of the clients
we use: `awscli`, `kubectl` and `docker`.

### To authenticate in AWS

1. Go to https://cb-sso.awsapps.com/start#/ and login with your credentials
2. Select the account you want to access and click on "Programatic access"
3. Either export your the AWS keys or add a profile to the ~/.aws/credentials
file

### To authenticate in EKS

To be able to interact with the Kubernetes API, normally using `kubectl`, you
need to update your `kubeconfig` file. This is made with the AWS CLI as follow:

```
CLUSTER_NAME=`aws eks list-clusters | jq -r '.clusters[0]'`
aws eks update-kubeconfig --name ${CLUSTER_NAME} [--region <region_name>]
```

### To authenticate in ECR

If you need to access (push or pull) docker images from the account's registry
(ECR), you need to authenticate your docker client as well. This is also made
with an AWS CLI command:

```
ACCOUNTID=<aws-account-id>
REGION=<aws-region>
aws ecr get-login-password \
    --region ${REGION} \
| docker login \
    --username AWS \
    --password-stdin ${ACCOUNTID}.dkr.ecr.${REGION}.amazonaws.com
```

An example of the command above could be:

```
aws ecr get-login-password \
    --region eu-west-1 \
| docker login \
    --username AWS \
    --password-stdin 647183375757.dkr.ecr.eu-west-1.amazonaws.com
```

### Change the context to a namespace

```
kubectl config set-context --current --namespace=<desired-namespace>
```

## Deploy an EKS cluster from scratch

We do the EKS installation following this documentation: https://docs.aws.amazon.com/eks/latest/userguide/create-cluster.html.

Even though it is recommended to follow the official documentation, here I pasted the most relevant commands for reference:

- Create the cluster

```
CLUSTER_NAME=<cluster-name>
eksctl create cluster \
 --name ${CLUSTER_NAME} \
 --version 1.21 \
 --with-oidc \
 --without-nodegroup
```

- Create a key

```
KEY_NAME=<key-name>
aws ec2 create-key-pair --region eu-west-1 --key-name ${KEY_NAME}
```

- Create the node group

```
NODEGROUP_NAME=<node-group-name>
eksctl create nodegroup \
  --cluster ${CLUSTER_NAME} \
  --region eu-west-1 \
  --name ${NODEGROUP_NAME} \
  --node-type t2.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 4 \
  --enable-ssm \
  --node-private-networking \
  --managed
```

## Configure the cluster

### Create an nginx ingress controller with an NLB

This option for ingress will deploy an nginx contoller in the cluster and an NLB in AWS.

Based on: https://www.appvia.io/blog/expose-kubernetes-service-eks-dns-tls

```
kubectl config set-context --current --namespace=ingress-nginx
```
With Helm:
https://github.com/kubernetes/ingress-nginx/tree/main/charts/ingress-nginx
```
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
```
helm install -f ./eks-manifests/ingress-nginx-values.yaml ingress-nginx ingress-nginx/ingress-nginx
```

#### Usage

See: https://kubernetes.github.io/ingress-nginx/user-guide/nginx-configuration/annotations to undesrtand how to configure your Ingress

You can see an example in the application Ingresses

### Configuring External DNS

To be able to use Route53 as the DNS zone for our cluster.

- Create an IAM policy with the following:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "route53:ChangeResourceRecordSets"
      ],
      "Resource": [
        "arn:aws:route53:::hostedzone/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "route53:ListHostedZones",
        "route53:ListResourceRecordSets"
      ],
      "Resource": [
        "*"
      ]
    }
  ]
}
```

The policy above is in a file in this repository, so it can be created with the following command:

```
aws iam create-policy --policy-name cb-eks-external-dns-policy --policy-document file://aws-resources/cb-eks-external-dns-policy.json
```

In this case it was named `cb-eks-external-dns-policy`

- With `eksctl` create an `iamserviceaccount`:

(Values are examples used in dev at some point)
```
SERVICE_ACCOUNT_NAME=cb-external-dns
NAMESPACE=kube-system
CLUSTER_NAME=cb-hector-eks-test-2
IAM_POLICY_ARN=arn:aws:iam::647183375757:policy/cb-eks-external-dns-policy
eksctl create iamserviceaccount \
    --name ${SERVICE_ACCOUNT_NAME} \
    --namespace ${NAMESPACE} \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn ${IAM_POLICY_ARN} \
    --approve \
    --override-existing-serviceaccounts
```

```
HOSTED_ZONE_ID=Z04485691H4TEXAXGOP2Y
HOSTED_ZONE_NAME=eks-dev.cryptobirds.com
NAMESPACE=kube-system
helm install external-dns \
  --set provider=aws \
  --set aws.zoneType=public \
  --set txtOwnerId=${HOSTED_ZONE_ID} \
  --set domainFilters\[0\]=${HOSTED_ZONE_NAME} \
  --set serviceAccount.name=${SERVICE_ACCOUNT_NAME} \
  --set serviceAccount.create=false \
  --set policy=sync \
  -n ${NAMESPACE} \
  bitnami/external-dns
```

#### References

- https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html
- https://github.com/kubernetes-sigs/external-dns/blob/master/docs/tutorials/aws.md
- https://github.com/bitnami/charts/tree/master/bitnami/external-dns

### Configure TLS

#### Install cert-manager 
https://cert-manager.io/docs/installation/kubernetes/

```
kubectl create namespace cert-manager
helm repo add jetstack https://charts.jetstack.io
helm repo update
```

```
helm install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.5.4 \
  --set installCRDs=true
```

#### Configure ACME 
https://cert-manager.io/docs/configuration/acme/

- Create a new IAM policy

```
aws iam create-policy --policy-name cb-eks-cert-manager-policy-dns01 --policy-document file://aws-resources/cb-eks-cert-manager-policy-dns01.json
```

- Override the service account created by cert-manager with the EKS IAM role

```
SERVICE_ACCOUNT_NAME=cert-manager
NAMESPACE=cert-manager
CLUSTER_NAME=cb-hector-eks-test-2
IAM_POLICY_ARN=arn:aws:iam::359744453235:policy/cb-eks-cert-manager-policy-dns01
eksctl create iamserviceaccount \
    --name ${SERVICE_ACCOUNT_NAME} \
    --namespace ${NAMESPACE} \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn ${IAM_POLICY_ARN} \
    --approve \
    --override-existing-serviceaccounts
```

- Get the ARN of the created role. You can find the role with:

```
aws iam list-roles
```

And get the ARN of the role that contains `cert-manager` in the conditions

- Edit the `serviceaccount` with as follow:

```
kubectl edit serviceaccount ${SERVICE_ACCOUNT_NAME}
```

And add the following annotation (change the ARN accordingly):

```
eks.amazonaws.com/role-arn: arn:aws:iam::359744453235:role/eksctl-cb-eks-prod-01-addon-iamserviceaccoun-Role1-1DT0AYBFGV5KK
```

- Edit the `cert-manager` deployment to include `fsGroup` as follow:

See: https://cert-manager.io/docs/configuration/acme/dns01/route53/

```
kubectl edit deployment cert-manager -n cert-manager
```

```
spec:
  template:
    spec:
      securityContext:
        fsGroup: 1001
```

Note: Remove the `nonRoot` parameter from the `securityContext`

- Deploy the issuers (HTTP01 and DNS01)

```
ENVIRONMENT=<dev or prod>
CHART=cb-cert-manager
helm install cb-cert-manager-issuers -f ./helm/$CHART/$ENVIRONMENT-values.yaml ./helm/$CHART
```

#### Usage
https://cert-manager.io/docs/usage/ingress/

### Environments

## Configure MongoDB Atlas

TODO (part of new BB)

### (DO NOT WORK YET) IAM access to MongoDB

- First, create a service account associated to an IAM role using `eksctl`:

```
NAMESPACE=cb
CLUSTER_NAME=cb-hector-eks-test-2
SA_NAME=cb-mongodb-atlas-dev
ROLE_NAME=cb-mongodb-atlas-dev-admin-role
POLICY_ARN=arn:aws:iam::aws:policy/AmazonWorkLinkReadOnly
eksctl create iamserviceaccount \
    --name ${SA_NAME} \
    --namespace ${NAMESPACE} \
    --cluster ${CLUSTER_NAME} \
    --approve \
    --attach-policy-arn ${POLICY_ARN} \
    --role-name ${ROLE_NAME} \
    --override-existing-serviceaccounts
```

- To test in an interactive pod:

```
kube run -it --attach ubuntu --image=ubuntu:20.04 --serviceaccount ${SA_NAME}

# Then, inside the container:
apt update && apt install -y wget
wget https://downloads.mongodb.com/compass/mongodb-mongosh_1.0.4_amd64.deb
apt install -y ./mongodb-mongosh_1.0.4_amd64.deb
```


## CryptoBirds workloads deployments

### Create the ECR repositories

ECR repositories are created in each AWS account, hence, in each environment.

Repositories names:
* admin
* api
* birdbrain
* birdbrain-python
* certificator
* corporate
* platform

To create the repositories:

```
for r in admin api birdbrain birdbrain-python certificator corporate platform; do
  aws ecr create-repository --repository-name $r
done
```

Once the registries have been created, the user has to authenticate in ECR (see the authentication section above). This is only needed to build and push manually.
Otherwise, build, push and even deployment are configured in GitHub actions.

```
ACCOUNTID=<account-id>
REPOSITORY=<repository>
docker build -t ${ACCOUNTID}.dkr.ecr.eu-west-1.amazonaws.com/${REPOSITORY} .
docker push ${ACCOUNTID}.dkr.ecr.eu-west-1.amazonaws.com/${REPOSITORY}
```
### Prepare IAM authentication for GitHub

- Create IAM policy. At the moment this policy allows push to all repositoires.
- The policy document is `aws-resources/cb-ecr-push-github-policy.json`

```
aws iam create-policy \
    --policy-name cb-ecr-push-github-policy \
    --policy-document file://aws-resources/cb-ecr-push-github-policy.json
```

- Create another policy. At the moment this policy allows list and describe Clusters.
- The policy document is `aws-resources/cb-eks-deploy-github-policy.json`

```
aws iam create-policy \
    --policy-name cb-eks-deploy-github-policy \
    --policy-document file://aws-resources/cb-eks-deploy-github-policy.json
```

- Create the IAM user as follow:

```
aws iam create-user --user-name cb-github-user
```

- Attach the policies to the user

```
aws iam attach-user-policy --user-name cb-github-user \
  --policy-arn arn:aws:iam::359744453235:policy/cb-ecr-push-github-policy


aws iam attach-user-policy --user-name cb-github-user \
  --policy-arn arn:aws:iam::359744453235:policy/cb-eks-deploy-github-policy
```

- Verify the policies have been attached:

```
aws iam list-attached-user-policies --user-name cb-github-user
```

- Copy the IAM keys and install them in GitHub

```
aws iam create-access-key --user-name cb-github-user
```

- Then, we need to map the IAM user with an authorized group inside EKS. To do that, we have following this document: https://docs.aws.amazon.com/eks/latest/userguide/add-user-role.html
- In summary, we edit a ConfigMap as follow:

```
kubectl edit -n kube-system configmap/aws-auth
```

- We need to add the `mapUsers` object. See the example below:
```
# Please edit the object below. Lines beginning with a '#' will be ignored,
# and an empty file will abort the edit. If an error occurs while saving this file will be
# reopened with the relevant failures.
#
apiVersion: v1
data:
  mapRoles: |
    - groups:
      - system:bootstrappers
      - system:nodes
      rolearn: arn:aws:iam::647183375757:role/eksctl-cb-hector-eks-test-2-nodeg-NodeInstanceRole-1SOYAF725VTHD
      username: system:node:{{EC2PrivateDNSName}}
  mapUsers: |
    - userarn: arn:aws:iam::647183375757:user/cb-github-user
      username: cb-github-user
      groups:
        -  system:masters
kind: ConfigMap
metadata:
  creationTimestamp: "2021-06-22T17:38:25Z"
  name: aws-auth
  namespace: kube-system
  resourceVersion: "37661732"
  uid: 30505643-013c-4794-89ce-579a25eb26eb
```

### Namespaces in Kubernetes

Create a namespace named `cb`.

```
kubectl create namespace cb
```
### Crypto Birds Platform

#### cb-platform-mysql

To provision the database:
- TODO: Create users, after we are in production, as in not urgent: https://cryptobirds.atlassian.net/jira/software/c/projects/CBP/boards/1/backlog?atlOrigin=eyJpIjoiNDQ1M2QzMTdjZGRhNGJiMTliY2VkMDFhMWUzMmIwYmEiLCJwIjoiaiJ9
- Create schema
```
CREATE SCHEMA `cryptobirds` ;
```
- Load data
Export data from the actual RDS database and import it from the client.
Related: Connect to the database using port forwarding

#### cb-platfrom (all components)


- Installation
```
ENVIRONMENT=<dev or prod>
CHART=cb-platform
helm install $CHART -n cb -f helm/${CHART}/${ENVIRONMENT}-values.yaml ./helm/${CHART}
```

- Upgrade

First, export some secrets:

```
export REDIS_PASSWORD=$(kubectl get secret --namespace "cb" cb-platform-redis -o jsonpath="{.data.redis-password}" | base64 --decode)

export MYSQL_ROOT_PASSWORD=$(kubectl get secret --namespace "cb" cb-platform-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
```

Then, run the upgrade:
```
ENVIRONMENT=dev
CHART=cb-platform
helm upgrade $CHART -n cb \
--set redis.auth.password=$REDIS_PASSWORD \
--set mysql.auth.rootPassword=$MYSQL_ROOT_PASSWORD \
-f helm/${CHART}/${ENVIRONMENT}-values.yaml \
./helm/${CHART}
```

### Secrets
#### cb-bb - GitHub
- Create the secret with the GitHub credentials

```
SECRET_NAME=cb-github-credentials-dev
kubectl create secret generic ${SECRET_NAME} --from-literal=cb-github-consumer-key="${GITHUB_TOKEN}" 
```
#### cb-bb - Twitter

- Create the secret with the Twitter credentials

```
SECRET_NAME=cb-twitter-credentials-dev
kubectl create secret generic ${SECRET_NAME} --from-literal=cb-twitter-consumer-key="${CONSUMER_KEY}" --from-literal=cb-twitter-consumer-secret="${CONSUMER_SECRET}" --from-literal=cb-twitter-bearer-token="${BEARER_TOKEN}"
```
#### cb-bb - Scores

- Create the secrets
```
API_AUTH_EMAIL="noreply-api-user@cryptobirds.com"
API_AUTH_PASSWORD="**"
kubectl create secret generic cb-api-user \
--from-literal=api-auth-email=${API_AUTH_EMAIL} \
--from-literal=api-auth-password=${API_AUTH_PASSWORD}
```
#### cb-bb - Launch certificator

- Create the secrets
```
WEB3J_CLIENT_ADDRESS="https://rinkeby.infura.io/v3/dabe0addb2b24cc8a798c65a574ccfb0"
ETH_PRIVATE_KEY="**"
kubectl create secret generic cb-certificator-secrets \
--from-literal=web3j-client-address=${WEB3J_CLIENT_ADDRESS} \
--from-literal=eth-private-key=${ETH_PRIVATE_KEY}
```
- To get/decode a secret: https://kubernetes.io/docs/tasks/configmap-secret/managing-secret-using-kubectl/#decoding-secret
...or install [`view-secret`](https://github.com/elsesiy/kubectl-view-secret) plugin
#### cb-bb-python - BB Pro
```
kubectl create secret generic cb-platform-mongo \
--from-literal=mongodb-user=${MONGODB_USER} \
--from-literal=mongodb-password=${MONGODB_PASSWORD} \
--from-literal=mongodb-cluster=${MONGODB_CLUSTER} \
--from-literal=mongodb-db=${MONGODB_DB}
```
#### cb-platform-rest-api - Rest API
```
kubectl create secret generic cb-platform-external \
--from-literal=bscscan-api-key=${BSCSCAN_API}
```
For Corporate and API:
```
kubectl create secret generic cb-platform-mailchimp \
--from-literal=api-key=${MAILCHIMP_API}
```
```
kubectl create secret generic cb-platform-smtp \
--from-literal=smtp-user=${SMTP_USER} \
--from-literal=smtp-password=${SMTP_PASS} \
--from-literal=email-secret=${EMAIL_SECRET}
```

## Automated build and deployment

### The process should be:

- (For dev team) Run the tests
- (done) Build a docker image
- (done) Tag the docker image with the commit hash
- (done) Push it to ECR (central ECR ?)
- (done) Deploy to dev cluster (manifests? helm? patch?)
- Deploy to prod cluster

## Monitoring and Operations

### Connect to the database using port forwarding

- First get the password from the secrets

```
ROOT_PASSWORD=$(kubectl get secret --namespace cb cb-platform-mysql -o jsonpath="{.data.mysql-root-password}" | base64 --decode)
```

- Create a port forwarding in a different session

```
kubectl port-forward service/cb-platform-mysql 3306:3306
```

- Use your favorite client to connect. For example:

```
mysql -uroot -p -h 127.0.0.1
```

### Operations with TLS certificates

- Install the `cmctl` tool from: https://cert-manager.io/docs/usage/cmctl/#installation



### Example: Observing a cronjob using kubectl

In this section, we describe how to observe the status of a cronjob previously created.

Firstly, I login into AWS and EKS as described at the top of this document.
Then, I change to the namespace where we are deploying our workloads and get
all cronjobs:

```
# Change to the namespace with name cb
kubectl config set-context --current --namespace=cb

# ... output ommited

# Get all cronjobs in the current namespace
kubectl get cj

NAME              SCHEDULE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
twitter-job   @daily     False     8        21h             4d5h
```

Alternatively, if we want to look for all cronjobs in the cluster, we can pass
the `-A` arguments to tell `kubectl` to look into all namespaces:

```
kubectl get cj -A

NAMESPACE   NAME              SCHEDULE      SUSPEND   ACTIVE   LAST SCHEDULE   AGE
cb          twitter-job   @daily        False     8        21h             4d5h
bb      fernandojob       */5 * * * *   False     0        2m31s           40d
bb      4chan-job     @daily        False     3        21h             41d
```

Let's say that I don't know what is going on the cronjob named `twitter-job`,
so I want to explore how it is configured and how it is behaving.

From the command above, I can see that the cronjob in question is scheduled to run
every day. In addition, something is alarming me because I see there are 8 active
jobs!! Is this really desired? I smell something wrong here ... let's explore deeper.

A `crojob` object is a kubernetes resource that creates a new `job` every time it is triggered.
A `job` deploys `pods` that are expected to carry out a task and finish at some point.
To explore the whole workload, let's find more information about the `cronjob`,
then we will look at the `jobs` and finally we will troubleshoot the `pods`.

To get more information about an object, we have the following commands:

```
# Describe the object, in this case the cronjob, in plain text
kubectl describe cronjob twitter-job

# Desctibe the object in yaml format. Useful when you want to see the same
format as the manifest
kubectl get cronjob twitter-job -o yaml
```

As output of the first command above, I got:

```
Name:                          twitter-job
Namespace:                     cb
Labels:                        <none>
Annotations:                   <none>
Schedule:                      @daily
Concurrency Policy:            Allow
Suspend:                       False
Successful Job History Limit:  3
Failed Job History Limit:      1
Starting Deadline Seconds:     <unset>
Selector:                      <unset>
Parallelism:                   <unset>
Completions:                   <unset>
Pod Template:
  Labels:  <none>
  Containers:
   bb-twitter:
    Image:      647183375757.dkr.ecr.eu-west-1.amazonaws.com/bb
    Port:       <none>
    Host Port:  <none>
    Command:
      node
      scripts/twitter.js
    Environment:
      MYSQL_USER:      root
      MYSQL_PASSWORD:  <set to the key 'mysql-root-password' in secret 'mysql'>  Optional: false
      MYSQL_SERVER:    mysql
      MYSQL_DATABASE:  cryptobirds
    Mounts:            <none>
  Volumes:             <none>
Last Schedule Time:    Mon, 02 Aug 2021 02:00:00 +0200
Active Jobs:           twitter-job-27126253, twitter-job-27126254, twitter-job-27126255, twitter-job-27126256, twitter-job-27126720, twitter-job-27128160, twitter-job-27129600, twitter-job-27131040
Events:                <none>
```

I see they are 8 active jobs ... Why? jobs are supposed to finish soon, there
should be a maximum of 1 active job, right? Let's take a look to the `jobs`.

```
kubectl get jobs
NAME                       COMPLETIONS   DURATION   AGE
twitter-job-27126253   0/1           4d5h       4d5h
twitter-job-27126254   0/1           4d5h       4d5h
twitter-job-27126255   0/1           4d5h       4d5h
twitter-job-27126256   0/1           4d5h       4d5h
twitter-job-27126720   0/1           3d21h      3d21h
twitter-job-27128160   0/1           2d21h      2d21h
twitter-job-27129600   0/1           45h        45h
twitter-job-27131040   0/1           21h        21h
```

Ok, as per the `COMPLETIONS` column, looks like these jobs never finished. Now,
I see the next step is to get more information about one of these jobs. Let's take
a look to the last one:

Again,

```
kubectl describe job twitter-job-27131040 
Name:           twitter-job-27131040
Namespace:      cb
Selector:       controller-uid=39474e16-cb47-4d45-ba7e-e0520d478e41
Labels:         controller-uid=39474e16-cb47-4d45-ba7e-e0520d478e41
                job-name=twitter-job-27131040
Annotations:    <none>
Controlled By:  CronJob/twitter-job
Parallelism:    1
Completions:    1
Start Time:     Mon, 02 Aug 2021 02:00:00 +0200
Pods Statuses:  1 Running / 0 Succeeded / 0 Failed
Pod Template:
  Labels:  controller-uid=39474e16-cb47-4d45-ba7e-e0520d478e41
           job-name=twitter-job-27131040
  Containers:
   bb-twitter:
    Image:      647183375757.dkr.ecr.eu-west-1.amazonaws.com/bb
    Port:       <none>
    Host Port:  <none>
    Command:
      node
      scripts/twitter.js
    Environment:
      MYSQL_USER:      root
      MYSQL_PASSWORD:  <set to the key 'mysql-root-password' in secret 'mysql'>  Optional: false
      MYSQL_SERVER:    mysql
      MYSQL_DATABASE:  cryptobirds
    Mounts:            <none>
  Volumes:             <none>
Events:                <none>
```

I see information about the job itself but, to be honest, I don't see much insight
about what's going on with the tasks. Let's jump directly the the pods:

```
kubectl get pods
```

Output:

```
kubectl get pods
NAME                                READY   STATUS                       RESTARTS   AGE
cb-admin-test-546969dfd6-kmnxl      1/1     Running                      0          12d
cb-api-test-6b7955cb55-9gmf4        1/1     Running                      0          51m
cb-database-mysql-0                 1/1     Running                      0          25d
cb-platform-test-7c887594f5-drkmm   1/1     Running                      0          5d6h
cb-redis-master-0                   1/1     Running                      0          25d
cb-redis-replicas-0                 1/1     Running                      0          25d
cb-redis-replicas-1                 1/1     Running                      0          25d
cb-redis-replicas-2                 1/1     Running                      0          25d
twitter-job-27126253-q77dw      0/1     CreateContainerConfigError   0          4d5h
twitter-job-27126254-p42jm      0/1     CreateContainerConfigError   0          4d5h
twitter-job-27126255-nsbxc      0/1     CreateContainerConfigError   0          4d5h
twitter-job-27126256-4lqz5      0/1     CreateContainerConfigError   0          4d5h
twitter-job-27126720-44dnt      0/1     CreateContainerConfigError   0          3d21h
twitter-job-27128160-7tkq5      0/1     CreateContainerConfigError   0          2d21h
twitter-job-27129600-grthj      0/1     CreateContainerConfigError   0          45h
twitter-job-27131040-pz7vd      0/1     CreateContainerConfigError   0          21h
```

I see 8 pods that failed to start the container. So the jobs never started. We can't
even look at the container logs because ... they were never created! Then,
let's take a look into the a `pod` description:

```
kubectl describe pod twitter-job-27131040-pz7vd
```

```
Name:         twitter-job-27131040-pz7vd
Namespace:    cb
Priority:     0
Node:         ip-192-168-43-136.eu-west-1.compute.internal/192.168.43.136
Start Time:   Mon, 02 Aug 2021 02:00:00 +0200
Labels:       controller-uid=39474e16-cb47-4d45-ba7e-e0520d478e41
              job-name=twitter-job-27131040
Annotations:  kubernetes.io/psp: eks.privileged
Status:       Pending
IP:           192.168.42.63
IPs:
  IP:           192.168.42.63
Controlled By:  Job/twitter-job-27131040
Containers:
  bb-twitter:
    Container ID:  
    Image:         647183375757.dkr.ecr.eu-west-1.amazonaws.com/bb
    Image ID:      
    Port:          <none>
    Host Port:     <none>
    Command:
      node
      scripts/twitter.js
    State:          Waiting
      Reason:       CreateContainerConfigError
    Ready:          False
    Restart Count:  0
    Environment:
      MYSQL_USER:      root
      MYSQL_PASSWORD:  <set to the key 'mysql-root-password' in secret 'mysql'>  Optional: false
      MYSQL_SERVER:    mysql
      MYSQL_DATABASE:  cryptobirds
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-xzb7h (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             False 
  ContainersReady   False 
  PodScheduled      True 
Volumes:
  kube-api-access-xzb7h:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type     Reason  Age                   From     Message
  ----     ------  ----                  ----     -------
  Warning  Failed  20m (x5919 over 21h)  kubelet  Error: secret "mysql" not found
  Normal   Pulled  8s (x6011 over 21h)   kubelet  Container image "647183375757.dkr.ecr.eu-west-1.amazonaws.com/bb" already present on machine
```

Et viol??, see the events when the pod tried to run the containers:

```
Warning  Failed  20m (x5919 over 21h)  kubelet  Error: secret "mysql" not found
```

Looks like the job is configured to pass a secret that is not found in the `namespace`.

- Copy paste error?
- Can you find the right secret that contains the database credentials in the same
namespace? Maybe describing other `deployments` that connect to the database?
- Once you fix this error. Will the job work? Will you be able to keep troubleshooting
until it is running successfully?

### List of things we want to see when observing cronjobs

- How many cronjobs are currently active?
- How many jobs are currently running?
- How can I see the history of jobs?
- How can I filter the history of the jobs?
- How can I see when a specific job is going to be created by a cronjob?
- How can I see the logs of a specific job?
- How can I see the last job of a specific cronjob?
- ...

### Running a cronjob manually

Sometimes you may want to manually run a cron(job), for example if a cronjob has failed and you want to run it again:
```
kubectl create job --from=cronjob/<name-of-cron-job> <name-of-job>
```
### Monitoring cronjobs using Kubernetes events

We can use Kubernetes events to monitor the Jobs lifecycles for our Birdbrains.

As an example, the following query demonstrate how can we query an event stream
from a namespace and filter them with `jq` to identify the events we are interested on,
like failed, created or completed jobs:

```
kubectl get events -n cb --sort-by=.metadata.creationTimestamp -o json | jq '.items[] | select(.involvedObject.kind=="Job") | {Result: .reason, Name: .involvedObject.name, FinishTime: .lastTimestamp, StartTime: .firstTimestamp, Namespace: .involvedObject.namespace, Message: .message}'
```

### Installing the Kubernetes Metrics Server

TODO:
- https://docs.aws.amazon.com/eks/latest/userguide/metrics-server.html

### Monitoring with Prometheus and Graphana
It has its own namespace: `cb-monitoring`
```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```
```
helm install prom-stack prometheus-community/kube-prometheus-stack
```
```
helm upgrade prom-stack prometheus-community/kube-prometheus-stack --install
```
#### Exporters:
[MySQL](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mysql-exporter)
```
helm install prom-mysql-exporter prometheus-community/prometheus-mysql-exporter
```
[MongoDB](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mongodb-exporter)
```
helm install prom-mongodb-exporter prometheus-community/prometheus-mongodb-exporter
```
[Redis](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-redis-exporter)
```
helm install prom-redis-exporter prometheus-community/prometheus-redis-exporter
```
### Alerting set-up and protocol

### Backups and restore

### Autoscaling

To set the kubernetes autoscaler, we followed this documentation:
https://docs.aws.amazon.com/eks/latest/userguide/cluster-autoscaler.html

In summary:

- Create this policy as `cb-eks-cluster-autoscaler-policy`, save as:
`cluster-autoscaler-policy.json`.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "autoscaling:DescribeAutoScalingGroups",
                "autoscaling:DescribeAutoScalingInstances",
                "autoscaling:DescribeLaunchConfigurations",
                "autoscaling:DescribeTags",
                "autoscaling:SetDesiredCapacity",
                "autoscaling:TerminateInstanceInAutoScalingGroup",
                "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*",
            "Effect": "Allow"
        }
    ]
}
```

```
aws iam create-policy \
    --policy-name cb-eks-cluster-autoscaler-policy \
    --policy-document file://cluster-autoscaler-policy.json
```

- Then, create the service account as usual:

```
SERVICE_ACCOUNT_NAME=cluster-autoscaler 
NAMESPACE=kube-system
CLUSTER_NAME=cb-hector-eks-test-2
IAM_POLICY_ARN=arn:aws:iam::647183375757:policy/cb-eks-cluster-autoscaler-policy
eksctl create iamserviceaccount \
    --name ${SERVICE_ACCOUNT_NAME} \
    --namespace ${NAMESPACE} \
    --cluster ${CLUSTER_NAME} \
    --attach-policy-arn ${IAM_POLICY_ARN} \
    --approve \
    --override-existing-serviceaccounts
```

- Deploy the autoscaler

```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
```

- Annotate with the role ARN previously created (found it using AWS console)

```
ROLE_ARN=arn:aws:iam::647183375757:role/eksctl-cb-hector-eks-test-2-addon-iamservice-Role1-1JKM41GYZDYNU
kubectl annotate serviceaccount cluster-autoscaler \
  -n kube-system \
  eks.amazonaws.com/role-arn=${ROLE_ARN}
```

- Patch

```
kubectl patch deployment cluster-autoscaler \
  -n kube-system \
  -p '{"spec":{"template":{"metadata":{"annotations":{"cluster-autoscaler.kubernetes.io/safe-to-evict": "false"}}}}}'
```

- Edit and replace <CLUSTER_NAME> by the cluster name

```
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
```

- Set the image version (please, follow aws instructions referenced above)

```
kubectl set image deployment cluster-autoscaler \
  -n kube-system \
  cluster-autoscaler=k8s.gcr.io/autoscaling/cluster-autoscaler:v1.21.0
```

- View the cluster autoscaler logs

```
kubectl -n kube-system logs -f deployment.apps/cluster-autoscaler
```