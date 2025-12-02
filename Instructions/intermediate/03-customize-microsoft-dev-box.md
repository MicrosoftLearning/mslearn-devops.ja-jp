---
lab:
  topic: Intermediate
  title: カスタム イメージとネットワークを使用して Microsoft Dev Box をカスタマイズする
  description: カスタム イメージ、プライベート ネットワーク、高度な構成を使って Microsoft Dev Box 環境をカスタマイズする方法を学びます。
---

# カスタム イメージとネットワークを使用して Microsoft Dev Box をカスタマイズする

このラボでは、Azure Image Builder でビルドされたカスタム イメージ、プライベート ネットワーク構成、高度なカスタマイズ機能を使って、Microsoft Dev Box 環境をカスタマイズする方法を学びます。 Azure コンピューティング ギャラリーの作成、カスタム イメージのビルド、仮想ネットワーク接続の構成、チーム固有の開発環境用のイメージ定義の実装を行います。

学習内容:

- Azure コンピューティング ギャラリーを作成して構成する
- Azure Image Builder を使用してカスタム開発ボックス イメージをビルドする
- 開発ボックス用のプライベート ネットワークを構成する
- イメージ定義とカスタマイズ カタログを実装する
- プライベート ネットワークを使用してカスタマイズされた開発ボックス プールを作成する

このラボの所要時間は約 **30** 分です。

## 開始する前に

このラボを終えるには、次のものが必要です。

