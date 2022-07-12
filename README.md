# SpringBoot APをEKS/Fargateで動作させCode系でCI/CDするCloudFormationサンプルテンプレート
## 構成
* システム構成図（最終形）
  * 最終目標の構成は、以下を想定しているが、まだできていない
![システム構成図](img/eks-rolling-update.png)

* システム構成図（現時点）
  * 現時点の構成は、以下の通り
![システム構成図](img/eks-rolling-update-current.png)

* ログの転送
  * コントロールプレーンのログをCloudWatch Logsへログを送信
  * データプレーンのAPログをFluent Bitをベースにした組み込みのログルーターを設定し、CloudWatch Logsへ送信

* メトリックスのモニタリング
  * AWS Distro for OpenTelemetry (ADOT)を使用してメトリックスをCloudWatch Container Insightsに送信
    
* オートスケーリング
  * 現状、未対応。今後対応予定。
  * Horizontal Pod AutoScaler
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/horizontal-pod-autoscaler.html
    * メトリックスサーバのインストールが必要
      * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/metrics-server.html

* CI/CD
  * CodePipeline、CodeBuildによる自動ビルド、ECRプッシュ、k8sのローリングアップデートに対応
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
### 1. アプリケーションのCodeCommit環境
* 別途、以下の2つのSpringBootAPのプロジェクトが以下のリポジトリ名でCodeCommitにある前提
  * backend-for-frontend
    * BFFのAP
    * backend-for-frontendという別のリポジトリに資材は格納
  * backend
    * BackendのAP
    * backendという別のリポジトリに資材は格納
* TBD: CD変更検知用のk8sマニフェストファイルのリポジトリ

### 2. ECRの作成
```sh
aws cloudformation validate-template --template-body file://cfn-ecr.yaml
aws cloudformation create-stack --stack-name ECR-Stack --template-body file://cfn-ecr.yaml
```

### 3. CI用CodeBuildのプロジェクト作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codebuild-ci.yaml
aws cloudformation create-stack --stack-name BFF-CodeBuild-CI-Stack --template-body file://cfn-bff-codebuild-ci.yaml

aws cloudformation validate-template --template-body file://cfn-backend-codebuild-ci.yaml
aws cloudformation create-stack --stack-name Backend-CodeBuild-CI-Stack --template-body file://cfn-backend-codebuild-ci.yaml
```
* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」

* 本当は、CloudFormationテンプレートのCodeBuildのSourceTypeをCodePipelineにするが、いったんDockerイメージ作成して動作確認したいので、今はCodeCommitになっている。動いてはいるので保留。

* TODO: Mavenのカスタムローカルキャッシュによるビルド時間短縮がうまく動いていない
  * ひょっとしたら、ローカルキャッシュではなくS3でないとうまくいかない？
    * https://docs.aws.amazon.com/ja_jp/codebuild/latest/userguide/build-caching.html  
    * https://aws.amazon.com/jp/blogs/devops/how-to-enable-caching-for-aws-codebuild/

### 3. ECRへ最初のDockerイメージをプッシュ
2つのCodeBuildプロジェクトが作成されるので、それぞれビルド実行し、ECRにDockerイメージをプッシュさせる。

## ネットワーク環境
* eksctlコマンドで、EKS専用のVPCを作成しクラスタ起動することもできるが、サンプルでは既存のVPCを使ってクラスタ起動する場合を想定した手順とする。
### 1. VPCおよびサブネット、Publicサブネット向けInternetGateway等の作成
* 以下のコマンドを実行する。
```sh
aws cloudformation validate-template --template-body file://cfn-vpc.yaml
aws cloudformation create-stack --stack-name EKS-VPC-Stack --template-body file://cfn-vpc.yaml

