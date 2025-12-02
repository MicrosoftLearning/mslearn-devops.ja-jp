---
lab:
  topic: Basic
  title: GitHub で作業フローを実装する
  description: リポジトリのフォーク、issue、ブランチ、pull request の作成など、GitHub を使用したリポジトリの実装および管理方法について説明します。
---

# GitHub で作業フローを実装する

このラボでは、GitHub を使用したリポジトリの実装および管理方法について説明します。 既存のリポジトリをフォークし、構成し、issue、ブランチ、pull request を作成し、変更を検証します。 このラボは、ソフトウェア開発環境でのバージョン管理とコラボレーションに GitHub を効果的に使用する方法を理解できるように設計されています。

このラボの各タスクは前のタスクを前提として構築されているため、順番に完了することが重要です。

学習内容:

- GitHub を使ってリポジトリを実装および管理する

このラボの所要時間は約 **30** 分です。

## 開始する前に

ラボを完了するには、以下が必要です。

- GitHub ユーザー アカウント。 アカウントをお持ちでない場合は、[新しいアカウントを作成](https://github.com/join)できます。 GitHub アカウントを作成する手順が必要な場合は、記事「[GitHub でのアカウントの作成](https://docs.github.com/get-started/quickstart/creating-an-account-on-github)」を参照してください。

## GitHub を使ってリポジトリを実装および管理する

この演習では、Git リポジトリをフォークし、GitHub を使用して管理します。

> **重要:** GitHub Copilot for pull request 機能の使用は、完全に任意です。 この機能を使用するには、**Copilot Enterprise サブスクリプション** を持つ企業のメンバーである必要があります。 アクセス権がない場合は、GitHub Copilot for pull request 機能に関連する手順をスキップできます。 GitHub Copilot for pull request 機能の詳細については、「[Copilot pull request summary について](https://docs.github.com/en/enterprise-cloud@latest/copilot/github-copilot-enterprise/copilot-pull-request-summaries/about-copilot-pull-request-summaries)」を参照してください。

> **注:**  このラボでは、まず、既存のリポジトリをフォークします。 フォークは、コードと可視性の設定を既存のアップストリーム リポジトリと共有するリポジトリです。 このアプローチは、オープンソース プロジェクトの更新を開発する場合、またはアップストリーム リポジトリへの書き込みアクセスが利用できないシナリオで頻繁に使用されます。 詳細については、「[フォークを使って作業する](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks)」を参照してください。

演習は次のタスクで構成されます。

- GitHub リポジトリをフォークする
- GitHub リポジトリを構成する
- 問題の作成
- ブランチを作成する
- ブランチ内のファイルを更新する
- pull request を作成してマージする
- issue を閉じる
- 変更を検証する

### GitHub リポジトリをフォークする

1. Web ブラウザーを起動し、[GitHub](https://github.com) ホーム ページに移動します。
1. 認証を求めるメッセージが表示されたら、GitHub ユーザー アカウントを使用してサインインします。
1. 同じブラウザー ウィンドウで別のタブを開き、[Spoon-Knife](https://github.com/octocat/Spoon-Knife) リポジトリに移動します。
1. **[Spoon-Knife]** リポジトリ ページで、**[フォーク]** を選択します。
1. **[新しいフォークの作成]** ページで、**[所有者]** ドロップダウン リスト エントリにお使いの GitHub ユーザー名が表示されていることを確認し、**[リポジトリ名]** テキスト ボックスで既定のエントリ **[Spoon-Knife]** を受け入れ、**[メイン ブランチのみをコピーする]** チェック ボックスをオンのままにして、**[フォークの作成]** を選択します。

   > **注:**  ブラウザー セッションが、フォークされたリポジトリに自動的にリダイレクトされます。

### GitHub リポジトリを構成する

1. フォークされた **[Spoon-Knife]** リポジトリ ページのツール バーで、**[設定]** を選択します。
1. **[設定]** タブの **[全般]** セクションで、既定のブランチが *_main_ に設定されていることを確認します。
1. **[全般]** セクションの **[機能]** 領域に移動し、**[Issues]** チェックボックスをオンにします。
1. 左側のナビゲーション メニューの **[コードと自動化]** グループで、**[ページ]** エントリを選択します。
1. **[GitHub Pages]** ペインの [ブランチ] セクションで、ドロップダウン リストの **[なし]** エントリを **[main]** に変更し、**[保存]** を選択します。

   > **注:**  GitHub Pages によって、リポジトリのコンテンツは、URL (`https://<your_GitHub_username>.github.io/Spoon-Knife/`) 経由でアクセスできるサイトに自動的に公開されます。

1. **GitHub Pages** ペインで、**[サイトにアクセスする]** ボタンを選択します。 これにより、別の Web ブラウザー タブが自動的に開き、index.html ファイルの現在の内容を表すページが表示されます。

   > **注:**  **[サイトにアクセスする]** ボタンとそのページが使用できるようになるまで、数分待つことが必要な場合があります。

   > **注:**  以前に **DevOps Core Intro Project** という GitHub プロジェクトを作成している場合は、必要に応じて以下の手順でこのリポジトリにリンクできます。 それ以外の場合は、「issue を作成する」セクションに進んでください。

1. フォークされた **[Spoon-Knife]** リポジトリ ページに戻り、ツール バーで **[プロジェクト]** を選択します。
1. **[新規プロジェクトへようこそ]** ペインで、**[プロジェクトのリンク]** を選択します。
1. 既存のプロジェクトの一覧で、**[DevOps Core Intro Project]** (使用できる場合) を選択します。

### 問題の作成

1. フォークされた **[Spoon-Knife]** ページで、**[Issues]** タブを選択します。
1. **[Issue]** ページで、**[新しい issue]** を選択します。
1. **[Add a title]** テキスト ボックスに「**`index.html looks rather austere`**」と入力します。
1. **[Add a description]** テキスト ボックスに「**`index.html file can use a modern touch`**」と入力します。
1. 現在のペインの [担当者] セクションで、**[担当者の追加...]** を選択し、**[候補]** セクションで、GitHub ユーザー名を選択します。
1. **[ラベル]** エントリの横にある歯車アイコンを選択し、ドロップダウン リストで **[拡張機能]** を選択します。
1. **[プロジェクト]** エントリの横にある歯車アイコンを選択し、ドロップダウン リストで **[DevOps Core Intro Project]** を選択します。
1. **[作成]** を選択して新しい issue を保存します。
1. 新しい issue をもう一度開きます。
1. **[index.html がやや質素に見える]** ペインの **[プロジェクト]** セクションで、**[状態]** を **[進行中]** に設定します。
1. フォークされた **[Spoon-Knife]** ページで、**[プロジェクト]** タブを選択します。
1. **[プロジェクトへようこそ]** ページで、**[DevOps Core Intro Project]** を選択します。
1. **DevOps Core Intro Project** のボード ビューで、**[進行中]** 列を確認し、新しく作成した issue が含まれていることを確認します。

### ブランチを作成する

1. フォークした **Spoon-Knife** ページに戻り、**[コード]** タブを選択します。
1. ページの左上隅にある **[main]** エントリを選択し、**[ブランチまたはタグの切り替え]** ドロップダウン リストを表示します。
1. **[ブランチの検索または作成...]** テキスト ボックスに「**`update index.html`**」と入力し、**[ブランチの作成: 'main' から index.html の更新]** エントリを選択して新しいブランチを作成します。

   > **注:**  これにより、ドロップダウン リストの内容に示されているように、新しく作成されたブランチが自動的に現在のブランチになります。

### ブランチ内のファイルを更新する

1. フォークされた **[Spoon-Knife]** リポジトリ ページで、ファイルの一覧から **[index.html]** を選択します。
1. **[Spoon-Knife/index.html]** ページで、コード エディター ツール バーの右側にある鉛筆アイコンを選択し、編集モードに切り替えます。
1. エディター ウィンドウで、ページの本文要素全体 (12 行目から 17 行目) を次の HTML コードに置き換えます。

   ```html
   <div id="octocat">
     <img
       src="https://octodex.github.com/images/NUX_Octodex.gif"
       alt="Octocat Image"
     />
   </div>

   <p>Ready to team up? Let's collaborate, @octocat!</p>
   ```

1. エディター ページの右上隅にある **[変更のコミット...]** を選択します。
1. **[Commit changes]** ウィンドウの **[Extended description]** テキスト ボックスに「**`Modified the image and paragraph text`**」と入力し、既定のコミット メッセージを受け入れて、**[Commit changes]** を選択します。

   > **注:**  この時点で、コミット用の新しいブランチを作成するオプションもあります。

1. 左側のリポジトリ ファイルの一覧で、**[styles.css]** を選択します。
1. **[Spoon-Knife/styles.css]** ページで、コード エディター ツール バーの右側にある鉛筆アイコンを選択し、編集モードに切り替えます。
1. エディター ウィンドウで、17 行目を次の HTML コード全体に置き換えます。

   ```css
     color: #333;
     line-height: 1.5;
     text-align: center;
   }

   body {
     font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
     background-color: #f8f9fa;
   }

   #octocat img {
     display: block;
     width: 100%;
     height: auto;
   }
   ```

1. エディター ページの右上隅にある **[変更のコミット...]** を選択します。
1. **[Commit changes]** ウィンドウの **[Extended description]** テキスト ボックスに「**`Modified CSS tags and selectors`**」と入力し、既定のコミット メッセージを受け入れ、**[Commit changes]** を選択して update-index.html ブランチの変更をコミットします。

### pull request を作成してマージする

1. フォークされた **[Spoon-Knife]** リポジトリ ページに戻ります。
1. ページの左上隅にあるドロップダウン リストにあるラベルで示されているように、現在 **update-index.html** ブランチが表示されていることを確認します。 このラベルに **main** と表示されている場合は、まずそれを選択し、ブランチの一覧を含むドロップダウン リストで、**[update-index.html]** を選択します。
1. フォークされた **[Spoon-Knife]** リポジトリ ページで、**[コントリビューション]** を選択し、**[pull request を開く]** を選択します。
1. **[pull request を開く]** で、エントリ **[ベース リポジトリ: octocat/Spoon-Knife]** を選択します。
1. **[ベース リポジトリの選択]** ドロップダウン リストで、このラボの開始時に作成した、フォークされたリポジトリの名前を選択します。

   > **注:**  この名前は、GitHub 名で始まり、その後にスラッシュ、**Spoon-Knife** が続きます。 名前を選択すると、エントリが、**ベース: main** に変更されます。

   > **注:**  この操作が必要なのは、フォークの作成元であるリポジトリのメイン ブランチではなく、フォークされたリポジトリのメイン ブランチを更新する必要があるためです。

1. **[タイトルの追加]** テキスト ボックスで、**index.html の更新** を **index.html と styles.css の更新** に置き換えます。

1. (省略可能) GitHub Copilot for pull request 機能にアクセスできる場合は、**[説明の追加]** テキスト ボックスで、**[Copilot アクション]** ボタンを選択し、**[概要]** を選択します (pull request の変更の概要を生成します)。

   1. GitHub Copilot for pull request 機能によって pull request の変更の概要が生成されます。

      ![GitHub Copilot for pull request 機能](media/github-pull-request-action.png)

   1. GitHub Copilot for pull request によって生成された概要を確認します。

   1. 概要が生成されたら、最初の行の **[Add a description]** に「**`Addressing #1`**」と入力し、**[Create pull request]** を選択します。

      ![GitHub Copilot の概要](media/github-pull-request-copilot-summary.png)

   > **注:**  GitHub Copilot for pull request 機能に対するアクセス権がない場合は、この手順をスキップできます。

   > **注:** GitHub Copilot for pull request 機能の使用を選択する場合は、次の手順をスキップできます。

1. **[Add a description]** テキスト ボックスに「**`Addressing #1`**」と入力し、**[Create pull request]** を選択します。

   > **注:**  **#1** を含めると、この pull request に関連付けられた最初の issue を参照できます。

1. 現在のブランチがベース ブランチと競合していないことを確認し、**[pull request のマージ]**、**[マージの確認]** の順に選択します。
1. pull request が正常にマージされ、閉じられたことを確認し、**[ブランチの削除]** を選択します。

### issue を閉じる

1. GitHub ページのツール バーで、**[Issues]** タブを選択します。
1. 最初の issue "**index.html がやや質素に見える**" の左側にあるチェックボックスをオンにし、**[としてマークする]** を選択し、ドロップダウン リストで **[完了]** を選択します。
1. **DevOps Core Intro Project** のボード ビューに戻り、issue が **[完了]** 列に表示されるようになったことを確認します。

### 変更を検証する

1. Web ブラウザー ウィンドウで、フォークされた **[Spoon-Knife]** リポジトリ ページに戻り、**[設定]** タブを選択し、左側のナビゲーション メニューの **[コードと自動化]** グループで **[Pages]** を選択し、**[GitHub Pages]** ペインを表示します。
1. **[GitHub Pages]** ペインで **[サイトにアクセスする]** を選択すると、別のブラウザー タブが開き、index.html ファイルの更新された内容が表示されます。
1. HTML および CSS ファイルで参照されるビジュアル要素を含むようにページが更新されていることを確認します。

> **注:**  この時点で、フォーク内のメイン ブランチへの変更を元のリポジトリに送信することになる場合があります。 これは通常、更新プログラムを開発するときに、オープンソース プロジェクトで共同作業する場合の次のステップになります。 ただし、元のリポジトリが保持されなくなるため、ここでは、この手順を適用しません。

## リソースをクリーンアップする

GitHub リポジトリとプロジェクトは参照およびポートフォリオ アイテムとして引き続き使用できるので、この演習のすべてのリソースはクリーンアップする必要はありません。

リポジトリを削除する場合は、次のドキュメントのようにしてできます:「[リポジトリの削除](https://docs.github.com/repositories/creating-and-managing-repositories/deleting-a-repository)」。
