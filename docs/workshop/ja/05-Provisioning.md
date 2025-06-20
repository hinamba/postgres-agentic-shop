# Azure Developer CLIを使ったリソースプロビジョニング

ローカルにクローンしたリポジトリには、Azure上にインフラおよびアプリケーションコードをデプロイするための設定が含まれています。ここでは**Azure Developer CLI (azd)** を用いて、必要なAzureリソースのプロビジョニングとアプリケーションのデプロイを行います。所要時間はおおよそ20〜30分程度ですが、Azureのリソース作成状況によっては変動します。

## `azd up`コマンドによるプロビジョニング

Azure Developer CLIを使うと、単一のコマンドでインフラの構築からデプロイまで実行できます。以下の手順で進めましょう。

1. **Azure CLIへのログイン確認（CloudShellでは不要）**: 事前準備でAzure CLIをインストール済みですが、念のためログイン状態を確認します。ターミナル上で `az account show` を実行し、サブスクリプション情報が表示されればログイン済みです。未ログインの場合は `az login` を実行してブラウザ認証を完了してください。加えて、`azd`自体もAzureへの認証が必要ですので、`azd auth login`を実行しておきます（ブラウザが自動起動しAzure認証画面が表示されます。うまくいかない場合 `azd auth login --use-device-code` を使用）。

2. **シェルスクリプトへの実行権限の付与**: CloudShell、Linux及びmacOS環境でデプロイする際は、`azd-hooks/predeploy.sh`に実行権限を付与しておく必要があります。

```sh
chmod +x azd-hooks/predeploy.sh
```

また、Windowsでデプロイする際に、PowerShellでスクリプト実行ポリシーに関するエラーが出る場合があります。一時的に以下のコマンドで回避できます。

```sh
PowerShell -ExecutionPolicy Bypass -Scope Process
```

3. **デプロイ実行**: 準備が整ったら、いよいよ `azd up` コマンドを実行します。このコマンドは以下の処理を自動で行います。

- Bicepテンプレート（`infra`ディレクトリ内）を用いてAzure上にリソースグループおよび必要な各種リソースを作成。
- 作成したリソースに対して、ポストデプロイのスクリプト（`azd-hooks/predeploy.sh`）を実行して、データベース拡張の有効化やAzure OpenAI設定の書き込みを実施。
- アプリケーションのコード（バックエンド/APIやフロントエンド）をビルドし、それぞれ指定されたAzureサービス（例: App ServiceやStatic Web Appsなど）にデプロイ。

4. **プロンプトへの入力**: `azd up`を実行すると、最初にいくつか入力を求められます。具体的には「デプロイ先のAzureサブスクリプション」「デプロイするAzureリージョン（ロケーション）」「Azure OpenAIモデルのデプロイ先リージョン」などです。ここで、前セクションで検討したリージョンを選択してください。Azure OpenAIについては利用可能なリージョンのみ候補に出ます。また、今回新規に作成するリソースグループ名も求められる場合があります。適宜入力しましょう。

> [!CAUTION] 警告
> OpenAIモデルのクォータに関する警告が表示されることがあります。「指定リージョンに十分なGPT-4oのTPMがない」等のメッセージが出た場合は、前述のクォータ条件を満たすように設定を変更するかリージョンを変更する必要があります。

5. **デプロイ進行のモニタ**: コマンドを実行すると、ターミナル上に各ステップの進行状況が表示されます。内部ではTerraformのように状態を追跡しつつ、順次リソースが構築されます。特にAzure Database for PostgreSQLやAzure OpenAIリソースの構築には数分単位の時間がかかります。気長に待ちましょう。Azure Portalを開いてリソースグループを確認すると、リアルタイムでリソースが増えていくのが確認できます。

6. **デプロイ完了の確認**: 全ての処理が完了すると、`azd`はデプロイ結果の概要を表示します。成功した場合、デプロイした各サービスのエンドポイントURLやリソースの名前などが一覧表示されるはずです（例えばWebアプリのURLなど）。

## プロビジョニングで作成される主なリソースと各ステップの説明

`azd up`によって自動的に構築・デプロイされる主なAzureリソースと、プロビジョニングプロセス内の各ステップについて簡単に説明します。

- **Resource Group（リソースグループ）**: 指定した名前で新規に作成されます。ハンズオン全体のリソースをまとめるコンテナです。

- **Azure Database for PostgreSQL – Flexible Server**: PostgreSQLデータベースサーバが作成されます。本プロジェクトの中核データベースであり、製品情報やレビューなどのテーブルデータ、および拡張機能によるベクトル・グラフデータを格納します。BicepテンプレートでSKUやストレージサイズ、管理ユーザ名などが定義されています（デフォルトでは安価なプランが選ばれているはずです）。

- **Azure OpenAI Service**: GPT-4やEmbeddingモデルを使用するためのAzure OpenAIリソースが作成されます。デプロイ時にモデル名やSKUも自動設定されます。例えばGPT-4oと、text-embedding-ada-002のデプロイがセットアップされます。これにより、データベースからAzure OpenAIにリクエストを送信できるようになります。

- **Container Apps**: バックエンドのアプリケーションコードをホストするためのコンピュートリソースが作られます。Pythonで書かれたAPI（エージェントのオーケストレーションロジック）と、TypeScript/JavaScriptで書かれたフロントエンドが存在します。Azure Developer CLIは`azure.yaml`に記載されたサービス定義に基づき、適切なホスティング先を構築・デプロイします。

- **接続設定・シークレット**: 構築された各リソース間の接続情報（例えばデータベースの接続文字列、OpenAIのAPIキーなど）は、Azure Developer CLIによりアプリの設定に組み込まれます。セキュリティをより強化する場合、Azure Key Vaultの利用も検討してください。

- **拡張機能のセットアップ**: PostgreSQLサーバが起動後、`azd-hooks/predeploy.sh` スクリプトが実行されます。このスクリプト内で、前セクションで述べた `CREATE EXTENSION` 文の実行や、Azure OpenAIエンドポイント・キーをデータベースに登録する `azure_ai.set_setting` 関数の呼び出しなどが行われます。Azure CLIの`rdbms-connect`拡張がここで活躍し、スクリプト内で `az postgres flexible-server connect` コマンドによりPostgreSQLに接続してSQL実行しています。

- **デプロイ後処理**: インフラ構築とアプリの配置が終わると、Azure Developer CLIは「デプロイ完了」としてエンドポイント情報を表示します。例えば「WebアプリURL: …」、「Azure OpenAIリソース名: …」などが出力されます。これらは`azure.yaml`で`output`として定義されている項目です。参加者はこの情報をメモするか、そのまま次の検証手順で利用します。

もしプロビジョニング中にエラーが発生した場合、エラーメッセージを確認してください。よくある問題としては、環境名に使用不可な文字が含まれていてバリデーションエラーが出る、Azure権限不足でリソース作成が拒否される、既存のリソース名と衝突する、といったことがあります。その場合はメッセージに従い対応するか、`azd up`を再実行すれば再試行されます。

> [!NOTE] トラブルシューティング
> DescriptionAzure Developer CLIに関する一般的なトラブルシューティングは[公式ドキュメント](https://github.com/Azure-Samples/postgres-agentic-shop)を参照してください。

[前へ](04-Repository.md) | [次へ](06-Post-provisioning.md)