#以下サイトにあるAWS提供のVPCとNATGateway等作成するCfnテンプレートをベースした場合の実行例も残しておく
#https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/creating-a-vpc.html
#aws cloudformation validate-template --template-body file://amazon-eks-vpc-private-subnets.yaml
#aws cloudformation create-stack --stack-name EKS-VPC-Stack --template-body file://amazon-eks-vpc-private-subnets.yaml
```
* なお、ロードバランサーよる自動サブネット検出のため、作成されるサブネットには以下のタグを付与されるようにしている。
  * 外部ロードバランサが使用するパブリックサブネット
    * kubernetes.io/role/elb : 1
  * 内部ロードバランサが使用するプライベートサブネット 
    * kubernetes.io/role/internal-elb : 1
  * 参考
    * https://aws.amazon.com/jp/premiumsupport/knowledge-center/eks-vpc-subnet-discovery

### 2. Security Groupの作成
* Batsion等で、必要なSecurity Groupの作成
```sh
aws cloudformation validate-template --template-body file://cfn-sg.yaml
aws cloudformation create-stack --stack-name EKS-SG-Stack --template-body file://cfn-sg.yaml
```
* 必要に応じて、端末の接続元IPアドレス等のパラメータを指定
    * 「--parameters ParameterKey=TerminalCidrIP,ParameterValue=X.X.X.X/X」
      * 後続の手順で環境変数「TERMINAL_CIDR=X.X.X.X/X」で指定するものと同じ

### 3. VPC Endpointの作成とプライベートサブネットのルートテーブル更新
* TODO: 現時点では未対応。完全Private化のため、NATGatewayの代わりにVPC Endpoint作成するように変更予定。
  * 参考
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/private-clusters.html   
    * なお、eksctlで完全Privateなクラスタ設定が可能だが、全てPrivate Subnetのみで構築となるので、今回のようなPublic SubnetにALBを配置し、以降はPrivate Subnet上のk8sリソース上のAPへアクセスするといったIngressの設定ができないので使わない。
       * https://eksctl.io/usage/eks-private-cluster/

### 4. NAT Gatewayの作成とプライベートサブネットのルートテーブル更新
* 現時点ではVPC Endpoint未対応のためNAT Gatewayを作成
* TODO: VPC Endpoint対応したら任意作成とする
```sh
aws cloudformation validate-template --template-body file://cfn-ngw.yaml
aws cloudformation create-stack --stack-name EKS-NATGW-Stack --template-body file://cfn-ngw.yaml
```

## DB環境
* TBD:　今後Aurora等のRDBリソースのサンプル作成を検討

## EKS環境
* 以降、envsubstコマンドを使うので、Windowsの場合には、Git bashで実行するとよい
### 1. 環境変数設定
* envsubstコマンド等での利用のため環境変数を設定しておく
```sh
AWS_ACCOUNT_ID=(AWSアカウントID)
AWS_REGION=(リージョン) #例：AWS_REGION=ap-northeast-1
EKS_CLUSTER_NAME=demo-eks-cluster #作成するEKSのクラスタ名
K8S_VERSION=1.22 #インストールしたkubectlのバージョン
#CloudFormation（EKS-VPC-Stack）の出力表示で「VpcId」の内容を確認
VPC_ID=(EKSクラスタのVPC_ID)
#CloudFormation（EKS-VPC-Stack）の出力表示で「PublicSubnetOneId」の内容を確認
PUB_SUBNET1_ID=（1つ目のパブリックサブネットのID）
#CloudFormation（EKS-VPC-Stack）の出力表示で「PublicSubnetTwoId」の内容を確認
PUB_SUBNET2_ID=（2つ目のパブリックサブネットのID）
#CloudFormation（EKS-VPC-Stack）の出力表示で「PrivateSubnetOneId」の内容を確認
PRIV_SUBNET1_ID=（1つ目のプライベートサブネットのID）
#CloudFormation（EKS-VPC-Stack）の出力表示で「PrivateSubnetTwoId」の内容を確認
PRIV_SUBNET2_ID=（2つ目のプライベートサブネットのID）
#ECRのベースアドレス
ECR_HOST=$AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
#CloudFormation（EKS-IAM-Stack）の出力表示で「EKSFargatePodExecutionRoleArn」の内容を確認
POD_EXECUTION_ROLE_ARN=(FargateのPod実行ロールのARN)
#端末の接続元IPアドレス等、PublicなALBへ接続制限したい場合のCIDR
TERMINAL_CIDR=X.X.X.X/X

export AWS_ACCOUNT_ID
export AWS_REGION
export EKS_CLUSTER_NAME
export K8S_VERSION
export VPC_ID
export PUB_SUBNET1_ID
export PUB_SUBNET2_ID
export PRIV_SUBNET1_ID
export PRIV_SUBNET2_ID
export ECR_HOST
export POD_EXECUTION_ROLE_ARN
export TERMINAL_CIDR
```
### 2. EKSクラスタの作成
* 以下のeksctlコマンドを実行しFargateでEKSクラスタを作成
  * 実行すると、CloudFormationのスタックとして作成される
```sh
#Dry Run
envsubst < ekscluster.yaml | eksctl create cluster -f - --dry-run

