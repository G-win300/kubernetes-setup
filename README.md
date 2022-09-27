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
