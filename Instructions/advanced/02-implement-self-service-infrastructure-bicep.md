---
lab:
  topic: Advanced
  title: Bicep を使用してセルフサービス インフラストラクチャを実装する
  description: セルフサービス プラットフォーム エンジニアリングのシナリオ向けに、Bicep を使用してコードとしてのインフラストラクチャを実装する方法を学習します。
---

# Bicep を使用してセルフサービス インフラストラクチャを実装する

このラボでは、Bicep を使用してコードとしてのインフラストラクチャ (IaC) を実装し、開発チーム向けのセルフサービス インフラストラクチャ機能を作成する方法を学習します。 プラットフォーム エンジニアリングのシナリオ向けに、Azure リソースのデプロイ、ポリシーとタグによるガバナンスの適用、自動スケーリングの実装を行う Bicep テンプレートを作成します。

学習内容:

- コードとしてのインフラストラクチャ用の Bicep を設定して使用する
- Azure リソースを定義してデプロイする Bicep テンプレートを作成する
- Bicep を使用して、SQL Database バックエンドを含む Azure App Service をデプロイする
- タグと Azure ポリシーを使用してガバナンスを適用する
- Bicep を使用して自動スケーリングを実装する

このラボの所要時間は約 **30** 分です。

## 開始する前に

このラボを終えるには、次のものが必要です。

- 少なくとも共同作成者レベルのアクセス権を持っている Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- Azure Cloud Shell、または Azure CLI がインストールされているローカル環境へのアクセス。
- Azure のサービスとコードとしてのインフラストラクチャの概念に関する基本的な知識。
- JSON と宣言構文に関する基本的な知識。

> **注:**  このラボでは Azure Cloud Shell を使用します。これには Azure portal からアクセスできるため、ローカル インストールは必要ありません。 ただし、Azure CLI と Visual Studio Code がインストールされたローカル環境を使用して、このラボを完了することもできます。

## Azure リソース用の Bicep テンプレートを作成してデプロイする

この演習では、SQL Database バックエンドを含む Azure App Service をデプロイする Bicep テンプレートを作成します。 このテンプレートには、セキュリティのベスト プラクティスとガバナンス コントロールが含まれます。

### Bicep 環境を設定する

1. Web ブラウザーを開始し、Azure portal (`https://portal.azure.com`) にアクセスします。
1. 認証を求められたら、Azure アカウントを使用してサインインします。
1. Azure portal で、上部のメニュー バーにある **[Cloud Shell]** アイコンを選択します。
1. **Bash** と **PowerShell** のどちらかを選択するように求められたら、**[Bash]** を選択します。
1. Cloud Shell を初めて使用する場合は、プロンプトに従ってストレージ アカウントを作成します。
1. Cloud Shell の準備ができたら、次のコマンドを実行して Bicep が使用可能であることを確認します。

   ```bash
   az bicep version
   ```

   > **注:**  Bicep は、Azure Cloud Shell にプレインストールされています。 Bicep CLI のバージョンを示す出力が表示されます。

1. 最新バージョンを使用していることを確認するには、次のコマンドを実行します。

   ```bash
   az bicep upgrade
   ```

### Bicep テンプレートを作成する

1. Cloud Shell で、新しい Bicep ファイルを作成します。

   ```bash
   code main.bicep
   ```

