# 🚀 Cluster Autoscaler Setup on Amazon EKS

This guide walks you through installing and configuring the **Cluster Autoscaler** on an Amazon EKS cluster using the AWS cloud provider.

---

## 1️⃣ Deploy Cluster Autoscaler

Apply the official Cluster Autoscaler manifest for your Kubernetes version (adjust `1.29.0` if needed):

```sh
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/cluster-autoscaler-1.29.0/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml
2️⃣ Verify the Pod
Check that the autoscaler pod is running in the kube-system namespace:


kubectl -n kube-system get pods -l app=cluster-autoscaler
Expected output:


NAME                                  READY   STATUS    RESTARTS   AGE
cluster-autoscaler-6889f6cf54-7pcsh   1/1     Running   0          2m
3️⃣ Edit Deployment (Add Cluster Name)
Edit the deployment to configure your cluster name:


kubectl -n kube-system edit deployment.apps/cluster-autoscaler
Inside the manifest, find the container args section and update:

yaml
Copy code
containers:
  - name: cluster-autoscaler
    command:
      - ./cluster-autoscaler
      - --v=4
      - --stderrthreshold=info
      - --cloud-provider=aws
      - --skip-nodes-with-local-storage=false
      - --expander=least-waste
      - --balance-similar-node-groups
      - --skip-nodes-with-system-pods=false
      - --nodes=2:6:ng-af5ac006
      - --cluster-name=naresh   # ✅ Add your cluster name here
Save & exit.

4️⃣ Configure IAM Permissions
Cluster Autoscaler requires IAM permissions to scale nodes.
Go to your EKS Node Group IAM Role and attach the following policy.

👉 Either attach AmazonEKSClusterAutoscalerPolicy (AWS Managed)
or create a custom IAM policy with the JSON below.

Example IAM Policy JSON


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
      "Effect": "Allow",
      "Resource": "*"
    }
  ]
}
Attach this to your Node Group Role.

5️⃣ Update Node Group Scaling Config
Set your min/max/desired node counts for the autoscaler:

aws eks update-nodegroup-config \
  --cluster-name naresh \
  --nodegroup-name ng-af5ac006 \
  --scaling-config minSize=2,maxSize=6,desiredSize=3
6️⃣ Check Autoscaler Logs
Watch the logs to confirm the autoscaler is working:


kubectl -n kube-system logs -f deployment/cluster-autoscaler
Look for lines like:


I0828 17:36:38.403432       1 scale_up.go:422] Pod default/nginx-deployment-12345 is unschedulable ...
I0828 17:36:38.403451       1 scale_up.go:423] Scale-up triggered ...
✅ Validation
Deploy a test workload with more pods than your current node capacity:


kubectl create deployment nginx --image=nginx --replicas=50
Check if new nodes are being added:


kubectl get nodes -w
Scale down pods and watch nodes reduce (if below maxSize and above minSize):


kubectl scale deployment nginx --replicas=1
📝 Notes
minSize ensures at least 2 nodes are always running.

maxSize sets the upper scaling limit.

desiredSize is the starting point but will be adjusted dynamically.

Ensure your Node Group IAM Role has autoscaling permissions, otherwise the pod will stay in Pending or fail to scale.

Only one Cluster Autoscaler pod should be running per cluster (it uses leader election).