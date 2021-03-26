# Blue Green Deployment

## Deploy EKS cluster

1.安装eksctl, kubectl, helm

```bash
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
curl -o kubectl https://amazon-eks.s3.us-west-2.amazonaws.com/1.18.8/2020-09-18/bin/linux/amd64/kubectl
curl -sSL https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3 | bash
helm repo add stable https://charts.helm.sh/stable

```

2.部署EKS集群

```bash
eksctl create cluster \
--name <my-cluster> \
--version <1.18> \
--region <cn-north-1> \
--nodegroup-name <linux-nodes> \
--nodes <3> \
--nodes-min <1> \
--nodes-max <4> \
--ssh-access \
--ssh-public-key <name-of-ec2-keypair> \
--managed
```

3.配置安装ALB controller

```bash
eksctl utils associate-iam-oidc-provider \
    --region <region-code> \
    --cluster <my-cluster> \
    --approve

curl -o iam-policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/main/docs/install/iam_policy_cn.json

aws iam create-policy \
    --policy-name <AWSLoadBalancerControllerIAMPolicy> \
    --policy-document file://iam_policy.json
eksctl create iamserviceaccount \
  --cluster=<my-cluster> \
  --namespace=kube-system \
  --name=aws-load-balancer-controller \
  --attach-policy-arn=arn:aws-cn:iam::<AWS_ACCOUNT_ID>:policy/<AWSLoadBalancerControllerIAMPolicy> \
  --override-existing-serviceaccounts \
  --approve
  
 kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"
 helm repo add eks https://aws.github.io/eks-charts
helm upgrade -i aws-load-balancer-controller eks/aws-load-balancer-controller \
  --set clusterName=<cluster-name> \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  -n kube-system
  
 kubectl edit deployment aws-load-balancer-controller -n kube-system
 找到
 spec:
      containers:
      - args:
        - --cluster-name=eks
        - --ingress-class=alb
  修改为:
  spec:
      containers:
      - args:
        - --cluster-name=eks
        - --ingress-class=alb
        - --enable-shield=false
        - --enable-waf=false
        - --enable-wafv2=false
```

4.返回以下输出说明ALB ingress controller部署成功

```bash
➜  blue-green-deployment kubectl get deployment -n kube-system aws-load-balancer-controller
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
aws-load-balancer-controller   1/1     1            1           74s
```

5.部署deployment01, 为version 0.0.1

```bash
cat <<EOF > 2_app-v1.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: app
  name: app-version-1
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: app
      version: 0.0.1
  template:
    metadata:
      labels:
        run: app
        version: 0.0.1
    spec:
      containers:
      - name: app
        image: herreraluis/basic_example:0.0.1
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
EOF

kubectl apply -f 2_app-v1.yaml
```

6.部署deployment02, 为version 0.0.2

```bash
cat <<EOF > 3_app-v2.yaml

apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    run: app
  name: app-version-2
  namespace: default
spec:
  replicas: 1
  selector:
    matchLabels:
      run: app
      version: 0.0.2
  template:
    metadata:
      labels:
        run: app
        version: 0.0.2
    spec:
      containers:
      - name: app
        image: herreraluis/basic_example:0.0.2
        imagePullPolicy: Always
        ports:
        - containerPort: 5000
EOF

kubectl apply -f 3_app-v2.yaml
```

7.为deployment01/02创建service，名为blue-green-service，使用selector默认指向run:app version: 0.0.1

```bash
cat <<EOF > 4_blue-green-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: blue-green-service
  namespace: default
  labels:
    app: blue-green-service
spec:
  type: NodePort
  selector:
    run: app
    version: 0.0.1
  ports:
  - name: http
    port: 80
    protocol: TCP
    targetPort: 5000
EOF

kubectl apply -f 4_blue-service.yaml
```

8.创建ALB Ingress, 并连接到blue-green-service

```bash
cat <<EOF > 5_ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  namespace:  default
  name: app-ingress
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
spec:
  rules:
    - http:
        paths:
        - path: /
          backend:
            serviceName: blue-green-service
            servicePort: 80
EOF

kubectl apply -f 6_ingress.yaml
```

7.通过kubectl get ingress app-ingress获取ELB域名

```bash
kubectl get ingress app-ingress
NAME          CLASS    HOSTS   ADDRESS                                                                     PORTS   AGE
app-ingress   <none>   *       k8s-default-appingre-24ecb13054-894412454.cn-north-1.elb.amazonaws.com.cn   80      10m
```

8.访问域名会返回以下输出

```bash
{
  "email": "lorem.ipsum+45@email.com",
  "id": "bb848ad4-2b16-11eb-b403-0a5623c0d2c6",
  "username": "The user 45",
  "version": "0.0.1"
}
```

8.修改blue-green-service, 修改selector指向run: app version:0.0.2指向deployment

```bash
kubectl edit svc blue-green-service
```

9.再次返回相同的域名会返回不同的输出，说明ALB已经指向了修改后的blue-service

```bash
{
  "date": "Fri, 20 Nov 2020 09:58:56 GMT",
  "email": "lorem.ipsum+88@email.com",
  "id": "012c8820-2b17-11eb-9e05-4a8189d334f9",
  "telephone": "+593 646 772 559",
  "username": "The user 88",
  "version": "0.0.2"
}
```
