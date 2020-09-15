# ci-cd-learning

# Description

CI/CDの勉強用リポジトリ

# リポジトリ構成

- Herokuへのデプロイを想定
- CIのテスト部分はGitHub Actionsを利用
- CIの残りの部分とCDはHerokuを利用
- アプリケーションとしてはExpressの雛形を利用

# CI/CDの概念

## 1. CI/CDとは

継続的インテグレーション (CI) と、継続的デリバリー/デプロイメント (CD) を組み合わせた言葉。  
双方を合わせた全体の概念として CI/CD と呼んだり、パイプライン (Pipeline) と呼んだりする。  

あまり言葉自体に細かい定義は無い。概念を表す言葉。

今日ではDocker等のコンテナ仮想化技術もあり、実現するためのコストが下がっている。

CI/CDを提供するサービスとしては

- CircleCI
- GitHub Actions
- Cloud Build
- Heroku Pipelines
  - Heroku CI

などが挙げられる。CI部分を中心としたもの、CD部分を中心としたもの、統合してサービスを提供するものと色々ある。

使うサービスによって文化や操作の違いがあるので、そこら辺は使うサービスに合わせて知る必要がある。

## 2. 継続的インテグレーション

開発された成果物をリリース用ブランチへ持ち込む為に必要な作業を自動化する。といった概念をCIと呼ぶ事が多い。

コードを書いて何かを実装したり修正したりした場合、その後はテストを書いてテストツールに掛けたり、linterを掛けてコードスタイルを矯正したりするが、そのような作業を自動化する。

具体的にはjsonやyaml等で`go test`や`gofmt`というようなタスクを定義しておき、紐付けたリポジトリのブランチに何かpushされると定義されたタスクを実行し、結果を通知してくれるものや、プルリクエストが投稿された場合に検証環境を切り出し、ビルド結果を確認する場所を設けてくれるもの等がある。

前者ではテストに掛かる手間の削減やlintによるコード品質の維持を、後者ではレビューを実施する際の環境依存部分の吸収や準備の手順削減を提供することができる。

余談になるが、前にGitHubの自動プルリクを飛ばしてきた`Dependabot`はCIに分類でき、プロジェクトで利用しているライブラリのバージョンをチェックしセキュリティ問題があった場合にバージョンアップを提案してくれる。

## 3. 継続的デリバリー/デプロイメント

リリース状態となったコード (Git-flowで言う**masterブランチ**) や、リリース一歩手前の状態となったコード (Git-flowで言う**developブランチ**) を、本番を模したステージング環境や本番環境にデプロイすることをCDと呼ぶ事が多い。

リリース用のブランチにpushがあったら、ビルドして配信先にデプロイ(展開)したり、本番環境であれば本番イメージのビルドやpush、サーバーを起動する作業が含まれることもある。

今回はプラットフォームをCGPに置くという話なので、恐らく利用するのはGCPのCloud Buildになる。  
Buildと言いつつ、色々出来る。  
ちなみにAWSのは**Code**Build

GitHub Pagesがまさにこれに当たる。intelligent-geckoでは、masterブランチにpushがあると、GitHub ActionsでビルドされたスライドがGitHub Pagesで自動的にホスティングされる。

# HerokuによるCI/CDの練習

## 1. Herokuとは

- salesforceが運営する、アプリケーションのホスティングをしてくれるPaaS
  - 主にWebアプリケーション

## 2. HerokuにおけるCI/CD

