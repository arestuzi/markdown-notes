# How to deploy EBS CSI on AWS EKS in China region?

## ISSUE

- Cannot access `k8s.gcr.io` repository in China region.

## Prerequisites

1. Fetch the Account ID and save it to variable AWS_ACCOUNT_ID.

    ```bash
    export AWS_ACCOUNT_ID=`aws sts get-caller-identity --output json | jq .Account | sed 's/"//g'`
    ```

## Solution

1. Use the command below to create a directory

    ```bash
    mkdir -p ~/eks/ebs-csi
    ```

2. Run the following code block to create a IAM policy that allows the CSI driver's service account to make calls to AWS APIs on your behalf.

    ```bash
    cat <<EoF > ~/eks/ebs-csi/ebs-csi-policy.json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
        "Effect": "Allow",
        "Action": [
            "ec2:AttachVolume",
            "ec2:CreateSnapshot",
            "ec2:CreateTags",
            "ec2:CreateVolume",
            "ec2:DeleteSnapshot",
            "ec2:DeleteTags",
            "ec2:DeleteVolume",
            "ec2:DescribeAvailabilityZones",
            "ec2:DescribeInstances",
            "ec2:DescribeSnapshots",
            "ec2:DescribeTags",
            "ec2:DescribeVolumes",
            "ec2:DescribeVolumesModifications",
            "ec2:DetachVolume",
            "ec2:ModifyVolume"
        ],
        "Resource": "*"
        }
    ]
    }
    EoF
    ```

3. Let's create the policy.

    ```bash
    cd ~/eks/ebs-csi && aws iam create-policy --policy-name AmazonEKS_EBS_CSI_Driver_Policy --policy-document file://ebs-csi-policy.json
    ```

4. Create an IAM role and attach the IAM policy to it.

    a. View your cluster's OIDC provider URL. Replace <cluster_name> (including <>) with your cluster name.

    ```bash
    export OIDC=`aws eks describe-cluster --name eks --query "cluster.identity.oidc.issuer" --output text | sed 's/https\:\/\///g'`
    ```

    b. Create the IAM role.

    ```bash
    cat <<EoF > ~/eks/ebs-csi/trust-policy.json
    {
        "Version": "2012-10-17",
        "Statement": [
            {
            "Effect": "Allow",
            "Principal": {
                "Federated": "arn:aws-cn:iam::${AWS_ACCOUNT_ID}:oidc-provider/${OIDC}"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                "${OIDC}:sub": "system:serviceaccount:kube-system:ebs-csi-controller-sa"
                }
            }
            }
        ]
    }
    EoF
    aws iam create-role --role-name AmazonEKS_EBS_CSI_DriverRole --assume-role-policy-document file://"trust-policy.json"
    aws iam attach-role-policy --policy-arn arn:aws-cn:iam::${AWS_ACCOUNT_ID}:policy/AmazonEKS_EBS_CSI_Driver_Policy --role-name AmazonEKS_EBS_CSI_DriverRole
    ```

5. Create an OpenIDConnect provider.  
a. Open the Amazon EKS console at [https://console.amazonaws.cn/eks/home#/clusters](https://console.amazonaws.cn/eks/home#/clusters)  
b. Select the name of your cluster and then select the `Configuration` tab.  
c. In the `Details` section, note the value of the `OpenID Connect provider URL`.  
d. Open the IAM console at [https://console.amazonaws.cn/iam/](https://console.amazonaws.cn/iam/)  
e. In the navigation panel, choose `Identity Providers`. If a `Provider` is listed that matches the URL for your cluster, then you already have a provider for your cluster. If a provider isn't listed that matches the URL for your cluster, then you must create one.  
f. To create a provider, choose `Add Provider`.  
g. For `Provider Type`, choose `OpenID Connect`.  
h. For `Provider URL`, paste the OIDC issuer URL for your cluster, and then choose `Get thumbprint`.  
i. For `Audience`, enter `sts.amazonaws.com` and choose Add provider.  

6. [Clone the Amazon EBS Container Storage Interface(CSI) driver](https://github.com/kubernetes-sigs/aws-ebs-csi-driver/) GitHub repository to your computer.

    ```bash
    git clone https://github.com/kubernetes-sigs/aws-ebs-csi-driver.git
    cat <<EoF > ~/eks/ebs-csi/aws-ebs-csi-driver/deploy/kubernetes/overlays/stable/kustomization.yaml
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    bases:
    - ../../base
    images:
    - name: k8s.gcr.io/provider-aws/aws-ebs-csi-driver
      newName: 918309763551.dkr.ecr.cn-north-1.amazonaws.com.cn/eks/aws-ebs-csi-driver
      newTag: v0.9.0
    - name: k8s.gcr.io/sig-storage/csi-provisioner
      newName: public.ecr.aws/eks-distro/kubernetes-csi/external-provisioner
      newTag: v2.0.3-eks-1-18-1
    - name: k8s.gcr.io/sig-storage/csi-attacher
      newName: public.ecr.aws/eks-distro/kubernetes-csi/external-attacher
      newTag: v3.0.1-eks-1-18-1
    - name: k8s.gcr.io/sig-storage/livenessprobe
      newName: public.ecr.aws/eks-distro/kubernetes-csi/livenessprobe
      newTag: v2.1.0-eks-1-18-1
    - name: k8s.gcr.io/sig-storage/csi-node-driver-registrar
      newName: public.ecr.aws/eks-distro/kubernetes-csi/node-driver-registrar
      newTag: v2.0.1-eks-1-18-1
    EoF
    ```

7. annotate service account.

    ```bash
    kubectl annotate serviceaccount ebs-csi-controller-sa -n kube-system \
    eks.amazonaws.com/role-arn=arn:aws-cn:iam::${AWS_ACCOUNT_ID}:role/AmazonEKS_EBS_CSI_DriverRole
    ```

8. Delete the driver pods. They're automatically redeployed with the IAM permissions from the IAM policy assigned to the role.

    ```bash
    kubectl delete pods -n kube-system -l=app=ebs-csi-controller
    ```

9. Check Deployment status by running.

    ```bash
    (base) ➜  ebs-csi kubectl get deployment -n kube-system
    NAME                 READY   UP-TO-DATE   AVAILABLE   AGE
    coredns              2/2     2            2           43h
    ebs-csi-controller   2/2     2            2           3m43s
    ```

10. Testing.

    ```bash
    kubectl apply -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/storageclass.yaml 
    kubectl apply -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/claim.yaml 
    kubectl apply -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/pod.yaml 

    kubectl exec -it app cat /data/out.txt
    ```

11. Clean resources.

    ```bash
    kubectl delete -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/storageclass.yaml 
    kubectl delete -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/claim.yaml 
    kubectl delete -f aws-ebs-csi-driver/examples/kubernetes/dynamic-provisioning/specs/pod.yaml 
    ```