1. 以下の Bicep コードをコピーし、次のファイルに貼り付けます。

   ```bicep
   param location string = 'eastus'
   param appServicePlanName string = 'asp-bicep-${uniqueString(resourceGroup().id)}'
   param webAppName string = 'app-bicep-${uniqueString(resourceGroup().id)}'
   param storageAccountName string = 'stbicep${uniqueString(resourceGroup().id)}'
   param sqlServerName string = 'sql-bicep-${uniqueString(resourceGroup().id)}'
   param sqlDatabaseName string = 'bicepdb'
   param sqlAdminUser string
   @secure()
   param sqlAdminPassword string

   targetScope = 'resourceGroup'

   // Storage Account
   resource storage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
     name: storageAccountName
     location: location
     sku: {
       name: 'Standard_LRS'
     }
     kind: 'StorageV2'
     properties: {
       accessTier: 'Hot'
       allowBlobPublicAccess: false
       minimumTlsVersion: 'TLS1_2'
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // App Service Plan
   resource appPlan 'Microsoft.Web/serverfarms@2023-01-01' = {
     name: appServicePlanName
     location: location
     sku: {
       name: 'F1'
       tier: 'Free'
     }
     properties: {
       reserved: false
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // Web App
   resource webApp 'Microsoft.Web/sites@2023-01-01' = {
     name: webAppName
     location: location
     properties: {
       serverFarmId: appPlan.id
       httpsOnly: true
       siteConfig: {
         minTlsVersion: '1.2'
         ftpsState: 'FtpsOnly'
       }
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // SQL Server
   resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
     name: sqlServerName
     location: location
     properties: {
       administratorLogin: sqlAdminUser
       administratorLoginPassword: sqlAdminPassword
       version: '12.0'
       minimalTlsVersion: '1.2'
       publicNetworkAccess: 'Enabled'
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // SQL Database
   resource sqlDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
     name: sqlDatabaseName
     location: location
     parent: sqlServer
     properties: {
       collation: 'SQL_Latin1_General_CP1_CI_AS'
       maxSizeBytes: 2147483648
       requestedServiceObjectiveName: 'Basic'
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // SQL Server Firewall Rule (Allow Azure Services)
   resource sqlFirewallRule 'Microsoft.Sql/servers/firewallRules@2023-05-01-preview' = {
     name: 'AllowAzureServices'
     parent: sqlServer
     properties: {
       startIpAddress: '0.0.0.0'
       endIpAddress: '0.0.0.0'
     }
   }

   // Outputs
   output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
   output storageAccountName string = storage.name
   output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
   ```

   > **注:**  このテンプレートは、`uniqueString()` 関数を使用してリソースの一意の名前を生成し、名前付けの競合が原因でデプロイが失敗しないようにします。

1. **Ctrl + S** キーを押してファイルを保存し、**Ctrl + Q** キーを押してコード エディターを閉じます。

### Bicep テンプレートをデプロイする

1. デプロイ用のリソース グループを作成します。

   ```bash
   az group create --name rg-bicep-infrastructure --location eastus
   ```

1. デプロイする前に Bicep テンプレートを検証します。

   ```bash
   az deployment group validate \
     --resource-group rg-bicep-infrastructure \
     --template-file main.bicep \
     --parameters sqlAdminUser='bicepAdmin' sqlAdminPassword='SecureP@ssw0rd123!'
   ```

   > **注:**  パスワードを、Azure SQL Database の要件を満たす強力なパスワードに置き換えます。

1. テンプレートをリソース グループにデプロイします。

   ```bash
   az deployment group create \
     --resource-group rg-bicep-infrastructure \
     --template-file main.bicep \
     --parameters sqlAdminUser='bicepAdmin' sqlAdminPassword='SecureP@ssw0rd123!'
   ```

   > **注:**  デプロイが完了するまでに 3 から 5 分かかる場合があります。

1. デプロイが完了するまで待ちます。 デプロイの成功を示す確認メッセージが表示されます。

1. デプロイの出力を表示します。

   ```bash
   az deployment group show \
     --resource-group rg-bicep-infrastructure \
     --name main \
     --query properties.outputs
   ```

1. リソース グループ内のリソースを一覧表示して、リソースが正常に作成されたことを確認します。

   ```bash
   az resource list --resource-group rg-bicep-infrastructure --output table
   ```

Bicep テンプレートを使用して、SQL Database バックエンドで Azure App Service を正常にデプロイしました。 このテンプレートには、HTTPS の適用、TLS バージョンの要件、ガバナンスのための適切なタグ付けなどのセキュリティのベスト プラクティスが含まれています。

## タグと Azure ポリシーを使用してガバナンスを適用する

セルフサービス インフラストラクチャ環境では、コンプライアンス、コスト管理、適切なリソース管理を確保するためにガバナンスを適用することが不可欠です。 Azure のタグとポリシーは、これを実現するための 2 つの重要なメカニズムです。 この演習では、インフラストラクチャ デプロイ用のガバナンス コントロールを実装します。

### リソースのタグ付けを確認する

