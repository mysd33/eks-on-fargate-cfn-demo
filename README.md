# SpringBoot APをEKS/Fargateで動作させCode系でCI/CDするCloudFormationサンプルテンプレート
## 構成
* システム構成図
  * TODO: 最終目標の構成で現状ここまでできていない
![システム構成図](img/eks-rolling-update.png)
  
## 事前準備
* CodePipeline、CodeBuildのArtifact用のS3バケットを作成しておく
* kubectl, eksctl, helmをインストールしておく
  * 参考
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/install-kubectl.html
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/eksctl.html
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/helm.html
## IAM
### 1. IAMの作成
```sh
aws cloudformation validate-template --template-body file://cfn-iam.yaml
aws cloudformation create-stack --stack-name EKS-IAM-Stack --template-body file://cfn-iam.yaml --capabilities CAPABILITY_IAM
```
## CI環境
* 別途、以下の2つのSpringBootAPのプロジェクトが以下のリポジトリ名でCodeCommitにある前提
  * backend-for-frontend
    * BFFのAP
  * backend
    * BackendのAP
  * TBD: CD変更検知用のマニフェストファイルのリポジトリ

### 1. ECRの作成
```sh
aws cloudformation validate-template --template-body file://cfn-ecr.yaml
aws cloudformation create-stack --stack-name ECR-Stack --template-body file://cfn-ecr.yaml
```

### 2. CodeBuildのプロジェクト作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codebuild.yaml
aws cloudformation create-stack --stack-name BFF-CodeBuild-Stack --template-body file://cfn-bff-codebuild.yaml
aws cloudformation validate-template --template-body file://cfn-backend-codebuild.yaml
aws cloudformation create-stack --stack-name Backend-CodeBuild-Stack --template-body file://cfn-backend-codebuild.yaml
```
* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」

* 本当は、CloudFormationテンプレートのCodeBuildのSorceTypeをCodePipelineにするが、いったんDockerイメージ作成して動作確認したいので、今はCodeCommitになっている。動いてはいるので保留。

* TBD: Mavenのカスタムローカルキャッシュによるビルド時間短縮がうまく動いていない
  * ひょっとしたら、ローカルキャッシュではなくS3でないとうまくいかない？
    * https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/build-caching.html  
    * https://aws.amazon.com/jp/blogs/devops/how-to-enable-caching-for-aws-codebuild/

### 3. ECRへ最初のDockerイメージをプッシュ
2つのCodeBuildプロジェクトが作成されるので、それぞれビルド実行し、ECRにDockerイメージをプッシュさせる

## ネットワーク環境
* TODO: ekctlコマンドで、事前に作成したVPC上にクラスタ作成するようにする
  * https://eksctl.io/usage/vpc-networking/
### 1. VPC、Security Group等の作成
* eksctlはデフォルトでEKS専用のVPCを作成しクラスタ起動することもできるが、サンプルでは既存のVPCを使ってクラスタ起動する場合を想定し、以下を実行
```sh
aws cloudformation validate-template --template-body file://amazon-eks-vpc-private-subnets.yaml
aws cloudformation create-stack --stack-name EKS-VPC-Stack --template-body file://amazon-eks-vpc-private-subnets.yaml
```
* TODO: 現状、以下のCfnサンプルテンプレートそのままなので、カスタマイズ
  * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/creating-a-vpc.html
    * Subnetに、kubernetes.io/role/elb、kubernetes.io/role/internal-elbのTag付けが必要
      * 内部ロードバランサーにプライベートサブネットを使用できるようにするには、VPC のすべてのプライベートサブネットにkubernetes.io/role/internal-elbをタグ付け
    * https://aws.amazon.com/jp/premiumsupport/knowledge-center/eks-vpc-subnet-discovery
      
## DB環境
* TBD:　今後Aurora等のRDBリソースのサンプル作成を検討

## EKS環境
### 1. EKSクラスタの作成
* TODO: ClusterConfigファイルでの実行（eksctl create cluster -f ekscluster.yaml）  
* TODO: 既存のVPC上での作成
  * https://eksctl.io/usage/vpc-networking/
* 以下のeksctlコマンドを実行
  * 参考
    * https://eksctl.io/usage/fargate-support/
    * https://eksctl.io/usage/schema/

```sh
set AWS_ACCOUNT_ID=(AWSアカウントID)
set AWS_REGION=(リージョン)
set EKS_CLUSTER_NAME=demo-eks-cluster

