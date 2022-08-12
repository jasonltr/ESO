following this [eso](https://aws.amazon.com/blogs/containers/leverage-aws-secrets-stores-from-eks-fargate-with-external-secrets-operator/)  
launch sandbox  
`eksctl create cluster --name fargate-cluster --region us-east-1 --zones=us-east-1a,us-east-1b,us-east-1d --fargate`  

create new IAM policy to enable all EKS actions  
create new user and attach EKS profile to that user  
get AWS Access Key ID
Get secret  
`rm ~/.aws/credentials`  
`rm ~/.aws/config`  
run aws configure  
user new user created  


then run eksctl commands  
```
eksctl create fargateprofile \
    --cluster fargate-cluster \
    --name externalsecrets2 \
    --namespace external-secrets2
```
By right you should manually select the subnets also
```
cloud_user
AKIAUB3MR75KNU7CUFWH
NLVhhfHyG0pFU3HBbz99V1ZvyoKPBpQTxgKMUbJI
```
```
eksadmin
AKIAUB3MR75KGYWUITV3
YkwQioKV+KumAroQkXpEvaHl8G3QcXhGe0aA32EV
```
```
jason@DEV-52WP6M3:~/Documents/eso1$ eksctl get fargateprofile --cluster fargate-cluster -o yaml
- name: externalsecrets
  podExecutionRoleARN: arn:aws:iam::278863675220:role/eksctl-fargate-cluster-clu-FargatePodExecutionRole-1RICZ8XT5Y7DO
  selectors:
  - namespace: external-secrets
  status: ACTIVE
  subnets:
  - subnet-062cfe330f0f8abcf
  - subnet-04a380b0a99d50e33
  - subnet-05d8e322f8075a288
- name: fp-default
  podExecutionRoleARN: arn:aws:iam::278863675220:role/eksctl-fargate-cluster-clu-FargatePodExecutionRole-1RICZ8XT5Y7DO
  selectors:
  - namespace: default
  - namespace: kube-system
  status: ACTIVE
  subnets:
  - subnet-062cfe330f0f8abcf
  - subnet-04a380b0a99d50e33
  - subnet-05d8e322f8075a288
  ```
may need to vary between cloud_user and eksadmin to run the different commands as cloud_user is cluster creator, while eksadmin will be able to run eksctl 
```
jason@DEV-52WP6M3:~/Documents/eso1$ helm install external-secrets \
>    external-secrets/external-secrets \
>    -n external-secrets \
>    --create-namespace \
>    --set installCRDs=true \
>    --set webhook.port=9443 
```
```
NAME: external-secrets
LAST DEPLOYED: Fri Aug 12 16:12:43 2022
NAMESPACE: external-secrets
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
external-secrets has been deployed successfully!

In order to begin using ExternalSecrets, you will need to set up a SecretStore
or ClusterSecretStore resource (for example, by creating a 'vault' SecretStore).

More information on the different types of SecretStores and how to configure them
can be found in our Github: https://github.com/external-secrets/external-secrets
```
```
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl get pods -n external-secretsNAME                                                READY   STATUS    RESTARTS   AGE
external-secrets-7886b578b4-kn5hh                   1/1     Running   0          97s
external-secrets-cert-controller-5578bf565f-x5kz8   0/1     Running   0          97s
external-secrets-webhook-84c7d457f4-jhcxl           0/1     Running   0          97s
```
```
`eksctl utils associate-iam-oidc-provider --cluster=fargate-cluster --approve`
```
```
2022-08-12 16:16:00 [ℹ]  will create IAM Open ID Connect provider for cluster "fargate-cluster" in "us-east-1"
2022-08-12 16:16:02 [✔]  created IAM Open ID Connect provider for cluster "fargate-cluster" in "us-east-1"
```
From the terminal where you have eksctl installed, run the following commands. IAMPOLICYARN will be replaced with the Amazon Resource Name (ARN) of the IAM policy that has permissions to access your AWS Secrets Manager secret.  
This service account will be attached to the policy you created earlier that allows the extraction of secrets from AWS secrets manager.
```
eksctl create iamserviceaccount \
    --name blogdemosa \
    --namespace default \
    --cluster fargate-cluster \
    --role-name "blogdemosa" \
    --attach-policy-arn arn:aws:iam::278863675220:policy/ReadOneSecret \
    --approve \
    --override-existing-serviceaccounts
```
```
2022-08-12 16:18:18 [ℹ]  1 iamserviceaccount (default/blogdemosa) was included (based on the include/exclude rules)  
2022-08-12 16:18:18 [!]  metadata of serviceaccounts that exist in Kubernetes will be updated, as --override-existing-serviceaccounts was set  
2022-08-12 16:18:18 [ℹ]  1 task: { 
    2 sequential sub-tasks: { 
        create IAM role for serviceaccount "default/blogdemosa",
        create serviceaccount "default/blogdemosa",
    } }2022-08-12 16:18:18 [ℹ]  building iamserviceaccount stack "eksctl-fargate-cluster-addon-iamserviceaccount-default-blogdemosa"
2022-08-12 16:18:18 [ℹ]  deploying stack "eksctl-fargate-cluster-addon-iamserviceaccount-default-blogdemosa"
2022-08-12 16:18:19 [ℹ]  waiting for CloudFormation stack "eksctl-fargate-cluster-addon-iamserviceaccount-default-blogdemosa"
2022-08-12 16:18:50 [ℹ]  waiting for CloudFormation stack "eksctl-fargate-cluster-addon-iamserviceaccount-default-blogdemosa"
2022-08-12 16:18:51 [ℹ]  created serviceaccount "default/blogdemosa"
```
check sa
```
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl get sa
NAME         SECRETS   AGE
blogdemosa   1         19s
default      1         95m
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl describe sa blogdemosa
Name:                blogdemosa
Namespace:           default
Labels:              app.kubernetes.io/managed-by=eksctl
Annotations:         eks.amazonaws.com/role-arn: arn:aws:iam::278863675220:role/blogdemosa
Image pull secrets:  <none>
Mountable secrets:   blogdemosa-token-dvcsh
Tokens:              blogdemosa-token-dvcsh
Events:              <none>
```
apply [secretstore.yaml](secretstore.yaml)
```
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl apply -f secretstore.yaml
secretstore.external-secrets.io/blogdemo created
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl get secretstore
NAME       AGE   STATUS   READY
blogdemo   25s   Valid    True
```
apply [externalsecret.yaml](externalsecret.yaml)
```
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl apply -f externalsecret.yaml
externalsecret.external-secrets.io/blogdemo created
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl describe secret blogdemosecret
Name:         blogdemosecret
Namespace:    default
Labels:       <none>
Annotations:  reconcile.external-secrets.io/data-hash: 58c4142861e3f9ff41537db547896cf3

Type:  Opaque

Data
====
admin-posgres-password:  5 bytes
admin-posgres-username:  5 bytes
```
apply [podexample.yaml](podexample.yaml)
```
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl apply -f podexample.yaml
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl get pods
NAME      READY   STATUS    RESTARTS   AGE
busybox   1/1     Running   0          66s
jason@DEV-52WP6M3:~/Documents/eso1$ kubectl exec --stdin --tty busybox -- /bin/sh
/ # echo $BLOG_SECRET_USERNAME
admin
/ # echo $BLOG_SECRET_PASSWORD
admin
```