![image](https://user-images.githubusercontent.com/38117745/93152159-e6c32780-f738-11ea-8a0a-f7fc7f39e20c.png)

- Heroku Pipelinesを中心とし、環境ごとのappを追加して作業を進めていく考えとなっている。
- Review Apps・Staging・Productionと3つの環境で1つのPipelineを構成する。
- Review Appsはプルリク用の環境となっていて、プルリクが来た場合に自動で検証環境を切り出してホスティングし、そこにアクセスするだけでプルリクをレビューすることが出来る。
  - プルリク単位での使い捨てが前提であり、sleep機能を持たない。
- Stagingはその名の通りステージング環境にあたり、Productionは本番環境となる。
  - ステージング環境から本番環境に移行する部分については、ポリシーによって自動でやらせることもあれば、一度人の目を通すこともある。

## 3. Herokuで試すCI/CD

- 今回の練習では、Herokuを使う
- 多くの言語に対応しており、使うライブラリの文化に従ったリポジトリ構成をしていれば、自動でプロジェクトを判別してくれる
- Herokuにデプロイしやすく、かつ結果が解りやすいWebアプリで試す
- WebUIを中心に操作を行うが、Herokuが用意しているCLIからの操作も可能
  - 本番ではコマンドのログを残す意味合いで、CLIから操作をするべき
- Free Dynoの消費があるので、注意

## 4. リポジトリとアプリの雛形作成

```
$ gh repo create --public {アカウント/組織}/{リポジトリ名}
```

- 今回は`gh`コマンドを利用し、GitHub側でのリポジトリ作成も同時に行った
- ghでリポジトリを作成すると、masterブランチが作成されないので注意
  - このまま別branchに移動すると、masterブランチが無いリポジトリとなる
    - Herokuではトラブルの原因となるので注意

```
$ npx express-generator
```

- Expressの雛形を作成
- npxを用いて使い捨てで実行

```
$ git add .
```

```
$ git commit -m "init"
```

```
git push --set-upstream origin master
```

- とりあえずmasterをpushしておく
  - Herokuはmasterが無いとトラブルが起こるので

## 5. Heroku CLIを使ったアプリの作成

```
$ heroku apps:create
```

- `heroku`コマンドのインストールとログインは済んでいる前提

## 6. Heroku側でGitHubとの連携

![image](https://user-images.githubusercontent.com/38117745/93157158-a073c580-f744-11ea-807d-483548c1a487.png)

- `Create new pipeline`を選択

![image](https://user-images.githubusercontent.com/38117745/93157437-30197400-f745-11ea-8621-20cc0c9b6057.png)

- Pipeline name: 適当
- Pipeline owner: 今回はテストなので個人アカウント

![image](https://user-images.githubusercontent.com/38117745/93157585-77a00000-f745-11ea-8c6d-e476d79de365.png)

- Connect to GitHub: 連携したいリポジトリに接続する

リポジトリと接続したら、`Create pipeline`でパイプラインを作成する

## 7. ステージング環境に紐付けるブランチを作成

```
$ git switch -c develop
```

- 今回はdevelopをステージング環境と紐付ける
  - **developにpushされたら、ステージング環境に自動デプロイする**
  - 本来のステージング環境は、本番と同じ環境となるので厳密にはdevelopと対応付けるのはどうなんだろうというのはある
    - けど、git-flowに従えばdevelopはmasterと同期されているので良い…のかも?
- developと紐付けようにも、リモートブランチにdevelopが無いので、一旦適当にpushしておく

```
$ vi views/index.jade
```

- 空コミットもアレなので、なんか書いておく

![image](https://user-images.githubusercontent.com/38117745/93159752-dd8e8680-f749-11ea-8bf7-158b56b01edc.png)

```
$ git add views/index.jade
```

```
$ git commit -m "CI/CD test"
```

```
$ git push --set-upstream origin develop
```

## 8. developブランチにpushがあれば、Stagingへデプロイする設定を行う

![image](https://user-images.githubusercontent.com/38117745/93158921-1f1e3200-f748-11ea-8e95-3a20ff23581e.png)

- Stagingの項目で、`Add app`を選択する

![image](https://user-images.githubusercontent.com/38117745/93159091-6e646280-f748-11ea-9632-c5e72369d413.png)

- Create new app...を選択

![image](https://user-images.githubusercontent.com/38117745/93159169-948a0280-f748-11ea-9491-7f089305a0a0.png)

- 適当なApp nameをつけて、`Create app`でappを作成する

![image](https://user-images.githubusercontent.com/38117745/93159269-cf8c3600-f748-11ea-8c3b-c6724b713466.png)

- 作成したapp名をクリックして、appの設定画面を開く

![image](https://user-images.githubusercontent.com/38117745/93159380-006c6b00-f749-11ea-97d1-0b7e7c97efba.png)

- メニューから`Deploy`を選択肢、デプロイに関する設定を開く

![image](https://user-images.githubusercontent.com/38117745/93160203-cf8d3580-f74a-11ea-9279-b0beae46ff5a.png)

- Automatic deploysの項目から、紐付けるブランチ(今回はdevelop)を設定して`Enable Automatic Deploys`を選択
  - これで、developに対するpushがあれば自動的にステージング環境にデプロイされるようになった

## 9. 最初のステージング環境へのデプロイを行う

最初の1回は、手動でdevelopをデプロイしておく必要がある。

![image](https://user-images.githubusercontent.com/38117745/93166827-40881980-f75a-11ea-8bb7-f8ef3a813c12.png)

- app設定画面、Deployメニューから`Manual deploy`を選択し、`Deploy Branch`を選択して最初のデプロイを行う

![image](https://user-images.githubusercontent.com/38117745/93167142-f0f61d80-f75a-11ea-9053-aa920d086914.png)

- `Your app was successfully deployed.`と表示されていればOK

## 10. 自動デプロイのテスト

```
$ vi views/index.jade
```

![image](https://user-images.githubusercontent.com/38117745/93160502-6a860f80-f74b-11ea-8694-63fae42e709b.png)

```
$ git add views/index.jade
```

```
$ git commit -m "Automatic deploy test"
```

```
$ git push
```

![image](https://user-images.githubusercontent.com/38117745/93160781-06b01680-f74c-11ea-9d0b-f6fcc745eb0f.png)

- ステージング環境のappから、`Open app`を選択すると、ホスティングされているアプリに飛べる

![image](https://user-images.githubusercontent.com/38117745/93160773-0152cc00-f74c-11ea-9932-4192a1ca1864.png)

- 追加した内容が反映されているのが分かる

## ここまでの確認

以上の手順で、"GitHub上にWebアプリケーションのリポジトリを作成し、ブランチを切って、developにpushがあればステージング環境へデプロイする"というCDが作成された。

あとはパイプラインのPRODUCTIONの方で、Automatic Deployの対象をmasterにしておけば、masterがdevelopの内容をマージした時点でプロダクション環境として自動デプロイする のような挙動になる。  
この部分を自動化するか、人の目を通すかは、ポリシー次第。

## 11. 実験的なCI

Heroku CIだと金を取られるのでGitHub Actionsにやらせて、プルリクを作成し、それをHerokuのReview Appsの機能で確認する形をとる。

### 実験的なCI: Heroku側の操作

![image](https://user-images.githubusercontent.com/38117745/93171111-6d8cfa00-f763-11ea-94cd-422cd1abdcdf.png)

- REVIEW APPSから`Enable Review Apps`を選択

![image](https://user-images.githubusercontent.com/38117745/93171225-a036f280-f763-11ea-8e11-da84b919d59f.png)

- `Create new review apps for new pull requests automatically` (プルリクが飛んできたら自動的に検証環境のappを動かす)と、  
`Destroy stale review apps automatically` (プルリクがクローズされたら自動的に検証環境を破棄する)の設定を入れて、`Enable Review Apps`を選択

![image](https://user-images.githubusercontent.com/38117745/93171436-0d4a8800-f764-11ea-9a91-1e4d905846d1.png)

- これで、プルリクが開かれたら検証環境にデプロイされるようになった

### 実験的なCI: GitHub Actionsの準備

```
$ git switch -c feat/github-actions
```

- CIテスト用のブランチを切り

```
$ mkdir -p .github/workflows/
```

- GitHub Actionsが指定するディレクトリ構造を作成する

```
$ vi .github/workflows/test.yml
```

- GitHub Actionsでは、タスクはYAMLで定義する
- 任意の名前のYAMLを作成

![image](https://user-images.githubusercontent.com/38117745/93168597-3d8f2800-f75e-11ea-9171-3d64305157ab.png)

- [構文ドキュメント](https://docs.github.com/ja/actions/reference/workflow-syntax-for-github-actions)や[Node.jsの例](https://docs.github.com/ja/actions/language-and-framework-guides/using-nodejs-with-github-actions)を参考にタスクを定義する
  - ここはサービスによって大きく変わる。Heroku CIなら、`app.json`を定義することとなる
  - ESLintのセットアップの手間を省くため、今回はPrettierを掛けてみることをテストとする

```
npm install -D prettier
```

- Prettierをインストールして

```
$ vi package.json
```

![image](https://user-images.githubusercontent.com/38117745/93169857-0706dc80-f761-11ea-97d6-d931d05395b1.png)

- `"scripts"`の項目に実行させるべきタスクである`prettier . -c`を追加
  - `"devDependencies"`の変更はprettierのインストール時に自動でやってくれる

```
$ npm run prettier
```

- prettierを走らせてみて

```
> ci-cd-docs@0.0.0 prettier /home/k017c1157/ci-cd-docs
> prettier . -c

(中略)

npm ERR! ci-cd-docs@0.0.0 prettier: `prettier . -c`
npm ERR! Exit status 1
```

- 走った上で、prettierが終了コード1を返していればOK

```
$ git add package.json package-lock.json .github/
```

- node_modules/以外の変更をaddして(邪魔なら`.gitignore`を用意するなり適当に)

```
$ git commit -m "feat: GitHub Actions"
```

```
git push -u origin feat/github-actions
```

- コミットしてpushする

```
remote: Create a pull request for 'feat/github-actions' on GitHub by visiting:
remote:      https://github.com/...
```

- プルリク作成用のURLが表示されるので、そこからプルリクを作成する

![image](https://user-images.githubusercontent.com/38117745/93172046-17b95180-f765-11ea-8bac-f81d1ef7ba42.png)

- 今回はprettierが終了コード1を返すようにしているので、failedとなり

![image](https://user-images.githubusercontent.com/38117745/93172503-f1e07c80-f765-11ea-985d-d2a5f3353a03.png)

- Review Appsの方で、検証環境が切り出されてデプロイのされていることが分かる。
  - `Open app`を選択することで、実際に動いてる状態で確認ができる
