---
lab:
  topic: Advanced
  title: YAML を使用してパイプラインをコードとして構成する
  description: YAML を使用してビルド パイプラインとリリース パイプラインを定義し、ソース コードと同様に管理されるマークアップ ファイルを使用してパイプライン機能にアクセスできるようにする方法を学習します。
---

# YAML を使用してパイプラインをコードとして構成する

**推定時間**: 45 分

YAML を使用してビルド パイプラインとリリース パイプラインを定義する方法を学習します。これにより、ビジュアル デザイナーを使用する場合と同じパイプライン機能にアクセスできますが、この場合は他のソース ファイルと同様に管理できるマークアップ ファイルを使用します。 これには、Azure DevOps での YAML を使用したコードとしての CI/CD パイプラインの構成、継続的デリバリー ステージの追加、環境の承認の実装が含まれます。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)
- **Azure サブスクリプション:** アクティブな Azure サブスクリプションが必要です。または新しいサブスクリプションを作成することもできます
- **Azure DevOps 組織:** お持ちでない場合は、[組織またはプロジェクト コレクションの作成](https://docs.microsoft.com/azure/devops/organizations/accounts/create-organization)に関する記事を参考にして作成してください
- **アカウントのアクセス許可:** 次のロールを持つ Microsoft アカウントまたは Microsoft Entra アカウントが必要です
  - Azure サブスクリプション内の所有者ロール
  - Microsoft Entra テナント内のグローバル管理者ロール
  - 詳細については、「[Azure portal を使用して Azure ロールの割り当てを一覧表示する](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)」および [Azure Active Directory での管理者ロールの表示と割り当て](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)に関する記事を参照してください

## チーム プロジェクトを作成して構成する

まず、このラボ用の Azure DevOps プロジェクトを作成します。

1. ブラウザーで、Azure DevOps 組織を開きます
1. **[新しいプロジェクト]** を選択します
1. プロジェクトに **eShopOnWeb_MultiStageYAML** という名前を付けます
1. 他のフィールドは既定値のままにしておきます
1. **[作成]** を選択します

   ![[新しいプロジェクトの作成] パネルのスクリーンショット](media/create-project.png)

## eShopOnWeb Git リポジトリをインポートする

次に、アプリケーション コードを含むサンプル リポジトリをインポートします。

1. Azure DevOps 組織で、**[eShopOnWeb_MultiStageYAML]** プロジェクトを開きます
1. **[リポジトリ] > [ファイル]** を選択します
1. **[リポジトリのインポート]** を選択します
1. **インポート** を選択する
1. **[Git リポジトリのインポート]** ウィンドウに、次の URL を貼り付けます: `https://github.com/MicrosoftLearning/eShopOnWeb.git`
1. **[インポート]** を選択します
   ![リポジトリのインポートのパネルを示すスクリーンショット](media/import-repo.png)

リポジトリは次のように構成されています。

- **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています
- **.devcontainer** フォルダーには、コンテナーを使用して開発するためのセットアップが含まれています
- **infra** フォルダーには、Bicep と ARM のコードとしてのインフラストラクチャのテンプレートが含まれています
- **.github** フォルダーには、YAML GitHub ワークフローの定義が含まれています
- **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています

1. **[リポジトリ] > [ブランチ]** に移動します
1. **main** ブランチにカーソルを合わせ、右側の省略記号を選択します
1. **[既定のブランチとして設定]** を選択します

> **注**:main ブランチが既に既定のブランチである場合、このオプションは淡色表示されます。手順を続行します。

## Azure リソースを作成する

アプリケーションのデプロイ先となる Azure Web アプリを作成します。

1. ラボ コンピューターから、[Azure portal](https://portal.azure.com) に移動します
1. Azure サブスクリプションに所有者ロールがあるユーザー アカウントでサインインします
1. Azure portal のツール バーで、**[Cloud Shell]** アイコン (検索ボックスの右側にあります) を選択します
1. **Bash** または **PowerShell** のいずれかを選択するように求められたら、**[Bash]** を選択します

> **注**:Cloud Shell を初めて起動すると、[作業の開始] ポップアップが表示されます。その場合、[ストレージ アカウントは不要] とお使いのサブスクリプションを選択し、[適用] を選択します。

1. Bash プロンプトから、次のコマンドを実行してリソース グループを作成します (`<region>` を 'centralus'、'westeurope' などの任意の Azure リージョンに置き換えます)

   ```bash
   LOCATION='<region>'
   RESOURCEGROUPNAME='az400m03l07-RG'
   az group create --name $RESOURCEGROUPNAME --location $LOCATION
   ```

1. Windows App Service プランを作成します

   ```bash
   SERVICEPLANNAME='az400m03l07-sp1'
   az appservice plan create --resource-group $RESOURCEGROUPNAME --name $SERVICEPLANNAME --sku B3
   ```

> **注**:サブスクリプションが名前空間 'Microsoft.Web' を使用するように登録されていないというエラーが発生した場合は、`az provider register --namespace Microsoft.Web` を実行してから、このコマンドを再試行してください。

1. 一意の名前を指定して Web アプリを作成します

   ```bash
   WEBAPPNAME=eshoponWebYAML$RANDOM$RANDOM
   az webapp create --resource-group $RESOURCEGROUPNAME --plan $SERVICEPLANNAME --name $WEBAPPNAME
   ```

> **注**:Web アプリの名前を記録します。 これはこのラボで後ほど必要になります。

1. Azure Cloud Shell を閉じますが、Azure portal は開いたままにします

## YAML を使用して CI/CD パイプラインをコードとして構成する

次に、YAML 定義を使用して CI/CD パイプラインを構成します。

### YAML ビルド定義を追加する

1. Azure DevOps の **[パイプライン]** セクションに移動します
1. **[最初のパイプラインの作成]** ウィンドウで、**[パイプラインの作成]** を選択します
1. **[コードの場所]** ペインで、**[Azure Repos Git (YAML)]** を選択します
1. **[リポジトリの選択]** ペインで **[eShopOnWeb_MultiStageYAML]** を選択します
1. **[パイプラインを構成する]** ペインで、下方向にスクロールして **[既存の Azure Pipelines YAML ファイル]** を選択します
1. **[既存の YAML ファイルを選択する]** ペインで、次のように指定します
   - [ブランチ]: **main**
   - パス: **.ado/eshoponweb-ci.yml**
1. **[続行]** を選択して、これらの設定を保存します
1. **[パイプライン YAML をレビューする]** 画面で **[実行]** を選択して、ビルド パイプラインを開始します
1. ビルド パイプラインが正常に完了するまで待ちます

> **注**:警告やエラーなど、YAML ファイルの各タスクをレビューできます。

### YAML 定義に継続的デリバリーを追加する

次に、継続的デリバリーを追加して自動デプロイを有効にします。

1. [パイプラインの実行] ウィンドウで、右上隅にある省略記号 (⋯) を選択します
1. ドロップダウン メニューで、**[パイプラインの編集]** を選択します
1. ファイルの末尾 (56 行目) に移動し、**Enter** キーを押して新しい空の行を追加します
1. **57** 行目で、**リリース** ステージを定義する次の内容を追加します

   ```yaml
   - stage: Deploy
     displayName: Deploy to an Azure Web App
     jobs:
       - job: Deploy
         pool:
           vmImage: "windows-latest"
         steps:
   ```

1. YAML 定義の末尾にある新しい行にカーソルを置きます
1. 右側のタスクの一覧で、**[Azure App Service のデプロイ]** タスクを検索して選択します
1. **[Azure App Service のデプロイ]** ペインで次の設定を指定して、**[追加]** を選択します
   - **Azure サブスクリプション**: お使いの Azure サブスクリプションを選択して **[承認]** を選択し、メッセージが表示されたら認証します
   - **App Service の名前**: 先ほどデプロイした Web アプリを選択します
   - **パッケージまたはフォルダー**: `$(Build.ArtifactStagingDirectory)/**/Web.zip` に更新します
   - **[アプリケーションと構成の設定]** を開き、**[アプリの設定]** で `-UseOnlyInMemoryDatabase true -ASPNETCORE_ENVIRONMENT Development` を追加します

追加されたコードは次のようになります。

```yaml
- task: AzureRmWebAppDeployment@4
  inputs:
    ConnectionType: "AzureRM"
    azureSubscription: "AZURE SUBSCRIPTION HERE (b999999abc-1234-987a-a1e0-27fb2ea7f9f4)"
    appType: "webApp"
    WebAppName: "eshoponWebYAML369825031"
    packageForLinux: "$(Build.ArtifactStagingDirectory)/**/Web.zip"
    AppSettings: "-UseOnlyInMemoryDatabase true -ASPNETCORE_ENVIRONMENT Development"
```

1. タスクが **steps** タスクの子として一覧に表示されていることを確認します。 これと異なる場合は、追加されたタスクのすべての行を選択し、**Tab** キーを 2 回押して適切にインデントします。

> **注**:既定により、2 つのステージは独立して実行されます。 ビルド出力を 2 番目のステージで使用できるようにするには、デプロイ成果物をダウンロードする新しいタスクを追加する必要があります。

1. **deploy** ステージの **steps** ノードの下の最初の行にカーソルを置き、**Enter** キーを押して新しい空の行を追加します
1. **[タスク]** ペインで **[ビルド成果物のダウンロード]** を検索して選択します
1. 次のパラメーターを指定します。
    - ダウンロードする成果物の生成元: **現在のビルド**
    - ダウンロードの種類: **特定の成果物**
    - 成果物名: **Web サイト**
    - ダウンロード先ディレクトリ: **$(Build.ArtifactStagingDirectory)**
1. **[追加]** を選択します

追加されたコードは次のようになります。

```yaml
- task: DownloadBuildArtifacts@1
  inputs:
    buildType: "current"
    downloadType: "single"
    artifactName: "Website"
    downloadPath: "$(Build.ArtifactStagingDirectory)"
```

1. YAML のインデントがオフになっている場合は、**Tab** キーを 2 回押して適切にインデントします
1. **[検証と保存]** を選択し、もう一度 **[保存]** を選択して、変更をメイン ブランチにコミットします
1. Azure DevOps の左側のメニューから **[パイプライン]** に移動し、もう一度 **[パイプライン]** を選択します
1. **[eShopOnWeb_MultiStageYAML]** パイプラインを開いて、**[パイプラインの実行]** を選択します
1. 表示されるペインで **[実行]** を確認します
1. 2 つの異なるステージ (**Build .Net Core Solution** と **Deploy to Azure Web App**) に注意してください
1. パイプラインでビルド ステージが正常に完了するまで待ちます
1. デプロイ ステージを開始すると、**[アクセス許可が必要です]** というプロンプトが表示されます
1. **ビュー**の選択
1. **[確認の待機中]** ペインで、**[許可する]** を選択します
1. メッセージを確認し、**[許可する]** を選択して確認します
1. デプロイ ステージが正常に完了するまで待ちます

### デプロイされたサイトをレビューする

1. Azure portal に戻り、Azure Web アプリに移動します
1. [Azure Web アプリ] ブレードで **[概要]** を選択します
1. [概要] ブレードで **[参照]** をクリックし、新しいブラウザー タブでサイトを開きます
1. デプロイされたサイトが期待どおりに読み込まれ、eShopOnWeb eコマース Web サイトが表示されることを確認します

## 承認の環境設定を構成する

YAML パイプラインには、クラシック リリース パイプラインのようなリリースおよび品質ゲートはありません。ただし、**[環境]** を使って同様の機能を構成できます。

### パイプラインの環境を設定する

1. Azure DevOps プロジェクト **[eShopOnWeb_MultiStageYAML]** から、**[パイプライン]** に移動します
1. 左側の [パイプライン] メニューで、**[環境]** を選択します
1. **[環境の作成]** を選択します
1. **[新しい環境]** ペインで、**approvals** という名前を追加します
1. **[リソース]** で **[なし]** を選択します
1. **[作成]** を選択します
1. 環境が作成されたら、**[承認とチェック]** タブを選択します
1. **[最初のチェックを追加]** で、**[承認]** を選択します
1. 自分の Azure DevOps ユーザー アカウント名を **[承認者]** フィールドに追加します
1. **[作成]** を選択します

### 承認の YAML パイプラインを更新する

1. **[リポジトリ]** に移動し、**.ado** フォルダーを参照して、**eshoponweb-ci.yml** ファイルを選択します
1. **[編集]** ボタンを選択して編集モードに切り替えます
1. **デプロイ ジョブ**の開始 (60 行目の `- job: Deploy`) に移動します
1. すぐ下に新しい空の行を追加して、次のスニペットを追加します。

   ```yaml
   environment: approvals
   ```

1. 環境はデプロイ ステージの設定であるため、`- job: Deploy` を `- deployment: Deploy` に変更します
1. vmImage 行の下に新しい空の行を追加し、次の YAML スニペットを貼り付けます。

   ```yaml
   strategy:
     runOnce:
       deploy:
   ```

1. 残りのスニペット (以降の手順) を選択し、**Tab** キーを使用してインデントを修正します

結果の YAML は次のようになります。

```yaml
- stage: Deploy
  displayName: Deploy to an Azure Web App
  jobs:
    - deployment: Deploy
      environment: approvals
      pool:
        vmImage: "windows-latest"
      strategy:
        runOnce:
          deploy:
            steps:
              - task: DownloadBuildArtifacts@1
                inputs:
                  buildType: "current"
                  downloadType: "single"
                  artifactName: "Website"
                  downloadPath: "$(Build.ArtifactStagingDirectory)"
              - task: AzureRmWebAppDeployment@4
                inputs:
                  ConnectionType: "AzureRM"
                  azureSubscription: "AZURE SUBSCRIPTION HERE"
                  appType: "webApp"
                  WebAppName: "eshoponWebYAML369825031"
                  packageForLinux: "$(Build.ArtifactStagingDirectory)/**/Web.zip"
                  AppSettings: "-UseOnlyInMemoryDatabase true -ASPNETCORE_ENVIRONMENT Development"
```

1. **[コミット]** を選択し、**[コミット]** をもう一度選択して、変更を保存します
1. **[パイプライン] > [パイプライン]** に移動し、**[EshopOnWeb_MultiStageYAML]** パイプラインを開きます
1. **[パイプラインの実行]** を選択して、新しい実行をトリガーします
1. ビルド ステージは、前と同じように完了します
1. デプロイ ステージの場合、**[待機中 (1 件のチェックが処理中)]** と表示され、承認が必要であることを示すメッセージも表示されます
1. **[確認]** ボタンを選択します
1. **[確認の待機中]** ペインで、**[承認する]** を選択します
1. これにより、デプロイ ステージは続行され、Azure Web アプリに正常にデプロイできます

> **注**:この例では承認のみを使用しましたが、Azure Monitor や REST API などの他のチェックも同様の方法で使用できます。

## リソースをクリーンアップする

不要な料金が発生しないように、Azure portal で作成されたリソースを必ず削除してください。

1. Azure portal で、**[az400m03l07-RG]** リソース グループに移動します
1. **[リソース グループの削除]** を選択します。
1. リソース グループの名前を入力して、削除を確認します
1. **[削除]** を選択します

## まとめ

このラボでは、Azure DevOps で YAML を使用して CI/CD パイプラインをコードとして構成しました。 以下の方法を学習しました。

- YAML を使用してビルド パイプラインとリリース パイプラインを定義する
- 継続的デリバリー ステージを YAML パイプラインに追加する
- デプロイ ステージの環境と承認を構成する
- YAML パイプラインを使用して Azure App Service にアプリケーションをデプロイする

YAML パイプラインは、他のソース ファイルと同様に管理でき、CI/CD プロセスのバージョン管理とコード レビュー機能が向上するため、多くのチームはこのパイプラインを好みます。
