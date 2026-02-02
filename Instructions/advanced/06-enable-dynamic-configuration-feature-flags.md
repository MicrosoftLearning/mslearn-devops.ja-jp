---
lab:
  topic: Advanced
  title: 動的構成と機能フラグを有効にする
  description: Azure App Configuration を使用して、動的な機能の切り替えによりアプリケーションの設定と機能フラグを一元的に管理する方法を学習します。
---

# 動的構成と機能フラグを有効にする

**推定時間**: 45 分

Azure App Configuration を使用して、アプリケーションの設定と機能フラグを一元的に管理する方法を学習します。 最新のクラウド アプリケーションが、アプリケーションの再デプロイを必要とせずに、一元化された構成管理と動的な機能の切り替えによってどのようなメリットを得られるかを探求します。

このラボの所要時間は約 **45** 分です。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://learn.microsoft.com/azure/devops/server/compatibility)
- **Azure サブスクリプション:** アクティブな Azure サブスクリプションが必要です。または新しいサブスクリプションを作成することもできます
- **Azure DevOps 組織:** お持ちでない場合は、[組織またはプロジェクト コレクションの作成](https://learn.microsoft.com/azure/devops/organizations/accounts/create-organization?view=azure-devops)に関する記事を参考にして作成してください
- **アカウントのアクセス許可:** Azure サブスクリプションの共同作成者または所有者ロールを持つ Microsoft アカウントまたは Microsoft Entra アカウントが必要です
  - 詳細については、「[Azure portal を使用して Azure ロールの割り当てを一覧表示する](https://learn.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)」および [Azure Active Directory での管理者ロールの表示と割り当て](https://learn.microsoft.com/azure/active-directory/roles/manage-roles-portal)に関する記事を参照してください

## Azure App Configuration について

[Azure App Configuration](https://learn.microsoft.com/azure/azure-app-configuration/overview) は、アプリケーションの設定と機能フラグを一元管理するサービスを提供します。 近年のプログラム、特にクラウドで実行されるものには、分散されたコンポーネントが多数存在するのが一般的です。 これらのコンポーネント全体に構成設定を分散させることは、トラブルシューティングすることの難しいエラーがアプリケーションのデプロイ中に発生する原因となります。 App Configuration を使用すると、1 か所にアプリケーションのすべての設定を格納して、それらのアクセスをセキュリティで保護することができます。

主な利点:

- アプリケーションの設定と機能フラグの**一元管理**
- アプリケーションの再起動を必要としない**動的構成**の変更
- 機能の制御されたロールアウトのための**機能フラグの管理**
- ロールバック シナリオのための**特定時点の構成**のスナップショット
- 機密性の高い設定のための Azure Key Vault との**統合**

## チーム プロジェクトを作成して構成する

まず、このラボ用の Azure DevOps プロジェクトを作成します。

1. ブラウザーで、Azure DevOps 組織を開きます
1. **[新しいプロジェクト]** を選択します
1. プロジェクトに **eShopOnWeb** という名前を付けます
1. **[作業項目プロセス]** ドロップダウンで **[スクラム]** を選択します
1. **[作成]** を選択します

## eShopOnWeb Git リポジトリをインポートする

次に、アプリケーション コードを含むサンプル リポジトリをインポートします。

1. Azure DevOps 組織で、**eShopOnWeb** プロジェクトを開きます
1. **[リポジトリ] > [ファイル]** を選択します
1. **インポート** を選択する
1. **[Git リポジトリのインポート]** ウィンドウに、次の URL を貼り付けます: `https://github.com/MicrosoftLearning/eShopOnWeb.git`
1. **インポート** を選択する

リポジトリは次のように構成されています。

- **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています
- **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています

1. **[リポジトリ] > [ブランチ]** に移動します
1. **main** ブランチにカーソルを合わせ、右側の省略記号を選択します
1. **[既定のブランチとして設定]** を選択します

## CI/CD パイプラインをインポートして実行する

CI/CD パイプラインをインポートし、eShopOnWeb アプリケーションをビルドしてデプロイします。 CI パイプラインは、アプリケーションをビルドしてテストを実行する準備が整っています。 CD パイプラインは、アプリケーションを Azure Web アプリにデプロイします。

### CI パイプラインをインポートして実行する

まず、[eshoponweb-ci.yml](https://github.com/MicrosoftLearning/eShopOnWeb/blob/main/.ado/eshoponweb-ci.yml) という CI パイプラインをインポートします。

1. **[パイプライン] - [パイプライン]** に移動します
1. **[パイプラインの作成]** (パイプラインがない場合) または **[新しいパイプライン]** (既に作成されたパイプラインがある場合) を選択します
1. **[Azure Repos Git (Yaml)]** を選びます
1. **eShopOnWeb** リポジトリを選びます
1. **[既存の Azure Pipelines の YAML ファイル]** を選択します
1. **main** ブランチと **/.ado/eshoponweb-ci.yml** ファイルを選択します
1. **[続行]** を選択します
1. **[実行]** ボタンを選択してパイプラインを実行します
1. パイプラインには、プロジェクト名に基づく名前が付けられます。 識別しやすくするために、**[名前の変更]** を行いましょう
1. **[パイプライン] > [パイプライン]** に移動し、先ほど作成したパイプラインを選択します
1. 省略記号と **[名前の変更または削除]** オプションを選択します
1. パイプラインに **eshoponweb-ci** という名前を付けて、**[保存]** を選択します

### サービス接続をセットアップする

Azure Resource Manager サービス接続を使用すると、パイプラインから Azure Key Vault などの Azure リソースに接続できます。 この接続により、パイプラインを使用して、毎回認証する必要なく、Azure App Service アプリなどの Azure リソースにデプロイできます。

1. Azure DevOps プロジェクトで、**[プロジェクト設定] > [サービス接続]** に移動します。
1. **[サービス接続の作成]**、**[Azure Resource Manager]**、**[次へ]** の順に選択します。
1. **[新しい Azure サービス接続]** ペインで、次の設定を確認し、**[保存]** を選択します
   - **ID の種類**: アプリの登録 (自動)
   - **Credential**:ワークロード ID フェデレーション
   - **[Scope level]\(スコープ レベル\)**:サブスクリプション
   - **[サブスクリプション]**: "このラボで使用しているサブスクリプションを選択します"__
   - **サービス接続名**: `azure subs`
   - **すべてのパイプラインにアクセス許可を付与**: 有効

### CD パイプラインをインポートして実行する

[eshoponweb-cd-webapp-code.yml](https://github.com/MicrosoftLearning/eShopOnWeb/blob/main/.ado/eshoponweb-cd-webapp-code.yml) という名前の CD パイプラインをインポートしてみましょう。

1. **[パイプライン] - [パイプライン]** に移動します
1. **[新しいパイプライン]** を選択します
1. **[Azure Repos Git (Yaml)]** を選びます
1. **eShopOnWeb** リポジトリを選びます
1. **[既存の Azure Pipelines の YAML ファイル]** を選択します
1. **main** ブランチと **/.ado/eshoponweb-cd-webapp-code.yml** ファイルを選択します
1. **[続行]** を選択します
1. YAML パイプライン定義で、変数セクションを次のように設定します
   - **resource-group**: リソース グループの名前 (例: **rg-az400-container-NAME**。NAME を置き換えてください)
   - **場所**: Azure リージョンの名前 (例: **southcentralus**)
   - **templateFile**: **'infra/webapp.bicep'**
   - **subscriptionid**: お使いの Azure サブスクリプション ID
   - **azureserviceconnection**: **'azure subs'**
   - **webappname**: Web アプリのグローバルに一意の名前 (例: **az400-webapp-NAME**)
   - **csmFile**: **'\$(Pipeline.Workspace)/eshoponweb-ci/Bicep/\$(templateFile)'**
   - **packageForLinux**: **'\$(Pipeline.Workspace)/eshoponweb-ci/Web.zip'**
1. **[保存して実行]** を選択します
1. パイプラインを開き、正常に実行されるまで待ちます

> **重要**: "この実行を続行するには、このパイプラインに、リソースにアクセスするためのアクセス許可が必要です" というメッセージが表示された場合は、[表示]、[許可] の順に選択し、さらにもう一度 [許可] を選択します。

1. 識別しやすくするために、パイプラインの名前を **eshoponweb-cd-webapp-code** に変更します

## Azure App Configuration サービスを作成する

アプリケーションの構成と機能フラグを一元的に保存する Azure App Configuration を作成します。

1. Azure portal で、**[App Configuration]** を検索し、**[App Configuration の作成]** を選択します
1. リソース グループを選択または作成します。
1. App Configuration リソースの場所を指定します
1. 構成ストアの名前を入力します (グローバルに一意である必要があります)
1. このラボでは、**[Standard]** 価格レベルを選択します (機能フラグに必要)
1. **[確認と作成]**、**[作成]** の順に選択します。
1. リソースが作成されたら、そのリソースに移動します

## App Configuration で構成キーを設定する

アプリケーションで使用する構成キーを追加します。

1. App Configuration サービスの左側のペインにある **[操作]** で、**[Configuration エクスプローラー]** を選択します
1. **[作成] > [キーと値]** を選択し、次の設定を追加します
   - **キー**: eShopOnWeb:Settings:ShowPipelineInfo
   - **値:** true
   - **ラベル**: 空のままにします
   - **コンテンツ タイプ**: 空のままにします
1. **[適用]** を選択し、プロセスを繰り返して、次のキーを追加します
   - **キー**: eShopOnWeb:Settings:ShowImageDevVersion, **値**: false
   - **キー**: eShopOnWeb:Settings:ShowImageProdVersion, **値**: true

## App Configuration で機能フラグを設定する

アプリケーションの機能を動的に制御する機能フラグを作成します。

1. App Configuration サービスの左側のペインで、**[機能マネージャー]** を選択します
1. **[作成]** を選択し、**[機能フラグ]** を追加します
   - **機能フラグを有効にする**: オン
   - **機能フラグ名**: ShoppingCart
   - **ラベル**: 空のままにします
   - **[説明]**: ショッピング カート機能を有効にする
1. **[適用]** を選択します
1. 繰り返して、別の機能フラグを作成します。
   - **機能フラグ名**: パイプライン
   - **[説明]**: パイプライン情報の表示を有効にする

## App Configuration を使用するようにアプリケーションを構成する

Azure App Configuration に接続するようにアプリケーションを変更します。

### App Configuration 接続文字列を追加する

1. Azure portal で、お使いの App Configuration リソースに移動します
1. 左側のメニューの **[設定]** で **[アクセスの設定]** を選択します
1. **[プライマリ]** 接続文字列をコピーします
1. Azure Web アプリ リソース (CD パイプラインによって作成されたもの) に移動します
1. 左側のメニューの **[設定]** で、**[環境変数]** を選択します
1. **[接続文字列]** タブを選択し、次の設定を追加します
   - **[名前]**: AppConfig
   - **値**: [App Configuration 接続文字列を貼り付けます]
   - **種類**: カスタム
   - **デプロイ スロットの設定**: オフのままにします
1. **[適用]** を選択し、さらにもう一度 **[適用]** を選択します

### アプリケーション コードを更新する

サンプル アプリケーションは、Azure App Configuration を使用するように既に構成されています。 主要な統合ポイントは次のとおりです。

1. **Program.cs** - アプリケーションは、App Configuration を使用するように構成されています

   ```csharp
   builder.Host.ConfigureAppConfiguration((hostingContext, config) =>
   {
       var settings = config.Build();
       config.AddAzureAppConfiguration(options =>
       {
           options.Connect(settings.GetConnectionString("AppConfig"))
                  .UseFeatureFlags();
       });
   });
   ```

1. **ビュー** - アプリケーションでは、機能フラグを使用してコンテンツを条件付きで表示します
   ```html
   <feature name="ShoppingCart">
     <div>Shopping cart feature is enabled!</div>
   </feature>
   ```

## 動的構成と機能フラグをテストする

設定を変更し、アプリケーションを再デプロイしないで、動的構成機能をテストします。

### 構成の変更をテストする

1. デプロイされた Web アプリケーションの URL に移動します
1. パイプライン情報の現在の表示を確認します
1. Azure portal で App Configuration に戻ります
1. **[Configuration エクスプローラー]** で、**[eShopOnWeb:Settings:ShowPipelineInfo]** キーを見つけます
1. その値を **true** から **false** に変更します
1. **[適用]** を選択します
1. Web アプリケーションを更新します (更新には最大 30 秒かかる場合があります)
1. パイプライン情報が表示されなくなったことに注意してください

### 機能フラグの変更をテストする

1. Web アプリケーションで、ショッピング カート機能が表示されているかどうかを確認します
1. Azure portal で App Configuration に戻ります
1. **[機能マネージャー]** で、**[ShoppingCart]** 機能フラグを見つけます
1. その状態を切り替えます (有効または無効)
1. **[適用]** を選択します
1. Web アプリケーションを更新します
1. フラグの状態に基づいてショッピング カート機能が表示または非表示になることに注意してください

## 高度な機能フラグのシナリオ

機能フラグは、次のようなより高度なシナリオをサポートします。

### 条件付きのアクティブ化

1. Azure portal で、お使いの App Configuration の **[機能マネージャー]** に移動します
1. **[パイプライン]** 機能フラグを選択します
1. **[フィルターの追加]** を選択します
1. **[ターゲット フィルター]** を選択します
1. 割合ベースのロールアウトを構成します
   - **既定の割合**:50
   - **グループ**:このデモでは空のままにします
1. **[適用]** を選択します

この構成により、ユーザーの 50% にランダムに機能が表示されます。

### 時間ベースのアクティブ化

1. **SpecialOffer** という名前の新しい機能フラグを作成します
1. **[時間枠]** フィルターを追加します
1. 機能をアクティブにする必要がある開始時刻と終了時刻を設定します
1. これにより、機能を時間に基づいて自動的に有効または無効にすることができます

## App Configuration の使用状況を監視する

アプリケーションで App Configuration がどのように使用されているかを監視できます。

1. Azure portal で、お使いの App Configuration リソースに移動します
1. 左側のメニューから **[監視]** を選択します
1. 表示する **[メトリック]** を選択します
   - **要求** - 構成要求の数
   - **調整された要求** - 調整された要求数
   - **ストレージ使用率** - 使用されているストレージの量

## リソースをクリーンアップする

不要な料金が発生しないように、Azure portal で作成されたリソースを必ず削除してください。

1. App Configuration と Web App のリソースを含むリソース グループを削除します
1. Azure portal で、リソース グループに移動します
1. **[リソース グループの削除]** を選択します。
1. リソース グループの名前を入力して、削除を確認します
1. **[削除]** を選択します

## まとめ

このラボでは、次の作業を行う方法について説明します。

- Azure App Configuration を使用して**動的構成を有効にする**
- 機能のロールアウトを制御するための**機能フラグを管理する**
- 一元化された構成を使用するように**アプリケーションを構成する**
- アプリケーションを再デプロイすることなく**構成の変更をテストする**
- 割合によるロールアウトや時間ベースのアクティブ化など、**高度な機能フラグのシナリオを実装する**

Azure App Configuration は、アプリケーションの設定と機能フラグを一元管理するための強力な方法を提供し、アプリケーションの再デプロイを必要とせずに、動的構成の変更や機能ロールアウトの制御を可能にします。 これにより、クラウド アプリケーションの柔軟性と保守性が向上します。
