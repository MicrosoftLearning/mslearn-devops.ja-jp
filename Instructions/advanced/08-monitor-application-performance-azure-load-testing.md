---
lab:
  topic: Advanced
  title: Azure Load Testing を使ってアプリケーション パフォーマンスを監視する
  description: Azure Load Testing を使用して、ライブ実行中の Web アプリケーションに対するパフォーマンス テストをさまざまな負荷シナリオでシミュレートする方法を学習します。
---

# Azure Load Testing を使ってアプリケーション パフォーマンスを監視する

**推定所要時間:** 60 分

Azure Load Testing を使用して、ライブ実行中の Web アプリケーションに対するパフォーマンス テストをさまざまな負荷シナリオでシミュレートする方法を学習します。 Azure Load Testing は、大規模な負荷を生成できるフル マネージドのロード テスト サービスであり、ロード テストを大規模に実行するために必要な複雑さとインフラストラクチャを抽象化します。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)
- **Azure DevOps 組織:** ない場合は作成します
- **Azure サブスクリプション:** 作成するか、既存のサブスクリプションを使用します
- Azure サブスクリプションに所有者ロールがあり、Microsoft Entra テナントに全体管理者ロールがある **Microsoft アカウントまたは Microsoft Entra アカウント**
- **Azure CLI**:[Azure CLI のインストール](https://docs.microsoft.com/cli/azure/install-azure-cli)

## Azure Load Testing について

Azure Load Testing は、大規模な負荷を生成できるフル マネージドのロード テスト サービスです。 サービスは、アプリケーションがどこにホストされているかにかかわらず、そのトラフィックをシミュレートします。 主な利点:

- インフラストラクチャの管理なしでの**スケーラブルな負荷の生成**
- テストを自動化するための **CI/CD パイプラインとの統合**
- **Apache JMeter テスト スクリプトのサポート**
- **リアルタイムの監視**と詳細な分析
- **複数の負荷パターン** (一定の負荷、ステップアップ、スパイク テスト)

## チーム プロジェクトを作成して構成する

まず、このラボ用の Azure DevOps プロジェクトを作成します。

1. ブラウザーで、Azure DevOps 組織を開きます
1. **[新しいプロジェクト]** を選択します
1. プロジェクトに **eShopOnWeb** という名前を付け、**[作業項目プロセス]** ドロップダウンで **[スクラム]** を選択します
1. **[作成]** を選択します

## eShopOnWeb Git リポジトリをインポートする

次に、アプリケーション コードを含むサンプル リポジトリをインポートします。

1. Azure DevOps 組織で、**eShopOnWeb** プロジェクトを開きます
1. **[リポジトリ] > [ファイル]**、**[インポート]** の順に選択します
1. **[Git リポジトリのインポート]** ウィンドウに、次の URL を貼り付けます: `https://github.com/MicrosoftLearning/eShopOnWeb.git`
1. **インポート** を選択する

リポジトリの構造は次のとおりです

- **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています
- **.devcontainer** フォルダーには、コンテナーを使用して開発するためのセットアップが含まれています
- **infra** フォルダーには、Bicep と ARM のコードとしてのインフラストラクチャのテンプレートが含まれています
- **.github** フォルダーには、YAML GitHub ワークフローの定義が含まれています
- **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています

1. **[リポジトリ] > [ブランチ]** に移動します
1. **main** ブランチにカーソルを合わせ、右側の省略記号を選択します
1. **[既定のブランチとして設定]** を選択します

## Azure リソースを作成する

Azure Cloud Shell を使用して、このラボに必要な Azure リソースを作成します。

1. [Azure portal](https://portal.azure.com) に移動し、サインインします
1. ツールバーで、検索テキスト ボックスのすぐ右側にある **[Cloud Shell]** アイコンを選択します
1. **Bash** または **PowerShell** のいずれかを選択するように求められたら、**[Bash]** を選択します

> **注**:Cloud Shell を初めて起動すると、"ストレージがマウントされていません" というメッセージが表示されます。その場合は、このラボで使用しているサブスクリプションを選択し、**[ストレージの作成]** を選択します。

1. **Bash** プロンプトから次のコマンドを実行してリソース グループを作成します (`<region>` を 'eastus' など、自分の場所に最も近い Azure リージョンの名前に置き換えます)

   ```bash
   RESOURCEGROUPNAME='az400m08l14-RG'
   LOCATION='<region>'
   az group create --name $RESOURCEGROUPNAME --location $LOCATION
   ```

1. Windows App Service プランを作成します

   ```bash
   SERVICEPLANNAME='az400l14-sp'
   az appservice plan create --resource-group $RESOURCEGROUPNAME \
       --name $SERVICEPLANNAME --sku B3
   ```

1. 一意の名前を指定して Web アプリを作成します

   ```bash
   WEBAPPNAME=az400eshoponweb$RANDOM$RANDOM
   az webapp create --resource-group $RESOURCEGROUPNAME --plan $SERVICEPLANNAME --name $WEBAPPNAME
   ```

> **注**:Web アプリの名前を記録します。 このラボで後ほど必要になります。

## YAML を使用して CI/CD パイプラインを構成する

Azure DevOps で YAML を使用して CI/CD パイプラインをコードとして構成します。

### YAML ビルドとデプロイ定義を追加する

1. **[パイプライン]** ハブで **[パイプライン]** ペインに移動します
1. **[新しいパイプライン]** (または、初めて作成する場合は [パイプラインの作成]) を選択します
1. **[コードの場所]** ペインで、**[Azure Repos Git (YAML)]** オプションを選択します
1. **[リポジトリの選択]** ペインで、**[eShopOnWeb]** を選択します
1. **[パイプラインの構成]** ペインで下方向にスクロールして、**[スターター パイプライン]** を選択します
1. [スターター パイプライン] のすべての行を**選択** し、削除します
1. 次のテンプレート パイプライン全体を**コピー**します

   ```yml
   #Template Pipeline for CI/CD
   # trigger:
   # - main

   resources:
     repositories:
       - repository: self
         trigger: none

   stages:
     - stage: Build
       displayName: Build .Net Core Solution
       jobs:
         - job: Build
           pool:
             vmImage: ubuntu-latest
           steps:
             - task: DotNetCoreCLI@2
               displayName: Restore
               inputs:
                 command: "restore"
                 projects: "**/*.sln"
                 feedsToUse: "select"

             - task: DotNetCoreCLI@2
               displayName: Build
               inputs:
                 command: "build"
                 projects: "**/*.sln"

             - task: DotNetCoreCLI@2
               displayName: Publish
               inputs:
                 command: "publish"
                 publishWebProjects: true
                 arguments: "-o $(Build.ArtifactStagingDirectory)"

             - task: PublishBuildArtifacts@1
               displayName: Publish Artifacts ADO - Website
               inputs:
                 pathToPublish: "$(Build.ArtifactStagingDirectory)"
                 artifactName: Website

     - stage: Deploy
       displayName: Deploy to an Azure Web App
       jobs:
         - job: Deploy
           pool:
             vmImage: "windows-2019"
           steps:
             - task: DownloadBuildArtifacts@1
               inputs:
                 buildType: "current"
                 downloadType: "single"
                 artifactName: "Website"
                 downloadPath: "$(Build.ArtifactStagingDirectory)"
   ```

1. 前のタスク レベルのインデントで、YAML 定義の末尾にある新しい行にカーソルを設定します
1. ポータルの右側にある **[アシスタントを表示する]** を選択します
1. タスクの一覧で、**[Azure App Service のデプロイ]** タスクを検索して選択します
1. **[Azure App Service のデプロイ]** ペインで次の設定を指定して、**[追加]** を選択します

    - **[Azure サブスクリプション]** ドロップダウン リストで、お使いの Azure サブスクリプションを選択します
    - **[App Service の種類]** で Windows 上の Web アプリが指定されていることを確認します
    - **[App Service の名前]** ドロップダウン リストで、先ほどデプロイした Web アプリの名前 (az400eshoponweb...) を選択します
    - **[パッケージまたはフォルダー]** テキスト ボックスで、既定値を `$(Build.ArtifactStagingDirectory)/**/Web.zip` に **[更新]** します
    - **[アプリケーションと構成の設定]** を展開し、[アプリの設定] テキスト ボックスで `-UseOnlyInMemoryDatabase true -ASPNETCORE_ENVIRONMENT Development` を追加します

1. **[追加]** ボタンを選択して、[アシスタント] ペインで設定を確認します

追加されたコードのスニペットは、次のようになります

```yml
- task: AzureRmWebAppDeployment@5
  inputs:
    ConnectionType: "AzureRM"
    azureSubscription: "SERVICE CONNECTION NAME"
    appType: "webApp"
    WebAppName: "az400eshoponWeb369825031"
    packageForLinux: "$(Build.ArtifactStagingDirectory)/**/Web.zip"
    AppSettings: "-UseOnlyInMemoryDatabase true -ASPNETCORE_ENVIRONMENT Development"
```

1. 更新を保存する前に、わかりやすい名前を付けます。 yaml エディター ウィンドウの上部で、**azure-pipelines-#.yml** の名前を **m08l14-pipeline.yml** に変更します
1. **[保存]** を選択し、**[保存]** ペインでもう一度 **[保存]** を選択して、メイン ブランチに変更を直接コミットします
1. **[パイプライン]** に移動し、もう一度 **[パイプライン]** を選択します。 **[すべて]** を選択して、すべてのパイプライン定義を開きます
1. パイプラインを選択し、**[パイプラインの実行]** を選択して実行することを確認し、もう一度 **[実行]** を選択して確認します
1. 2 つの異なるステージ (**Build .Net Core Solution** と **Deploy to Azure Web App**) に注意してください
1. パイプラインが正常に完了するまで待ちます

### デプロイされたサイトをレビューする

1. Azure portal に戻り、[Azure Web アプリ] ブレードに移動します
1. [Azure Web アプリ] ブレードで **[概要]**、**[参照]** の順に選択して、新しいブラウザー タブでサイトを開きます
1. デプロイされたサイトが期待どおりに読み込まれ、eShopOnWeb eコマース Web サイトが表示されることを確認します

## Azure Load Testing をデプロイしてセットアップする

Azure Load Testing リソースをデプロイし、さまざまな Load Testing シナリオを構成します。

> **重要**: Azure Load Testing は、**有料サービスです**。 ロード テストを実行するためのコストが発生します。 追加コストが発生しないように、ラボの完了後に必ずリソースをクリーンアップしてください。 詳細については、[Azure Storage の価格に関するページ](https://azure.microsoft.com/pricing/details/load-testing)を参照してください。

### Azure Load Testing をデプロイする

1. Azure Portal で、**[Azure リソースの作成]** に移動します
1. 検索フィールドに、「**Azure Load Testing**」と入力します
1. 検索結果から (Microsoft が公開している) **[Azure Load Testing]** を選択します
1. **[作成]** を選択してデプロイ プロセスを開始します
1. リソースのデプロイに必要な詳細を指定します
   - **サブスクリプション**: Azure サブスクリプションを選びます
   - **リソース グループ**: Web App Service のデプロイに使用したリソース グループを選択します
   - **名前**: `eShopOnWebLoadTesting`
   - **[リージョン]**: 自分のリージョンに近いリージョンを選択します

> **注**: Azure Load Testing サービスは、すべての Azure リージョンで使用できるわけではありません。

1. **[確認と作成]** を選択して、設定を検証します
1. **[作成]** を選択して確認し、Azure Load Testing リソースをデプロイします
1. 展開が正常に完了するまで待機します
1. **[リソースに移動]** を選択して、Azure Load Testing リソースの **eShopOnWebLoadTesting** に移動します

### Azure Load Testing テストを作成する

異なる負荷構成設定を使用して、さまざまな Azure Load Testing テストを作成します。

#### 最初のロード テストを作成する (仮想ユーザー)

1. **[eShopOnWebLoadTesting]** Azure Load Testing リソース ブレードから、**[テスト]** の下にある **[テスト]** に移動します
1. **[+ 作成]** メニュー オプションを選択し、**[URL ベースのテストの作成]** を選択します
1. 次のパラメーターと設定を完成します

   - **[Test URL] (テストの URL)**: デプロイした Azure App Service の URL (az400eshoponweb...azurewebsites.net) を **https:// **を含めて入力します
   - **Specify Load (負荷の指定)** : 仮想ユーザー
   - **Number of Virtual Users (仮想ユーザーの数)** : 50
   - **[Test Duration (minutes)] (テストの継続時間 (分))**: 5
   - **[Ramp-up time (minutes)] (ランプアップ時間 (分))**: 1

1. **[確認と作成]**、**[作成]** の順に選択します
1. テストは 5 分間実行されます

#### 2 番目のロード テストを作成する (1 秒あたりの要求数)

1. 上部のメニューで、**[作成]**、**[URL ベースのテストの作成]** を選択します
1. 次のパラメーターと設定を完成します

   - **[Test URL] (テストの URL)**: Azure App Service の URL を入力します (https:// を含めます)
   - **Specify Load (負荷の指定)** : 1 秒あたりの要求数 (RPS)
   - **Requests per second (RPS) (1 秒あたりの要求数 (RPS))** : 100
   - **Response time (milliseconds) (応答時間 (ミリ秒))** : 500
   - **[Test Duration (minutes)] (テストの継続時間 (分))**: 5
   - **[Ramp-up time (minutes)] (ランプアップ時間 (分))**: 1

1. **[確認と作成]**、**[作成]** の順に選択します
1. テストは約 5 分間実行されます

### Azure Load Testing の結果を検証する

両方のテストが完了したら、Azure Load Testing の TestRuns の結果を検証します。

1. **[Azure Load Testing]** から **[テスト]** に移動します
1. いずれかのテスト定義を選択し、テストの 1 つを**選択**して、詳細なビューを開きます
1. 表示された一覧から **TestRun_mm/dd/yy-hh:hh** を選択します
1. 詳細な **[TestRun]** ページから、シミュレーションの実際の結果を特定します

   - 負荷と合計要求数
   - Duration
   - 応答時間 (90 パーセンタイル応答時間)
   - 1 秒あたりの要求数のスループット

1. 下にあるダッシュボードの折れ線グラフとグラフ ビューを確認する
1. 時間をかけて、両方のシミュレート テストの**結果を相互に比較**し、より多くのユーザーが App Service のパフォーマンスに与える**影響を特定**します

## Azure Pipelines で CI/CD を使用してロード テストを自動化する

構成ファイルをエクスポートし、Azure Pipelines を構成して、ロード テストの自動化を CI/CD パイプラインに追加します。

### Azure DevOps サービス接続の詳細を特定する

Azure DevOps サービス接続に必要なアクセス許可を付与します。

1. **Azure DevOps ポータル**から、**[eShopOnWeb]** プロジェクトに移動します
1. 左下隅で **[プロジェクトの設定]** を選択します
1. **[パイプライン]** セクションで、**[サービス接続]** を選択します
1. お使いの Azure サブスクリプションの名前が付いたサービス接続に注意してください
1. **[サービス接続] を選びます**。 **[概要]** タブから **[詳細]** に移動し、**[サービス接続の管理]** を選択します
1. これで Azure portal にリダイレクトされ、[アクセス制御 (IAM)] ブレードにリソース グループの詳細が表示されます

### Azure Load Testing リソースへのアクセス許可を付与する

Azure Load Testing では、Azure RBAC を使用して、ロード テスト リソースで特定のアクティビティを実行するためのアクセス許可を付与します。 **ロード テスト共同作成者**ロールを Azure DevOps サービス接続に付与します。

1. **[+ 追加]**、**[ロールの割り当ての追加]** の順に選択します
1. **[ロール] タブ**の職務権限ロールの一覧で、**[ロード テスト共同作成者]** を選択します
1. **[メンバー]** タブで、**[メンバーの選択]** を選択し、自分のユーザー アカウントを見つけて選択し、**[選択]** を選択します
1. **[確認と割り当て]** タブで **[確認と割り当て]** を選択して、ロールの割り当てを追加します

Azure Pipelines ワークフロー内のサービス接続を使用して、Azure Load Testing リソースにアクセスできるようになりました。

### ロード テスト入力ファイルをエクスポートし、Azure Repos にインポートする

CI/CD ワークフローで Azure Load Testing を使用してロード テストを実行するには、ロード テスト構成設定と入力ファイルをソース管理リポジトリに追加する必要があります。

1. **Azure portal** で **[Azure Load Testing]** リソースに移動します
1. 左側のペインで **[テスト]** を選択してロード テストの一覧を表示し、**対象のテスト**を選択します
1. テスト実行の横にある**省略記号 (...)** を選択し、**[入力ファイルのダウンロード]** を選択します
1. ブラウザーで、ロード テストの入力ファイルを含む zip 形式のフォルダーがダウンロードされます
1. 任意の zip ツールを使用して、入力ファイルを抽出します。  フォルダーには、以下が含まれています。

   - _config.yaml_: ロード テストの YAML 構成ファイル
   - _quick_test.jmx_: JMeter テスト スクリプト

1. **Azure DevOps ポータル**、**[eShopOnWeb]** DevOps プロジェクトの順に移動します
1. **[リポジトリ]** を選びます。 ソース コード フォルダー構造内の **tests** サブフォルダーに注意してください
1. 省略記号 (...) に注意し、**[新規作成] > [フォルダー]** の順に選択します
1. フォルダー名として **jmeter**、ファイル名として **placeholder.txt** を指定します
1. **[コミット]** を選択して、プレースホルダー ファイルと jmeter フォルダーの作成を確認します
1. **[フォルダー構造]** から、新しく作成された **jmeter** サブフォルダーに移動します
1. **省略記号 (...)** を選択し、**[ファイルのアップロード]** を選択します
1. **[参照]** オプションを使用して、抽出した zip ファイルの場所に移動し、**config.yaml** と **quick_test.jmx** の両方を選択します
1. **[コミット]** を選択して、ソース管理へのファイルのアップロードを確認します
1. [リポジトリ] 内で、**/tests/jmeter** サブフォルダーに移動します
1. Load Testing の **config.yaml** ファイルを開きます。 **[編集]** を選択して、編集を許可します
1. **displayName** 属性と **testId** 属性を **ado_load_test** という値に置き換えます
1. **[コミット]** を選択して変更を保存します

### CI/CD ワークフロー YAML 定義ファイルを更新する

1. Azure DevOps Marketplace で [[Azure Load Testing タスク拡張機能]](https://marketplace.visualstudio.com/items?itemName=AzloadTest.AzloadTesting) を開き、**[無料で入手]** を選択します
1. Azure DevOps 組織を選択し、**[インストール]** を選択して拡張機能をインストールします
1. Azure DevOps ポータルから **[パイプライン]** に移動し、先ほど作成したパイプラインを選択します。 **編集**を選択する
1. YAML スクリプトで、**行 56** に移動し、Enter または Return キーを押して新しい空の行を追加します (デプロイ ステージの直前)
1. 57 行目で、[タスク アシスタント] を選択し、**[Azure Load Testing]** を検索します (必ず前のタスク レベルのインデントにカーソルを置いてください)。
1. 正しい設定でグラフィカル ペインを完成します

   - Azure サブスクリプション: Azure リソースを実行するサブスクリプションを選びます
   - ロード テスト ファイル: '$(Build.SourcesDirectory)/tests/jmeter/config.yaml'
   - ロード テスト リソース グループ: Azure Load Testing リソースを保持するリソース グループ
   - ロード テスト リソース名: `eShopOnWebLoadTesting`
   - ロード テストの実行名: ado_run
   - ロード テストの実行の説明: ADO からのロード テスト

1. **[追加]** を選択して、パラメーターを YAML スニペットとして挿入します
1. 必要に応じてインデントを修正します (スニペットを正しく配置するために 2 個のスペースまたはタブを追加します)

YAML コードは次のようになります

```yml
- task: AzureLoadTest@1
  inputs:
    azureSubscription: "AZURE DEMO SUBSCRIPTION"
    loadTestConfigFile: "$(Build.SourcesDirectory)/tests/jmeter/config.yaml"
    resourceGroup: "az400m08l14-RG"
    loadTestResource: "eShopOnWebLoadTesting"
    loadTestRunName: "ado_run"
    loadTestRunDescription: "load testing from ADO"
```

1. 挿入した YAML スニペットの下に、新しい空の行を追加します
1. Azure Load Testing タスクの結果を表示する発行タスクのスニペットを追加します

```yml
- publish: $(System.DefaultWorkingDirectory)/loadTest
  artifact: loadTestResults
```

1. 必要に応じて、インデントを修正します
1. **変更を保存する**
1. 保存したら、**[実行]** を選択してパイプラインをトリガーします
1. ブランチ (main) を確認し、**[実行]** を選択してパイプラインの実行を開始します
1. パイプラインの状態ページで、**[ビルド]** ステージを選択し、詳細ログを開きます
1. パイプラインが **AzureLoadTest** タスクに到着するまで待ちます
1. タスクが実行されている間に、Azure Portal で **Azure Load Testing** を参照して、パイプラインによって **adoloadtest1** という名前の新しい RunTest がどのように作成されるかを確認します
1. Azure DevOps の CI/CD パイプライン実行ビューに戻ります。**AzureLoadTest タスク**は正常に完了しています

### Load Testing パイプラインに失敗と成功の条件を追加する

ロード テストの失敗条件を使用して、アプリケーションが品質要件を満たさない場合にアラートが生成されます。

1. Azure DevOps から eShopOnWeb プロジェクトに移動し、**[リポジトリ]** を開きます
1. [リポジトリ] 内で、**/tests/jmeter** サブフォルダーに移動します
1. Load Testing の **config.yaml** ファイルを開きます。 **[編集]** を選択して、編集を許可します
1. `failureCriteria: []` が存在する場合は置き換えます。それ以外の場合は次のスニペットを追加します

   ```yaml
   failureCriteria:
     - avg(response_time_ms) > 300
     - percentage(error) > 50
   ```

1. **[コミット]** を選択して変更を保存し、もう一度 [コミット] を選択します
1. **[パイプライン]** に戻り、**eShopOnWeb** パイプラインをもう一度実行します
1. 数分後、**AzureLoadTest** タスクの **[失敗]** 状態で実行が完了します
1. パイプラインの詳細ログ ビューを開き、**AzureLoadtest** の詳細を検証します

出力には次のような内容が示されます

```text
-------------------Test Criteria ---------------
Results          :1 Pass 1 Fail

Criteria                     :Actual Value        Result
avg(response_time_ms) > 300                       1355.29               FAILED
percentage(error) > 50                                                  PASSED

##[error]TestResult: FAILED
```

1. 最後の行に **##[error]TestResult: FAILED** と書かれていることに注意してください。 平均応答時間が 300 を超える**FailCriteria** を定義したので、応答時間がこのしきい値を超えるとタスクは失敗のフラグが付けられます

> **注**:実際のシナリオでは、App Service のパフォーマンスを検証し、パフォーマンスが特定のしきい値を下回る場合は、追加のデプロイまたはスケーリング アクションをトリガーできます。

## まとめ

このラボでは、次の作業を行う方法について説明します。

- Azure Pipelines を使用して **Azure App Service に Web アプリをデプロイする**
- さまざまなテスト シナリオで **Azure Load Testing をデプロイして構成する**
- パフォーマンス検証を自動化するために**ロード テストを CI/CD パイプラインに統合する**
- アプリケーションの品質を確保するためにロード テストの**成功および失敗の基準を定義する**
- **ロード テストの結果を分析**してアプリケーションのパフォーマンス特性を理解する

Azure Load Testing は、CI/CD ワークフローとシームレスに統合されるパフォーマンス テスト用の強力なプラットフォームを提供し、運用環境にデプロイする前にアプリケーションが予想される負荷を確実に処理できるようにします。

## クリーンアップ

不要な料金が発生しないように、このラボで作成された Azure リソースを必ず削除してください。

1. Azure portal から **az400m08l14-RG** リソース グループを削除します
1. これにより、App Service、App Service プラン、Load Testing リソースなど、関連付けられているすべてのリソースが削除されます
