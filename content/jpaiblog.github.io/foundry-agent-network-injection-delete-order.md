---
title: Foundry Agent Service のネットワーク分離構成を正しく削除する方法 (InUseSubnetCannotBeDeleted への対処)
date: 2026-09-10 00:00:00
categories:
- Azure OpenAI
- Microsoft Foundry
tags:
- Microsoft Foundry
- Foundry Agent Service
- Virtual Network
---
仮想ネットワーク挿入 (network injection) を構成した Microsoft Foundry リソースを削除した後、エージェント用サブネットが `InUseSubnetCannotBeDeleted` で削除できなくなることがあります。この記事では、その原因と正しい削除順序、同じ名前で再作成できない場合の対処、および設計段階での回避方法について、公開ドキュメントに基づいてご紹介します。

<!-- more -->
<br>

***
## 何が起きているのか

次のどちらかのエラーに遭遇した場合、原因は同じです。

サブネットまたは仮想ネットワークを削除しようとしたとき:

```
(InUseSubnetCannotBeDeleted) Subnet <agent-subnet> is in use by /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/virtualNetworks/<vnet-name>/subnets/<agent-subnet>/serviceAssociationLinks/legionservicelink and cannot be deleted.
```

同じサブネットに新しい Foundry リソースを再デプロイしようとしたとき (こちらは公式ドキュメントのトラブルシューティング ガイドにも掲載されています):

```
Subnet requires any of the following delegation(s) [Microsoft.App/environments] to reference service association link /subscriptions/<subscription-id>/resourceGroups/<resource-group>/providers/Microsoft.Network/virtualNetworks/<vnet-name>/subnets/<agent-subnet>/serviceAssociationLinks/legionservicelink.
```

仮想ネットワーク挿入では、エージェント用サブネットを `Microsoft.App/environments` に委任し、プラットフォーム側から **サービス関連付けリンク (service association link、以下 SAL)** が張られます。Foundry Agent Service の場合、この SAL の名前が `legionservicelink` です。SAL が残っている限り、そのサブネットは使用中として扱われ、削除も再利用もできません。

登場するリソースの関係は次のとおりです。**capability host** は、ネットワーク挿入を構成すると Foundry アカウントに自動的に作成される、エージェントの実行基盤です。利用者が明示的に作成しなくても存在するため、削除時に見落とされがちなポイントです。

```text
[Foundry 側]

  Foundry アカウント (Microsoft.CognitiveServices/accounts)
   |
   +-- プロジェクト
   |     +-- capability host (プロジェクト側)
   |
   +-- capability host (アカウント側)
         既定名: <accountName>@aml_aiagentservice
         |
         | networkInjections.scenario='agent' により、
         | 下のサブネット上に Azure Container Apps 環境を作成
         v

[ネットワーク側]

  仮想ネットワーク
   |
   +-- エージェント サブネット
         Microsoft.App/environments に委任
         |
         +-- serviceAssociationLinks/legionservicelink
               Container Apps 環境がこのサブネットを掴んでいる印。
               これが残っている間は削除も再利用もできない。
```

片付ける向きは、この図の上から下です。**上にある capability host を解放すると、下の SAL が外れ、サブネットを削除できるようになります。** 逆に、下のサブネットから先に消そうとしても失敗します。

