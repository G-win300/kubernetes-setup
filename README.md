# kubernetes-setup

# create Cluster
eksctl create cluster --name=gwineksdemo1 \
                      --region=us-east-1 \
                      --zones=us-east-1a,us-east-1b \
                      --without-nodegroup 
       TO DELETE             
eksctl delete cluster gwineksdemo1
                      
                      
eksctl utils associate-iam-oidc-provider \
    --region us-east-1 \
    --cluster gwineksdemo1 \
    --approve
    
    
# Create Public Node Group   

  eksctl create nodegroup --cluster=gwineksdemo1 \
                       --region=us-east-1 \
                       --name=gwineksdemo1-ng-private1 \
                       --node-type=t3.medium \
                       --nodes=2 \
                       --nodes-min=2 \
                       --nodes-max=4 \
                       --node-volume-size=20 \
                       --ssh-access \
                       --ssh-public-key=kube-demo \
                       --managed \
                       --asg-access \
                       --external-dns-access \
                       --full-ecr-access \
                       --appmesh-access \
                       --alb-ingress-access \
                       --node-private-networking    /*for public subnet...just removed the line*/ 
                       
   # TO DELETE                   
  eksctl delete nodegroup --cluster=gwineksdemo1  --name=gwineksdemo1-ng-private1

		
		## STEP 2
	# Download IAM Policy
	## Download latest
curl -o iam_policy_latest.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy.json

	## Verify latest
  ls -lrta 

	## Download specific version
  curl -o iam_policy_v2.3.1.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.3.1/docs/install/iam_policy.json

(....#take note of policyARN)

		STEP 2B
	# Create IAM Policy using policy downloaded 
aws iam create-policy \
    --policy-name AWSLoadBalancerControllerIAMPolicy \
    --policy-document file://iam_policy_latest.json
    (#takenote: sometimes, the policy will have already been created sef. hence, no need downloading fresh iam_policy_v2.3.1.json file 
    of step 2a...except you're using new instance)
    
    
 		STEP3
# Replaced name, cluster and policy arn (Policy arn we took note in step-02)
eksctl create iamserviceaccount \
  --cluster=gwineksdemo1 \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws:iam::282024636277:policy/AWSLoadBalancerControllerIAMPolicy \
  --override-existing-serviceaccounts \
  --approve
  
  	VERIFY
  eksctl get iamserviceaccount --cluster gwineksdemo1


		STEP 4a
    #note: get your VPC id and input in the command below
    also, if deploying to any other region, use this link to get the "set image repo" (https://docs.aws.amazon.com/eks/latest/userguide/add-ons-images.html)
	

		STEP 4b
# Install the AWS Load Balancer Controller.
## Template
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=gwineksdemo1 \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=vpc-0510ac48d05ad364f \
  --set image.repository=602401143452.dkr.ecr.us-east-1.amazonaws.com/amazon/aws-load-balancer-controller

		
    COMMANDS TO Verify that the controller is installed and Webhook Service created
 # Verify that the controller is installed.
kubectl -n kube-system get deployment 
kubectl -n kube-system get deployment aws-load-balancer-controller
kubectl -n kube-system describe deployment aws-load-balancer-controller


		 STEP 5
                       FOR EXTERNAL DNS
This IAM policy will allow external-dns pod to add, remove DNS entries (Record Sets in a Hosted Zone) in AWS Route53 service
go to policy in IAM and click on create new policy
copy and paste this code
 -----------------------------------------------------
 
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

--------------------------------------------------------------

give this policy the following credentials:
Name: AllowExternalDNSUpdates
Description: Allow access to Route53 Resources for ExternalDNS
 
 take note of the policy ARN=arn:aws:iam::282024636277:policy/AllowExternalDNSUpdates
