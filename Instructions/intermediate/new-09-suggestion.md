---
lab:
  topic: Intermediate
  title: Azure Artifacts によるパッケージ管理
  description: Azure Artifacts を使用してマイクロサービス アーキテクチャ用の内部 NuGet パッケージをビルドおよび管理し、実際のエンタープライズ シナリオをシミュレートします。
  duration: 40 minutes
  level: 400
  islab: true
  primarytopics:
    - Azure
    - Azure Artifacts
---

# Azure Artifacts を使用した Package Management: 内部共有ライブラリのビルド

## シナリオの概要

あなたは、モノリシック アプリケーションからマイクロサービス アーキテクチャに移行する中規模の eコマース企業である **Contoso Retail** の DevOps エンジニアです。 開発チームは、ログ、認証ヘルパー、API 応答モデル、ユーティリティ関数といった共通の機能が、複数のサービス間で重複していることを特定しました。

マネージャーからは次のタスクが割り当てられています。

> "マイクロサービス間でのコードのコピーを停止する必要があります。 チームが Azure Artifacts を通じて実行できる共有ライブラリを作成してください。 最初は共通の API 応答モデルとログ ユーティリティから始めてください。 ライブラリは適切にバージョン管理され、すべてのサービスで簡単に更新できるようにしてください。"

このラボでは、次のことを行います。

- 組織の内部パッケージの Azure Artifacts フィードを設定する
- 現実的なユーティリティ コードを使用して .NET 10 共有ライブラリを作成する
- バージョン コントロールのために Azure Repos にコードをプッシュする
- セマンティック バージョニングを実装し、パッケージを公開する
- シミュレートされたマイクロサービスでパッケージを実行する
- パッケージの公開と実行を自動化する CI/CD パイプラインを作成する
- パッケージを更新し、バージョンの依存関係を管理する