eksctl create cluster ^
--name %EKS_CLUSTER_NAME% ^
--version 1.22 ^
--region %AWS_REGION% ^
--fargate
#--dry-runオプションでDry Run可能

#TODO: コマンドオプションの追加検討
#--with-oidc ^
#--vpc-cidr XXX ^
#--vpc-public-subnets XXX ^
#--vpc-private-subnets XXX ^
#--alb-ingress-access ^
#--full-ecr-access
```

* EKSの動作確認
```sh
#クラスタノードの表示
kubectl get nodes -o wide
#クラスタ上のPodの表示
kubectl get pods --all-namespaces -o wide
```

* アプリケーションの名前空間用のFargateプロファイルの作成
  * TODO: ClusterConfigファイル化
```sh
eksctl create fargateprofile --cluster %EKS_CLUSTER_NAME% ^
--region %AWS_REGION% ^
--name fp-demo-app ^
--namespace demo-app

#Fargateプロファイルの表示
#デフォルト作成（default、kube-system名前空間用）のものと
#前述の手順で作成されたプロファイルが２つ表示される
eksctl get fargateprofile --cluster %EKS_CLUSTER_NAME% -o yaml

```

* TBD: クラスタの完全Privateクラスタ化
  * https://eksctl.io/usage/eks-private-cluster/
  * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/private-clusters.html

* TBD: コントロールプレーンのCloudWatch Logsの有効化
  * https://eksctl.io/usage/cloudwatch-cluster-logging/
  * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/control-plane-logs.html

### 2. ALBの作成
* Fargateでは最も簡単なCLBのロードバランサが使えない。IngressとしてALBを作成するための設定が必要。
  * 参考
    * https://aws.amazon.com/jp/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/aws-load-balancer-controller.html
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/alb-ingress.html
* クラスターがサービスアカウントにIAMを使用することを許可する
  * 参考
    * https://eksctl.io/usage/iamserviceaccounts/
```sh
eksctl utils associate-iam-oidc-provider --cluster %EKS_CLUSTER_NAME% --approve
```
* AWS Load Balancer Controller用のIAMポリシーを作成する  
```sh
curl -o iam_policy.json https://raw.githubusercontent.com/kubernetes-sigs/aws-load-balancer-controller/v2.2.0/docs/install/iam_policy.json

aws iam create-policy --policy-name AWSLoadBalancerControllerIAMPolicy ^
--policy-document file://iam_policy.json
```
*  kube-system名前空間にaws-load-balancer-controllerという名前のサービスアカウントを作成
```sh
eksctl create iamserviceaccount ^
  --cluster=%EKS_CLUSTER_NAME% ^
  --namespace=kube-system ^
  --name=aws-load-balancer-controller ^
  --attach-policy-arn=arn:aws:iam::%AWS_ACCOUNT_ID%:policy/AWSLoadBalancerControllerIAMPolicy ^
  --override-existing-serviceaccounts ^
  --approve

#作成されたサービスアカウントの確認
eksctl get iamserviceaccount --cluster %EKS_CLUSTER_NAME% --name aws-load-balancer-controller --namespace kube-system
```

* helmを使用してAWS Load Balancer Controllerをインストール
```sh
#Amazon EKSチャートレポをHelmに追加
helm repo add eks https://aws.github.io/eks-charts
helm repo update

#TargetGroupBinding カスタムリソース定義 (CRD) をインストール
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

set VPC_ID=(EKSクラスタのVPC_ID)

#Helm チャートをインストール
#us-west-2以外のAWSリージョンにデプロイする場合は「--set image.repository」
#オプションで以下のサイトにあるリポジトリを指定
#https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/add-ons-images.html
helm install aws-load-balancer-controller eks/aws-load-balancer-controller ^
    --set clusterName=%EKS_CLUSTER_NAME% ^
    --set serviceAccount.create=false ^
    --set region=%AWS_REGION% ^
    --set vpcId=%VPC_ID% ^
    --set image.repository=602401143452.dkr.ecr.%AWS_REGION%.amazonaws.com/amazon/aws-load-balancer-controller ^
    --set serviceAccount.name=aws-load-balancer-controller ^
    -n kube-system

