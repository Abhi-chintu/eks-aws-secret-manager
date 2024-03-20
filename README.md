# eks-aws-secret-manager

**There are multiple ways we can use AWS Secret Manager to EKS:**
1. Write custom code using AWS SDK and load secrets from the Secret Manager during start-up of the Pod.
2. Write custom Kubernetes Controller which shall sync secrets from AWS Secrets Manager with Kubernetes Native Secret objects.
   which can then be leveraged by the Pods via Environment variables or Mounted Volumes.
3. Write MutatingAdmissionWebhook to inject Sidecar container into the Application Pod to load secrets from the AWS Secrets Manager 
   which can then be used by the application container using Mounted Volumes approach.
4. Using Kubernetes Secrets Store CSI Driver — It can load secrets from AWS Secrets Manager and mount them on the Pod using Mounted Volumes.

We follow using K8s Secrets Store CSI Driver.

Workflow:
- the user make request to create a pod
- EKS API Server receives the request and using Pod Scheduler schedules pod to a Node
- Kubelet running on the Node gets the request to create Pod. Kubelet scans the Pod manifest file and finds volume to be mounted using CSI Driver.
- Kubelet invokes Secret Store CSI Driver via RPC call to mount the volume in the Pod
- Secret Store CSI Driver creates the volume and invokes the AWS Secrets and Config Provider (ASCP) to further retrieve the secrets from the AWS Secrets Manager
- ASCP uses SecretProviderClass manifest file to find which secrets to be loaded and creates the secrets files in the mounted volume
- Pod gets into Started state and files are read from the mounted volume to get the secrets and perform the business.

Pre-requistie:
--------------
 - Eks cluster
 - OIDC provider

1. Install CSI Secrets Store Driver

        helm repo add secrets-store-csi-driver https://kubernetes-sigs.github.io/secrets-store-csi-driver/charts
   
        helm install csi-secrets-store secrets-store-csi-driver/secrets-store-csi-driver --namespace kube-system --set syncSecret.enabled=true

2. Install AWS Secrets and Config Provider

        kubectl apply -f https://raw.githubusercontent.com/aws/secrets-store-csi-driver-provider-aws/main/deployment/aws-provider-installer.yaml

3. Create Secret inside AWS Secrets Manager 

        REGION=ap-south-1
        CLUSTERNAME=eksCluster
        SECRET_ARN=$(aws --query ARN --output text secretsmanager  create-secret --name DBCredentials --secret-string '{"dbusername":"arun", "dbpassword":"pass@123"}' --    region "$REGION")

4. Create IAM policy

         POLICY_ARN=$(aws --region "$REGION" --query Policy.Arn --output text iam create-policy --policy-name secret-store-iam-policy --policy-document '{
          "Version": "2012-10-17",
          "Statement": [ {
              "Effect": "Allow",
              "Action": ["secretsmanager:GetSecretValue", "secretsmanager:DescribeSecret"],
              "Resource": ["$SECRET_ARN"]
          } ]
       }')

5. Create Service Account

  Service Account will be associated with Policy we created above. This concept is called IAM Role for Service Account (IRSA).

      eksctl create iamserviceaccount --name secret-store-service-account \
      --region="$REGION" --cluster "$CLUSTERNAME" \
      --attach-policy-arn "$POLICY_ARN" \
      --approve --override-existing-serviceaccounts

6. Create AWS Secret Provider Class

       kubectl apply -f secret-store-provider.yaml

7. Create a deployment

       kubectl apply -f secret-store-deployment.yaml