#実行
envsubst < ekscluster.yaml | eksctl create cluster -f -
```
* EKSクラスタの作成確認
```sh
#クラスタノードの表示
kubectl get nodes -o wide
#クラスタ上のPodの表示
kubectl get pods --all-namespaces -o wide
```
* Fargateプロファイルの作成確認
```sh
eksctl get fargateprofile --cluster $EKS_CLUSTER_NAME -o yaml
```

* 参考
  * eksctlコマンドの実行
    * https://eksctl.io/introduction/#getting-started
  * ClusterConfigファイルの仕様
    * https://eksctl.io/usage/schema/
  * クラスターがサービスアカウントにIAMを使用する
    * https://eksctl.io/usage/iamserviceaccounts/
  * 既存のVPC上での作成
    * https://eksctl.io/usage/vpc-networking/
  * コントロールプレーンのCloudWatch Logsの有効化
    * https://eksctl.io/usage/cloudwatch-cluster-logging/
  * Fargate対応
    * https://eksctl.io/usage/fargate-support/    

* TODO: ClusterConfigファイルに、addonsでaws-load-balancer-controllerの記載を追加するとどうなるのか試してみる

* TODO: クラスタの完全Privateクラスタ化(VPC endpoint化)
  * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/private-clusters.html
  * https://aws.amazon.com/jp/blogs/containers/de-mystifying-cluster-networking-for-amazon-eks-worker-nodes/
  * https://eksctl.io/usage/eks-private-cluster/


### 3. ALBの作成
* Fargateでは最も簡単なCLBのロードバランサが使えない。IngressとしてALBを作成するための設定が必要。
  * 参考
    * https://aws.amazon.com/jp/premiumsupport/knowledge-center/eks-alb-ingress-controller-fargate/
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/aws-load-balancer-controller.html
    * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/alb-ingress.html

* クラスタ作成時に、AWS LoadBalancer Controller用のサービスアカウントも作成されたかの確認
```sh
eksctl get iamserviceaccount --cluster $EKS_CLUSTER_NAME --name aws-load-balancer-controller --namespace kube-system
```

* helmを使用してAWS Load Balancer Controllerをインストール
```sh
#Amazon EKSチャートレポをHelmに追加
helm repo add eks https://aws.github.io/eks-charts
helm repo update

#TargetGroupBinding カスタムリソース定義 (CRD) をインストール
kubectl apply -k "github.com/aws/eks-charts/stable/aws-load-balancer-controller//crds?ref=master"

#Helm チャートをインストール
#us-west-2以外のAWSリージョンにデプロイする場合は「--set image.repository」オプションで
#以下のサイトにあるリポジトリを指定
#https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/add-ons-images.html
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
    --set clusterName=$EKS_CLUSTER_NAME \
    --set serviceAccount.create=false \
    --set region=$AWS_REGION \
    --set vpcId=$VPC_ID \
    --set image.repository=602401143452.dkr.ecr.$AWS_REGION.amazonaws.com/amazon/aws-load-balancer-controller \
    --set serviceAccount.name=aws-load-balancer-controller \
    -n kube-system