このラボの所要時間は約 **40** 分です。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)
- **Azure DevOps 組織:** ない場合は作成します
- **Visual Studio Code** と [C# 開発キット拡張機能](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit)
- **.NET 10 SDK:** [.NET 10 SDK をダウンロードしてインストールします](https://dotnet.microsoft.com/download/dotnet/10.0)
- **Azure Artifacts 資格情報プロバイダー:**[資格情報プロバイダーをダウンロードしてインストールする](https://go.microsoft.com/fwlink/?linkid=2099625)

## エンタープライズ シナリオでの Azure Artifacts について

Azure Artifacts は、マイクロサービスを採用している組織や、共通のコードを共有する複数のアプリケーションを維持する組織にとって不可欠です。 エンタープライズでの主な利点は次のとおりです。

- **コードの再利用性:** コピーと貼り付けなしでチーム間で共通ライブラリが共有される
- **バージョン コントロール:** セマンティック バージョニングを使用してパッケージのバージョンが管理される
- **セキュリティ:** パッケージの公開と実行ができるユーザーが制御される
- **監査可能性:** プロジェクト間のパッケージの使用と依存関係が追跡される
- **CI/CD の統合:** ビルド パイプラインからパッケージが自動的に公開される

一般的なエンタープライズ シナリオでは、次のようなものが含まれます。

| フィード | パーパス | サンプル パッケージ |
|------|---------|------------------|
| `contoso-shared` | 横断的関心事 | ログ、認証、ユーティリティ |
| `contoso-domain` | ビジネス ドメイン モデル | 注文モデル、顧客モデル |
| `contoso-infra` | インフラストラクチャ コード | データベース ヘルパー、メッセージ バス クライアント |

## タスク 1: Azure DevOps プロジェクトを設定する

まず、共有ライブラリをホストする Azure DevOps プロジェクトを作成します。

1. ブラウザーで、Azure DevOps 組織を開きます
1. **[+ 新しいプロジェクト]** を選択します
1. プロジェクトを構成します。
   - **名前**: `Contoso.Microservices`
   - **説明**: `Internal shared libraries and microservice projects for Contoso Retail`
   - **可視性:** プライベート
   - **バージョン コントロール:** Git
   - **作業項目プロセス:** アジャイル
1. **[作成]** を選択します

## タスク 2: Azure Artifacts フィードを作成する

組織の内部パッケージの専用フィードを設定します。

1. Azure DevOps プロジェクトの左側のナビゲーションから、**[Artifacts]** を選択します
1. **[+ フィードの作成]** を選択します
1. 次のようにして、フィードを構成します。
   - **名前**: `contoso-internal`
   - **可視性:** `Members of your Microsoft Entra Tenant` (組織を選択します)
   - **アップストリーム ソース:** **[共通のパブリック ソースからのパッケージを含める]** を選択します
   
   > **なぜアップストリーム ソースを含めるのですか?** これにより、開発者は内部パッケージとパブリック NuGet パッケージの両方を同じフィードから復元できるため、構成が簡略化されます。
   
   - **スコープ:** `Project: Contoso.Microservices`
1. **[作成]** を選択します
1. 作成後、**[フィードに接続]** を選択します
1. [NuGet] セクションで **[dotnet]** を選択します
1. **Artifacts の URL** (表示されている nuget.config xml スニペットの値パラメーター) をコピーします。 次のように表示されます。

   ```
   https://pkgs.dev.azure.com/<your-org>/Contoso.Microservices/_packaging/contoso-internal/nuget/v3/index.json

   ```

1. **戻る**矢印をクリックして、メインの Artifacts フィード ページに戻ります。

> **注**: 既定では、"<DevOps Project> ビルド サービス" ユーザーには、**Artifact フィード コラボレーター ([フィードとアップストリームの閲覧者])** のアクセス許可があります。 後でビルド パイプラインを実行するときに、パッケージを_使用する_だけでなく、_公開_するために、アクセス許可を更新する必要があります。

1. **Artifacts** / **フィード** ページ (_開始するにはフィードに接続してください_と記載されたページ) から、**フィードの設定** (小さな歯車) を選択し、**[アクセス許可]** タブに移動します
1. **[Contoso.MicroServices Build Service (ADO 組織)]** ユーザーを選択します
1. **[編集]** をクリックします
1. アクセス許可を [フィードとアップストリームの閲覧者] から **[フィード公開元 (共同作成者)]** に変更します
1. **変更を保存する**

## タスク 3: 共有ライブラリ プロジェクトを作成する

現実的な共有ユーティリティを使用して.NET 10 クラス ライブラリを作成します。

### ソリューション構造を初期化する

1. ターミナルを開き、次のようにしてプロジェクト ディレクトリを作成します。
   ```powershell
   mkdir C:\ContosoMicroservices
   cd C:\ContosoMicroservices

   ```

1. 次のようにして新しいソリューションとクラス ライブラリを作成します。
   ```powershell
   dotnet nuget add source https://api.nuget.org/v3/index.json --name nuget.org
   dotnet new sln --name Contoso.Shared
   dotnet new classlib --name Contoso.Shared.Core --framework net10.0
   dotnet sln add Contoso.Shared.Core
   dotnet new gitignore

   ```

1. VS Code でプロジェクトを開きます。
   ```powershell
   code .

   ```

### パッケージ メタデータを追加する

1. `Contoso.Shared.Core/Contoso.Shared.Core.csproj` ファイルを開いて、その内容を次に置き換えます。

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    
    <!-- Package metadata -->
    <PackageId>Contoso.Shared.Core</PackageId>
    <Version>1.0.0</Version>
    <Authors>Contoso DevOps Team</Authors>
    <Company>Contoso Retail</Company>
    <Description>Core shared utilities for Contoso microservices including API response models, logging helpers, and common extensions.</Description>
    <PackageTags>contoso;shared;utilities;microservices</PackageTags>
    <RepositoryType>git</RepositoryType>
  </PropertyGroup>

</Project>
```

### API 応答モデルを作成する

エンタープライズ アプリケーションでは、API 応答を標準化することで、すべてのマイクロサービスの一貫性が確保されます。 次の `ApiResponse<T>` クラスは、成功または失敗状態、分散トレースの相関関係 ID、標準化されたエラー情報を含めてすべての応答をラップします。

1. `Contoso.Shared.Core` フォルダー内の既定の `Class1.cs` ファイルを削除します
1. `Contoso.Shared.Core` の内部に `Models` という名前の新しいフォルダーを作成します
1. 次のコードを使用して新しいファイル `Models/ApiResponse.cs` を作成します。

```csharp
namespace Contoso.Shared.Core.Models;

public class ApiResponse<T>
{
    public bool Success { get; set; }
    public T? Data { get; set; }
    public ApiError? Error { get; set; }
    public string CorrelationId { get; set; } = Guid.NewGuid().ToString();
    public DateTime Timestamp { get; set; } = DateTime.UtcNow;

    public static ApiResponse<T> Ok(T data, string? correlationId = null)
    {
        return new ApiResponse<T>
        {
            Success = true,
            Data = data,
            CorrelationId = correlationId ?? Guid.NewGuid().ToString()
        };
    }

    public static ApiResponse<T> Fail(string errorCode, string message, string? correlationId = null)
    {
        return new ApiResponse<T>
        {
            Success = false,
            Error = new ApiError(errorCode, message),
            CorrelationId = correlationId ?? Guid.NewGuid().ToString()
        };
    }
}

public record ApiError(string Code, string Message)
{
    public string? Details { get; init; }
    public Dictionary<string, string[]>? ValidationErrors { get; init; }
}
```

### ログ拡張機能を作成する

`LogContext` クラスは、マイクロサービス間で一貫性のあるログ エントリと分散トレースを実現する、構造化されたログ コンテキストを提供します。 マルチテナント アプリケーションの相関関係 ID、サービス名、およびオプションのユーザーまたはテナント情報がキャプチャされます。

1. `Contoso.Shared.Core` の内部に `Logging` という名前の新しいフォルダーを作成します
1. 新しい `Logging/LogContext.cs` ファイルを作成します。

```csharp
namespace Contoso.Shared.Core.Logging;

public class LogContext
{
    public string CorrelationId { get; set; } = Guid.NewGuid().ToString();
    public required string ServiceName { get; set; }
    public string? Operation { get; set; }
    public string? UserId { get; set; }
    public string? TenantId { get; set; }

    public Dictionary<string, object?> ToDictionary()
    {
        return new Dictionary<string, object?>
        {
            ["CorrelationId"] = CorrelationId,
            ["ServiceName"] = ServiceName,
            ["Operation"] = Operation,
            ["UserId"] = UserId,
            ["TenantId"] = TenantId,
            ["Timestamp"] = DateTime.UtcNow.ToString("O")
        };
    }
}
```

### 文字列拡張ユーティリティを作成する

これらの拡張メソッドは、サービス間で使用される共通の文字列操作を提供します。表示目的の `Truncate`、メールやクレジット カードなどの機密データを非表示にする `Mask`、URL に適した文字列にするための `ToSlug` などがあります。

1. `Contoso.Shared.Core` の内部に `Extensions` という名前の新しいフォルダーを作成します
1. 新しい `Extensions/StringExtensions.cs` ファイルを作成します。

```csharp
namespace Contoso.Shared.Core.Extensions;

public static class StringExtensions
{
    public static string Truncate(this string value, int maxLength, string suffix = "...")
    {
        if (string.IsNullOrEmpty(value)) return value;
        if (maxLength <= 0) return string.Empty;
        if (value.Length <= maxLength) return value;

        return string.Concat(value.AsSpan(0, maxLength - suffix.Length), suffix);
    }

    public static string Mask(this string value, int visibleChars = 4, char maskChar = '*')
    {
        if (string.IsNullOrEmpty(value)) return value;
        if (value.Length <= visibleChars * 2) return new string(maskChar, value.Length);

        var start = value[..visibleChars];
        var end = value[^visibleChars..];
        var masked = new string(maskChar, value.Length - (visibleChars * 2));

        return $"{start}{masked}{end}";
    }

    public static string ToSlug(this string value)
    {
        if (string.IsNullOrEmpty(value)) return value;

        return value
            .ToLowerInvariant()
            .Replace(" ", "-")
            .Replace("_", "-");
    }
}
```

### プロジェクトをビルドする

1. ターミナルで、次のようにしてソリューションをビルドします。
   ```powershell
   dotnet build

   ```
1. ビルド エラーがないことを確認します

### コードを Azure Repos にプッシュします

共有ライブラリが作成されたら、バージョン コントロールのために Azure Repos にプッシュします。 これは、チームでのコラボレーションを可能にし、後で CI/CD 自動化の際に基礎となります。

1. Azure DevOps で、`Contoso.Microservices` プロジェクトの **Repos** に移動します
1. リポジトリが空であるため、セットアップ手順が表示されます。 **クローン URL** (HTTPS) をコピーすると、次のようになります。

   ```
   https://dev.azure.com/<your-org>/Contoso.Microservices/_git/Contoso.Microservices
   ```

1. ターミナルで Git を初期化し、コードをプッシュし、**DevOps 組織名**で URL を更新します。

   ```powershell
   cd C:\ContosoMicroservices
   git init
   git add .
   git commit -m "Initial commit: Contoso.Shared.Core library"
   git remote add origin https://dev.azure.com/<your-org>/Contoso.Microservices/_git/Contoso.Microservices
   git push -u origin main

   ```

   > **注:** 認証を求められる場合があります。 Azure DevOps 資格情報を使用します。

1. Azure DevOps で、**Repos** ページを更新して、コードが表示されていることを確認します

## タスク 4: パッケージを公開するための CI パイプラインを作成する

DevOps 環境では、パッケージは手動ではなく CI パイプラインを介して自動的に公開されます。 これにより、一貫性、追跡可能性、適切なバージョン コントロールが保証されます。

### パイプライン ファイルを作成する

1. VS Code で、ソリューション ルート (`C:\ContosoMicroservices`) に新しいファイル `azure-pipelines.yml` を作成します。

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - Contoso.Shared.Core/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  projectPath: 'Contoso.Shared.Core/Contoso.Shared.Core.csproj'

stages:
- stage: Build
  displayName: 'Build and Pack'
  jobs:
  - job: BuildJob
    displayName: 'Build Library'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET 10 SDK'
      inputs:
        packageType: 'sdk'
        version: '10.x'

    - task: DotNetCoreCLI@2
      displayName: 'Restore packages'
      inputs:
        command: 'restore'
        projects: '$(projectPath)'

    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        projects: '$(projectPath)'
        arguments: '--configuration $(buildConfiguration) --no-restore'

    - task: DotNetCoreCLI@2
      displayName: 'Pack NuGet package'
      inputs:
        command: 'pack'
        packagesToPack: '$(projectPath)'
        configuration: '$(buildConfiguration)'
        packDirectory: '$(Build.ArtifactStagingDirectory)'

    - publish: '$(Build.ArtifactStagingDirectory)'
      artifact: 'nuget-package'
      displayName: 'Publish artifact'

- stage: Publish
  displayName: 'Publish to Azure Artifacts'
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - job: PublishJob
    displayName: 'Push to Feed'
    steps:
    - download: current
      artifact: 'nuget-package'

    - task: NuGetAuthenticate@1
      displayName: 'Authenticate to Azure Artifacts'

    - task: DotNetCoreCLI@2
      displayName: 'Push to contoso-internal feed'
      inputs:
        command: 'push'
        packagesToPush: '$(Pipeline.Workspace)/nuget-package/*.nupkg'
        nuGetFeedType: 'internal'
        publishVstsFeed: 'Contoso.Microservices/contoso-internal'
```

このパイプラインは、変更が `Contoso.Shared.Core` フォルダー内の `main` ブランチにプッシュされたときにトリガーされます。 NuGet パッケージがビルドされ、パックされ、Azure Artifacts フィードに自動的に公開されます。

> **主な利点:** `NuGetAuthenticate@1` タスクで認証が自動的に処理されます。パイプラインの公開に個人用アクセス トークンは必要ありません。

### パイプラインをコミットしてプッシュし、トリガーする

1. 次のようにしてパイプライン ファイルを追加してプッシュします。

   ```powershell
   cd C:\ContosoMicroservices
   git add azure-pipelines.yml
   git commit -m "Add CI pipeline for package publishing"
   git push

   ```

### Azure DevOps でパイプラインを作成する

1. Azure DevOps で、**[パイプライン]** に移動します
1. **[パイプラインの作成]** (または **[新しいパイプライン]**) を選択します
1. **[Azure Repos Git]** を選択します
1. **[Contoso.Microservices]** リポジトリを選択します
1. `/azure-pipelines.yml` が自動的に読み込まれます
1. [実行] をクリックします****

1. 次のようにして、パイプラインが両方のステージで実行されるのを確認します。
   - **[ビルドしてパック]**: ライブラリがコンパイルされ、NuGet パッケージが作成されます
   - **[Azure Artifacts に公開]**: パッケージがフィードにプッシュされます

1. 完了したら、**[Artifacts]** > **[contoso-internal]** に移動します
1. `Contoso.Shared.Core` バージョン `1.0.0` がフィードに表示されることを確認します

> **DevOps のベスト プラクティス:** パイプラインを介してパッケージを公開すると、すべてのパッケージ バージョンが特定のコミットとビルドにリンクされ、完全な追跡可能性が得られます。

## タスク 5: マイクロサービスでパッケージを実行する

パッケージがパイプラインを介して公開されたので、それを実行するマイクロサービスを作成します。

### 注文サービス プロジェクトを作成する

1. ターミナルで、ソリューション フォルダーに移動し、次のようにして新しい Web API プロジェクトを作成します。

   ```powershell
   cd C:\ContosoMicroservices
   dotnet nuget add source https://api.nuget.org/v3/index.json --name nuget.org
   dotnet new webapi --name Contoso.OrderService --framework net10.0 --use-controllers
   dotnet sln add Contoso.OrderService

   ```

### Azure Artifacts を使用するように NuGet を構成する

ローカル開発と CI/CD の両方のパイプラインで Azure Artifacts フィードからパッケージを復元できるように、`nuget.config` ファイルを作成します。

1. ソリューション ルート (`C:\ContosoMicroservices`) に `nuget.config` を作成します: (この xml は、Artifacts フィード/フィードに接続/Nuget/dotnet - プロジェクト設定セクションでも確認できます)

   ```xml
   <?xml version="1.0" encoding="utf-8"?>
   <configuration>
     <packageSources>
       <clear />
       <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
       <add key="contoso-internal" value="https://pkgs.dev.azure.com/<your-org>/Contoso.Microservices/_packaging/contoso-internal/nuget/v3/index.json" />
     </packageSources>
   </configuration>
   ```

   > **注:** `<your-org>` は Azure DevOps 組織名に置き換えてください。

### 共有パッケージをインストールする

> **重要:** 続行する前に、タスク 4 のパイプラインが正常に完了し、パッケージがフィードに存在することをご確認ください。

> 1. Azure DevOps で、**[Artifacts]** > **[contoso-internal]** に移動します
> 2. パッケージが存在しない場合は、`Contoso.Shared.Core` バージョン `1.0.0` が一覧に表示されていることを確認し、**[パイプライン]** のパイプライン実行にエラーがないかどうかを確認します。

1. まだインストールしていない場合は、Azure Artifacts 資格情報プロバイダーをインストールします (これにより、ローカル開発の認証が有効になります)。
   - [https://github.com/microsoft/artifacts-credprovider/releases](https://github.com/microsoft/artifacts-credprovider/releases) から最新リリースをダウンロードします
   - `Microsoft.Net.Providers.CredentialProvider.zip` (.NET 用) をダウンロードします 
   - zip ファイルを `%USERPROFILE%\.nuget\plugins` に展開します (フォルダーが存在しない場合は作成してください)
   
   または、PowerShell コマンドが環境内で動作する場合は、次のようにします。

   ```powershell
   iex "& { $(irm https://aka.ms/install-artifacts-credprovider.ps1) }"

   ```

1. パッケージ参照を注文サービスに追加します。 VS Code で `Contoso.OrderService/Contoso.OrderService.csproj` を開き、既存の `ItemGroup` タグに次の行を追加します。

   ```xml
    <PackageReference Include="Contoso.Shared.Core" Version="1.0.0" />
   ```

1. 次に、同じ `Contoso.OrderService.csproj` で、[PackageReference ItemGroup] セクションの下に次の新しい [ItemGroup] セクションを追加して、Contoso.Shared.Core へのプロジェクト参照を追加します。

    ```xml
    <ItemGroup>
    <ProjectReference Include="..\Contoso.Shared.Core\Contoso.Shared.Core.csproj" />
    </ItemGroup>
    ```

1. 次のようにして、プロジェクトを復元してビルドします。

   ```powershell
   cd C:\ContosoMicroservices
   dotnet restore --interactive
   dotnet build

   ```

   > **注:** 初めて復元するときは、ブラウザー ウィンドウで Azure DevOps を認証するように求められます。 アカウントを選択し、アクセスを許可します。 資格情報プロバイダーが、将来使用するために資格情報をキャッシュします。

### 共有ライブラリを使用する

共有パッケージがインストールされたら、それを使用するコントローラーを作成します。 次のコントローラーは、実行するサービスがどのように共有ライブラリを使用するかを示しています。 `ApiResponse<T>` がどのようにして一貫した応答の書式設定を提供し、`Mask` と `Truncate` の拡張メソッドが機密データと表示書式を処理するかをご確認ください。

1. プロジェクトから `Controllers/WeatherForecastController.cs` と `WeatherForecast.cs` を削除します
1. 次のコードを使用して新しいファイル `Controllers/OrdersController.cs` を作成します。

```csharp
using Contoso.Shared.Core.Models;
using Contoso.Shared.Core.Extensions;
using Microsoft.AspNetCore.Mvc;

namespace Contoso.OrderService.Controllers;

[ApiController]
[Route("api/[controller]")]
public class OrdersController : ControllerBase
{
    [HttpGet("{id}")]
    public ActionResult<ApiResponse<OrderDto>> GetOrder(int id)
    {
        if (id <= 0)
        {
            return BadRequest(ApiResponse<OrderDto>.Fail(
                "INVALID_ORDER_ID",
                "Order ID must be a positive number"));
        }

        if (id == 999)
        {
            return NotFound(ApiResponse<OrderDto>.Fail(
                "ORDER_NOT_FOUND",
                $"Order with ID {id} was not found"));
        }

        var order = new OrderDto
        {
            OrderId = id,
            CustomerName = "John Doe",
            CustomerEmail = "john.doe@example.com".Mask(3),
            TotalAmount = 299.99m,
            Status = "Processing",
            Description = "This is a sample order with a very long description that should be truncated".Truncate(50)
        };

        return Ok(ApiResponse<OrderDto>.Ok(order));
    }

    [HttpGet]
    public ActionResult<ApiResponse<List<OrderDto>>> GetOrders()
    {
        var orders = new List<OrderDto>
        {
            new() { OrderId = 1, CustomerName = "John Doe", TotalAmount = 299.99m, Status = "Completed" },
            new() { OrderId = 2, CustomerName = "Jane Smith", TotalAmount = 149.50m, Status = "Processing" }
        };

        return Ok(ApiResponse<List<OrderDto>>.Ok(orders));
    }
}

public class OrderDto
{
    public int OrderId { get; set; }
    public string CustomerName { get; set; } = string.Empty;
    public string? CustomerEmail { get; set; }
    public decimal TotalAmount { get; set; }
    public string Status { get; set; } = string.Empty;
    public string? Description { get; set; }
}
```

1. 次のようにして、プロジェクトをビルドして実行します。

   ```powershell
   cd c:\ContosoMicroServices\Contoso.OrderService
   dotnet run

   ```

1. ターミナルに表示されている **http://localhost:port>** の URL を開いて API をテストします。

   - `/api/orders/1` に移動すると、成功応答が返されるはずです
   - 共有拡張機能を使用してメールがどのようにマスクされ、説明が切り捨てられるかをご確認ください
   - `/api/orders/999` に移動すると、ORDER_NOT_FOUND エラーが返されるはずです

## タスク 6: パッケージを更新する (バグ修正のシミュレーション)

チームから、`Mask` 拡張メソッドがエッジ ケースを適切に処理しないことが報告されました。 バグを修正し、バージョンを更新し、パイプラインを介して更新プログラムを公開しましょう。

### 修正を行う

更新されたメソッドでは、`minMaskedChars` パラメーターを追加して、短い文字列が過剰にデータを公開せず、適切にマスクされるようになります。

1. VS Code で、`Contoso.Shared.Core/Extensions/StringExtensions.cs` を開きます
1. `Mask` メソッドを更新します。

> **注**: 他のコード スニペットは変更しないでください。 また、コード スニペットのインデントも考慮してください。

```csharp
public static string Mask(this string value, int visibleChars = 4, char maskChar = '*', int minMaskedChars = 3)
{
    if (string.IsNullOrEmpty(value)) return value;
    if (visibleChars < 0) visibleChars = 0;
    
    if (value.Length <= visibleChars * 2 + minMaskedChars)
    {
        return new string(maskChar, Math.Max(value.Length, minMaskedChars));
    }

    var start = value[..visibleChars];
    var end = value[^visibleChars..];
    var maskedLength = Math.Max(value.Length - (visibleChars * 2), minMaskedChars);
    var masked = new string(maskChar, maskedLength);

    return $"{start}{masked}{end}";
}
```

> **この修正プログラムにより、何が行われますか?** 元の v1.0.0 `Mask` メソッドでは、短い文字列でエッジ ケースの問題が発生しました。 たとえば、既定の `visibleChars=4` では、"ab@c.io" (7 文字) などの短いメールでは、先頭の 4 文字、末尾の 4 文字が表示され、文字列全体が公開されていました。 この修正により、`minMaskedChars` パラメーターが追加され、少なくとも 3 文字が常にマスクされます。 これで、機密データが表示されず、短い文字列が完全にマスクされるようになりました (たとえば、`*******`)。

### バージョンを更新する

これは[セマンティック バージョニング](https://semver.org/)に続くパッチ リリース (破壊的変更のないバグ修正) です。

1. `Contoso.Shared.Core/Contoso.Shared.Core.csproj` ファイルを開きます
1. 次のようにしてバージョンを更新し、リリース ノートを追加します。

> **注**: `<PropertyGroup>` セクションのみを上書きし、`<Project>` タグは配置したままにしてください。

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    
    <PackageId>Contoso.Shared.Core</PackageId>
    <Version>1.0.1</Version>
    <Authors>Contoso DevOps Team</Authors>
    <Company>Contoso Retail</Company>
    <Description>Core shared utilities for Contoso microservices including API response models, logging helpers, and common extensions.</Description>
    <PackageTags>contoso;shared;utilities;microservices</PackageTags>
    <RepositoryType>git</RepositoryType>
    <PackageReleaseNotes>
      v1.0.1: Fixed edge case in Mask extension, added minMaskedChars parameter
    </PackageReleaseNotes>
</PropertyGroup>
```

### パイプラインを介して公開する

1. 次のようにして変更をコミットしてプッシュし、パイプラインをトリガーします。

   ```powershell
   cd C:\ContosoMicroservices
   git add .
   git commit -m "Fix: Handle edge cases in Mask extension (v1.0.1)"
   git push

   ```

1. Azure DevOps で、**[パイプライン]** に移動し、ビルドの実行を監視します
1. パイプラインが完了したら、**[Artifacts]** > **[contoso-internal]** に移動します
1. **[Contoso.Shared.Core]** をクリックしてパッケージの詳細を開きます
1. **[バージョン]** タブを選択すると、使用可能なすべてのバージョンが表示されます。`1.0.0` と `1.0.1` の両方が表示されるはずです

### 注文サービスを更新する

1. 注文サービスでパッケージ参照を更新します。 `Contoso.OrderService/Contoso.OrderService.csproj` を開き、次のようにしてバージョンを変更します。

   ```xml
   <PackageReference Include="Contoso.Shared.Core" Version="1.0.1" />
   ```

1. 復元とビルド:

   ```powershell
   cd C:\ContosoMicroservices
   dotnet restore
   dotnet build

   ```

> **注**: この手順を実行するときに "エラー NU1102: パッケージが見つかりません" というエラー メッセージが表示された場合は、次のコマンドを使用して Nuget のローカル キャッシュをクリアし、dotnet restore を再度実行することで解決します (このエラーの詳細については、**[こちら](https://learn.microsoft.com/en-us/nuget/consume-packages/managing-the-global-packages-and-cache-folders)** をご覧ください)。

  ```powershell
   cd C:\ContosoMicroservices
   dotnet nuget locals http-cache --clear
   dotnet restore

   ```


1. 注文サービスを実行して、更新されたパッケージが動作していることを確認します。

   ```powershell
   cd C:\ContosoMicroservices\Contoso.OrderService
   dotnet run

   ```

1. API アプリ (ターミナルに表示される URL) を開き、`/api/orders/1` エンドポイントをテストします。 応答にはまだマスクされたメールが表示され、更新されたパッケージが新しいエッジ ケース処理で正しく動作することを確認できます。

1. 実行中のアプリケーションを停止します (ターミナルで `Ctrl+C` キーを押します)

## タスク 7: 注文サービスのパイプラインを作成する

注文サービスのパイプラインを作成して、DevOps ワークフローを完了します。 これは、実行中のアプリケーションがどのようにして CI/CD 中に Azure Artifacts からパッケージを復元するかを示しています。

### パイプライン ファイルを作成する

1. ソリューション ルートに新しいファイル `azure-pipelines-orderservice.yml` を作成します。

```yaml
trigger:
  branches:
    include:
      - main
  paths:
    include:
      - Contoso.OrderService/**

pool:
  vmImage: 'ubuntu-latest'

variables:
  buildConfiguration: 'Release'
  projectPath: 'Contoso.OrderService/Contoso.OrderService.csproj'

stages:
- stage: Build
  displayName: 'Build Order Service'
  jobs:
  - job: BuildJob
    displayName: 'Build and Test'
    steps:
    - task: UseDotNet@2
      displayName: 'Use .NET 10 SDK'
      inputs:
        packageType: 'sdk'
        version: '10.x'

    - task: NuGetAuthenticate@1
      displayName: 'Authenticate to Azure Artifacts'

    - task: DotNetCoreCLI@2
      displayName: 'Restore packages (including from Azure Artifacts)'
      inputs:
        command: 'restore'
        projects: '$(projectPath)'
        feedsToUse: 'config'
        nugetConfigPath: 'nuget.config'

    - task: DotNetCoreCLI@2
      displayName: 'Build'
      inputs:
        command: 'build'
        projects: '$(projectPath)'
        arguments: '--configuration $(buildConfiguration) --no-restore'

    - task: DotNetCoreCLI@2
      displayName: 'Publish'
      inputs:
        command: 'publish'
        projects: '$(projectPath)'
        arguments: '--configuration $(buildConfiguration) --output $(Build.ArtifactStagingDirectory)'
        publishWebProjects: false

    - publish: '$(Build.ArtifactStagingDirectory)'
      artifact: 'orderservice'
      displayName: 'Publish artifact'
```

このパイプラインは、`NuGetAuthenticate` タスクが、前に作成した `nuget.config` ファイルを使用して、ビルド エージェントがプライベートの Azure Artifacts フィードからどのようにパッケージを復元できるようにするかを示しています。

### パイプラインを作成する

1. 次のようにしてすべての変更をコミットしてプッシュします。

   ```powershell
   cd C:\ContosoMicroServices
   git add .
   git commit -m "Add Order Service pipeline and update nuget.config"
   git push

   ```

1. Azure DevOps で、**[パイプライン]** に移動します
1. **[新しいパイプライン]** を選択します
1. **[Azure Repos Git]** > **[Contoso.Microservices]** を選択します
1. **[既存の Azure Pipelines YAML ファイル]** を選択します。
1. 以下を選択します。
   - [ブランチ]: **Main**
   - パス: **`/azure-pipelines-orderservice.yml`**
1. **[続行]** を選択します
1. **[実行]** を選択してパイプラインを開始します
1. 実行中のパイプラインから、**[注文サービスのビルド]** ステージを選択して、**[ジョブの詳細]** を開きます。
1. パイプラインで発生しているいくつかの手順を次のようにして確認します。
   - ローカルで使用されているのと同じ Azure Artifacts 資格情報プロバイダーを使用して Azure Artifacts を認証します
   - プライベート フィードから `Contoso.Shared.Core` を復元します
   - 注文サービスをビルドして公開します

> **DevOps のベスト プラクティス:** CI/CD パイプラインで Azure Artifacts を使用することで、すべてのビルドで一貫性のあるバージョン管理された依存関係が使用されるようになります。 これにより、"自分のマシンでは動く" という問題が解消され、どのパッケージ バージョンがデプロイされているかを完全に追跡できます。

## リソースをクリーンアップする

このラボ専用で作成したリソースが不要になった場合:

1. Azure DevOps で、自分のプロジェクトに移動し、**[プロジェクトの設定]** を選択します
1. **[全般]** で **[概要]** を選択します
1. ページの下部で **[削除]** を選択します
1. 確認するプロジェクト名を入力します
1. **[削除]** を選択します

または、将来のラボや実験のためにプロジェクトを保持します。

## まとめ

このラボでは、DevOps エンジニアが Azure DevOps を使用してマイクロサービス アーキテクチャの内部パッケージを作成して管理する実際のシナリオをシミュレートしました。