#AWS Load Balancer Controllerがインストールされていることを確認
kubectl get deployment -n kube-system aws-load-balancer-controller    
#問題がある場合にログを見るとよい
kubectl logs -n kube-system deployment.apps/aws-load-balancer-controller
```

### 3. ログ、メトリックス出力の設定
* EKS/Fargateの場合  
  * Fluent Bitをベースにした組み込みのログルーターを設定し、CloudWatch Logsへログを送信
  * AWS Distro for OpenTelemetry (ADOT)を使用してメトリクスをCloudWatch Containe Insightsに送信  
  * 参考
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/fargate-logging.html
    * https://blog.mmmcorp.co.jp/blog/2021/08/11/post-1704/
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/monitoring-fargate-usage.html
    * https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/Container-Insights-EKS-otel.html#Container-Insights-EKS-otel-Fargate
    
* TBD: 手順

### 4. k8sリソースの作成
* envsubstコマンドを使うので、Windowsの場合には、Git bashで実行するとよい

* ECRのアドレスの環境変数
```sh
AWS_ACCOUNT_ID=(AWSアカウントID)
AWS_REGION=(リージョン)
ECR_HOST=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
echo $ECR_HOST
export ECR_HOST
```
* Namespaceの作成
```sh
cd k8s
kubectl apply -f k8s-demo-app-namespace.yaml
```

* Backend APの作成
  * TODO: BackendのALBを内部ロードバランサ的にできるか？
```sh
#Deploymentの作成
envsubst < k8s-backend-deployment.yaml | kubectl apply -f -

#暫定手順：CLBロードバランサの作成（on EC2のみで動作、Fagateだと動かない）
#kubectl apply -f k8s-backend-clb.yaml

#Serviceの作成
kubectl apply -f k8s-backend-service.yaml

#Ingressの作成
kubectl apply -f k8s-backend-ingress.yaml

#Ingressの作成確認
kubectl get ingress/backend-app-ingress -n demo-app
```

* BFF APの作成
```sh
#Deploymentの作成
#kubectl get ingress -n demo-appでADDRESSからBackend向けのALBのDNSを取得し設定
BACKEND_LB_DNS=(Backend APのロードバランサ)
#例：BACKEND_LB_DNS=a5027f47adc0d4c10bc2da33b708b8fc-1647541580.ap-northeast-1.elb.amazonaws.com
export BACKEND_LB_DNS
envsubst < k8s-bff-deployment.yaml | kubectl apply -f -

#暫定手順：CLBロードバランサの作成（on EC2のみで動作、Fagateは動かない）
#kubectl apply -f k8s-bff-clb.yaml

#Serviceの作成
kubectl apply -f k8s-bff-service.yaml

#Ingressの作成
kubectl apply -f k8s-bff-ingress.yaml

#Ingressの作成確認
kubectl get ingress/bff-app-ingress -n demo-app
```

### 5. APの実行確認
* Backend AP
  * VPC内のbationを作成し、curlコマンドで動作確認
```sh
curl http://(ロードバランサのDNS名)/backend/api/v1/users
```

* Bff AP
  * ブラウザで確認
    * サンプルAPは、ECS用のCloudFormationサンプルと共用しているので、画面に「Hello! AWS ECS sample!」と表示されるが気にしなくてよい。
```sh
http://(ロードバランサのDNS名)/backend-for-frontend/index.html
```

## CD環境
### 1. CodePipelineの作成
* TBD

### 2. CodePipelineの確認
* TBD

### 3. ソースコードの変更
* TBD

## k8sリソースとEKSクラスタ等の削除
```sh
kubectl delete ingress bff-app-ingress -n demo-app
kubectl delete ingress backend-app-ingress -n demo-app
kubectl delete service bff-app-service -n demo-app
kubectl delete service backend-app-service -n demo-app
kubectl delete deployment bff-app -n demo-app
kubectl delete deployment backend-app -n demo-app


helm uninstall aws-load-balancer-controller -n kube-system

eksctl delete fargateprofile --name my-profile --cluster %EKS_CLUSTER_NAME%
eksctl delete cluster --name %EKS_CLUSTER_NAME%

aws aws iam delete-policy --policy-arn arn:aws:iam::%AWS_ACCOUNT_ID%:policy/AWSLoadBalancerControllerIAMPolicy

```