1. Cloud Shell で、リソースが適切なタグを付けてデプロイされたことを確認します。

   ```bash
   az resource list --resource-group rg-bicep-infrastructure \
     --query "[].{Name:name, Type:type, Tags:tags}" \
     --output table
   ```

1. リソース グループ自体のタグを確認します。

   ```bash
   az group show --name rg-bicep-infrastructure --query tags
   ```

1. 所有権と環境を識別するタグをリソース グループに追加します。

   ```bash
   az group update --name rg-bicep-infrastructure \
     --set tags.Environment='Development' tags.Owner='PlatformEngineering' tags.Project='BicepInfrastructure'
   ```

1. タグが適用されたことを確認します。

   ```bash
   az group show --name rg-bicep-infrastructure --query tags
   ```

### ガバナンスに関する Azure ポリシーを作成して実装する

1. 必要なタグを適用するポリシー定義ファイルを作成します。

   ```bash
   cat > tagging-policy.json << EOF
   {
     "if": {
       "allOf": [
         {
           "field": "type",
           "notIn": [
             "Microsoft.Resources/subscriptions/resourceGroups"
           ]
         },
         {
           "anyOf": [
             {
               "not": {
                 "field": "tags[Environment]",
                 "exists": "true"
               }
             },
             {
               "not": {
                 "field": "tags[Owner]",
                 "exists": "true"
               }
             }
           ]
         }
       ]
     },
     "then": {
       "effect": "deny"
     }
   }
   EOF
   ```

1. サブスクリプション ID を取得します。

   ```bash
   SUBSCRIPTION_ID=$(az account show --query id --output tsv)
   echo "Subscription ID: $SUBSCRIPTION_ID"
   ```

1. ポリシー定義を作成します。

   ```bash
   az policy definition create \
     --name 'require-tags-policy' \
     --display-name 'Require Environment and Owner tags' \
     --description 'Deny creation of resources without Environment and Owner tags' \
     --rules tagging-policy.json \
     --mode All
   ```

1. そのポリシーをリソース グループに割り当てます。

   ```bash
   az policy assignment create \
     --name 'require-tags-assignment' \
     --display-name 'Require tags on resources' \
     --policy "/subscriptions/$SUBSCRIPTION_ID/providers/Microsoft.Authorization/policyDefinitions/require-tags-policy" \
     --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-bicep-infrastructure"
   ```

1. ポリシー割り当てを確認します。

   ```bash
   az policy assignment list --scope "/subscriptions/$SUBSCRIPTION_ID/resourceGroups/rg-bicep-infrastructure" --output table
   ```

### ポリシーの適用をテストする

1. ポリシーをテストするために、必要なタグを含まない単純な Bicep テンプレートを作成します。

   ```bash
   cat > test-policy.bicep << EOF
   param location string = 'eastus'
   param storageAccountName string = 'testpolicy\${uniqueString(resourceGroup().id)}'

   resource testStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
     name: storageAccountName
     location: location
     sku: {
       name: 'Standard_LRS'
     }
     kind: 'StorageV2'
     // Note: No tags defined - this should be blocked by policy
   }
   EOF
   ```

1. 必要なタグを含まないテンプレートのデプロイを試みます。

   ```bash
   az deployment group create \
     --resource-group rg-bicep-infrastructure \
     --template-file test-policy.bicep
   ```

   > **注:**  このデプロイは、ポリシーが適用されたために失敗します。これは、ガバナンス コントロールが正しく機能していることを示します。

1. 次に、必要なタグを含む準拠バージョンを作成します。

   ```bash
   cat > test-policy-compliant.bicep << EOF
   param location string = 'eastus'
   param storageAccountName string = 'testcomp\${uniqueString(resourceGroup().id)}'

   resource testStorage 'Microsoft.Storage/storageAccounts@2023-01-01' = {
     name: storageAccountName
     location: location
     sku: {
       name: 'Standard_LRS'
     }
     kind: 'StorageV2'
     tags: {
       Environment: 'Testing'
       Owner: 'PlatformEngineering'
     }
   }
   EOF
   ```

1. 準拠テンプレートをデプロイします。

   ```bash
   az deployment group create \
     --resource-group rg-bicep-infrastructure \
     --template-file test-policy-compliant.bicep
   ```

   > **注:**  このデプロイは、必要なタグが含まれているため成功します。

