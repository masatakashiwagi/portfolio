+++
author = "Masataka Kashiwagi"
title = "Github Actions と CodePipeline を使って Step Functions を動かす CI/CD を構築"
description = "Github Actions と CodePipeline で Step Functions を回す CI/CD を構築した話"
date = 2022-07-18T09:58:08+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["Dev", "AWS", "MLOps"]
+++

## はじめに

2ヶ月以上ご無沙汰になってしまいましたが，久しぶりのテックブログになります．今回はタイトルにもあるように，Github Actions と CodePipeline を使ってマージトリガーで Step Functions のパイプラインを動かす CI/CD を構築したお話になります．

今回のモチベーションは，3つほどあります．

1. ML 系のモデル学習パイプラインの構築と DryRun 的なものを毎回手動で実行するのがいいかげんめんどくさくなってきた
2. 人数が少ない ML チームだと担当者が対応できない場合に，属人化したものを代わりにオペレーションするのが大変でオペミスが発生する可能性がある
    - この辺りはドキュメント整備やチーム内での共有といった部分を整理しておく必要があるのは理解しつつ...
3. CI/CD 周りの設定含めてもう少し知識を付けて MLOps のレベルを上げたかった

ML 系プロジェクトにおいて，CI/CD 整備の優先度が低かったり，そもそもソフトウェアエンジニアに比べてこの辺りの経験や知識が豊富でないということで後回しにされがちですが，MLOps を考える上で CI/CD は大事なファクターの1つなのでしっかり取り組むべきだと思います（CI/CD の自動化は Google が定義している MLOps level 2: CI/CD pipeline automation に相当する部分）．また，少人数チームの場合は尚のこと，人手をかけられない+属人化を排除する意味でも取り入れていくのが良いかなと思います．

<!-- 継続的な取り組みとしてのContinuous IntegrationとContinuous Delivery & Deployは品質・運用・スピードを高めるためにも大事にすべき要素だと思います． -->

以下のリポジトリにソースコードなどを置いてあります．

<iframe class="hatenablogcard" style="width:100%;height:155px;max-width:680px;" title="teamaya/cicd-pipeline at main · masatakashiwagi/teamaya" src="https://hatenablog-parts.com/embed?url=https://github.com/masatakashiwagi/teamaya/tree/main/cicd-pipeline" width="300" height="150" frameborder="0" scrolling="no"></iframe>

## CI/CD パイプラインの構成

※ はじめに，CodePipeline の設定や IAM ロールの必要な権限は詳細に説明しないのでご了承くださいませ．公式ドキュメントや巷にある詳細な説明がされているブログをご覧下さい．

今回のゴールは <span class="marker_yellow">**Github のブランチマージから最終的に Step Functions のパイプラインを動かす**</span>ところまでになります．本来は Step Functions 内で ML のモデル学習を行うパイプラインを構築しますが，今回はサンプルとして SageMaker ProcessingJob を単発で動かすだけになります．

今回構築した CI/CD パイプラインは以下のような感じになります．

![CI/CD パイプライン](../../img/cicd-img1.png "CI/CD パイプライン")

ディレクトリ構成は以下になります．

```text
.
├── .github
│   └── workflows
│       └── sam-codepipeline.yaml
├── .gitignore
├── README.md
├── async-processing
├── cicd-pipeline
│   ├── README.md
│   ├── config
│   │   ├── buildspec.yml
│   │   └── dev-codepipeline-ver1.json
│   ├── container
│   │   ├── Dockerfile
│   │   ├── app
│   │   │   └── src
│   │   │       ├── hello.py
│   │   │       └── logger.py
│   │   ├── docker-compose.yml
│   │   ├── requirements.lock
│   │   └── requirements.txt
│   └── sam
│       ├── env
│       │   ├── dev
│       │   │   ├── samconfig.toml
│       │   │   └── template.yaml
│       │   └── prod
│       │       ├── samconfig.toml
│       │       └── template.yaml
│       └── statemachine
│           └── sample-ml-pipelines-ver1.asl.json
└── teamaya
```

