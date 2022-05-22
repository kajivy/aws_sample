# ファイル名と説明
## CloudFormationテンプレートをCI/CDするCludFormation
① GitHubへ展開済みのインフラであるCloudFormationテンプレートの変更内容をPushすると、GitHubActionによりcodecommitへミラーリング <br>
②codecommitで変更が検知されるとcodepipelineにより、テンプレートの文法チェック（validation）が行われ、手動承認後、S3へArtifactsとして吐き出されます。<br>
③CloudFormation ChangeSetにより差分をチェック後、変更後のテンプレート内容が展開されます。
<details><summary>
aws_sampleリポジトリ</summary>
.github/workflows
<br>└ main.yml ・・・githubの内容をcodecommitへミラーリング
<br>├check_template・・・このディレクトリにあるテンプレートをvalidationで構文チェックする。また、cfn.ymlの子スタック（ネストされたスタック）。今回は、ネットワークレイヤ、サブネットレイヤ、アプリケーションレイヤの３つに分かれています。
<br>├params
<br>└ params.json・・・CI_CDpipleine.ymlの外部読み込みパラメーター
<br>├pipeline
<br>└ CI_CDpipeline.yml・・・CloudFormationテンプレートをCI/CDするtemplate
<br>├ECR.yml・・・先にECRを展開し、imageをpushしないと、check_template配下のテンプレートが展開されないため、ネストされたスタックに含めずに、独立させています。
<br>├builespec.yml・・・CI_CDpipeline.ymlと対応するbuildspec
<br>├cfn.yml・・・check_template配下にあるテンプレートに対する親スタック。パラメーターもこちらで設定しています。
<br>├requirements.txt・・・builespec.ymlの外部読み込みパラメーター<br>
<br>
</details>
<br>

## インフラストラクチャ（CloudFormationテンプレート）
今回展開するインフラは、いくつか特徴があります。
<br>
・ECSのCloudFormation化
<br>
・CloudFormationベストプラクティスの一つであるレイヤーの区分け
<br>
  →具体的には、ネットワークレイヤ、サブネットレイヤ、アプリケーションレイヤ
<br>
・レイヤーごとにテスト環境と開発環境の構築。
<br>
→ネストされたスタックを使用することで、レイヤーごとのパラメーター設定をすることなく一発でいずれかの環境構築
<br>
  →マネジメントコンソールでテンプレートをアップロードすると、いずれかの環境が選択可能
<br>
・LamdaによるDBのユーザー名、パスワードをSystemManagerにて自動ローテーション
<br>
・ネストされたスタックの使用

# 構成
dev環境
<br>

![](Library/Mobile%20Documents/com~apple~CloudDocs/Documents/aws/ecs.png)
# Ho to use

<details>
<summary>  １　IAMロールを用いたGithubActionsによるCodeCmmitミラーリング
<br></summary>
※GitHubを使わずに、CodeCommitでリポジトリを管理する場合は、この作業は不要となります。
<br>
<br>

[こちらを参考](https://book-reviews.blog/authenticate-using-IAM-Role-instead-of-persistent-credentials-on-Github-Actions/)にIAMロールを作成。作成したロールのARNをコピーして、.github/workflows配下のmain.ymlの`env`の`AWS_ROLE_ARN`欄に貼り付け
<br>

<br>

  今回の技術選定理由は、 CodeCommitよりGitHubの方が用途として一般的であること、CodeCommitへのミラーリング方法として、永続的なクレデンシャルを使用することもできますが、クレデンシャルよりIAMロールを用いた方が漏えいリスクを低減できることが主なものとなります。
<br>
</details>
<br>

<details>
<summary>２  CI_CDpipeline.ymlをマネジメントコンソールから展開<br></summary>
 パラメーター設定<br>
  ApplicationName:CodeCommitのリポジトリ名<br>
  BranchName:デフォルトは、 "main"となっているのでご自身のGitHub環境で適宜変更をしてください。
<br>
</details>
<br>
<details>
<summary>3 　ECR.ymlをマネジメントコンソールから展開し、イメージをpush</summary>
<br>
  まず、各自サンプルアプリをご用意願います。
<br>ECR.ymlを展開し、サンプルアプリをプッシュします。
<br>
cliを使ってECRにログイン〜プッシュまでは、こちらhttps://think-memo.com/ecr-push/が参考になります。
</details>
<br>
<details>
<summary>4 gitリポジトリへpush</summary>
<br>
  今回作成しましたリポジトリを各環境へクローンしていただき、ご自身の作成したリポジトリにpushしてください。
<br>
GitHubリポジトリへpush→codecommitリポジトリへミラーリング→CI／CDpipelineが起動し、check_template配下にあるテンプレートが構文チェックされる→S3へテンプレートがアップされ親スタックであるcfn.ymlが起動→テンプレート内容のレビュー後、手動承認→CloudFormation changesetにより差分チェック後、環境構築
<br>
以降、テンプレート内容に変更をし、リポジトリへpushする度に上記フローで展開していきます。
</details>

# 作成にあたって　

　今回は、 ECS環境をCloudFormaiton化することと、その環境をCI/CDパイプラインで構築することの２本立てとなっております。
通常は、アプリ開発者さんのためにECSにアプリをプッシュしてCI/CDパイプラインを構築する流れとなりますが、インフラ屋さん（AWSエンジニア、クラウドエンジニアetc）のためのパイプライン構築もあっていいのでは無いかということで作成しました。<br>
 　テンプレートのレイヤー化をした際に、test環境、dev環境を選択できるようにしていますが、各テンプレートでmappingを使い、それぞれの環境を選択できるようにしたところまでは、スムーズにできたのですが、その環境選択を一度設定したら、他のスタックにも適用できるようにする術が分からずにいましたが、参考文献に記載させていただいているクラスメソッドさんの記事に記載されているテンプレートを解析している時に、CloudFormationによるネストされたスタックの記述がありましたので、これを深掘りし、今回のポートフォリオの一部とさせていただきました。

# 苦労した点
 　buildspec.ymlの記述でS3の/TestOutputにpackaged.ymlが出力できないというエラー解消に２週間ほど時間を費やしてしましました。現役のエンジニア様にみていただいたところ、buildフェーズのymlの記法が違っているとのことで、validationチェックしたところ、確かにエラーが出ていましたので、yml記法に修正したところ、当該エラーは解消されました。
　一見、人間の目では間違っていると見分けられないものでも、エラーチェックのシステムでは、エラーがあるかどうか、どのようなエラーがあるか見分けることができるので、プラグインを導入するなどして、システムに依存して排除できるエラーは、極力出さないようにすることが大切だと思いました。

# 　参考文献
https://zenn.dev/trkdkjm/articles/f8fcc38c3cf690
<br>
https://dev.classmethod.jp/articles/developing-cloudformation-ci-cd-pipeline-with-github-codebuild-codepipeline/
<br>
https://techblog.nhn-techorus.com/archives/17674
<br>
https://kws-cloud-tech.com/
<br>