1. テスト用のストレージ アカウントをクリーンアップします。

   ```bash
   az storage account delete \
     --name "testcompliant$(az group show --name rg-bicep-infrastructure --query id --output tsv | sed 's/.*\///' | tr -d '\n' | openssl dgst -sha1 | cut -c1-13)" \
     --resource-group rg-bicep-infrastructure \
     --yes
   ```

ガバナンス コントロールを正常に実装し、Azure ポリシーを使用してテストしました。 このポリシーにより、リソース管理に関する組織の標準が適用され、リソース グループにデプロイされるすべてのリソースに、必要な環境タグと所有者タグが確実に含まれるようになります。

## Bicep を使用して自動スケーリングを実装する

この演習では、Bicep テンプレートを拡張して、Azure App Service の自動スケーリング機能を含めます。 これにより、プラットフォームでは需要に基づいてリソースを自動的に調整でき、パフォーマンスとコスト効率が向上します。

### 自動スケーリングを含む拡張 Bicep テンプレートを作成する

1. 自動スケーリング機能を含む新しい Bicep テンプレートを作成します。

   ```bash
   cat > main-autoscale.bicep << 'EOF'
   param location string = 'eastus'
   param appServicePlanName string = 'asp-autoscale-${uniqueString(resourceGroup().id)}'
   param webAppName string = 'app-autoscale-${uniqueString(resourceGroup().id)}'
   param sqlServerName string = 'sql-autoscale-${uniqueString(resourceGroup().id)}'
   param sqlDatabaseName string = 'autoscaledb'
   param sqlAdminUser string
   @secure()
   param sqlAdminPassword string

   targetScope = 'resourceGroup'

   // App Service Plan (Standard tier required for autoscaling)
   resource appPlan 'Microsoft.Web/serverfarms@2023-01-01' = {
     name: appServicePlanName
     location: location
     sku: {
       name: 'S1'
       tier: 'Standard'
       capacity: 1
     }
     properties: {
       perSiteScaling: false
       maximumElasticWorkerCount: 10
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // Web App
   resource webApp 'Microsoft.Web/sites@2023-01-01' = {
     name: webAppName
     location: location
     properties: {
       serverFarmId: appPlan.id
       httpsOnly: true
       siteConfig: {
         minTlsVersion: '1.2'
         ftpsState: 'FtpsOnly'
         alwaysOn: true
       }
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // SQL Server
   resource sqlServer 'Microsoft.Sql/servers@2023-05-01-preview' = {
     name: sqlServerName
     location: location
     properties: {
       administratorLogin: sqlAdminUser
       administratorLoginPassword: sqlAdminPassword
       version: '12.0'
       minimalTlsVersion: '1.2'
       publicNetworkAccess: 'Enabled'
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // SQL Database
   resource sqlDb 'Microsoft.Sql/servers/databases@2023-05-01-preview' = {
     name: sqlDatabaseName
     location: location
     parent: sqlServer
     properties: {
       collation: 'SQL_Latin1_General_CP1_CI_AS'
       maxSizeBytes: 2147483648
       requestedServiceObjectiveName: 'S1'
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // Autoscale Settings
   resource autoscaleSetting 'Microsoft.Insights/autoscalesettings@2022-10-01' = {
     name: 'autoscale-${appServicePlanName}'
     location: location
     properties: {
       targetResourceUri: appPlan.id
       enabled: true
       profiles: [
         {
           name: 'defaultProfile'
           capacity: {
             minimum: '1'
             maximum: '5'
             default: '1'
           }
           rules: [
             {
               metricTrigger: {
                 metricName: 'CpuPercentage'
                 metricResourceUri: appPlan.id
                 operator: 'GreaterThan'
                 threshold: 75
                 timeAggregation: 'Average'
                 timeGrain: 'PT1M'
                 timeWindow: 'PT5M'
                 statistic: 'Average'
               }
               scaleAction: {
                 direction: 'Increase'
                 type: 'ChangeCount'
                 value: '1'
                 cooldown: 'PT5M'
               }
             }
             {
               metricTrigger: {
                 metricName: 'CpuPercentage'
                 metricResourceUri: appPlan.id
                 operator: 'LessThan'
                 threshold: 25
                 timeAggregation: 'Average'
                 timeGrain: 'PT1M'
                 timeWindow: 'PT10M'
                 statistic: 'Average'
               }
               scaleAction: {
                 direction: 'Decrease'
                 type: 'ChangeCount'
                 value: '1'
                 cooldown: 'PT10M'
               }
             }
           ]
         }
       ]
     }
     tags: {
       Environment: 'Development'
       Owner: 'PlatformEngineering'
     }
   }

   // Outputs
   output webAppUrl string = 'https://${webApp.properties.defaultHostName}'
   output sqlServerFqdn string = sqlServer.properties.fullyQualifiedDomainName
   output autoscaleSettingName string = autoscaleSetting.name
   EOF
   ```