流れを説明していくと，
- <span class="marker_yellow">**Github Actions パート**</span>
    <details>
    <summary>sam-codepipeline.yaml</summary>

    ```yml
    name: sam-stepfunctions-codepipeline

    on:
      pull_request:
        branches:
          - dev
          - main
        types: [opened]
        # paths:
        #   - 'cicd-pipeline/config/dev-codepipeline-ver1.json'
        #   - './github/workflows/sam-codepipeline.yaml'
      workflow_dispatch:

    jobs:
      Build-Deploy-SAM:
        name: Build & Deploy SAM for Pipeline
        runs-on: ubuntu-latest
        timeout-minutes: 5
        steps:
          - name: Checkout
            uses: actions/checkout@v2

          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: ap-northeast-1

          # samによるdev/prod環境のAWSリソース更新
          - name: Build SAM & Deploy SAM
            run: |
              if ${{ github.base_ref == 'dev' }}; then
                cd sam/env/dev
                sam build
                sam deploy --fail-on-empty-changeset --no-confirm-changeset
              elif ${{ github.base_ref == 'main' }}; then
                cd sam/env/prod
                sam build
                sam deploy --fail-on-empty-changeset --no-confirm-changeset
              else
                echo "Invalid branch name."
                exit 1
              fi

      Update-CodePipeline:
        name: Update Codepipeline for Step Functions
        runs-on: ubuntu-latest
        timeout-minutes: 5
        needs: Build-Deploy-SAM
        steps:
          - name: Checkout
            uses: actions/checkout@v2

          - name: Configure AWS credentials
            uses: aws-actions/configure-aws-credentials@v1
            with:
              aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
              aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
              aws-region: ap-northeast-1

          - name: Update Codepipeline
            run: aws codepipeline update-pipeline --pipeline file://config/codepipeline-ver1.json

    ```

    </details>

    1. Pull Reqest が Open したタイミングで Github Actions が走る
    2. AWS Serverless Application Model (SAM) の Build と Deploy を行う
        - マージ先のブランチに応じて，切り替わるようにしています．dev マージでは dev 用のリソースを作成し，main マージでは prod 用のリソースを作成します．
    3. AWS CLI を使って CodePipeline の action configuration を更新する
        - Step Functions の StateMachineArn が ML のモデル学習のバージョンによって変更されることがあるので，後続の CodePipeline で動かす対象の Step Functions を更新します．

- <span class="marker_yellow">**CodePipeline パート**</span>
    <details>
    <summary>buildspec.yml</summary>

    ```yml
    version: 0.2

    env:
      variables:
        ENV: "dev"
        REPOSITORY_NAME: <ECRのリポジトリ名>
        IMAGE_TAG: "latest"
        REGION: "ap-northeast-1"
      parameter-store:
        AWS_ACCOUNT_ID: "/CodeBuild/common/AWS_ACCOUNT_ID"
        AWS_ACCESS_KEY_ID: "/CodeBuild/common/AWS_ACCESS_KEY_ID"
        AWS_SECRET_ACCESS_KEY: "/CodeBuild/common/AWS_SECRET_ACCESS_KEY"
    phases:
      pre_build:
        commands:
          #  - echo Login to Docker
          #  - docker login --username $AWS_ACCESS_KEY_ID --password $AWS_SECRET_ACCESS_KEY
          - echo Set ECR repository URI
          - REPOSITORY_URI=$AWS_ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com
          - aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $REPOSITORY_URI
      build:
        commands:
          - echo Build started
          - echo Building the Docker Image
          - docker build -t $REPOSITORY_URI/$REPOSITORY_NAME:$IMAGE_TAG container
      post_build:
        commands:
          - echo Login to Amazon ECR
          - aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $REPOSITORY_URI
          - echo Pushing the Docker Image to ECR started
          - docker push $REPOSITORY_URI/$REPOSITORY_NAME:$IMAGE_TAG

    ```

    </details>

    1. PR が dev/main にマージされたタイミングで AWS で事前に設定している CodePipeline が走る
        - 事前に CodePipeline 上で組んでいるフローが動く
    2. CodeBuild が起動し，Docker Image の Build が行われ，Image を ECR に push します
    3. Step Functions が起動し，パイプラインが走る
        - ECR に登録している Image を使って，SageMaker ProcessingJob が動く

    細かい部分で言うと，`AWS_ACCOUNT_ID`, `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` などの機密情報は，パラメータストア（[AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/ja_jp/systems-manager/latest/userguide/systems-manager-parameter-store.html)）へ登録しておき，それを参照する形で使うようにしています．

