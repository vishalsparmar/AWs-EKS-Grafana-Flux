Required GitLab variables
Set these in GitLab → CI/CD → Variables:
TFC_TOKEN        = <Terraform Cloud API token>
TFC_ORG          = <your org name>
TFC_WORKSPACE    = <your workspace name>
🧠 What this solves
✅ If initial run fails → pipeline waits
✅ If you rerun in TFC UI → picks latest run automatically
✅ GitLab reflects final outcome, not first attempt
⚙️ Optional improvement (recommended)
Add timeout protection


ince you already have a Transit Gateway in place, you're in a great position. The TGW is the hard part—now you just need to verify that your tools cluster can route to the target EKS cluster's private API endpoint.

Here is a step-by-step guide to test connectivity to one target EKS cluster using the AWS Console and CLI.

Step 1: Verify the Target EKS Cluster Endpoint Configuration

First, ensure the target cluster has its private endpoint enabled so it can receive requests from your tools cluster via the Transit Gateway.

Open the Amazon EKS Console and select the target cluster you want to test.
Click on the Networking tab.
In the Cluster endpoint section, click Manage endpoint access.
Verify or change the following settings:

Private endpoint access: Must be Enabled (this is the key setting for TGW connectivity) .
Public endpoint access: Can remain Enabled (since you need it for other tools) .
Click Save changes.
Why this matters: With private access enabled, the cluster's API server gets a DNS entry (*.eks.amazonaws.com) that resolves to a private IP address inside its own VPC . Your goal is to reach that private IP via the TGW.
Step 2: Configure Route Tables in the Transit Gateway

Your TGW must know how to direct traffic from your tools cluster's VPC to the target EKS cluster's VPC.

Open the Transit Gateway Console.
In the left navigation pane, select Transit Gateway Route Tables.
Select the route table associated with the attachment for your tools cluster's VPC.
Click the Routes tab, then Create static route.

CIDR: Enter the CIDR block of the target EKS cluster's VPC.
Blackhole: Leave this unchecked.
Attachment: Select the TGW attachment for the target EKS cluster's VPC.
Click Create static route.
Tip: You should also check the target EKS cluster's subnet route tables to ensure they have a route back to your tools cluster's VPC CIDR via the TGW.
Step 3: Test Network Reachability (The curl Test)

This is the most direct way to confirm the TGW is routing correctly. You need a pod in your tools cluster.

A. Deploy a test pod in your tools cluster

Save this as test-pod.yaml:

yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-test
spec:
  containers:
  - name: alpine
    image: alpine:latest
    command: ["sleep", "3600"]
  restartPolicy: Never
Apply it to your tools cluster:

bash
kubectl apply -f test-pod.yaml
B. Find the Target Cluster's Private IP

In the EKS Console, select your target cluster.
Go to the Networking tab.
Under Cluster endpoint, you will see Private endpoint. Copy the hostname (e.g., XXXXXXXXX.gr7.us-east-1.eks.amazonaws.com).
From your local machine, ping this hostname or use nslookup to resolve it to its private IP address:

bash
nslookup XXXXXXXXX.gr7.us-east-1.eks.amazonaws.com
The result will be one or two private IP addresses .
C. Run the connectivity test

Exec into the test pod in your tools cluster and try to reach the target cluster's API server:

bash
# Get a shell in the pod
kubectl exec -it network-test -- /bin/sh

# Inside the pod, attempt to connect to the target cluster's private IP on port 443
wget --spider --timeout=5 <Private-IP-Address-of-Target-Cluster>

# If wget is not available, try:
nc -zv <Private-IP-Address-of-Target-Cluster> 443
Success: The command exits without a timeout error. Your TGW routing is working.
Failure (Timeout): The connection hangs. Double-check your TGW route tables and Security Group rules.
Step 4: Configure kubectl from Your Tools Cluster

Once the network test passes, configure kubectl to use the private endpoint.

A. Generate the kubeconfig

From a pod in your tools cluster that has the AWS CLI installed (like your TFE agent), run:

bash
aws eks update-kubeconfig \
    --region <target-cluster-region> \
    --name <target-cluster-name> \
    --role-arn <IAM-role-ARN-with-access-to-the-target-cluster>
Note: The --role-arn is critical for cross-account access. This IAM role must have permissions to describe the cluster and be mapped in the target cluster's aws-auth ConfigMap .
B. Verify kubectl connectivity

bash
kubectl get namespaces
If this command succeeds, your TFE agent is now authenticated and communicating with the target cluster's API via the private endpoint and Transit Gateway.

Troubleshooting: The "Timeout" Error

If the kubectl command times out, it's almost always one of two issues:

Security Group (Most Common): The target EKS cluster's security group needs an inbound rule to allow traffic on port 443 from your tools cluster's VPC CIDR or the specific security group of your TFE agents .

Fix: In the EKS Console for the target cluster, go to the Networking tab. Click on the Cluster security group. Add an Inbound Rule:

Type: HTTPS (443)
Source: <CIDR-of-your-tools-cluster-VPC> or <ID-of-TFE-agents-sg>
IAM Permissions: The IAM role used by update-kubeconfig does not have access to the target cluster.

Fix: Ensure the IAM role's ARN is listed as a principal in the target cluster's Access entries (EKS API) or the aws-auth ConfigMap .

