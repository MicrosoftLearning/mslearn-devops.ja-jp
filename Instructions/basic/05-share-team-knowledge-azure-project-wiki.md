---
lab:
  topic: Basic
  title: Azure プロジェクトの Wiki を使用してチームの知識を共有する
  description: Markdown のコンテンツの管理や Mermaid ダイアグラムの作成など、Azure DevOps で Wiki を作成および構成する方法について説明します。
---

# Azure プロジェクトの Wiki を使用してチームの知識を共有する

**推定時間**: 45 分

Markdown コンテンツの管理や Mermaid ダイアグラムの作成など、Azure DevOps で Wiki を作成および構成する方法について説明します。 Azure DevOps Wiki は、チームが知識を共有し、プロセスを文書化し、プロジェクト情報を維持するための一元化された場所を提供します。

## 開始する前に

必要なもの:

- **Microsoft Edge** または [Azure DevOps 対応ブラウザー](https://docs.microsoft.com/azure/devops/server/compatibility)
- **Azure DevOps 組織:** お持ちでない場合は作成してください
- **eShopOnWeb プロジェクト:** 以前のラボのサンプル プロジェクトを使用するか、新しいプロジェクトを作成します

## Azure DevOps Wiki の概要

Azure DevOps には、次の 2 種類の Wiki が用意されています。

1. **プロジェクト Wiki** - リポジトリとは別に存在する Wiki
1. **コード Wiki** - Git リポジトリに格納されているコンテンツから作成された Wiki

主な機能:

- 豊富な書式設定機能を備えた **Markdown サポート**
- フローチャートやシーケンス ダイアグラムを作成するための **Mermaid ダイアグラム**
- ドラッグ アンド ドロップ機能を備えた**画像サポート**
- 変更履歴を備えた**バージョン管理**
- 作業項目、コード、他の Wiki への**相互参照リンク**
- 同時ユーザー サポートを備えた**共同編集**

## プロジェクトとリポジトリを設定する

まず、このラボ用の eShopOnWeb プロジェクトの準備ができていることを確認します。

### Azure DevOps 組織を設定する (必要な場合)

Azure DevOps 組織がまだない場合は、次の手順に従ってください。

1. プライベート ブラウザー セッションを使って、`https://account.microsoft.com` で新しい**個人用 Microsoft アカウント (MSA)** を取得します (既にお持ちの場合はスキップしてください)。
1. 同じブラウザー セッションを使って、`https://azure.microsoft.com/free` で無料の Azure サブスクリプションにサインアップします (既にお持ちの場合はスキップしてください)。
1. ブラウザーを開き、`https://portal.azure.com` の Azure portal に移動し、Azure portal 画面の上部で **Azure DevOps** を検索します。 表示されたページで、**[Azure DevOps 組織]** を選択します。
1. 次に、**[自分の Azure DevOps 組織]** というラベルの付いたリンクを選択するか、`https://aex.dev.azure.com` に直接移動します。
1. **[We need a few more details](詳細情報をいくつか入力する必要があります)** ページで、 **[続行]** を選びます。
1. 左側のドロップダウン ボックスで、**Microsoft アカウント**の代わりに **[既定のディレクトリ]** を選びます。
1. "詳細情報をいくつか入力する必要があります" というメッセージが表示されたら、名前、メール アドレス、場所を入力して、**[続行]** を選択します。__
1. **[既定のディレクトリ]** を選択して `https://aex.dev.azure.com` に戻り、青い **[新しい組織の作成]** ボタンを選択します。
1. **[続行]** を選択して「サービス使用条件」を承諾します。__
1. プロンプト ( _[Almost done](ほぼ完了)_ ) が表示されたら、Azure DevOps 組織の名前を既定のままにし (グローバルに一意の名前である必要があります)、一覧から最寄りのホスティング場所を選びます。
1. 新しく作成した組織が **Azure DevOps** で開いたら、左下隅にある **[組織設定]** を選びます。
1. **[組織設定]** 画面で、**[課金情報]** を選びます (この画面を開くには数秒かかります)。
1. **[課金の設定]** を選び、画面の右側で **[Azure サブスクリプション]**、**[保存]** の順に選んでサブスクリプションを組織にリンクします。
1. 画面の上部にリンクされた Azure サブスクリプション ID が表示されたら、**MS Hosted CI/CD** の**有料並列ジョブ**の数を 0 から **1** に変更します。 次に、下部にある **[保存]** を選びます。

   > **注**: **CI/CD 機能を使用する前に数分間待つ**と、新しい設定がバックエンドに反映されます。 それができなかった場合、 _"ホストされた並列処理は購入も許可もされていません"_ というメッセージが依然として表示されます。

1. **[組織の設定]** の **[パイプライン]** セクションに移動して、**[設定]** を選択します。
1. **[クラシック ビルド パイプラインの作成を無効にする]** と **[クラシック リリース パイプラインの作成を無効にする]** を **[オフ]** に切り替えます。
1. **[組織の設定]** で、**[セキュリティ]** セクションに移動して **[ポリシー]** を選択します。
1. **[パブリック プロジェクトを許可します]** を **[オン]** に切り替えます。

### Azure DevOps プロジェクトを作成して構成する (必要な場合)

1. ブラウザーを開き、Azure DevOps 組織に移動します。
1. **[新しいプロジェクト]** オプションを選び、次の設定を使用します。
   - 名前: **eShopOnWeb**
   - 可視性: **プライベート**
   - 詳細設定: バージョン コントロール: **Git**
   - 詳細設定: 作業項目プロセス: **スクラム**
1. **［作成］** を選択します

   ![新しいプロジェクトの作成パネルのスクリーンショット。](media/create-project.png)

### eShopOnWeb Git リポジトリをインポートする (必要な場合)

1. 前に作成した **eShopOnWeb** プロジェクトを開きます。
1. **[Repos] > [ファイル]** を選び、**[リポジトリをインポートする]**、**[インポート]** の順に選びます。
1. **[Git リポジトリをインポートする]** ウィンドウで、URL `https://github.com/MicrosoftLearning/eShopOnWeb.git` を貼り付けて、**[インポート]** を選びます。

   ![リポジトリのインポート パネルのスクリーンショット。](media/import-repo.png)

1. リポジトリは次のように編成されています。

   - **.ado** フォルダーには、Azure DevOps の YAML パイプラインが含まれています。
   - **.devcontainer** フォルダーには、コンテナーを使って開発するためのセットアップが含まれています (VS Code でローカルに、または GitHub Codespaces で)。
   - **.azure** フォルダーには、Bicep & ARM インフラストラクチャがコード テンプレートとして含まれています。
   - **.github** フォルダーには、YAML GitHub ワークフローの定義が含まれています。
   - **src** フォルダーには、ラボ シナリオで使用される .NET 8 Web サイトが含まれています。

1. Web ブラウザーのウィンドウは開いたままにしておきます。
1. **[リポジトリ]、[ブランチ]** に移動します。
1. **main** ブランチをポイントし、列の右側にある省略記号を選択します。
1. **[既定のブランチとして設定]** を選択します。

### 後で使用するためにブランド画像をダウンロードする

1. **[ファイル]** ペインで **src** フォルダーを展開し、**Web > wwwroot > images** サブフォルダーに移動します
1. **Images** サブフォルダーにある **brand.png** エントリを見つけます
1. 右端にマウス ポインターを移動すると、縦型の省略記号 (3 つのドット) メニューが表示されます
1. **[ダウンロード]** を選択して **brand.png** ファイルをローカル コンピューターにダウンロードします

> **注**:このイメージは次の演習で使用します。

### ドキュメント フォルダーを作成する

1. **Repos** 内から、**[ファイル]** を選択します
1. フォルダー構造の上部に **eShopOnWeb** というリポジトリ タイトルがあることに注意してください
1. **省略記号 (3 つのドット) を選択し**、**[新規] > [フォルダー]** を選択します
1. 新しいフォルダー名のタイトルとして「**`Documents`**」を入力します
1. リポジトリでは空のフォルダーが許可されていないため、新しいファイル名として「**`README.md`**」を指定します
1. **[作成]** を選択して作成を確定します
1. README.md ファイルが閲覧モードで開きます
1. **[コミット]** ボタンを選択して変更を保存します
1. [コミット] ウィンドウで、**[コミット]** を押して確定します

## コードを Wiki として公開する

Git リポジトリのコンテンツを Wiki として公開できます。 これは、コードと一緒にドキュメントを保守するのに役立ちます。

### ブランチを Wiki として公開する

1. 左側の Azure DevOps 縦型メニューで **[概要]** を選択します
1. **[概要]** セクションで **[Wiki]** を選択します
1. **[Wiki としてコードを公開する]** を選択します
1. **[Wiki としてコードを公開する]** ペインで、次の設定を指定して **[公開]** を選択します。

   | 設定    | 値                        |
   | ---------- | ---------------------------- |
   | リポジトリ | **eShopOnWeb**               |
   | [Branch]\(ブランチ)     | **main**                     |
   | Folder     | **/Documents**               |
   | Wiki 名  | **`eShopOnWeb (Documents)`** |

これにより、エディターで Wiki セクションが自動的に開きます。

> **注**:エディターが自動的に開かない場合は、エクスプローラーの下部にある **[+ 新しいページ]** を選択します。

### Wiki コンテンツを作成する

1. Wiki ページの **[タイトル]** フィールドに「`Welcome to our Online Retail Store!`」と入力します

1. Wiki ページの本文に次のコンテンツを貼り付けます。

   ```markdown
   ## Welcome to Our Online Retail Store!

   At our online retail store, we offer a **wide range of products** to meet the **needs of our customers**. Our selection includes everything from _clothing and accessories to electronics, home decor, and more_.

   We pride ourselves on providing a seamless shopping experience for our customers. Our website offers the following benefits:

   1. user-friendly,
   1. and easy to navigate,
   1. allowing you to find what you're looking for,
   1. quickly and easily.

   We also offer a range of **_payment and shipping options_** to make your shopping experience as convenient as possible.

   ### About the team

   Our team is dedicated to providing exceptional customer service. If you have any questions or concerns, our knowledgeable and friendly support team is always available to assist you. We also offer a hassle-free return policy, so if you're not completely satisfied with your purchase, you can easily return it for a refund or exchange.

   ### Physical Stores

   | Location    | Area                  | Hours           |
   | ----------- | --------------------- | --------------- |
   | New Orleans | Home and DIY          | 07.30am-09.30pm |
   | Seattle     | Gardening             | 10.00am-08.30pm |
   | New York    | Furniture Specialists | 10.00am-09.00pm |

   ## Our Store Qualities

   - We're committed to providing high-quality products
   - Our products are offered at affordable prices
   - We work with reputable suppliers and manufacturers
   - We ensure that our products meet our strict standards for quality and durability.
   - Plus, we regularly offer sales and discounts to help you save even more.

   # Summary

   Thank you for choosing our online retail store for your shopping needs. We look forward to serving you!
   ```

1. このサンプル テキストは、いくつかの一般的な Markdown 構文機能を示しています。

   - **タイトルとサブタイトル** (## と ###)
   - **太字テキスト** (\*\*)
   - **斜体テキスト** (\*)
   - **番号付きリスト** (1.)
   - **箇条書きリスト** (-)
   - ヘッダーとデータを含む**テーブル**

1. 完了したら、右上隅の **[保存]** ボタンを押します

1. ブラウザーを**更新**するか、他の DevOps ポータル オプションを選択して Wiki セクションに戻ります

1. **eShopOnWeb (ドキュメント)** Wiki が表示され、**ホームページ**として **[Welcome to our Online Retail Store]** が表示されます

### 公開した Wiki コンテンツを管理する

1. 左側の縦型メニューで **[リポジトリ]** を選択します
1. ドロップダウン メニューに **eShopOnWeb** リポジトリと **main** ブランチが表示されていることを確認します
1. リポジトリのフォルダー階層で **Documents** フォルダーを選択します
1. **Welcome-to-our-Online-Retail-Store!.md** ファイルを選択します
1. Markdown 形式が生のテキストとして表示され、ここからファイルの内容を編集できることに注目してください

> **注**:Wiki ソース ファイルはソース コードとして処理されるため、従来のソース管理手法 (クローン、pull request、承認など) をすべて Wiki ページに適用できます。

## プロジェクト Wiki の作成と管理

既存のリポジトリとは独立して Wiki を作成および管理できます。 これにより、コードと共にバージョン管理する必要のないドキュメントをより柔軟に扱えます。

### Mermaid ダイアグラムを使用してプロジェクト Wiki を作成する

1. Azure DevOps ポータルで、**eShopOnWeb** プロジェクトの **Wiki** ペインに移動します
1. **eShopOnWeb (ドキュメント)** Wiki コンテンツを選択し、上部のドロップダウン リスト ヘッダーを選択します
1. ドロップダウン リストで、**[新しいプロジェクト Wiki を作成する]** を選択します
1. **[ページ タイトル]** テキスト ボックスに「`Project Design`」と入力します
1. ページの本文にカーソルを置きます
1. ツール バーの一番左のアイコン (ヘッダー設定) を選択し、**Header 1** を選択します
1. これにより、行の先頭にハッシュ文字 (**#**) が追加されます
1. **#** 文字の後に「`Authentication and Authorization`」と入力し、**Enter** キーを押します
1. ヘッダー設定アイコンをもう一度選択し、**Header 2** を選択します
1. **##** 文字の後に「`Azure DevOps OAuth 2.0 Authorization Flow`」と入力し、**Enter** キーを押します

### Mermaid ダイアグラムを追加する

Mermaid は、Markdown にヒントを得たテキスト定義をレンダリングしてダイアグラムを動的に作成するダイアグラム作成ツールです。

1. 次のコードを**コピーして貼り付け**、Mermaid ダイアグラムを挿入します。

   ```text
   ::: mermaid
   sequenceDiagram
    participant U as User
    participant A as Your app
    participant D as Azure DevOps
    U->>A: Use your app
    A->>D: Request authorization for user
    D-->>U: Request authorization
    U->>D: Grant authorization
    D-->>A: Send authorization code
    A->>D: Get access token
    D-->>A: Send access token
    A->>D: Call REST API with access token
    D-->>A: Respond to REST API
    A-->>U: Relay REST API response
   :::
   ```

> **注**:Mermaid 構文の詳細については、[Mermaid の概要](https://mermaid-js.github.io/mermaid/#/)に関するページを参照してください

1. プレビュー ウィンドウで **[ダイアグラムを読み込む]** を選択し、結果を確認します
1. 出力は、[OAuth 2.0 認可フロー](https://docs.microsoft.com/azure/devops/integrate/get-started/authentication/oauth)を示すフローチャートのようになります

### リビジョン メッセージと一緒に保存します

1. エディター ペインの右上隅で、**[保存]** ボタンの横にある下向きのキャレットを選択します
1. ドロップダウン メニューで、**[リビジョン メッセージと一緒に保存します]** を選択します
1. **[ページの保存]** ダイアログ ボックスで「`Authentication and authorization section with the OAuth 2.0 Mermaid diagram`」と入力します
1. **[保存]** を選びます。

### 画像セクションを追加する

1. カーソルを Mermaid 要素の末尾に置き、**Enter** キーを押して新しい行を追加します
1. ヘッダー設定アイコンを選択し、**Header 2** を選択します
1. **##** 文字の後に「`User Interface`」と入力し、**Enter** キーを押します
1. ツール バーで、**[ファイルの挿入]** を表すクリップ アイコンを選択します
1. **[開く]** ダイアログ ボックスで、**brand.png** ファイルをダウンロードした場所に移動します
1. **brand.png** ファイルを選択し、**[開く]** を選択します
1. プレビュー ウィンドウを確認し、画像が正しく表示されることを確認します
1. **[保存]** の横にある下向きキャレットを選択し、**[リビジョン メッセージと一緒に保存します]** を選択します
1. **[ページの保存]** ダイアログ ボックスで「`User Interface section with the eShopOnWeb image`」と入力します
1. **[保存]** を選びます。
1. 右上隅にある **[閉じる]** を選択します

## プロジェクト Wiki の管理

次に、変更の取り消しやページの整理など、Wiki コンテンツの管理方法について説明します。

### 変更履歴を使用して変更を元に戻す

1. **プロジェクト設計** Wiki を選択した状態で、右上隅にある縦型の省略記号を選択します
1. ドロップダウン メニューで **[リビジョンを表示する]** を選択します
1. **[リビジョン]** ペインで、最新の変更を表すエントリを選択します
1. 表示されたペインで、以前のバージョンと現在のバージョンの比較を確認します
1. **[元に戻す]** を選択します
1. 確認を求められたら、もう一度 **[元に戻す]** を選択します
1. 次に、**[ページの参照]** を選択します
1. **[プロジェクト設計]** ペインに戻り、変更が正常に元に戻されたことを確認します

### Wiki ページの追加と整理

1. **[プロジェクト設計]** ペインの左下隅にある **[+ 新しいページ]** を選択します
1. **[ページ タイトル]** テキスト ボックスに「`Project Design Overview`」と入力します
1. **[保存]** を選択し、**[閉じる]** を選択します
1. **プロジェクト設計**プロジェクト Wiki 内のページを一覧表示するペインに戻り、"**プロジェクト設計の概要**" エントリを見つけます
1. マウス ポインターで選択し、**[プロジェクト設計]** ページ エントリの上にドラッグ アンド ドロップします
1. "**プロジェクト設計の概要**" エントリがトップ レベル ページとして一覧表示され、ホーム アイコンが Wiki のホーム ページとして指定されていることを確認します

## Wiki 管理のベスト プラクティス

- ページとセクションには**明確でわかりやすいタイトルを使用する**
- 適切な見出しレベルを使用して、**コンテンツを階層的に整理する**
- Wiki 構文 `[[Page Name]]` を使用して**ページ間をリンクする**
- 理解を深めるために**画像やダイアグラムを含める**
- 構造化されたデータのプレゼンテーションには**テーブルを使用する**
- 書式とスタイルの**整合性を維持する**
- **コンテンツを定期的に確認して更新**し、最新の状態を維持する
- **リビジョン メッセージを使用**して変更を意味のある形で追跡する

## まとめ

このラボでは、次の作業を行う方法について説明します。

- Git リポジトリから**コードを作成して Wiki として公開する**
- リッチ フォーマットを使用して **Markdown コンテンツを管理する**
- 視覚的なドキュメントを作成するために **Mermaid ダイアグラムを作成する**
- Wiki ページに**画像を追加して整理する**
- コード リポジトリとは独立して**プロジェクト Wiki を管理する**
- **リビジョン履歴を使用**して変更の追跡と元に戻す操作を行う
- 階層構造で **Wiki ページを整理する**

Azure DevOps Wiki は、Markdown の柔軟性と整理されたドキュメントの構造を組み合わせた、チームの知識共有のための強力なプラットフォームを提供します。 開発ワークフローとシームレスに統合されており、効果的なチーム コミュニケーションに必要なコラボレーション機能も提供されます。