### AWS Serverless Application Model
AWS Serverless Application Model (SAM) とは AWS でサーバーレスアプリケーションを簡単に構築することがフレームワークになります．CloudFormation の拡張で CloudFormation で利用できるリソースは SAM でも使用することができます．YAML もしくは JSON 形式のテンプレートを使って簡単に環境構築ができますし，CLI も提供されてます．詳しくは公式の「[AWS Serverless Application Model (AWS SAM) とは](https://docs.aws.amazon.com/ja_jp/serverless-application-model/latest/developerguide/what-is-sam.html)」を見て頂ければと思います．

設定するファイルは2つあります．

1. samconfig.toml
2. template.yaml

dev 環境の設定ファイルを見ていくと，

- <span class="marker_yellow">**samconfig.toml**</span>
    <details>
    <summary>samconfig.toml</summary>

    ```yml
    version = 0.1
    [default]
    [default.deploy.parameters]
    stack_name = "dev-sample-codepipeline"
    s3_bucket = "aws-sam-cli-managed-samclisourcebucket-dev-sample-codepipeline"
    s3_prefix = "dev-sample-codepipeline"
    region = "ap-northeast-1"
    confirm_changeset = true
    capabilities = "CAPABILITY_IAM"
    disable_rollback = true
    ```

    </details>

    - こちらのファイルは1つのファイルに dev と prod の両方の設定を実装することもできますが，今回は環境毎にファイルを分けています．（後述するテンプレートと一緒に管理する必要があるが，微妙に環境毎で変数が違う部分もあるのでこのファイルも分けて管理した方が良いかなと思い分けています）
    - `[default.deploy.parameters]` は `sam deploy` コマンドが実行された時に渡される引数になります
    - 注意としては，`s3_bucket` は事前に作成しておかないとデプロイした際に `S3 Bucket does not exist.` といったエラーが発生します．
        - S3 には，ビルドしたテンプレートファイルとリソースファイルが保存されます

- <span class="marker_yellow">**template.yaml**</span>
    <details>
    <summary>template.yaml</summary>

    ```yml
    AWSTemplateFormatVersion: "2010-09-09"
    Transform: AWS::Serverless-2016-10-31
    Description: >
      Create Resource
      - StepFunctions
      - EventBridge

    Parameters:
      EnvironmentVariable:
        Description: 環境変数
        Type: String
        Default: dev
      VersionVariable:
        Description: バージョン番号
        Type: String
        Default: ver1
      StepFunctionsExecutionRole:
        Description: Step Functionsの実行ロール
        Type: String
        Default: arn:aws:iam::<AWSアカウントID>:role/StepFunctionsExecutionRole
      SageMakerProcessingImage:
        Description: SageMakerのProcessingJobを動かすImage
        Type: String
        Default: <AWSアカウントID>.dkr.ecr.ap-northeast-1.amazonaws.com/<ECRのリポジトリ名>:latest

    Resources:
      # =======Step Functions for ProcessingJob======== #
      DevMLPipelinesStateMachine:
        Type: AWS::Serverless::StateMachine
        Properties:
          Name: !Sub ${EnvironmentVariable}-sample-ml-pipelines-${VersionVariable}
          DefinitionUri: ../../statemachine/sample-ml-pipelines-ver1.asl.json
          DefinitionSubstitutions:
            ProcessingJobRole: !Ref StepFunctionsExecutionRole
            ProcessingImage: !Ref SageMakerProcessingImage
            ProcessingEnvironment: !Ref EnvironmentVariable
          Role: !Ref StepFunctionsExecutionRole
          Events:
            Schedule:
              Type: Schedule
              Properties:
                Description: パイプライン用のスケジューラー
                Enabled: False
                Name: !Sub ${EnvironmentVariable}-sample-ml-pipelines-${VersionVariable}
                Schedule: "cron(0 16 * * ? *)"
    ```

    </details>

    - JSON ではなく，YAML 形式で書けるので良きですね！
    - Parameters ブロックでは，値を変数化できるので，共通設定や dev/prod で動的に変わる部分だったりを書いておくと使い回しやすいかなと思います．
    - Resources ブロックでは，Parameters ブロックで定義した変数を `!Ref` や `!Sub` で使うことができます．
        - `!Sub` は値の一部に変数を使用したい時に使うことができます．
    - Role の設定もできますが，今回は事前に設定しておいた IAM ロールを使用しています．
    - 今回は Step Functions だけを定義したので，定義ファイル `sample-ml-pipelines-ver1.asl.json` を次に見ていきます．

- <span class="marker_yellow">**sample-ml-pipelines-ver1.asl.json**</span>
    <details>
    <summary>sample-ml-pipelines-ver1.asl.json</summary>

    ```yml
    {
      "Comment": "Sample ML pipelines",
      "StartAt": "SageMaker-Hello-World",
      "States": {
        "SageMaker-Hello-World": {
          "Comment": "Hello Worldを出力する",
          "Type": "Task",
          "Resource": "arn:aws:states:::sagemaker:createProcessingJob.sync",
          "Parameters": {
            "RoleArn": "${ProcessingJobRole}",
            "ProcessingJobName.$": "States.Format('{}', $$.Execution.Name)",
            "AppSpecification": {
              "ImageUri": "${ProcessingImage}",
              "ContainerEntrypoint": [
                "python3",
                "/opt/program/src/hello.py"
              ]
            },
            "ProcessingResources": {
              "ClusterConfig": {
                "InstanceCount": 1,
                "InstanceType": "ml.t3.medium",
                "VolumeSizeInGB": 10
              }
            },
            "Environment": {
              "PYTHON_ENV": "${ProcessingEnvironment}"
            },
            "StoppingCondition": {
              "MaxRuntimeInSeconds": 86400
            }
          },
          "Catch": [
            {
              "ErrorEquals": [
                "States.ALL"
              ],
              "Next": "FailState"
            }
          ],
          "End": true
        },
        "FailState": {
          "Type": "Fail",
          "Cause": "Error",
          "Error": "Error"
        }
      }
    }
    ```

    </details>

    - JSON 形式で書かれた Step Functions の定義ファイルになります．
    - Hello World を出力するだけの内容になっていますが，それを SageMaker ProcessingJob で動かしています．今回は単純な処理ですが，ML モデルを構築するためのパイプラインをここに実装すれば，その内容が SAM で構築されます．
    - パイプラインを構築する際は，Step Functions の Workflow studio で直感的な GUI で簡単に作成できるので，それで作成した後に JSON の定義ファイルをDLすれば同じものを本番環境用にサクッと構築することができます．
    - ProcessingJob を動かす Image は CodeBuild 時に ECR に push したものを使っています．
    - また，`ProcessingJobName` は一意でないとエラーになるので，[Context オブジェクト](https://docs.aws.amazon.com/ja_jp/step-functions/latest/dg/input-output-contextobject.html)の `$$.Execution.Name` を使用しています．

### AWS CodePipeline

CodePipeline は複数のステージというものが用意されていて，それを繋ぎ合わせて一連のパイプラインを構築し CI/CD を自動化するものになります．詳しくは公式の「[AWS CodePipeline とは](https://docs.aws.amazon.com/ja_jp/codepipeline/latest/userguide/welcome.html)」を見て頂ければと思います．

今回の場合，Source → Build → Execute (Invoke) のような流れになっています．

- Source: Github のブランチマージトリガーで起動する
- Build: Docker ImageのBuild と ECR への push を行う
- Execute (Invoke): Step Functions を起動する

![CodePipeline](../../img/cp-img1.png "CodePipeline")

- 処理が正常に終了すると，Step Functions の実行結果と CloudWatch Logs のログは以下のようになります．

![Step Functions の結果](../../img/sf-img1.png "Step Functions の結果")

![CloudWatch Logs の結果](../../img/cwl-img1.png "CloudWatch Logs の結果")

- CodePipeline を動かす時の注意としては，権限周りのエラーがよく発生するので CodePipeline から何を動かす必要があるかをチェックして動かすアクションの権限を与えてやる必要があります．
  - 今回の場合:
    - CodePipeline では，CodeBuild と Step Functions を動かすためのポリシーが必要
    - CodeBuild では，ECR と SystemsManager を操作するためのポリシーが必要
    - Step Functions では，SageMaker の操作と ECR へのアクセスを行うポリシーが必要
  - エラー周りは参考に挙げたブログが役に立ちました．

## おわりに

今回は Github Actions と CodePipeline を使って Step Functions を動かす CI/CD パイプラインを構築したお話でした．

Step Functions の中身をMLモデル学習のパイプラインとすれば，Github Actions の PR が Open とブランチマージをトリガーとして，CodePipeline が走ることで，一連の流れを CI/CD で実現できることになります．

ここにテストを追加したり，例えば，ECS で ML-API が動いている場合には，Step Functions のパイプラインが正常終了した後に，承認プロセスを入れて ECS のタスク定義とサービスの更新を入れることで，デプロイまで持っていくことができるかなと思います！

もう少し発展させることでより良い開発体験が生み出すことができるのと，MLOps の成熟度も上がって運用の自動化も一歩前進すると考えています．

## 参考

- [CodeBuild・CodePipelineを使ってデリバリーパイプラインを導入（AWSでWebアプリ構築 part3）](https://rinoguchi.net/2021/08/aws-fullstack-sample-application-part3.html)
- [Solve - Policy has Prohibited field Principal AWS Error](https://bobbyhadz.com/blog/aws-policy-has-prohibited-field-principal)
- [Troubleshooting AWS CodeBuild](https://docs.aws.amazon.com/codebuild/latest/userguide/troubleshooting.html)
- [Troubleshooting CodePipeline](https://docs.aws.amazon.com/codepipeline/latest/userguide/troubleshooting.html)