1. 拡張 Bicep テンプレートを検証します。

   ```bash
   az deployment group validate \
     --resource-group rg-bicep-infrastructure \
     --template-file main-autoscale.bicep \
     --parameters sqlAdminUser='autoscaleAdmin' sqlAdminPassword='SecureP@ssw0rd123!'
   ```

### 自動スケーリング インフラストラクチャをデプロイする

1. 自動スケーリング機能を含む拡張テンプレートをデプロイします。

   ```bash
   az deployment group create \
     --resource-group rg-bicep-infrastructure \
     --template-file main-autoscale.bicep \
     --parameters sqlAdminUser='autoscaleAdmin' sqlAdminPassword='SecureP@ssw0rd123!'
   ```

   > **注:**  このデプロイでは、既存の App Service プランを Standard レベルにアップグレードし、自動スケーリング規則を追加します。

1. デプロイが正常に完了したことを確認します。

   ```bash
   az deployment group show \
     --resource-group rg-bicep-infrastructure \
     --name main-autoscale \
     --query properties.provisioningState
   ```

1. 自動スケーリング設定が作成されたことを確認します。

   ```bash
   az monitor autoscale list \
     --resource-group rg-bicep-infrastructure \
     --output table
   ```

1. 自動スケーリング構成の詳細を表示します。

   ```bash
   AUTOSCALE_NAME=$(az monitor autoscale list --resource-group rg-bicep-infrastructure --query "[0].name" --output tsv)
   az monitor autoscale show \
     --resource-group rg-bicep-infrastructure \
     --name "$AUTOSCALE_NAME" \
     --query "{Name:name, Enabled:enabled, MinCapacity:profiles[0].capacity.minimum, MaxCapacity:profiles[0].capacity.maximum, Rules:length(profiles[0].rules)}"
   ```

### 自動スケーリング構成を検証する

1. Azure portal に移動し、自動スケーリング構成を確認します。

   - リソース グループ `rg-bicep-infrastructure` に移動します
   - App Service プランを選択します
   - 左側のメニューで、**[スケールアウト (App Service プラン)]** を選択します
   - 定義した規則で自動スケーリングが有効になっていることを確認します

1. 現在のインスタンス数とスケーリング履歴を表示します。

   ```bash
   APP_SERVICE_PLAN=$(az appservice plan list --resource-group rg-bicep-infrastructure --query "[0].name" --output tsv)
   az appservice plan show \
     --resource-group rg-bicep-infrastructure \
     --name "$APP_SERVICE_PLAN" \
     --query "{Name:name, Sku:sku.name, CurrentInstances:numberOfWorkers, MaxWorkers:maximumNumberOfWorkers}"
   ```

   > **注:**  自動スケーリングの動作を完全にテストするには、アプリケーションに負荷を生成して CPU しきい値をトリガーする必要があります。 これはロード テスト ツールを使用して行うことができますが、このラボでは構成を確認するだけで十分です。

Bicep を使用して、Azure App Service の自動スケーリングを正常に実装しました。 この自動スケーリング構成の内容は次のとおりです。

- CPU 使用率が 5 分間で 75% を超えた場合にスケールアウトする (インスタンスを追加する)
- CPU 使用率が 10 分間で 25% を下回った場合にスケールインする (インスタンスを削除する)
- インスタンス数は 1 から 5 個までを維持する
- 急速なスケーリング アクションを防ぐために、クールダウン期間を含める

これにより、アプリケーションはコストを最適化しながら、さまざまなトラフィック負荷を効率的に処理できるようになります。

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。
