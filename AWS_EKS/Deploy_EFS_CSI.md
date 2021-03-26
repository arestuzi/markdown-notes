# How to deploy EFS CSI in China region?

1. clone `aws-efs-csi-driver` to the EC2 instance.

    ```bash
    git clone https://github.com/kubernetes-sigs/aws-efs-csi-driver
    ```

2. replace `aws-efs-csi-driver/deploy/kubernetes/overlays/stable/kustomization.yaml` with following contents.

    ```bash
    apiVersion: kustomize.config.k8s.io/v1beta1
    kind: Kustomization
    bases:
    - ../../base
    images:
    - name: amazon/aws-efs-csi-driver
        newTag: v1.1.0
    - name: quay.io/k8scsi/livenessprobe
        newName: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/quay/k8scsi/livenessprobe
        newTag: v2.0.0
    - name: quay.io/k8scsi/csi-node-driver-registrar
        newName: 048912060910.dkr.ecr.cn-northwest-1.amazonaws.com.cn/quay/k8scsi/csi-node-driver-registrar
        newTag: v1.3.0
    ```

3. Create the EFS CSI.

    ```bash
    kubectl apply -k aws-efs-csi-driver/deploy/kubernetes/overlays/stable/
    ```