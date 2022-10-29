+++
author = "Masataka Kashiwagi"
title = "Step FunctionsでのOpensearch Packageの更新は直列or並列？"
description = "複数の辞書更新を直列で行うか並列で行うか"
date = 2022-10-29T10:38:31+09:00
draft = false
share = true
showLicense = false
support = true
tags = ["AWS", "Dev"]
+++

## はじめに
OpenSearchのPackage更新，つまりユーザー辞書やシノニム辞書の更新をStep Functionsで行う場合に，直列で行うのが良いか並列で行うのが良いかメモ程度の備忘録として残しておきます．

結論としては，OpenSearchのPackage更新は同時に複数更新すると，関連付けでエラーが発生する可能性があるので，**直列**でパイプラインを組みのが良いと思いました．特にインスタンススペックが低いとドメインへの負荷でエラーになる可能性が高いと思います．

以下のようなエラーが発生する場合があります．

![関連付けエラー](../../img/opensearch-error-img1.png "関連付けエラー")

## Step Functionsによる辞書更新パイプライン
まず初めにOpenSearchのPackageをAPIで更新する手順を説明すると

1. `OpenSearch: UpdatePackage`
    - [update-package](https://docs.aws.amazon.com/ja_jp/cli/latest/reference/opensearch/update-package.html)
    - APIパラメータとして以下のJSONを渡す感じです
        ```json
        {
          "PackageID": "<OpenSearchのドメインに関連付けられたパッケージの内部ID>",
          "PackageSource": {
            "S3BucketName": "<パッケージが置かれているバケット名>",
            "S3Key": "<パッケージのファイル名>"
          }
        }
        ```
2. `OpenSearch: AssociatePackage`
    - [associate-package](https://docs.aws.amazon.com/ja_jp/cli/latest/reference/opensearch/associate-package.html)
    - APIパラメータとして以下のJSONを渡す感じです
        ```json
        {
          "DomainName": "<関連付けを行うOpenSearchのドメイン名>",
          "PackageID": "<OpenSearchのドメインに関連付けられたパッケージの内部ID>"
        }
        ```

この2つを実行するだけになります．

一方で，それぞれのAPIを実行すると処理が走るが，更新や関連付けには一定の時間が必要になります．そのため，`OpenSearch: ListDomainsForPackage`でドメインとパッケージの状態を確認し，`ACTIVE`状態になったら次の処理を実行する必要があります．

これ踏まえて，ユーザー辞書とシノニム辞書の2つを更新する処理を直列と並列それぞれ実装してみます．

### 直列でパイプラインを組んだ場合
ユーザー辞書を更新した後にシノニム辞書の更新を行うパイプラインになります．

<details>
<summary>直列パイプライン - DSL</summary>

```json
{
  "Comment": "全量データの同期と辞書更新のジョブ",
  "StartAt": "Update-Package-User-Dict",
  "States": {
    "Update-Package-User-Dict": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>",
        "PackageSource": {
          "S3BucketName": "<バケット名>",
          "S3Key": "<ファイル名>"
        }
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:updatePackage",
      "Comment": "パッケージのアップデート",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "NotifySlackFailure"
        }
      ],
      "Next": "List-Domains-For-Package-User1"
    },
    "List-Domains-For-Package-User1": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
      "Comment": "パッケージのステータス確認",
      "Next": "Choice-Package-Active-Check-User1"
    },
    "Choice-Package-Active-Check-User1": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ACTIVE",
          "Next": "Associate-Package-User"
        }
      ],
      "Default": "Wait-User1-10s",
      "Comment": "パッケージのステータスに応じた処理の分岐"
    },
    "Associate-Package-User": {
      "Type": "Task",
      "Parameters": {
        "DomainName": "<ドメイン名>",
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:associatePackage",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "NotifySlackFailure"
        }
      ],
      "Next": "List-Domains-For-Package-User2",
      "Comment": "パッケージの関連付けを行う"
    },
    "List-Domains-For-Package-User2": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
      "Comment": "パッケージのステータス確認",
      "Next": "Choice-Package-Active-Check-User2"
    },
    "Choice-Package-Active-Check-User2": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ACTIVE",
          "Next": "Pass"
        },
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ASSOCIATION_FAILED",
          "Next": "NotifySlackFailure"
        }
      ],
      "Default": "Wait-User2-60s"
    },
    "Pass": {
      "Type": "Pass",
      "Next": "Update-Package-Synonym-Dict"
    },
    "Update-Package-Synonym-Dict": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>",
        "PackageSource": {
          "S3BucketName": "<バケット名>",
          "S3Key": "<ファイル名>"
        }
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:updatePackage",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "NotifySlackFailure"
        }
      ],
      "Next": "List-Domains-For-Package-Synonym1",
      "Comment": "パッケージのアップデート"
    },
    "List-Domains-For-Package-Synonym1": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
      "Comment": "パッケージのステータス確認",
      "Next": "Choice-Package-Active-Check-Synonym1"
    },
    "Choice-Package-Active-Check-Synonym1": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ACTIVE",
          "Next": "Associate-Package-Synonym"
        }
      ],
      "Default": "Wait-Synonym1-10s"
    },
    "Associate-Package-Synonym": {
      "Type": "Task",
      "Parameters": {
        "DomainName": "<ドメイン名>",
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:associatePackage",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "Next": "NotifySlackFailure"
        }
      ],
      "Next": "List-Domains-For-Package-Synonym2",
      "Comment": "パッケージの関連付けを行う"
    },
    "List-Domains-For-Package-Synonym2": {
      "Type": "Task",
      "Parameters": {
        "PackageID": "<パッケージID>"
      },
      "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
      "Comment": "パッケージのステータス確認",
      "Next": "Choice-Package-Active-Check-Synonym2"
    },
    "Choice-Package-Active-Check-Synonym2": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ACTIVE",
          "Next": "Success-Associate-Package-Synonym"
        },
        {
          "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
          "StringEquals": "ASSOCIATION_FAILED",
          "Next": "NotifySlackFailure"
        }
      ],
      "Default": "Wait-Synonym2-60s",
      "Comment": "パッケージのステータスに応じた処理の分岐"
    },
    "Success-Associate-Package-Synonym": {
      "Type": "Succeed"
    },
    "Wait-Synonym2-60s": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "List-Domains-For-Package-Synonym2"
    },
    "Wait-Synonym1-10s": {
      "Type": "Wait",
      "Seconds": 10,
      "Comment": "待機",
      "Next": "List-Domains-For-Package-Synonym1"
    },
    "Wait-User2-60s": {
      "Type": "Wait",
      "Seconds": 60,
      "Next": "List-Domains-For-Package-User2"
    },
    "Wait-User1-10s": {
      "Type": "Wait",
      "Seconds": 10,
      "Comment": "待機",
      "Next": "List-Domains-For-Package-User1"
    },
    "NotifySlackFailure": {
      "Type": "Task",
      "Resource": "arn:aws:states:::lambda:invoke",
      "OutputPath": "$.Payload",
      "Parameters": {
        "Payload.$": "$",
        "FunctionName": "<LambdaのARN>"
      },
      "Retry": [
        {
          "ErrorEquals": [
            "Lambda.ServiceException",
            "Lambda.AWSLambdaException",
            "Lambda.SdkClientException"
          ],
          "IntervalSeconds": 2,
          "MaxAttempts": 6,
          "BackoffRate": 2
        }
      ],
      "Comment": "処理失敗のslack通知",
      "Next": "FailState"
    },
    "FailState": {
      "Type": "Fail",
      "Error": "Error",
      "Cause": "Error"
    }
  }
}
```

</details>

![直列パイプライン](../../img/stepfunctions-graph-img2.png "直列パイプライン")

### 並列でパイプラインを組んだ場合
ユーザー辞書とシノニム辞書の更新を同時に実行するパイプラインになります．

<details>
<summary>並列パイプライン - DSL</summary>

```json
{
  "Comment": "全量データの同期と辞書更新のジョブ",
  "StartAt": "Parallel",
  "States": {
    "Parallel": {
      "Type": "Parallel",
      "Branches": [
        {
          "StartAt": "Update-Package-User-Dict",
          "States": {
            "Update-Package-User-Dict": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>",
                "PackageSource": {
                  "S3BucketName": "<バケット名>",
                  "S3Key": "<ファイル名>"
                }
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:updatePackage",
              "Comment": "パッケージのアップデート",
              "Next": "List-Domains-For-Package-User1",
              "Catch": [
                {
                  "ErrorEquals": [
                    "States.ALL"
                  ],
                  "Next": "NotifySlackFailure1"
                }
              ]
            },
            "NotifySlackFailure1": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "OutputPath": "$.Payload",
              "Parameters": {
                "Payload.$": "$",
                "FunctionName": "<LambdaのARN>"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 6,
                  "BackoffRate": 2
                }
              ],
              "Comment": "処理失敗のslack通知",
              "Next": "FailState1"
            },
            "FailState1": {
              "Type": "Fail",
              "Error": "Error",
              "Cause": "Error"
            },
            "List-Domains-For-Package-User1": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
              "Comment": "パッケージのステータス確認",
              "Next": "Choice-Package-Active-Check-User1"
            },
            "Choice-Package-Active-Check-User1": {
              "Type": "Choice",
              "Choices": [
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ACTIVE",
                  "Next": "Associate-Package-User"
                }
              ],
              "Default": "Wait-User1-10s",
              "Comment": "パッケージのステータスに応じた処理の分岐"
            },
            "Associate-Package-User": {
              "Type": "Task",
              "Parameters": {
                "DomainName": "<ドメイン名>",
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:associatePackage",
              "Next": "List-Domains-For-Package-User2",
              "Catch": [
                {
                  "ErrorEquals": [
                    "States.ALL"
                  ],
                  "Next": "NotifySlackFailure1"
                }
              ]
            },
            "List-Domains-For-Package-User2": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
              "Next": "Choice-Package-Active-Check-User2",
              "Comment": "パッケージのステータス確認"
            },
            "Choice-Package-Active-Check-User2": {
              "Type": "Choice",
              "Choices": [
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ACTIVE",
                  "Next": "Success-Associate-Package-User"
                },
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ASSOCIATION_FAILED",
                  "Next": "NotifySlackFailure1"
                }
              ],
              "Default": "Wait-User2-60s",
              "Comment": "パッケージのステータスに応じた処理の分岐"
            },
            "Success-Associate-Package-User": {
              "Type": "Succeed"
            },
            "Wait-User2-60s": {
              "Type": "Wait",
              "Seconds": 60,
              "Next": "List-Domains-For-Package-User2"
            },
            "Wait-User1-10s": {
              "Type": "Wait",
              "Seconds": 10,
              "Comment": "待機",
              "Next": "List-Domains-For-Package-User1"
            }
          }
        },
        {
          "StartAt": "Update-Package-Synonym-Dict",
          "States": {
            "Update-Package-Synonym-Dict": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>",
                "PackageSource": {
                  "S3BucketName": "<バケット名>",
                  "S3Key": "<ファイル名>"
                }
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:updatePackage",
              "Comment": "パッケージのアップデート",
              "Next": "List-Domains-For-Package-Synonym1",
              "Catch": [
                {
                  "ErrorEquals": [
                    "States.ALL"
                  ],
                  "Next": "NotifySlackFailure2"
                }
              ]
            },
            "NotifySlackFailure2": {
              "Type": "Task",
              "Resource": "arn:aws:states:::lambda:invoke",
              "OutputPath": "$.Payload",
              "Parameters": {
                "Payload.$": "$",
                "FunctionName": "<LambdaのARN>"
              },
              "Retry": [
                {
                  "ErrorEquals": [
                    "Lambda.ServiceException",
                    "Lambda.AWSLambdaException",
                    "Lambda.SdkClientException"
                  ],
                  "IntervalSeconds": 2,
                  "MaxAttempts": 6,
                  "BackoffRate": 2
                }
              ],
              "Next": "FailState2",
              "Comment": "処理失敗のslack通知"
            },
            "FailState2": {
              "Type": "Fail",
              "Error": "Error",
              "Cause": "Error"
            },
            "List-Domains-For-Package-Synonym1": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
              "Comment": "パッケージのステータス確認",
              "Next": "Choice-Package-Active-Check-Synonym1"
            },
            "Choice-Package-Active-Check-Synonym1": {
              "Type": "Choice",
              "Choices": [
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ACTIVE",
                  "Next": "Associate-Package-Synonym"
                }
              ],
              "Default": "Wait-Synonym1-10s",
              "Comment": "パッケージのステータスに応じた処理の分岐"
            },
            "Associate-Package-Synonym": {
              "Type": "Task",
              "Parameters": {
                "DomainName": "<ドメイン名>",
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:associatePackage",
              "Next": "List-Domains-For-Package-Synonym2",
              "Catch": [
                {
                  "ErrorEquals": [
                    "States.ALL"
                  ],
                  "Next": "NotifySlackFailure2"
                }
              ]
            },
            "List-Domains-For-Package-Synonym2": {
              "Type": "Task",
              "Parameters": {
                "PackageID": "<パッケージID>"
              },
              "Resource": "arn:aws:states:::aws-sdk:opensearch:listDomainsForPackage",
              "Next": "Choice-Package-Active-Check-Synonym2",
              "Comment": "パッケージのステータス確認"
            },
            "Choice-Package-Active-Check-Synonym2": {
              "Type": "Choice",
              "Choices": [
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ACTIVE",
                  "Next": "Success-Associate-Package-Synonym"
                },
                {
                  "Variable": "$.DomainPackageDetailsList[0].DomainPackageStatus",
                  "StringEquals": "ASSOCIATION_FAILED",
                  "Next": "NotifySlackFailure2"
                }
              ],
              "Default": "Wait-Synonym2-60s",
              "Comment": "パッケージのステータスに応じた処理の分岐"
            },
            "Success-Associate-Package-Synonym": {
              "Type": "Succeed"
            },
            "Wait-Synonym2-60s": {
              "Type": "Wait",
              "Seconds": 60,
              "Next": "List-Domains-For-Package-Synonym2"
            },
            "Wait-Synonym1-10s": {
              "Type": "Wait",
              "Seconds": 10,
              "Comment": "待機",
              "Next": "List-Domains-For-Package-Synonym1"
            }
          }
        }
      ],
      "End": true
    }
  }
}
```

</details>

![並列パイプライン](../../img/stepfunctions-graph-img1.png "並列パイプライン")

パイプライン内では`OpenSearch: ListDomainsForPackage`でパッケージの状況を確認し，ACTIVE状態でない場合は数十秒の待機処理を入れてから再度確認する方法で更新を行っています．

辞書更新にかかる処理時間は5分以内ぐらいなので，より安全に実施できる直列で更新する方法に落ち着きました．

## おわりに
今回はOpenSearchの複数の辞書更新をStep Functionsで行う場合に，直列で更新処理を組むのが良いか，並列で更新処理を組むのが良いかをというところで，より安全に実行するという観点で直列を選択しました．

パイプラインが長くなってしまいますが，辞書更新用のStep Functionsを用意することで，このSFをデータ同期のパイプラインのステップの1つとして組み込めば，処理単位でモジュール化することができるので，修正やデバッグもやりやすくなると思います．