#AWS Load Balancer Controllerがインストールされていることを確認
kubectl get deployment -n kube-system aws-load-balancer-controller
#問題がある場合にログを見るとよい
kubectl logs -n kube-system deployment.apps/aws-load-balancer-controller
```

### 4. ログ、メトリックス出力の設定
* コントロールプレーンのログ
  * クラスタ作成時のClusterConfigファイルの設定により以下のロググループにログ出力
    * /aws/eks/demo-eks-cluster/cluster

* データプレーンのログ
  * EKS/Fargateの場合、以下の対応が必要
    * Fluent Bitをベースにした組み込みのログルーターを設定し、CloudWatch Logsへ以下のロググループへログを送信
      * /eks/logs/fluent-bit-cloudwatch
    * 参考
      * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/fargate-logging.html
      * https://blog.mmmcorp.co.jp/blog/2021/08/11/post-1704/

  * Namespaceの作成
```sh
cd k8s
kubectl apply -f k8s-aws-observability-namespace.yaml
```  
  * CloudWatchへのログ送信のためのConfigMap作成
    * ここでは、Go言語で書かれた出力プラグイン（cloudwatch）を使った例にしている
```sh
envsubst < k8s-aws-logging-cloudwatch-configmap.yaml | kubectl apply -f -
```

* データプレーンのメトリックス
  * EKS/Fargateの場合、以下の対応が必要
    * AWS Distro for OpenTelemetry (ADOT)を使用してメトリクスをCloudWatch Container Insightsに送信  
      * 参考                  
        * https://aws.amazon.com/jp/blogs/news/introducing-amazon-cloudwatch-container-insights-for-amazon-eks-fargate-using-aws-distro-for-opentelemetry/
        * https://aws-otel.github.io/docs/getting-started/container-insights/eks-fargate
        * https://docs.aws.amazon.com/ja_jp/eks/latest/userguide/monitoring-fargate-usage.html

  * クラスタ作成時に、ADOT Collector用のサービスアカウントも作成されたかの確認
```sh
eksctl get iamserviceaccount --cluster $EKS_CLUSTER_NAME --name adot-collector --namespace fargate-container-insights
```

  * ADOT Collectorのインストール
```sh
envsubst < k8s-otel-fargate-container-insights.yaml | kubectl apply -f -
```

### 5. AP用のk8sリソースの作成、APの起動
* Namespaceの作成
```sh
cd k8s
kubectl apply -f k8s-demo-app-namespace.yaml
```
* Backend APの作成
```sh
#Deploymentの作成
envsubst < k8s-backend-deployment.yaml | kubectl apply -f -

#Podの起動確認しておくとよい
kubectl get pod -n demo-app

#Serviceの作成
kubectl apply -f k8s-backend-service.yaml

#Ingressの作成
kubectl apply -f k8s-backend-ingress.yaml

#Ingressの作成確認
kubectl get ingress/backend-app-ingress -n demo-app
```

* BFF APの作成
```sh
#kubectl get ingress/backend-app-ingress -n demo-appでADDRESSからBackend向けのALBのDNSを取得し設定
BACKEND_LB_DNS=(Backend APのロードバランサ)
#例：BACKEND_LB_DNS=internal-k8s-demoapp-backenda-52b135456b-62915499.ap-northeast-1.elb.amazonaws.com
export BACKEND_LB_DNS

#Deploymentの作成
envsubst < k8s-bff-deployment.yaml | kubectl apply -f -

#Podの起動確認しておくとよい
kubectl get pod -n demo-app

#Serviceの作成
kubectl apply -f k8s-bff-service.yaml

#Ingressの作成
#なお、マニフェストにALBへの接続元端末のCIDR制限が環境変数TERMINAL_CIDRで設定されている
envsubst < k8s-bff-ingress.yaml | kubectl apply -f -

#Ingressの作成確認
kubectl get ingress/bff-app-ingress -n demo-app
```

### 6. APの実行確認
* VPCのパブリックサブネット上にBationのEC2を起動
```sh
cd ..
aws cloudformation validate-template --template-body file://cfn-bastion-ec2.yaml
aws cloudformation create-stack --stack-name Demo-Bastion-Stack --template-body file://cfn-bastion-ec2.yaml
```
  * 必要に応じてキーペア名等のパラメータを指定
    * 「--parameters ParameterKey=KeyPairName,ParameterValue=myKeyPair」
  * BastionのEC2のアドレスは、CloudFormationの「Demo-Bastion-Stack」スタックの出力「BastionDNSName」のURLを参照    

* Backendアプリケーションの確認
  * EC2にSSHでログインし、以下のコマンドを「curl http://(Private ALBのDNS名)/backend/api/v1/users」を入力するとバックエンドサービスAPのJSONレスポンスが返却    
  * EC2、curlコマンドで動作確認

* BFFアプリケーションの確認
  * ブラウザで「http://(Public ALBのDNS名)/backend-for-frontend/index.html」を入力しフロントエンドAPの画面が表示される
    * サンプルAPは、ECS用のCloudFormationサンプルと共用しているので、画面に「Hello! AWS ECS sample!」と表示されるが気にしなくてよい。

* APログの確認
  * うまく動作しない場合、APログ等にエラーが出ていないか確認するとよい
  * CloudWatch Logsの以下のロググループ
    * /eks/logs/fluent-bit-cloudwatch
      * from-fluent-bit-kube.var.log.containers.bff-app-XXXX
      * from-fluent-bit-kube.var.log.containers.backend-app-XXXX
## CD環境

### 1. CodeBuildのIAMロールのEKS接続設定
* CodeBuildからEKS接続できるよう設定する
  * 参考
    * https://aws.amazon.com/jp/premiumsupport/knowledge-center/codebuild-eks-unauthorized-errors/
    * https://eksctl.io/usage/iam-identity-mappings/

```sh
#CloudFormation（EKS-IAMC-Stack）の出力表示で「CodeBuildRoleArn」の内容を確認
#CodeBuildのIAMロールのARNから「/service-role」を除いた文字列
#例： arn:aws:iam::999999999999:role/service-role/EKS-IAM-Stack-CodeBuildRole-NTM312K33QOU → arn:aws:iam::999999999999:role/EKS-IAM-Stack-CodeBuildRole-NTM312K33QOU
CODE_BUILD_ROLE_ARN=(CodeBuildのIAMロールのARN)
#例： EKS-IAM-Stack-CodeBuildRole-NTM312K33QOU
CODE_BUILD_ROLE=(CodeBuildのIAMロール名)
eksctl create iamidentitymapping --cluster $EKS_CLUSTER_NAME --region=$AWS_REGION --arn $CODE_BUILD_ROLE_ARN --group system:masters --username $CODE_BUILD_ROLE

