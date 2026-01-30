---
lab:
  topic: Advanced
  title: Azure Bicep テンプレートを使用したデプロイ
  description: Azure Bicep テンプレートを作成し、Azure Bicep モジュールの概念と Azure YAML パイプラインを使用して、そのテンプレートをモジュール化する方法を学習します。
---

# Azure Bicep テンプレートを使用したデプロイ

**推定時間**: 45 分

Azure Bicep テンプレートを作成し、Azure Bicep モジュールの概念を使用して、そのテンプレートをモジュール化する方法を学習します。 モジュールを使用するようにメイン デプロイ テンプレートを変更し、Azure YAML パイプラインを使用して、すべてのリソースを Azure にデプロイします。

このラボの所要時間は約 **30** 分です。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)
- **Azure サブスクリプション:** アクティブな Azure サブスクリプションが必要です。または新しいサブスクリプションを作成することもできます
- **Azure DevOps 組織:** お持ちでない場合は、[組織またはプロジェクト コレクションの作成](https://docs.microsoft.com/azure/devops/organizations/accounts/create-organization)に関する記事を参考にして作成してください
- **アカウントのアクセス許可:** 次のロールを持つ Microsoft アカウントまたは Microsoft Entra アカウントが必要です
  - Azure サブスクリプション内の所有者ロール
  - Microsoft Entra テナント内のグローバル管理者ロール
  - 詳細については、「[Azure portal を使用して Azure ロールの割り当てを一覧表示する](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)」および [Azure Active Directory での管理者ロールの表示と割り当て](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)に関する記事を参照してください

## Azure Bicep について

Azure Bicep は、宣言型の構文を使用して Azure リソースをデプロイするドメイン固有言語 (DSL) です。 Azure Resource Manager (ARM) テンプレートのすべての機能を維持しながら、JSON テンプレートと比較してより簡潔な構文を提供します。

Bicep の主な利点:

- **簡略化された構文:** JSON ARM テンプレートよりも読みやすく、書きやすい
- **モジュール設計:** 再利用可能なコンポーネントを作成するためのモジュールをサポート
- **厳密な型指定:** IntelliSense のサポートとコンパイル時の検証
- **状態管理なし:** 宣言型構文は、目的を達成する方法ではなく、何をしたいかを記述することを意味します

## チーム プロジェクトを作成して構成する

まず、このラボ用の Azure DevOps プロジェクトを作成します。

1. ブラウザーで、Azure DevOps 組織を開きます
1. **[新しいプロジェクト]** を選択します
1. プロジェクトに **eShopOnWeb** という名前を付けます
1. 他のフィールドは既定値のままにしておきます
1. **[作成]** を選択します

   ![[新しいプロジェクトの作成] パネルのスクリーンショット](media/create-project.png)

## eShopOnWeb Git リポジトリをインポートする

次に、アプリケーション コードとインフラストラクチャのテンプレートを含むサンプル リポジトリをインポートします。

1. Azure DevOps 組織で、**eShopOnWeb** プロジェクトを開きます
1. **[リポジトリ] > [ファイル]** を選択します
1. **[リポジトリのインポート]** を選択します
1. **インポート** を選択する
1. **[Git リポジトリのインポート]** ウィンドウに、次の URL を貼り付けます: `https://github.com/MicrosoftLearning/eShopOnWeb.git`
1. **インポート** を選択する

   ![リポジトリのインポート パネルのスクリーンショット](media/import-repo.png)

リポジトリは次のように構成されています。

- **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています
- **.devcontainer** フォルダーには、コンテナーを使用して開発するためのセットアップが含まれています
- **infra** フォルダーには、このラボで使用される Bicep&ARM のコードとしてのインフラストラクチャ テンプレートが含まれています
- **.github** フォルダーには、YAML GitHub ワークフローの定義が含まれています
- **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています

1. **[リポジトリ] > [ブランチ]** に移動します
1. **main** ブランチにカーソルを合わせ、右側の省略記号を選択します
1. **[既定のブランチとして設定]** を選択します

## Azure Bicep テンプレートを理解し、再利用可能なモジュールを使って簡略化する

Azure Bicep テンプレートを確認し、再利用可能なモジュールを使って簡略化します。

### Azure Bicep テンプレートを作成する

Azure DevOps エディターを使用して、Azure Bicep テンプレートを調べます。

1. Azure DevOps プロジェクトで、**[リポジトリ]**、**[ファイル]** の順に移動します
1. `infra` フォルダーを開き、`simple-windows-vm.bicep` ファイルを見つけます

   ![simple-windows-vm.bicep のファイル パスのスクリーンショット](media/browsebicepfile.png)

1. テンプレートを確認し、その構造をより深く理解します

 テンプレートには次のものが含まれます。

- 型、既定値、検証を含む**パラメーター**
- 計算値の**変数**
- 次の種類の**リソース**:
  - Microsoft.Storage/storageAccounts
  - Microsoft.Network/publicIPAddresses
  - Microsoft.Network/virtualNetworks
  - Microsoft.Network/networkInterfaces
  - Microsoft.Compute/virtualMachines

リソース定義のシンプルさと、テンプレート全体を通して明示的な `dependsOn` ではなく暗黙的にシンボリック名を参照できることに注意してください。

### ストレージ リソースの Bicep モジュールを作成する

ストレージ テンプレート モジュール **storage.bicep** を作成します。このモジュールは、ストレージ アカウントのみを作成し、メイン テンプレートによってインポートされます。

1. まず、メイン テンプレートからストレージ リソースを削除します。 **[編集]** ボタンを選択します

   ![パイプラインの [編集] ボタンのスクリーンショット](media/edit.png)

1. ストレージ リソースを削除します

   ```bicep
   resource storageAccount 'Microsoft.Storage/storageAccounts@2022-05-01' = {
     name: storageAccountName
     location: location
     sku: {
       name: 'Standard_LRS'
     }
     kind: 'Storage'
   }
   ```

1. 20 行目の `publicIPAllocationMethod` パラメーターの既定値を `Dynamic` から `Static` に変更します
1. 27 行目の `publicIpSku` パラメーターの既定値を `Basic` から `Standard` に変更します
1. ファイルをコミットします (まだこれで終わりではありません)

   ![ファイルの [コミット] ボタンのスクリーンショット](media/commit.png)

1. 次に、`Infra` フォルダーにマウスのポインターを置き、省略記号アイコンを選択します
1. **[新規]**、**[ファイル]** の順に選択します
1. 名前に「**storage.bicep**」と入力し、**[作成]** を選択します

   ![新しいファイルのメニューのスクリーンショット](media/newfile.png)

1. そのファイルに次のコード スニペットをコピーし、変更をコミットします

   ```bicep
   @description('Location for all resources.')
   param location string = resourceGroup().location

   @description('Name for the storage account.')
   param storageAccountName string

   resource storageAccount 'Microsoft.Storage/storageAccounts@2022-05-01' = {
     name: storageAccountName
     location: location
     sku: {
       name: 'Standard_LRS'
     }
     kind: 'Storage'
   }

   output storageURI string = storageAccount.properties.primaryEndpoints.blob
   ```

### テンプレート モジュールを使用するように simple-windows-vm テンプレートを変更する

作成したテンプレート モジュールを参照するように、`simple-windows-vm.bicep` テンプレートを変更します。

1. `simple-windows-vm.bicep` ファイルに戻り、**[編集]** を選択します
1. 変数の後に、次のコードを追加します

   ```bicep
   module storageModule './storage.bicep' = {
     name: 'linkedTemplate'
     params: {
       location: location
       storageAccountName: storageAccountName
     }
   }
   ```

1. 仮想マシン リソース内のストレージ アカウント BLOB URI への参照を、モジュールの出力を使用するように変更します
1. 仮想マシン リソースを見つけて、diagnosticsProfile セクションを次の内容に置き換えます

   ```bicep
   diagnosticsProfile: {
     bootDiagnostics: {
       enabled: true
       storageUri: storageModule.outputs.storageURI
     }
   }
   ```

1. メイン テンプレートで以下の詳細をレビューします:

   - メイン テンプレートのモジュールは、別のテンプレートにリンクするために使用されます
   - このモジュールには、依存関係の構成に使用される `storageModule` というシンボリック名が含まれています
   - テンプレート モジュールを使用する際に使用できるデプロイ モードは、**[増分]** のみです
   - テンプレート モジュールには相対パスを使用します
   - パラメーターは、メイン テンプレートからテンプレート モジュールに値を渡します

1. テンプレートをコミットします

## YAML パイプラインを使用してテンプレートを Azure にデプロイする

Azure DevOps YAML パイプラインを使用して、テンプレートを Azure 環境にデプロイします。

### YAML パイプラインによってリソースを Azure にデプロイする

1. **[パイプライン]** ハブで **[パイプライン]** に戻ります
1. **[最初のパイプラインの作成]** ウィンドウで、**[パイプラインの作成]** を選択します
1. **[コードの場所]** ペインで、**[Azure Repos Git (YAML)]** を選択します
1. **[リポジトリの選択]** ペインで、**[eShopOnWeb]** を選択します
1. **[パイプラインを構成する]** ペインで、下方向にスクロールして **[既存の Azure Pipelines YAML ファイル]** を選択します
1. **[既存の YAML ファイルを選択する]** ペインで、次のように指定します
   - [ブランチ]: **main**
   - パス: **.ado/eshoponweb-cd-windows-cm.yml**
1. **[続行]** を選択して、これらの設定を保存します
1. [変数] セクションで、次の手順を実行します
   - リソース グループの名前を選択します
   - 目的の場所を設定します
   - サービス接続の値を、既存のサービス接続のいずれかに置き換えます
1. 右上隅で **[保存して実行]** ボタンを選択します

   ![[保存して実行] ボタンのスクリーンショット](media/saveandrun.png)

1. コミット ダイアログが表示されたら、**[保存して実行]** をもう一度選択します
1. デプロイが完了するまで待ってから、結果を確認します

    ![YAML パイプラインを使った Azure へのリソースのデプロイに成功したことを示すスクリーンショット](media/deploy.png)

> **注**:メッセージが表示されたら、前に作成したサービス接続を使用するためのアクセス許可をパイプラインに付与することを忘れないでください。

## リソースをクリーンアップする

不要な料金が発生しないように、Azure portal で作成されたリソースを必ず削除してください。

1. Azure portal で、作成したリソース グループに移動します
1. **[リソース グループの削除]** を選択します。
1. リソース グループの名前を入力して、削除を確認します
1. **[削除]** を選択します

## まとめ

このラボでは、次の作業を行う方法について説明します。

- パラメーター、変数、リソースなど、**Azure Bicep テンプレートの構造を理解する**
- ストレージ リソース用の**再利用可能な Bicep モジュールを作成する**
- モジュールと更新された依存関係を使用するように**メイン テンプレートを変更する**
- YAML パイプラインを使用して**テンプレートを Azure にデプロイする**

Azure Bicep は、Azure のコードとしてのインフラストラクチャを定義してデプロイするための強力な方法を提供します。 モジュールを使用すると、再利用可能なコンポーネントを作成でき、インフラストラクチャ テンプレートの管理と整理がより容易になります。 Azure DevOps パイプラインと統合すると、アプリケーションと共にインフラストラクチャを自動的にデプロイできるようになります。