- 少なくとも共同作成者レベルのアクセス権を持っている Azure サブスクリプション。 まだお持ちでない場合は、[サインアップ](https://azure.microsoft.com/)できます。
- Azure サブスクリプションで設定された Microsoft Dev Box 環境。 まだ設定していない場合は、ラボ「[開発者セルフサービス用に Microsoft Dev Box を実装する](../intermediate/02-implement-microsoft-dev-box.md)」を参照するか、「[作業の開始テンプレートを使用して Microsoft Dev Box を構成する](https://learn.microsoft.com/azure/dev-box/quickstart-get-started-template)」の手順に従って行ってください。
- Microsoft Dev Box のデプロイに関係する 3 つの異なるロールを表す、事前に作成された 3 つのユーザー アカウント (および必要に応じて事前に作成された 3 つの Microsoft Entra グループ) を含む Microsoft Entra テナント。 わかりやすくするため、ラボの手順のユーザー名とグループ名は、次の表の情報と一致させています。

  | User              | グループ                        | 役割                  |
  | ----------------- | ---------------------------- | --------------------- |
  | platformegineer01 | DevCenter_Platform_Engineers | プラットフォーム エンジニア     |
  | devlead01         | DevCenter_Dev_Leads          | 開発チーム リーダー |
  | devuser01         | DevCenter_Dev_Users          | 開発者             |

- ユーザーとグループを作成するための適切なアクセス許可を持つ Microsoft Entra テナント
- Azure サブスクリプションと同じ Microsoft Entra テナントに関連付けられている Microsoft Intune サブスクリプション
- GitHub ユーザー アカウント。 アカウントがない場合は、[新しいアカウントを作成](https://github.com/join)できます。 GitHub アカウントを作成する手順が必要な場合は、記事「[GitHub でのアカウントの作成](https://docs.github.com/get-started/quickstart/creating-an-account-on-github)」を参照してください。

### 必要な GitHub リポジトリをフォークする

1. 新しいブラウザー タブを開き、[https://github.com/MicrosoftLearning/contoso-co-eShop](https://github.com/MicrosoftLearning/contoso-co-eShop) に移動します。
1. メッセージが表示されたら、ご自分のアカウントで GitHub にサインインします。
1. **[Fork]** を選んで、ご自分の GitHub アカウントにリポジトリのフォークを作成します。

> **注:**  このリポジトリには、カスタマイズに使われるイメージ定義ファイルが含まれています。

## Microsoft Dev Box 環境をカスタマイズする

この演習では、Microsoft Dev Box 環境の機能をカスタマイズします。 このアプローチの焦点となるのは、Azure コンピューティング ギャラリー、カスタム イメージ、プライベート ネットワークを使ってカスタム開発者セルフサービス ソリューションを実装するときに適用できる変更の範囲です。

演習は次のタスクで構成されます。

- Azure コンピューティング ギャラリーを作成してデベロッパー センターにアタッチする
- Azure Image Builder の認証と認可を構成する
- Azure Image Builder を使用してカスタム イメージを作成する
- Azure デベロッパー センターのネットワーク接続を作成する
- Azure デベロッパー センター プロジェクトにイメージ定義を追加する
- カスタマイズした開発ボックス プールを作成する
- カスタマイズした開発ボックスを評価する

### Azure コンピューティング ギャラリーを作成してデベロッパー センターにアタッチする

このタスクでは、Azure コンピューティング ギャラリーを作成し、それをデベロッパー センターにアタッチします。 ギャラリーは Azure サブスクリプションに備わっているリポジトリで、カスタム イメージに関連する構造と体系を構築するのに役立ちます。 コンピューティング ギャラリーをデベロッパー センターにアタッチしてイメージで事前設定すると、そのコンピューティング ギャラリーに格納されているイメージに基づいて開発ボックスの定義を作成できます。

1. Web ブラウザーを開始し、Azure portal (`https://portal.azure.com`) にアクセスします。
1. 認証を求められたら、Microsoft アカウントを使用してサインインします。
1. Azure portal の **[検索]** テキスト ボックスで、「**`Azure compute galleries`**」を検索して選択します。
1. **[Azure コンピューティング ギャラリー]** ページで、**[+ 作成]** を選択します。
1. **[Azure コンピューティング ギャラリーの作成]** ページの **[基本]** タブで、以下の設定を指定し、**[次へ: 共有方法]** を選択します。

   | 設定        | 値                                                        |
   | -------------- | ------------------------------------------------------------ |
   | サブスクリプション   | このラボで使用している Azure サブスクリプションの名前 |
   | リソース グループ | **新しい**リソース グループ **rg-devcenter-custom** の名前 |
   | Name           | **compute_gallery_custom**                                   |
   | リージョン         | **(米国) 米国東部**                                             |

1. **[Azure コンピューティング ギャラリーの作成]** ページの **[共有方法]** タブで、**[ロールベースのアクセス制御 (RBAC)]** オプションが選択されていることを確認し、**[確認および作成]** を選択します。
1. **[レビューと作成]** タブで、検証が完了するのを待ち、**[作成]** を選択します。

   > **注:** プロジェクトがプロビジョニングされるまで待ってください。 Azure コンピューティング ギャラリーの作成にかかる時間は 1 分未満のはずです。

1. Azure portal で、**`Dev centers`** を検索して選択します。
1. **[デベロッパー センター]** ページで、**[+ 作成]** を選択します。
1. **[デベロッパー センターの作成]** ページの **[基本]** タブで、以下の設定を指定し、**[次へ: 設定]** を選択します。

   | 設定                                                                 | 値                                                        |
   | ----------------------------------------------------------------------- | ------------------------------------------------------------ |
   | サブスクリプション                                                            | このラボで使用している Azure サブスクリプションの名前 |
   | リソース グループ                                                          | **rg-devcenter-custom**                                      |
   | Name                                                                    | **devcenter-custom**                                         |
   | 場所                                                                | **(米国) 米国東部**                                             |
   | クイック スタート カタログを添付する: Azure デプロイ環境の定義 | Enabled                                                      |
   | クイック スタート カタログを添付する: 開発ボックスのカスタマイズ タスク              | Enabled                                                      |

1. **[デベロッパー センターの作成]** ページの **[設定]** タブで、以下の設定を指定し、**[レビューと作成]** を選択します。

   | 設定                                    | Value   |
   | ------------------------------------------ | ------- |
   | プロジェクトごとにカタログを有効にする                | Enabled |
   | プロジェクトで Microsoft がホストするネットワークを許可する | Enabled |
   | Azure Monitor エージェントのインストールを有効にする    | Enabled |

1. **[レビューと作成]** タブで、検証が完了するのを待ち、**[作成]** を選択します。

   > **注:**  デベロッパー センターがプロビジョニングされるまで待ちます。 これには 1 分ほどかかる場合があります。

1. **[デプロイが完了しました]** ページで、 **[リソースに移動]** を選択します。
1. **[devcenter-custom]** ページの左側にある縦のナビゲーション メニューで、**[開発ボックスの構成]** セクションを展開し、**[Azure コンピューティング ギャラリー]** を選びます。
1. **[devcenter-custom | Azure コンピューティング ギャラリー]** ページで、**[+ 追加]** を選びます。
1. **[Azure Compute Gallery の追加]** ペインの **[ギャラリー]** ドロップダウン リストで、**compute_gallery_custom** を選んでから **[追加]** を選びます。

   > **注:** _"このデベロッパー センターには、システム割り当て ID またはユーザー割り当て ID がありません。ID が割り当てられるまでギャラリーを追加できません。"_ というエラー メッセージが表示された場合は、 デベロッパー センターにシステム割り当て ID を割り当てる必要があります。
   > そうするには、Azure portal の **[devcenter-custom]** ページの左側にある縦のナビゲーション メニューで、[設定] の下にある **[ID]** を選び、**[システム割り当て]** タブで **[状態]** スイッチを **[オン]** に設定してから、**[保存]** を選びます。

### Azure Image Builder の認証と認可を構成する

このタスクでは、前のタスクで作成した Azure コンピューティング ギャラリーにイメージを追加するために Azure Image Builder によって使用されるユーザー割り当てマネージド ID を作成します。 また、カスタムのロールベースのアクセス制御 (RBAC) ロールを作成し、それをマネージド ID に割り当てることで、必要なアクセス許可を構成します。 こうすることで、次のタスクで Azure Image Builder を使用してカスタム イメージをビルドできます。

1. Azure portal で、**Cloud Shell** ツール バー アイコンを選択して Cloud Shell ウィンドウを開き、必要に応じて **[PowerShell に切り替える]** を選択して PowerShell セッションを開始し、**[Cloud Shell の PowerShell に切り替える]** ダイアログ ボックスで **[確認]** を選択します。

   > **注:** Cloud Shell を初めて開く場合は、**[Azure Cloud Shell へようこそ]** ダイアログ ボックスで **[PowerShell]** を選択し、**[作業の開始]** ペインでオプション **[ストレージ アカウントは不要です]** を選択し、**[サブスクリプション]** ドロップダウン リストで、このラボで使用している Azure サブスクリプションの名前を選択します。

1. Cloud Shell ウィンドウの PowerShell セッションで、次のコマンドを実行して、必要なすべてのリソース プロバイダーが登録されていることを確認します。

   ```powershell
   Register-AzResourceProvider -ProviderNamespace Microsoft.VirtualMachineImages
   Register-AzResourceProvider -ProviderNamespace Microsoft.Storage
   Register-AzResourceProvider -ProviderNamespace Microsoft.Compute
   Register-AzResourceProvider -ProviderNamespace Microsoft.KeyVault
   Register-AzResourceProvider -ProviderNamespace Microsoft.Network
   ```

1. 次のコマンドを実行して、必要な PowerShell モジュールをインストールします (プロンプトが表示されたら、「**A**」を入力して **Enter** キーを押します)。

   ```powershell
   'Az.ImageBuilder', 'Az.ManagedServiceIdentity' | ForEach-Object {Install-Module -Name $_ -AllowPrerelease}
   ```

1. 次のコマンドを実行して、イメージ ビルド プロセス全体で参照される変数を設定します。

   ```powershell
   $currentAzContext = Get-AzContext
   # the target Azure subscription ID
   $subscriptionID=$currentAzContext.Subscription.Id
   # the target Azure resource group name
   $imageResourceGroup='rg-devcenter-custom'
   # the target Azure region
   $location='eastus'
   # the reference name assigned to the image created by using the Azure Image Builder service
   $runOutputName="aibWinImgCustom"
   # image template name
   $imageTemplateName="templateWinVSCodeCustom"
   # the Azure compute gallery name
   $computeGallery = 'compute_gallery_custom'
   ```

1. 次のコマンドを実行して、ユーザー割り当てマネージド ID を作成します (VM Image Builder では、指定されたユーザー ID を使用して、対象の Azure コンピューティング ギャラリーにイメージを格納します)。

   ```powershell
   # Install the Azure PowerShell module to support AzUserAssignedIdentity
   Install-Module -Name Az.ManagedServiceIdentity
   # Generate a pseudo-random integer to be used for resource names
   $timeInt=$(get-date -UFormat "%s")

   # Create an identity
   $identityName='identityAIBCustom' + $timeInt
   New-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $identityName -Location $location
   $identityNameResourceId=$(Get-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $identityName).Id
   $identityNamePrincipalId=$(Get-AzUserAssignedIdentity -ResourceGroupName $imageResourceGroup -Name $identityName).PrincipalId
   ```

1. 次のコマンドを実行して、新しく作成されたユーザー割り当てマネージド ID に、**rg-devcenter-custom** リソース グループにイメージを格納するのに必要なアクセス許可を付与します。

   ```powershell
   # Set variables
   $imageRoleDefName = 'Custom Azure Image Builder Image Def Custom' + $timeInt
   $aibRoleImageCreationUrl = 'https://raw.githubusercontent.com/azure/azvmimagebuilder/master/solutions/12_Creating_AIB_Security_Roles/aibRoleImageCreation.json'
   $aibRoleImageCreationPath = 'aibRoleImageCreation.json'

   # Customize the role definition file
   Invoke-WebRequest -Uri $aibRoleImageCreationUrl -OutFile $aibRoleImageCreationPath -UseBasicParsing
   ((Get-Content -path $aibRoleImageCreationPath -Raw) -Replace '<subscriptionID>', $subscriptionID) | Set-Content -Path $aibRoleImageCreationPath
   ((Get-Content -path $aibRoleImageCreationPath -Raw) -Replace '<rgName>', $imageResourceGroup) | Set-Content -Path $aibRoleImageCreationPath
   ((Get-Content -path $aibRoleImageCreationPath -Raw) -Replace 'Azure Image Builder Service Image Creation Role', $imageRoleDefName) | Set-Content -Path $aibRoleImageCreationPath

   # Create a role definition
   New-AzRoleDefinition -InputFile  ./aibRoleImageCreation.json

   # Assign the role to the VM Image Builder user-assigned managed identity within the scope of the **rg-devcenter-custom** resource group
   New-AzRoleAssignment -ObjectId $identityNamePrincipalId -RoleDefinitionName $imageRoleDefName -Scope "/subscriptions/$subscriptionID/resourceGroups/$imageResourceGroup"
   ```

### Azure Image Builder を使用してカスタム イメージを作成する

このタスクでは、Azure Image Builder を使い、Chocolatey と Visual Studio Code が自動的にインストールされる Windows 11 Enterprise イメージを定義する既存の Azure Resource Manager (ARM) テンプレートに基づいて、カスタム イメージを作成します。 Azure VM Image Builder を使用すると、VM イメージの定義とプロビジョニングのプロセスが大幅に簡略化されます。 これは、自動イメージング パイプラインを構成するために指定したイメージ構成に依存します。 その後に、開発者はこのようなイメージを使用して開発ボックスをプロビジョニングできるようになります。

1. Cloud Shell ウィンドウの PowerShell セッションで、次のコマンドを実行して、この演習の最初のタスクで作成した Azure コンピューティング ギャラリーに追加するイメージ定義を作成します。

   ```powershell
   # ensure that the image definition security type property is set to 'TrustedLaunch'
   $securityType = @{Name='SecurityType';Value='TrustedLaunch'}
   $features = @($securityType)
   # Image definition name
   $imageDefName = 'imageDefDevBoxVSCodeCustom'

   # Create the image definition
   New-AzGalleryImageDefinition -GalleryName $computeGallery -ResourceGroupName $imageResourceGroup -Location $location -Name $imageDefName -OsState generalized -OsType Windows -Publisher 'Contoso' -Offer 'vscodedevbox' -Sku '1-0-0' -Feature $features -HyperVGeneration 'V2'
   ```

   > **注:** 開発ボックス イメージは、第 2 世代、Hyper-V v2、Windows 10 または 11 Enterprise バージョン 20H2 以降の使用を含む多数の要件を満たす必要があります。 完全なリストについては、Microsoft Learn の記事「[Microsoft Dev Box 用に Azure Compute Gallery を構成する](https://learn.microsoft.com/azure/dev-box/how-to-configure-azure-compute-gallery)」をご覧ください。

1. 次のコマンドを実行して、template.json という名前の空のファイルを作成します。このファイルには、Chocolatey と Visual Studio Code が自動的にインストールされる Windows 11 Enterprise イメージを定義する ARM テンプレートが格納されます。

   ```powershell
   Set-Location -Path ~
   $templateFile = 'template.json'
   Set-Content -Path $templateFile -Value ''
   ```

1. Cloud Shell の PowerShell セッションで、nano テキスト エディターを使用して、新しく作成されたファイルに次のコンテンツを追加します。

   > **注:** nano テキスト エディターを開くには、コマンド `nano ./template.json` を実行します。 変更を保存して nano テキスト エディターを終了するには、**Ctrl + X** キー、**Y** キー、**Enter** キーの順に押します。

   ```json
   {
     "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
     "contentVersion": "1.0.0.0",
     "parameters": {
       "imageTemplateName": {
         "type": "string"
       },
       "api-version": {
         "type": "string"
       },
       "svclocation": {
         "type": "string"
       }
     },
     "variables": {},
     "resources": [
       {
         "name": "[parameters('imageTemplateName')]",
         "type": "Microsoft.VirtualMachineImages/imageTemplates",
         "apiVersion": "[parameters('api-version')]",
         "location": "[parameters('svclocation')]",
         "dependsOn": [],
         "tags": {
           "imagebuilderTemplate": "win11multi",
           "userIdentity": "enabled"
         },
         "identity": {
           "type": "UserAssigned",
           "userAssignedIdentities": {
             "<imgBuilderId>": {}
           }
         },
         "properties": {
           "buildTimeoutInMinutes": 100,
           "vmProfile": {
             "vmSize": "Standard_D2s_v3",
             "osDiskSizeGB": 127
           },
           "source": {
             "type": "PlatformImage",
             "publisher": "MicrosoftWindowsDesktop",
             "offer": "Windows-11",
             "sku": "win11-21h2-ent",
             "version": "latest"
           },
           "customize": [
             {
               "type": "PowerShell",
               "name": "CreateBuildPath",
               "inline": [
                 "mkdir c:\\buildArtifacts",
                 "echo Azure-Image-Builder-Was-Here  > c:\\buildArtifacts\\azureImageBuilder.txt"
               ]
             },
             {
               "type": "PowerShell",
               "name": "InstallChocolatey",
               "inline": [
                 "Set-ExecutionPolicy Bypass -Scope Process -Force",
                 "[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072",
                 "iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))"
               ]
             },
             {
               "type": "PowerShell",
               "name": "InstallVSCode",
               "inline": ["choco install vscode -y"]
             }
           ],
           "distribute": [
             {
               "type": "SharedImage",
               "galleryImageId": "/subscriptions/<subscriptionID>/resourceGroups/<rgName>/providers/Microsoft.Compute/galleries/<sharedImageGalName>/images/<imageDefName>",
               "runOutputName": "<runOutputName>",
               "artifactTags": {
                 "source": "azureVmImageBuilder",
                 "baseosimg": "windows11"
               },
               "replicationRegions": ["<region1>", "<region2>"]
             }
           ]
         }
       }
     ]
   }
   ```

1. 次のコマンドを実行して、template.json のプレースホルダーを実際の Azure 環境に固有の値に置き換えます。

   ```powershell
   $replRegion2 = 'eastus2'
   $templateFilePath = '.\template.json'
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<subscriptionID>', $subscriptionID | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<rgName>', $imageResourceGroup | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<runOutputName>', $runOutputName | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<imageDefName>', $imageDefName | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<sharedImageGalName>', $computeGallery | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<region1>', $location | Set-Content -Path $templateFilePath
   (Get-Content -Path $templateFilePath -Raw ) -Replace '<region2>', $replRegion2 | Set-Content -Path $templateFilePath
   ((Get-Content -Path $templateFilePath -Raw) -Replace '<imgBuilderId>', $identityNameResourceId) | Set-Content -Path $templateFilePath
   ```

1. 次のコマンドを実行して、テンプレートを Azure Image Builder サービスに送信します (このサービスは、スクリプトなどの依存成果物をダウンロードし、カスタム仮想マシン イメージをビルドするための、名前に **IT\_** プレフィックスが含まれるステージング リソース グループに格納することで、送信されたテンプレートを処理します)。

   ```powershell
   New-AzResourceGroupDeployment -ResourceGroupName $imageResourceGroup -TemplateFile $templateFilePath -Api-Version "2020-02-14" -imageTemplateName $imageTemplateName -svclocation $location
   ```

1. 次のコマンドを実行して、イメージ ビルド プロセスを起動します。

   ```powershell
   Invoke-AzResourceAction -ResourceName $imageTemplateName -ResourceGroupName $imageResourceGroup -ResourceType Microsoft.VirtualMachineImages/imageTemplates -ApiVersion "2020-02-14" -Action Run -Force
   ```

1. 次のコマンドを実行して、イメージのプロビジョニング状態を判定します。

   ```powershell
   Get-AzImageBuilderTemplate -ImageTemplateName $imageTemplateName -ResourceGroupName $imageResourceGroup | Select-Object -Property Name, LastRunStatusRunState, LastRunStatusMessage, ProvisioningState
   ```

   > **注:** 次の出力は、ビルド プロセスが正常に完了したことを示します。

   ```powershell
   Name                    LastRunStatusRunState LastRunStatusMessage ProvisioningState
   ----                    --------------------- -------------------- -----------------
   templateWinVSCodeCustom Succeeded                                  Succeeded
   ```

1. または、ビルドの進行状況を監視するには、次の手順を使用します。

   1. Azure portal で、**`Image templates`** を検索して選択します。
   1. **[イメージ テンプレート]** ページで、**templateWinVSCodeCustom** を選びます。
   1. **[templateWinVSCodeCustom]** ページの **[要点]** セクションで、**[ビルドの実行状態]** エントリの値を確認します。

   > **注:** ビルド プロセスは約 30 分かかる場合があります。 時間を節約するため、ビルドが行われている間に次のタスクに進み、後で戻って完了を確認してもかまいません。

1. ビルドが完了したら、Azure portal で「**`Azure compute galleries`**」を検索して選択します。
1. **[Azure コンピューティング ギャラリー]** ページで、**compute_gallery_custom** を選びます。
1. **[compute_gallery_custom]** ページで、**[定義]** タブが選ばれていることを確認し、定義の一覧で **imageDefDevBoxVSCodeCustom** を選びます。
1. **[imageDefDevBoxVSCodeCustom]** ページで **[バージョン]** タブを選び、**[1.0.0 (最新バージョン)]** エントリが一覧に表示され、**[プロビジョニングの状態]** が **[成功]** になっていることを確認します。
1. **[1.0.0 (最新バージョン)]** エントリを選択します。
1. **[1.0.0 (compute_gallery_custom/imageDefDevBoxVSCodeCustom/1.0.0)]** ページで、VM イメージのバージョン設定を確認します。

### Azure デベロッパー センターのネットワーク接続を作成する

このタスクでは、Azure 仮想ネットワーク内でホストされているリソースへのプライベート接続を必要とするシナリオで使用するための、Azure デベロッパー センター ネットワークを構成します。 これまでのラボで使ったかもしれない Microsoft がホストするネットワークとは異なり、仮想ネットワーク接続では、ハイブリッド シナリオ (オンプレミス リソースへの接続を提供) と Azure 開発ボックスの Microsoft Entra ハイブリッド参加も (Microsoft Entra 参加のサポートに加えて) サポートされます。

1. Azure portal が表示されている Web ブラウザーの **[検索]** テキスト ボックスで、「**`Virtual networks`**」を検索して選択します。
1. **[仮想ネットワーク]** ページで、**[+ 作成]** を選択します。
1. **[仮想ネットワークの作成]** ページの **[基本]** タブで、以下の設定を指定し、**[次へ]** を選択します。

   | 設定        | 値                                                        |
   | -------------- | ------------------------------------------------------------ |
   | サブスクリプション   | このラボで使用している Azure サブスクリプションの名前 |
   | リソース グループ | **rg-devcenter-custom**                                      |
   | Name           | **vnet-custom**                                              |
   | 場所       | **(米国) 米国東部**                                             |

1. **[仮想ネットワークの作成]** ページの **[セキュリティ]** タブで、既定値を変更せずに既存の設定を確認し、**[次へ]** を選択します。
1. **[仮想ネットワークの作成]** ページの **[IP アドレス]** タブで、既定値を変更せずに既存の設定を確認し、**[確認 + 作成]** を選択します。
1. **[仮想ネットワークの作成]** ページの **[確認および作成]** タブで、**[作成]** を選択します。

   > **注:** 仮想ネットワークが作成されるまで待ってください。 これにかかる時間は 1 分未満です。

1. Azure portal の **[検索]** テキスト ボックスで、「**`Network connections`**」を検索して選択します。
1. **[ネットワーク接続]** ページで、**[+ 作成]** を選択します。
1. **[ネットワーク接続の作成]** ページの **[基本]** タブで、以下の設定を指定し、**[レビューと作成]** を選択します。

   | 設定         | 値                                                        |
   | --------------- | ------------------------------------------------------------ |
   | サブスクリプション    | このラボで使用している Azure サブスクリプションの名前 |
   | リソース グループ  | **rg-devcenter-custom**                                      |
   | Name            | **network-connection-vnet-custom**                           |
   | 仮想ネットワーク | **vnet-custom**                                              |
   | Subnet          | **default**                                                  |

1. **[仮想ネットワークの作成]** ページの **[確認および作成]** タブで、**[作成]** を選択します。

   > **注:** ネットワーク接続が作成されるまで待ってください。 これには 1 分ほどかかる場合があります。

1. Azure portal で、「**`Dev centers`**」を検索して選び、**[デベロッパー センター]** ページで **devcenter-custom** を選びます。
1. **[devcenter-custom]** ページの左側にある縦のナビゲーション メニューで、**[開発ボックスの構成]** セクションを展開して **[ネットワーク]** を選びます。
1. **[devcenter-custom | ネットワーク]** ページで、**[+ 追加]** を選びます。
1. **[ネットワーク接続を追加]** ペインの **[ネットワーク接続]** ドロップダウン リストで、**network-connection-vnet-custom** を選んでから **[追加]** を選びます。

   > **注:**  ネットワーク接続が追加されるまで待たずに、次のタスクに進んでください。 ネットワーク接続の追加には 1 分程度かかることがあります。

### Azure デベロッパー センター プロジェクトにイメージ定義を追加する

このタスクでは、イメージ定義を Azure デベロッパー センター プロジェクトに追加します。 イメージ定義では、Azure Marketplace またはカスタム イメージと、基になるイメージに適用する追加の変更を定義する構成可能なタスクが組み合わされます。 イメージ定義を使用して、新しいイメージ (タスクによって適用されたものを含むすべての変更を含む) をビルドしたり、開発ボックス プールを直接作成したりできます。 再利用可能なイメージを作成すると、開発ボックスのプロビジョニングに必要な時間が最小限に抑えられます。

Microsoft Dev Box チームのカスタマイズ用にイメージングを構成するには、プロジェクト レベルのカタログ (デベロッパー センターの作成時に構成しました) を有効にする必要があります。 このタスクでは、プロジェクトのカタログ同期設定を構成します。 この作業には、イメージ定義ファイルを含むカタログのアタッチが含まれます。

1. Azure portal が表示されている Web ブラウザーの **[devcenter-custom]** ページの左側にある縦のナビゲーション メニューで、**[管理]** セクションを展開して **[プロジェクト]** を選びます。
1. **[devcenter-custom | プロジェクト]** ページで、**[+ 作成]** を選びます。
1. **[プロジェクトを作成]** ページの **[基本]** タブで、以下の設定を指定し、**[次へ: 開発ボックスの管理]** を選択します。

   | 設定        | 値                                                        |
   | -------------- | ------------------------------------------------------------ |
   | サブスクリプション   | このラボで使用している Azure サブスクリプションの名前 |
   | リソース グループ | **rg-devcenter-custom**                                      |
   | デベロッパー センター     | **devcenter-custom**                                         |
   | Name           | **devcenter-project-custom**                                 |
   | 説明    | **カスタム デベロッパー センター プロジェクト**                                |

1. **[プロジェクトを作成]** ページの **[開発ボックスの管理]** タブで、以下の設定を指定し、**[次へ: カタログ]** を選択します。

   | 設定                 | Value |
   | ----------------------- | ----- |
   | Dev Box の制限を有効にする   | はい   |
   | 開発者あたりの Dev Box 数 | **2** |

1. **[プロジェクトを作成]** ページの **[カタログ]** タブで、以下の設定を指定し、**[レビューと作成]** を選択します。

   | 設定                            | Value   |
   | ---------------------------------- | ------- |
   | デプロイ環境の定義 | Enabled |
   | イメージ定義                  | Enabled |

1. **[プロジェクトの作成]** ページの **[確認と作成]** タブで、**[作成]** を選びます。

   > **注:** プロジェクトが作成されるまで待ってください。 これにかかる時間は 1 分未満です。

1. **[デプロイが完了しました]** ページで、 **[リソースに移動]** を選択します。
1. **[devcenter-project-custom]** ページの左側にある縦のナビゲーション メニューで、**[設定]** セクションを展開して **[カタログ]** を選びます。
1. **[devcenter-project-custom | カタログ]** ページで、**[+ 追加]** を選びます。
1. **[カタログの追加]** ペインで、**[名前]** テキスト ボックスに「**`image-definitions-custom`**」と入力し、**[カタログ ソース]** セクションで **[GitHub]** を選択し、**[認証の種類]** で **[GitHub アプリ]** を選択し、**[このカタログを自動的に同期する]** チェックボックスをオンのままにして **[GitHub でサインイン]** を選択します。
1. メッセージが表示されたら、**[GitHub でサインイン]** ウィンドウで GitHub 資格情報を入力し、**[サインイン]** を選択します。
1. "アカウントに関連付けられた GitHub リポジトリが見つからなかった" ことを伝えるメッセージと、**リポジトリを構成する**ためのリンクが表示される場合、通常、これは初回セットアップの間、または GitHub アプリが以前に構成されていない場合に発生します。 Microsoft DevCenter GitHub アプリを設定するには、次の追加手順を完了します。

   1. **リポジトリを構成する**リンクを選択します。 これにより、GitHub に移動する新しいブラウザー タブまたはウィンドウが開きます。
   1. GitHub の **[Install Microsoft DevCenter]** ページで、個人用アカウントにアプリをインストールするよう求められます。
   1. **[Install on your personal account]** セクションで、次のいずれかのオプションを選びます。
      - アカウント内の現在および将来のすべてのリポジトリへのアクセスを許可するには、**[All repositories]** を選びます。
      - 特定のリポジトリを選ぶには、**[Only select repositories]** を選びます。 先に進む前に、**contoso-co-eShop** リポジトリを自分の GitHub アカウントにフォークしていることを確認します。 このオプションを選んだ場合は、**[Select repositories]** ドロップダウンを使って、**contoso-co-eShop** リポジトリ (または Azure DevCenter で使用できるようにする他のリポジトリ) を選びます。
   1. **[with these permissions]** セクションで付与されるアクセス許可を確認します。通常は、[Read access to code and metadata] が含まれます。
   1. **[Install]** を選んで、GitHub アプリのインストールを完了します。
   1. Azure portal にリダイレクトされます。 リダイレクトが自動的に行われない場合は、GitHub のタブを閉じて Azure portal に戻ります。
   1. Azure portal に戻り、**[カタログの追加]** ページで、**[最新の情報に更新]** を選ぶか、ページを更新してリポジトリの一覧を読み込み直します。

   > **注:** この手順を完了するには、https://github.com/MicrosoftLearning/contoso-co-eShop リポジトリを GitHub アカウントにフォークする必要があります。

1. メッセージが表示されたら、**[Microsoft デベローッパー センターを承認する]** ウィンドウで、**[Microsoft デベローッパー センターを承認する]** を選択します。
1. **[カタログの追加]** ペインに戻って、**[リポジトリ]** ドロップダウン リストで **contoso-co-eShop** を選択し、**[ブランチ]** ドロップダウン リストで **[既定のブランチ]** エントリを受け入れ、**[フォルダー パス]** に「**`.devcenter/catalog/image-definitions`**」と入力し、**[追加]** を選択します。
1. **[devcenter-project-custom | カタログ]** ページに戻り、**[状態]** 列のエントリを監視して同期が正常に完了することを確認します。
1. **[ステータス]** 列の **[同期に成功しました]** リンクを選択し、結果の通知ウィンドウを確認し、3 つの項目がカタログに追加されたことを確認し、右上隅の **[x]** 記号を選択してウィンドウを閉じます。
1. **[devcenter-project-custom | カタログ]** ページに戻り、**image-definitions-custom** を選んで、そこに **ContosoBaseImageDefinition**、**backend-eng**、**frontend-eng** という名前の 3 つのエントリがあることを確認します。
1. Azure portal で **[devcenter-project-custom]** ページに戻り、左側にある縦のナビゲーション メニューで **[管理]** セクションを展開し、**[イメージ定義]** を選んで、このタスクで前に確認したものと同じ 3 つのイメージ定義がページに表示されることを確認します。

### カスタマイズした開発ボックス プールを作成する

このタスクでは、新しくプロビジョニングされたイメージ定義を使用して開発ボックス プールを作成します。 このプールでは、この演習で前に設定したネットワーク接続も利用します。

1. **[devcenter-project-custom | イメージ定義]** ページが表示されている Azure portal で、左側にある縦のナビゲーション メニューで **[管理]** セクションの **[開発ボックス プール]** を選びます。
1. **[devcenter-project-custom | 開発ボックス プール]** ページで、**[+ 作成]** を選びます。
1. **[開発ボックス プールの作成]** ページの **[基本]** タブで、以下の設定を指定し、**[作成]** を選択します。

   | 設定                                                                                                           | 値                              |
   | ----------------------------------------------------------------------------------------------------------------- | ---------------------------------- |
   | 名前                                                                                                              | **devbox-pool-custom**             |
   | Definition                                                                                                        | **frontend-eng**                   |
   | ネットワーク接続                                                                                                | **network-connection-vnet-custom** |
   | シングル サインオンの有効化                                                                                             | Enabled                            |
   | Dev box Creator Privileges (開発ボックス作成者の特権)                                                                                        | **ローカル管理者**            |
   | 自動停止を有効にする                                                                                      | Enabled                            |
   | 停止時刻                                                                                                         | **07:00 PM**                       |
   | タイム ゾーン                                                                                                         | 現在使用しているタイム ゾーン             |
   | 切断時の休止状態を有効にする                                                                                    | Enabled                            |
   | 猶予時間 (分)                                                                                           | **60**                             |
   | この組織に Azure ハイブリッド特典 ライセンスがあり、このプール内のすべての開発ボックスに適用されることを確認しました | Enabled                            |

   > **注:** 開発ボックス プールが作成されるまで待ってください。 これには 2 分ほどかかる場合があります。

### カスタマイズした開発ボックスを評価する

このタスクでは、カスタム イメージとプライベート ネットワークを使って開発ボックスを作成し、カスタマイズ後の開発ボックスの機能を評価します。

> **注:**  このタスクには、前のラボの開発者ユーザー アカウントの資格情報が必要です。または、ラボ環境の準備セクションの手順に従って、新しいユーザー アカウントを作成してもかまいません。

1. 最初に、開発者が新しい開発ボックス プールを使用できるようにアクセス許可を割り当てる必要があります。 Azure portal で **[devcenter-project-custom]** ページに移動します。
1. 左側にある縦のナビゲーション メニューで **[アクセスの制御 (IAM)]** を選びます。
1. **[devcenter-project-custom | アクセス制御 (IAM)]** ページで、**[+ 追加]** を選び、ドロップダウン リストの **[ロールの割り当ての追加]** を選びます。
1. **[ロールの割り当ての追加]** ページの **[ロール]** タブで、**[職務ロール]** タブが選択されていることを確認し、ロールの一覧で **[DevCenter Dev Box Users]** を選択し、**[次へ]** を選択します。
1. **[ロールの割り当ての追加]** ページの **[メンバー]** タブで、**[ユーザー、グループ、またはサービス プリンシパル]** オプションが選ばれていることを確認し、**[+ メンバーの選択]** を選びます。
1. **[メンバーの選択]** ペインで、「**`DevCenter_Dev_Users`**」を検索して選び、**[選択]** を選びます。
1. **[ロールの割り当ての追加]** ページの **[メンバー]** タブに戻り、**[次へ]** を選択します。
1. **[ロールの割り当ての追加]** ページの **[確認と割り当て]** タブで、**[確認と割り当て]** を選択します。
1. Web ブラウザーのシークレット/プライベート モードを開始し、Microsoft Dev Box 開発者ポータル (`https://aka.ms/devbox-portal`) にアクセスします。
1. サインインを求められたら、**devuser01** ユーザー アカウントの資格情報を指定します。
1. Microsoft Dev Box 開発者ポータルの **[ようこそ、devuser01 さん]** ページで、**[+ 新しい開発ボックス]** を選択します。
1. **[開発ボックスを追加する]** ペインで、次の設定を指定します。

   | 設定 | 値                        |
   | ------- | ---------------------------- |
   | 名前    | **devuser01custombox01**     |
   | プロジェクト | **devcenter-project-custom** |
   | プール    | **devbox-pool-custom**       |

1. プールの指定、休止状態のサポート状態、スケジュールされたシャットダウンのタイミングなど、**[開発ボックスを追加する]** ペインに表示される他の情報を確認します。 使用できるカスタマイズ オプションに注意してください。
1. **[開発ボックスを追加する]** ペインで、**[作成]** を選択します。

   > **注:**  開発ボックスの作成プロセスには 30 から 65 分かかる場合があります。 このラボでの時間を節約するには、プロビジョニングの完了を待つ必要はありませんが、カスタム イメージとプライベート ネットワーク構成で開発ボックスが作成されていることを確認する必要があります。

1. プロビジョニングの状態を確認し、次のものを使って開発ボックスが作成されていることに注意してください。
   - カスタム **frontend-eng** イメージ定義
   - プライベート ネットワーク接続 **network-connection-vnet-custom**
   - イメージ定義カタログからのカスタム構成

## リソースをクリーンアップする

これで演習が完了したので、不要なリソース使用を避けるために、作成したクラウド リソースを削除してください。

1. お使いのブラウザーで Azure portal ([https://portal.azure.com](https://portal.azure.com)) に移動し、メッセージに応じて Azure 資格情報を使用してサインインします。
1. 作成したリソース グループに移動し、この演習で使用したリソースの内容を表示します。
1. ツール バーの **[リソース グループの削除]** を選びます。
1. リソース グループ名を入力し、削除することを確認します。

> **注:** リソース グループを削除すると、その中のすべてのリソースが削除されます。 この演習で既存のリソース グループを選択した場合は、この演習の範囲外にある既存のリソースも削除されます。

GitHub リポジトリまたはプロジェクトは参照およびポートフォリオ アイテムとして引き続き使用できるので、これらをクリーンアップする必要はありません。

リポジトリを削除したい場合は、次のドキュメントのようにしてできます: 「[リポジトリの削除](https://docs.github.com/repositories/creating-and-managing-repositories/deleting-a-repository)」。