サブネットの委任そのものについても、[Azure Virtual Network でのサブネット委任](https://learn.microsoft.com/ja-jp/azure/virtual-network/subnet-delegation-overview) に次の記載があります。

```
サブネットまたは仮想ネットワークを削除する前に、委任を削除する必要がある。
```

ここで重要なのは、**この SAL を利用者側から直接削除する手段がない** ことです。[Service Association Links の REST API リファレンス](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/service-association-links?view=rest-virtualnetwork-2025-07-01) を見ても、用意されている操作は `List` のみです。

```
Operations
List — Gets a list of service association links for a subnet.
```

つまり SAL は消しに行くものではなく、**リンク元である Foundry リソース側を正しく片付けることで、プラットフォームが非同期に解放する** ものです。

ご自身の状況に応じて、読むべき節は次のとおりです。

| 現在の状況 | 読む節 |
| --- | --- |
| これから環境を削除する | 「正しい削除順序」 |
| 削除したのにサブネットが削除できない | 「消去 (purge) を実行する」の後に「SAL の解放を確認する」 |
| 同じ名前で作り直せない | 「同じ名前で再作成できない場合」 |
| アカウントは残して作り直したい | 「capability host を削除する」 |
| これから構築する / 作り直しを繰り返す | 「設計で回避する」 |

参考ドキュメント:
- [Azure Virtual Network でのサブネット委任](https://learn.microsoft.com/ja-jp/azure/virtual-network/subnet-delegation-overview)
- [Service Association Links - REST API (Azure Virtual Networks)](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/service-association-links?view=rest-virtualnetwork-2025-07-01)

<br>

***
## 正しい削除順序

| 順序 | 操作 | 補足 |
| --- | --- | --- |
| 1 | プロジェクトの capability host を削除 | アカウント側より先に削除します |
| 2 | アカウントの capability host を削除 | 既定名は `<accountName>@aml_aiagentservice` |
| 3 | プロジェクトを削除 | 入れ子のプロジェクトが残っているとアカウントを削除できません |
| 4 | Foundry リソースを削除 (delete) | この時点ではまだソフト削除の状態です |
| 5 | Foundry リソースを消去 (purge) | ここまで実施して初めて後片付けが走ります |
| 6 | SAL の解放を待つ | サブネットから `legionservicelink` が消えるまで待ちます |
| 7 | サブネット / 仮想ネットワークを削除 | 最後に削除します |

[Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks) の「制限」にも、この順序が明記されています。

```
ネットワーク挿入の削除: Foundry リソースと Standard Agent をセキュリティで保護されたネットワーク セットアップで削除する場合は、
Foundry リソースと仮想ネットワークを最後に削除します。仮想ネットワークを削除する前に、Foundry リソースを削除して消去します。
```

順序を守らずに削除してしまった場合の復旧方法も、同ドキュメントのトラブルシューティング ガイドに記載があります。

```
1 つの解決策は、Azure ポータルで Foundry リソース ページに移動し、削除されたリソースの管理を選択することです。
そこから、エージェントがこの仮想ネットワークに関連付けられたリソースを消去します。
```

公開されている [クリーンアップ スクリプト (cleanup.ps1)](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup/cleanup.ps1) の README には、この順序が必要な理由が 3 点にまとめられています。

```
Deleting a Foundry private network deployment is not as simple as az group delete.

- Capability hosts must be deleted in order - project-level first, then account-level. Deleting in the wrong order can leave the
  account in a failed state.
- SALs block subnet reuse - subnets with active SALs cannot be re-delegated or deleted. SAL cleanup happens asynchronously after
  caphost deletion and can take up to 24 hours.
- Soft-deleted accounts block redeployment - Cognitive Services accounts are soft-deleted for 48 hours. A new deployment with the
  same name will fail unless the old account is purged.
```

なお、**環境ごと破棄する場合は capability host を手動で削除する必要はありません**。消去 (purge) が capability host の削除を連鎖させるためです。[Bicep サンプル](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup) の README に次の記載があります。

```
To completely remove an account, you must delete and purge the account. Simply deleting the account is not sufficient, you must purge
so that deletion of the associated capability host is triggered. The service will automatically handle the removal of the capability
host and any linked resources in the background.
```

用途別に整理すると次のようになります。

- **環境ごと破棄する**: 削除 (delete) と消去 (purge) を実施すれば、上記の手順 1 から 3 はプラットフォーム側で処理されます。
- **アカウントは残して capability host だけ作り直す**: 手順 1 と 2 を手動で実施します。次節のコマンド例、またはサンプルに同梱の [deleteCapHost.sh](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup/deleteCapHost.sh) を使います。
- **検証環境の作り直しを繰り返す**: [cleanup.ps1](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup/cleanup.ps1) が順序と待機を自動化してくれるため、手作業より確実です。実行時は `-DryRun` で検出結果を確認してから本実行します。

参考ドキュメント:
- [Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks)
- [Foundry Private Network Cleanup (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup)
- [Standard Agent Setup with E2E Network Isolation (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup)

<br>

***
## capability host を削除する

アカウントを残したまま capability host だけを作り直す場合の手順です。環境ごと破棄する場合は、この節を飛ばして次の消去 (purge) に進んで構いません。

capability host の一覧取得と削除は ARM の REST API で行います。アカウント側とプロジェクト側で別の操作グループとして公開されており、本記事では API バージョン `2025-06-01` を使用します (公開サンプルの [deleteCapHost.sh](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup/deleteCapHost.sh) と [cleanup.ps1](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup/cleanup.ps1) は `2025-04-01-preview` を使用しています)。

以下のコマンド例は `az rest` を使います。ログイン中の資格情報でトークンが自動付与されます。URL が長くなるため、先に共通部分を変数にしておきます。書き換えるのはこの 3 行だけです。

```bash
SUB=<subscription-id>
RG=<foundry-rg>
ACC=<foundry-account-name>

API=2025-06-01
BASE="https://management.azure.com/subscriptions/$SUB/resourceGroups/$RG/providers/Microsoft.CognitiveServices/accounts/$ACC"
```

URL に `?` が含まれるため、`--url` の値は必ずクォートで囲んでください。

**1. 削除対象を洗い出す**

```bash
# アカウント配下のプロジェクト一覧
az cognitiveservices account project list \
  --name "$ACC" --resource-group "$RG" --query "[].name" -o tsv

# プロジェクトの capability host
az rest --method GET --url "$BASE/projects/<project-name>/capabilityHosts?api-version=$API"

# アカウントの capability host
az rest --method GET --url "$BASE/capabilityHosts?api-version=$API"
```

アカウント側の応答には `properties.customerSubnet` が含まれ、どのサブネットに紐づいているかをその場で確認できます。後で SAL の解放を確認する対象はこのサブネットです。

```json
{
  "value": [
    {
      "name": "<account-caphost-name>",
      "type": "Microsoft.CognitiveServices/accounts/capabilityHosts",
      "properties": {
        "customerSubnet": "/subscriptions/<subscription-id>/resourceGroups/<vnet-rg>/providers/Microsoft.Network/virtualNetworks/<vnet-name>/subnets/<agent-subnet>",
        "provisioningState": "Succeeded"
      }
    }
  ]
}
```

**2. プロジェクトの capability host を削除する**

```bash
az rest --method DELETE --url "$BASE/projects/<project-name>/capabilityHosts/<project-caphost-name>?api-version=$API"
```

**3. アカウントの capability host を削除する**

プロジェクト側をすべて削除してから実行します。自動作成されたアカウント側の既定名は `<accountName>@aml_aiagentservice` です。

```bash
az rest --method DELETE --url "$BASE/capabilityHosts/<account-caphost-name>?api-version=$API"
```

**4. 応答を確認する**

削除要求に対して返るステータスと、その後にすべきことは次のとおりです。

| ステータス | 意味 | 次にすること |
| --- | --- | --- |
| `204 No Content` | 同期的に削除が完了しました。 | 次の手順に進みます。 |
| `202 Accepted` | 非同期で受け付けられました。削除はまだ進行中です。 | 進行状況を確認します (下記)。 |
| `404 Not Found` | 対象の capability host が存在しません。 | すでに削除済みです。次の手順に進んで問題ありません。 |
| `409 Conflict` | 競合しています。 | capability host が失敗状態になっている可能性があります。プロジェクト側が残っていないかを先に確認します。 |

`202` が返った場合の進行状況は、削除した capability host を GET して確認するのが簡単です。`provisioningState` は削除中は `Deleting` を返し、削除が完了すると対象が見つからない状態になります。

```bash
az rest --method GET --url "$BASE/capabilityHosts/<account-caphost-name>?api-version=$API" \
  --query "properties.provisioningState" -o tsv
```

`provisioningState` が取り得る値は `Succeeded` / `Failed` / `Canceled` / `Creating` / `Updating` / `Deleting` です。生の HTTP を扱う場合は、[deleteCapHost.sh](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup/deleteCapHost.sh) と同様に応答の `Azure-AsyncOperation` ヘッダーの URL をポーリングする方法もあります。

アカウント側の削除は時間がかかります。[cleanup.ps1](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup/cleanup.ps1) のポーリング上限にも目安がコメントされています。

```
$maxPolls = 60  # 30 min max (account caphosts can take 15-20 min)
```

参考ドキュメント:
- [Account Capability Hosts - REST API (Microsoft Foundry)](https://learn.microsoft.com/en-us/rest/api/microsoftfoundry/accountmanagement/account-capability-hosts)
- [Project Capability Hosts - REST API (Microsoft Foundry)](https://learn.microsoft.com/en-us/rest/api/microsoftfoundry/accountmanagement/project-capability-hosts)
- [az cognitiveservices account](https://learn.microsoft.com/ja-jp/cli/azure/cognitiveservices/account?view=azure-cli-latest)
- [Foundry Private Network Cleanup (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup)

<br>

***
## 消去 (purge) を実行する

削除 (delete) はリソースを即座に完全消滅させるものではなく、48 時間はソフト削除の状態で保持されます。[削除された Microsoft Foundry リソースを回復または消去する](https://learn.microsoft.com/ja-jp/azure/ai-services/recover-purge-resources) に次の記載があります。

```
リソースを削除すると、同じ名前のリソースを 48 時間作成することはできません。同じ名前のリソースを作成するには、削除したリソースを消去する必要があります。
```

課金の観点でも消去 (purge) は重要です。同ドキュメントには次の記載もあります。

```
削除されたリソースでプロビジョニングされたデプロイの料金は、リソースが消去されるまで続行されます。
```

**Azure ポータルの場合**: Foundry のリソース ページから「削除されたリソースの管理 (Manage deleted resources)」を開き、対象のサブスクリプションを選択して「消去 (Purge)」を実行します。

**Azure CLI の場合**: ポータルの一覧から探すよりも確実です。

```bash
# サブスクリプション内のソフト削除状態のリソースを一覧表示する
az cognitiveservices account list-deleted -o table

# 消去 (purge) を実行する
az cognitiveservices account purge \
  --name <foundry-account-name> \
  --resource-group <foundry-rg> \
  --location <region>

# 消去できたことを確認する (対象が見つからない状態になれば完了)
az cognitiveservices account show-deleted \
  --name <foundry-account-name> \
  --resource-group <foundry-rg> \
  --location <region>
```

REST API を直接呼び出す場合のエンドポイントも、同ドキュメントに記載されています。

```
DELETE https://management.azure.com/subscriptions/{subscriptionID}/providers/Microsoft.CognitiveServices/locations/{location}/resourceGroups/{resourceGroup}/deletedAccounts/{resourceName}?Api-Version=2021-04-30
```

消去は元に戻せない操作です。実行するとリソースに関連付けられたすべてのデータとキーが失われます。

**対象が「削除されたリソースの管理」に表示されない場合**

削除したはずのリソースが一覧に出てこない、というご相談をいただくことがあります。まず確認していただきたいのは、操作しているアカウントの **ロール割り当てスコープ** です。同ドキュメントに次の記載があります。

```
Contributor を使用してリソースを消去する場合、ロールはサブスクリプション レベルで割り当てる必要があります。
ロールの割り当てがリソースまたはリソース グループ レベルでのみ存在する場合は、消去機能にアクセスできません。
```

リソース グループ レベルで共同作成者 (Contributor) を持っていても、消去機能にはアクセスできません。**サブスクリプション スコープ** で Contributor もしくは Cognitive Services Contributor が割り当てられている必要があります。CLI では見えているのにポータルでは見えない場合は、この権限スコープか、ポータルで選択しているサブスクリプションのどちらかが原因であることがほとんどです。

参考ドキュメント:
- [削除された Microsoft Foundry リソースを回復または消去する](https://learn.microsoft.com/ja-jp/azure/ai-services/recover-purge-resources)
- [az cognitiveservices account purge](https://learn.microsoft.com/ja-jp/cli/azure/cognitiveservices/account?view=azure-cli-latest#az-cognitiveservices-account-purge)
- [az cognitiveservices account list-deleted](https://learn.microsoft.com/ja-jp/cli/azure/cognitiveservices/account?view=azure-cli-latest#az-cognitiveservices-account-list-deleted)

<br>

***
## SAL の解放を確認する

消去 (purge) を実行しても、SAL はその瞬間に消えるわけではありません。サブネットの `serviceAssociationLinks` プロパティを次のコマンドで確認し、出力が空になればサブネットの削除・再利用が可能な状態です。

```bash
az network vnet subnet show \
  --resource-group <vnet-rg> \
  --vnet-name <vnet-name> \
  --name <agent-subnet-name> \
  --query "serviceAssociationLinks[].name" -o tsv
```

待ち時間の目安は、[クリーンアップ スクリプト](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup) の README の記述が参考になります。

```
Waits for service association links to be removed from subnets (up to 20 min). SAL removal happens asynchronously after caphost
deletion. If SALs are still present after 20 minutes, the script warns you to check again later - backend cleanup can take up to 24 hours.
```

まず 20 分ほど待ち、それでも残っている場合はバックエンドのクリーンアップ待ちとして最大 24 時間を見込みます。20 分経過した時点で SAL が残っていても、それだけでは不具合とは限りません。

参考ドキュメント:
- [az network vnet subnet show](https://learn.microsoft.com/ja-jp/cli/azure/network/vnet/subnet?view=azure-cli-latest#az-network-vnet-subnet-show)
- [Service Association Links - REST API (Azure Virtual Networks)](https://learn.microsoft.com/en-us/rest/api/virtualnetwork/service-association-links?view=rest-virtualnetwork-2025-07-01)
- [Foundry Private Network Cleanup (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/deployment-tools/cleanup)

<br>

***
## 同じ名前で再作成できない場合

**「消去 (purge) してから 48 時間待てば同じ名前で作成できる」わけではありません。** 48 時間は削除 (delete) を起点としたソフト削除の保持期間であり、消去を実施した時点で名前の予約は解放されます。逆に、消去していなければ 48 時間の経過を待っても同じ名前では作成できません。前述の公式ドキュメントの記載も、待つことではなく消去することを条件としています。

それでも同じ名前での作成が失敗する場合は、作成が途中で失敗したときの capability host がアカウント側に残っている可能性があります。とくに `Microsoft.App` と `Microsoft.ContainerService` のリソース プロバイダーが未登録のまま作成した場合、アカウントの作成要求はいったん受け付けられた後に失敗します。[Bicep サンプル](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup) の README に次の警告があります。

```
Microsoft.App and Microsoft.ContainerService are mandatory for network injection. When networkInjections.scenario='agent' is used,
the capability host is created on the agent subnet's Azure Container Apps environment. If either provider is in NotRegistered, the
failure surfaces after the Foundry account resource is already accepted - the account reaches provisioningState: Failed ... and must
be cleaned up before retrying.
```

この状態で残ったリソースの解放も非同期に進むため、いつ同じ名前で作成できるようになるかを事前に見積もることはできません。**作成を急ぐ場合は、別のアカウント名で作り直すのが最短です。** 元の名前は、残存リソースが解放されてから改めてご利用ください。

リソース プロバイダーは、作成前に登録しておきます。登録が必要なプロバイダーの全一覧は、[Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks) の前提条件に記載されています。

```bash
az provider register --namespace 'Microsoft.App'
az provider register --namespace 'Microsoft.ContainerService'
```

**デプロイ前の preflight チェック**

同じ失敗を繰り返さないための [preflight チェック スクリプト (preflight-check.ps1)](https://github.com/microsoft-foundry/foundry-samples/blob/main/infrastructure/infrastructure-setup-bicep/deployment-tools/preflight/preflight-check.ps1) も公開されています。

```
ARM template deployments can fail 10-20 minutes in with opaque error messages. By that point resources may be partially created,
leaving your environment in an inconsistent state that requires manual cleanup.
```

検査項目には、この記事で扱ってきた内容がそのまま含まれています。

- リソース プロバイダーの登録状態 (`Microsoft.App` / `Microsoft.ContainerService` を含む)
- ソフト削除状態の Cognitive Services アカウント
- サブネットに残っている SAL
- エージェント サブネットの `Microsoft.App/environments` への委任

README の検査項目表には、本記事の主題に直結する項目が次のように記載されています。

```
Soft-deleted Cognitive Services accounts | Name collision failures when a new deployment tries to create an account with the same
name as a soft-deleted one

No Service Association Links (SALs) on subnets | Deployment will fail - the platform cannot inject into a subnet already owned by
another resource
```

作り直しの前に一度実行しておくと、10 分から 20 分かけてデプロイしたうえで失敗する、という事態を避けられます。

参考ドキュメント:
- [Preflight Check for Foundry Private Network Deployments (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/deployment-tools/preflight)
- [Standard Agent Setup with E2E Network Isolation (foundry-samples)](https://github.com/microsoft-foundry/foundry-samples/tree/main/infrastructure/infrastructure-setup-bicep/15-private-network-standard-agent-setup)
- [Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks)

<br>

***
## 設計で回避する

この事象は、いったん発生すると待つ以外の選択肢がほとんどありません。作り直しを繰り返す検証環境ほど、設計側で回避しておく価値があります。

**1. 仮想ネットワークを Foundry とは別のリソース グループに置く**

同一リソース グループである必要はないことが、[Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks) の FAQ に明記されています。

```
仮想ネットワークは Foundry リソースと同じリソース グループに存在する必要がありますか?

いいえ。仮想ネットワークと Foundry リソースは同じリソース グループ内に存在する必要はありませんが、同じリージョンに存在する必要があります。
```

仮想ネットワークを常設のリソース グループに分離しておけば、Foundry 側のリソース グループを一括削除しても仮想ネットワークには一切触れないため、`InUseSubnetCannotBeDeleted` でリソース グループの削除自体が失敗する事態を避けられます。

**2. エージェント サブネットを共有しない**

同ドキュメントの「制限」にも記載があります。

```
エージェント サブネットの排他性: エージェント サブネットを複数の Foundry リソースで共有することはできません。
各 Foundry リソースでは、専用エージェント サブネットを使用する必要があります。
```

仮想ネットワークは複数の Foundry リソースで共有できますが、エージェント サブネットは共有できません。作成時の最小要件は /27 以上、推奨サイズは /24 です。

**3. 予備のエージェント サブネットを用意しておく**

SAL の解放を待てない状況で再デプロイしたい場合は、あらかじめ用意しておいた別のサブネット (たとえば `snet-agent-01` と `snet-agent-02`) に切り替えるのが最も速い回避策です。前のサブネットの SAL はバックエンドで解放されるまで放置し、解放を確認してから片付けます。

なお、サブネットを再利用する場合、`Microsoft.App/environments` への委任はそのまま残して構いません。新しい Foundry リソースでも同じ委任が必要になるためです。委任の削除が必要になるのは、サブネット自体を削除する場合のみです。

参考ドキュメント:
- [Foundry Agent Service のプライベート ネットワークを設定する](https://learn.microsoft.com/ja-jp/azure/foundry/agents/how-to/virtual-networks)

<br>

***
`変更履歴`
`2026/09/10 created by KazuyaOnuki`

※ 本記事は 「[jpaiblog について](https://jpaiblog.github.io/blog/2020/01/01/about-jpaiblog/)」 の留意事項に準じます。  
※ 併せて 「[ホームページ](https://jpaiblog.github.io/blog/)」 および 「[記事一覧](https://jpaiblog.github.io/blog/archives/)」 もご参照いただければ幸いです。  
