---
lab:
  topic: Intermediate
  title: CI/CD のために GitHub Actions を実装する
  description: サービス プリンシパル認証と GitHub 環境を使って Azure Web アプリをデプロイする GitHub Actions ワークフローを実装する方法を学びます。
---

# CI/CD のために GitHub Actions を実装する

このラボでは、Azure Web アプリをデプロイする GitHub Actions ワークフローを実装する方法を学習します。 GitHub リポジトリ、Azure サービス プリンシパル、自動デプロイ ワークフローを使って作業します。

学習内容:

- CI/CD 用に GitHub リポジトリをインポートして構成します。
- GitHub Actions 認証のための Azure サービス プリンシパルを作成します。
- CI/CD のために GitHub Actions ワークフローを実装します。
- 手動承認ゲートを使って GitHub 環境を設定します。
- GitHub Actions を使って Azure にアプリケーションをデプロイします。

このラボの所要時間は約 **40** 分です。

## 開始する前に

このラボを終えるには、次のものが必要です。

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)。
- **Azure サブスクリプション**:まだ Azure サブスクリプションをお持ちでない場合は、[Azure 無料アカウント](https://azure.microsoft.com/free) ページで無料アカウントにサインアップしてください。
- Azure サブスクリプションで**共同作成者**または**所有者**のロールを持つ Microsoft アカウントまたは Microsoft Entra アカウントを持っていることを確認します。 詳細については、[「Azure portal を使用して Azure ロールの割り当てを一覧表示する」](https://docs.microsoft.com/azure/role-based-access-control/role-assignments-list-portal)および[「Azure Active Directory で管理者ロールを表示して割当てる」](https://docs.microsoft.com/azure/active-directory/roles/manage-roles-portal)を参照してください。
- **GitHub アカウント**:このラボに使用できる GitHub アカウントをまだお持ちでない場合は、[新しい GitHub アカウントのサインアップ](https://github.com/join) ページにある手順に従って作成してください。

## eShopOnWeb を GitHub リポジトリにインポートする

このセクションでは、既存の [eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) リポジトリのコードをご自分の GitHub パブリック リポジトリにインポートします。

リポジトリは次のように編成されています。

- **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています。
- **.devcontainer** フォルダーには、コンテナーを使って開発するためのセットアップが含まれています (VS Code でローカルに、または GitHub Codespaces で)。
- **infra** フォルダーには、一部のラボ シナリオで使用される Bicep および ARM のコードとしてのインフラストラクチャ テンプレートが含まれています。
- **.github** フォルダーには、YAML GitHub ワークフローの定義が含まれています。
- **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています。

### GitHub でパブリック リポジトリを作成して eShopOnWeb をインポートする

このタスクでは、空のパブリック GitHub リポジトリを作成し、既存の [eShopOnWeb](https://github.com/MicrosoftLearning/eShopOnWeb) リポジトリをインポートします。

1. ラボ コンピューターから Web ブラウザーを起動して、[GitHub Web サイト](https://github.com/)に移動したら、ご自分のアカウントを使ってサインインし、**[New]** を選んで新しいリポジトリを作成します。

   ![新しいリポジトリを作成するボタンのスクリーンショット。](media/github-new.png)

1. **[Create a new repository]** ページで、**[Import a repository]** リンク (ページ タイトルの下) を選びます。

   > **注**: インポート Web サイトは、<https://github.com/new/import> で直接開くこともできます。

1. **[Import your project to GitHub]** ページ上で:

   | フィールド                              | 値                                             |
   | ---------------------------------- | ------------------------------------------------- |
   | ソース リポジトリの URL | <https://github.com/MicrosoftLearning/eShopOnWeb> |
   | Owner                              | アカウントの別名                                |
   | リポジトリ名                    | eShopOnWeb                                        |
   | プライバシー                            | **公開企業**                                        |

1. **[Begin Import]** を選び、リポジトリの準備ができるのを待ちます。

1. リポジトリ ページで **[Settings]** に移動し、**[Actions] > [General]** を選んで、**[Allow all actions and reusable workflows]** オプションを選びます。 **[保存]** を選択します。

   ![GitHub Actions を有効にするオプションのスクリーンショット。](media/enable-actions.png)

## GitHub リポジトリと Azure アクセスを設定する

このセクションでは、GitHub が GitHub Actions から Azure サブスクリプションにアクセスするのを認可するために Azure サービス プリンシパルを作成します。 また、Web サイトをビルド、テストし、Azure にデプロイする GitHub ワークフローも設定します。

### Azure サービス プリンシパルを作成し、GitHub シークレットとして保存する

このタスクでは、GitHub が目的のリソースをデプロイするために使用する Azure サービス プリンシパルを作成します。 代わりに、シークレットレス認証メカニズムとして [Azure の OpenID Connect](https://docs.github.com/actions/deployment/security-hardening-your-deployments/configuring-openid-connect-in-azure) を使用することもできます。

1. ラボ コンピューターのブラウザー ウィンドウで、Azure portal (`https://portal.azure.com`) を開きます。
1. ポータルで、**[リソース グループ]** を探して選びます。
1. **[+ 作成]** を選んで、演習用の新しいリソース グループを作成します。
1. **[リソース グループを作成します]** タブで、リソース グループに **rg-eshoponweb-NAME** という名前を付けます (NAME を一意の別名に置き換えてください)。 **[確認と作成] > [作成]** を選びます。
1. Azure Portal で、(検索バーの横にある) **Cloud Shell** を開きます。

   > **注**: Azure portal からストレージの作成を求められた場合は、**[ストレージ アカウントは不要です]** オプションを選び、ご自分のサブスクリプションを選んで **[適用]** ボタンを選びます

1. ターミナルが **Bash** モードで実行されていることを確認し、次のコマンドを実行します。**SUBSCRIPTION-ID** と **RESOURCE-GROUP** を独自の識別子に置き換えてください (どちらも [リソース グループ] の **[概要]** ページで確認できます)。

   ```bash
   az ad sp create-for-rbac --name GH-Action-eshoponweb --role contributor --scopes /subscriptions/SUBSCRIPTION-ID/resourceGroups/RESOURCE-GROUP --sdk-auth
   ```

   > **注**: これは必ず 1 行で入力するか貼り付けてください。

   > **注**: このコマンドにより、前に作成したリソース グループへの共同作成者アクセス権を持つサービス プリンシパルが作成されます。 これにより、GitHub Actions には、このリソース グループと対話するために必要なアクセス許可のみが付与されます (サブスクリプションの残りの部分は含まれません)

1. コマンドによって JSON オブジェクトが出力されます。後でそれをワークフローの GitHub シークレットとして使います。 その JSON をコピーしておきます。 この JSON には、Microsoft Entra ID (サービス プリンシパル) の名前で Azure に対する認証に使用される識別子が含まれます。

   ```JSON
       {
           "clientId": "<GUID>",
           "clientSecret": "<GUID>",
           "subscriptionId": "<GUID>",
           "tenantId": "<GUID>",
           (...)
       }
   ```

1. (既に登録済みの場合はスキップしてください) また、次のコマンドを実行して、後でデプロイする **Azure App Service** のリソース プロバイダーを登録する必要もあります。

   ```bash
   az provider register --namespace Microsoft.Web
   ```

1. ブラウザー ウィンドウで、**eShopOnWeb** GitHub リポジトリに戻ります。
1. リポジトリ ページで **[Settings]** に移動し、**[Secrets and variables] > [Actions]** を選びます。 **[New repository secret]** を選びます

   - 名前: **`AZURE_CREDENTIALS`**
   - シークレット: **前にコピーした JSON オブジェクトを貼り付けます** (GitHub では、[azure/login](https://github.com/Azure/login) アクションで使用される複数のシークレットを同じ名前で保持できます)

1. **[Add secret]** を選びます。 これで、GitHub Actions ではリポジトリ シークレットを使用して、サービス プリンシパルを参照できるようになりました。

### GitHub ワークフローを変更して実行する

このタスクでは、指定された GitHub ワークフローを変更し、それを実行して独自のサブスクリプションにソリューションをデプロイします。

1. ブラウザー ウィンドウで、**eShopOnWeb** GitHub リポジトリに戻ります。
1. リポジトリ ページで、 **[Code]** に移動し、**eShopOnWeb/.github/workflows/eshoponweb-cicd.yml** というファイルを開きます。 このワークフローは、指定された .NET 8 Web サイト コード用の CI/CD プロセスを定義します。
1. **on** セクションのコメントを解除します ("#"を削除します)。 このワークフローはメイン ブランチへのプッシュのたびにトリガーされ、手動トリガー ("workflow_dispatch") も実行できます。
1. **env** セクションで、次の変更を行います。
   - **RESOURCE-GROUP** 変数の **rg-eshoponweb-NAME** を置き換えます。 これは、前の手順で作成したのと同じリソース グループである必要があります。
   - (省略可能) **LOCATION** には、最も近い [Azure リージョン](https://azure.microsoft.com/explore/global-infrastructure/geographies)を選択できます。 たとえば、"eastus"、"eastasia"、"westus" などです。
   - **SUBSCRIPTION-ID** の **YOUR-SUBS-ID** を置き換えます。
   - **WEBAPP-NAME** の **eshoponweb-webapp-NAME** を一意のエイリアスに置き換えます。 これは、Azure App Service を使用してグローバルに一意の Web サイトを作成するために使用されます。
1. ワークフローを注意深くお読みください。理解に役立つコメントが記載されています。

1. 右上の **[Commit changes...]** を選びます。**[Commit changes]** は既定値 (メイン ブランチを変更する) のままです。 ワークフローが自動的に実行されます。

### GitHub ワークフローの実行を確認する

このタスクでは、GitHub ワークフローの実行を確認します。

1. ブラウザー ウィンドウで、**eShopOnWeb** GitHub リポジトリに戻ります。
1. リポジトリ ページで、 **[Actions]** に移動すると、実行前のワークフロー設定が表示されます。 それを選択してください。

   ![進行中の GitHub ワークフローのスクリーンショット。](media/gh-actions.png)

1. ワークフローが完了するまで待ちます。 **[Summary]** で、実行から保持される 2 つのワークフロー ジョブ、状態、成果物を確認できます。 各ジョブを選んでログを確認できます。

   ![正常に終了したワークフローのスクリーンショット。](media/gh-action-success.png)

1. ブラウザー ウィンドウで、Azure portal (`https://portal.azure.com`) に戻ります。 前に作成したリソース グループを開きます。 bicep テンプレートを使用した GitHub アクションによって、Azure App Service プランと App Service が作成されていることがわかります。 発行された Web サイトで App Service が開かれ **[参照]** が選ばれていることがわかります。

   ![WebApp 参照のスクリーンショット。](media/browse-webapp.png)

### (省略可能) GitHub 環境を使って手動承認の事前デプロイを追加する

このタスクでは、ワークフローのデプロイ ジョブで定義されているアクションを実行する前に、GitHub 環境を使って手動承認を求めます。

1. リポジトリ ページで、 **[Code]** に移動し、**eShopOnWeb/.github/workflows/eshoponweb-cicd.yml** というファイルを開きます。
1. **[デプロイ]** ジョブ セクションには、**Development** という**環境**への参照があります。 GitHub で使用される[環境](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)では、ターゲットの保護規則 (およびシークレット) が追加されます。

1. リポジトリ ページで **[Settings]** に移動し、**[Environments]** を開いて **[New environment]** を選びます。
1. それに **`Development`** という名前を指定し、**[Configure Environment]** を選びます。

   > **注**:**[Environments]** の一覧に **Development** という名前の環境が既に存在する場合は、環境名を選んでその構成を開きます。

1. **[Configure Development]** タブで、オプション **[Required Reviewers]** と、レビュー担当者としての GitHub アカウントをオンにします。 **[Save protection rules]** を選びます。
1. 次に、保護規則をテストしましょう。 リポジトリ ページで **[Actions]** に移動して、**[eShopOnWeb Build and Test]** ワークフローを選び、**[Run workflow] > [Run workflow]** を選んで手動で実行します。

   ![ワークフローの手動トリガーのスクリーンショット。](media/gh-manual-run.png)

1. 開始されたワークフローの実行を選び、**buildandtest** ジョブが完了するまで待ちます。 **deploy** ジョブに達すると、レビュー要求が表示されます。

1. **[Review deployments]** を選び、**[Development]** をオンにして、**[Approve and deploy]** を選びます。

   ![アクションの承認のスクリーンショット。](media/gh-approve.png)

1. ワークフローは **deploy** ジョブの実行に続き、完了します。

## リソースをクリーンアップする

ラボを終えたら、不要な料金が発生しないように Azure リソースをクリーンアップすることが重要です。

### Azure リソースを削除する

1. Azure portal (`https://portal.azure.com`) で **[リソース グループ]** セクションに移動します。
1. 作成した **rg-eshoponweb-NAME** リソース グループを見つけて選びます。
1. [リソース グループ] ページで、**[リソース グループの削除]** を選択します。
1. リソース グループ名を入力して削除を確認し、**[削除]** を選びます。
1. 削除プロセスが完了するまで待ちます。

### GitHub リソースをクリーンアップする (省略可能)

GitHub リポジトリをクリーンアップする場合:

1. GitHub で **eShopOnWeb** リポジトリに移動します。
1. **[Settings]** に移動し、**[Danger Zone]** まで下にスクロールします。
1. **[Delete this repository]** を選び、プロンプトに従って削除を確定します。

> **注:** リポジトリを削除すると、すべてのコード、issue、pull request、その他のリポジトリ データが完全に削除されます。

> **重要**:不要な料金が発生しないように、Azure リソースを必ず削除してください。 GitHub リポジトリは、ご自分のポートフォリオの一部として残すことができます。