eksctl get iamidentitymapping --cluster $EKS_CLUSTER_NAME --region=$AWS_REGION --arn $CODE_BUILD_ROLE_ARN
```

### 2. CD用CodeBuildの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codebuild-cd.yaml
aws cloudformation create-stack --stack-name BFF-CodeBuild-CD-Stack --template-body file://cfn-bff-codebuild-cd.yaml --parameters ParameterKey=ClusterName,ParameterValue=$EKS_CLUSTER_NAME ParameterKey=BackendLbDns,ParameterValue=$BACKEND_LB_DNS

aws cloudformation validate-template --template-body file://cfn-backend-codebuild-cd.yaml
aws cloudformation create-stack --stack-name Backend-CodeBuild-CD-Stack --template-body file://cfn-backend-codebuild-cd.yaml --parameters ParameterKey=ClusterName,ParameterValue=$EKS_CLUSTER_NAME
```

### 3. CodePipelineの作成
```sh
aws cloudformation validate-template --template-body file://cfn-bff-codepipeline.yaml
aws cloudformation create-stack --stack-name Bff-CodePipeline-Stack --template-body file://cfn-bff-codepipeline.yaml

aws cloudformation validate-template --template-body file://cfn-backend-codepipeline.yaml
aws cloudformation create-stack --stack-name Backend-CodePipeline-Stack --template-body file://cfn-backend-codepipeline.yaml
```

* Artifact用のS3バケット名を変えるには、それぞれのcfnスタック作成時のコマンドでパラメータを指定する
    * 「--parameters ParameterKey=ArtifactS3BucketName,ParameterValue=(バケット名)」

### 4. CodePipelineの確認
  * CodePipelineの作成後、パイプラインが自動実行されるので、デプロイ成功することを確認する
### 5. ソースコードの変更
  * 何らかのソースコードの変更を加えて、CodeCommitにプッシュする
  * CodePipelineのパイプラインが実行され、新しいAPがデプロイされることを確認する

## k8sリソースとEKSクラスタ等の削除
```sh
cd k8s
kubectl delete ingress bff-app-ingress -n demo-app
kubectl delete ingress backend-app-ingress -n demo-app
kubectl delete service bff-app-service -n demo-app
kubectl delete service backend-app-service -n demo-app
kubectl delete deployment bff-app -n demo-app
kubectl delete deployment backend-app -n demo-app

envsubst < k8s-otel-fargate-container-insights.yaml | kubectl delete -f -

envsubst < k8s-aws-logging-cloudwatch-configmap.yaml | kubectl delete -f -

helm uninstall aws-load-balancer-controller -n kube-system

eksctl delete cluster --name $EKS_CLUSTER_NAME
```

## その他のCloudFormationのスタック削除
* 上記のコマンドでk8sリソースとEKSクラスタ等の削除後、残ったCloudFormationのスタックを削除