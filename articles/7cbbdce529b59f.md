---
title: "Docker×FastAPI×React(TypeScript) on AWS ECS【AWS前編】"
emoji: "😸"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [docker, python, react, typescript, aws]
published: false
---


## 前回のfrontend編からの続きです
なんのこっちゃ。って人は以下記事のぜひともご覧ください。

[backend編](https://zenn.dev/daisukesasaki/articles/f18dd6554f94e3)
[frontnd編](https://zenn.dev/daisukesasaki/articles/9620f7fd0ca348)

※AWS_CLIが前提になるのでAWS_CLIの設定がまだ済んでいない方は以下のリンクから設定を先に済ませておきましょう。
(この記事ではv1で行っているので、v2ではすんなりいかないかも、、そのうち検証します。)
[AWS_CLI v1](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv1.html)

記事作成時のAWS_CLI verison
`aws-cli/1.20.6`

ようやくAWS編に入るのですが、いきなりECSの話に入る前にVPCの準備から始めましょう。
※いや、別にdefaultVPCでいいわ。って方はその辺ぶっ飛ばしていいです。
と、その前に構成図一回確認します。
![](/images/aws-ark.png)

## VPC
[VPCについてはこちら](https://aws.amazon.com/jp/vpc/?vpc-blogs.sort-by=item.additionalFields.createdDate&vpc-blogs.sort-order=desc)
**[始める前に注意]**
リージョンはap-northeast-○○でお願いします。(1 or 2)

それではまずVPCを作成します。

`名前タグ : お好きな名前で`（記事ではecs-vpcとしますので適宜読み替えて下さい）
`IPv4 CIDRブロック : 10.0.0.0/21`
「VPCを作成」ボタンを押す。

![](/images/aws-vpc-st01.png)
作成できたことが確認できたら次に移ります。


## Subnet
次にパブリック用、プライベート用のサブネットを2つずつ用意します。

- パブリック用1つ目
`VPC ID : ecs-vpc`
`サブネット名 : public-subnet-01`
`アベイラビリティーゾーン : ap-northeast-1a`
`IPv4 CIDRブロック : 10.0.0.0/24`
「サブネットを作成」ボタンを押す。

- パブリック用2つ目
`VPC ID : ecs-vpc`
`サブネット名 : public-subnet-02`
`アベイラビリティーゾーン : ap-northeast-1c`
`IPv4 CIDRブロック : 10.0.1.0/24`

同様手順でプライベート用も作成します。
- プライベート用1つ目
`VPC ID : ecs-vpc`
`サブネット名 : private-subnet-01`
`アベイラビリティーゾーン : ap-northeast-1a`
`IPv4 CIDRブロック : 10.0.2.0/24`

- プライベート用2つ目
`VPC ID : ecs-vpc`
`サブネット名 : private-subnet-02`
`アベイラビリティーゾーン : ap-northeast-1c`
`IPv4 CIDRブロック : 10.0.3.0/24`

![](/images/aws-vpc-subnet.png)

すべて終わると以下のように確認できると思います。

![](/images/aws-subnet-result.png)

## InternetGateWay

続けてインターネットゲートウェイを作成します。

`名前タグ : お好きな名前で`（記事ではecs-igwとしますので適宜読み替えて下さい）
「インターネットゲートウェイを作成」ボタンを押す。
![](/images/aws-igw-create.png)

インターネットゲートウェイが作成出来たら、先程作成したecs-vpcにアタッチします。
Nameの横にあるチェックマークをonにした状態で、アクション→VPCにアタッチを選択します。
![](/images/igw-attach-vpc.png)

アタッチ先を選択します。
`使用可能な VPC : ecs-vpc`
インターネットゲートウェイのアタッチを押す。
![](/images/igw-attach-vpc2.png)

## RouteTable
前述で作成したサブネットのルートテーブルを編集します。
Name`public-subnet-01`の横にあるチェックマークをonにした状態で、ルートテーブルタグを選択すると、
ルートが確認できると思いますが、現状は1件だけになっています。ここにルートを追加していきます。
![](/images/subnet-root.png)

ルートテーブルのリンク先へ遷移→ルートタブを選択し、「ルートを編集」を押します。
![](/images/root-table01.png)

「ルートを追加」を押して、
`送信先 : 0.0.0.0/0`
`ターゲット : ecs-igw`
上記のように設定し、 「変更を保存」を押します。
![](/images/root-table02.png)

ここまででVPC側の設定は終了です。
次は、一旦ローカル環境に移って、後に紹介するECRにアップする本番環境用のイメージを作成します。

## Production image
本番環境用のイメージとして、新たにDockerfile.prodを用意します。
### project/backend/Dockerfile.prod
```Dockerfile
#official base-image python
FROM python:3.9.6-slim-buster

#working directroy
WORKDIR /usr/src/

#environment variables
ENV PYTHONDONTWRITEBYTECODE 1
ENV PYTHONUNBUFFERED 1

#install system depedencies
RUN apt-get update \
  && apt-get -y install netcat \
  && apt-get clean

#install python depedencies
RUN pip install --upgrade pip
COPY ./requirements.txt .
RUN pip install -r requirements.txt

#add app
COPY . .

#add entrypoint.sh
CMD gunicorn app.main:app -w 4 -k uvicorn.workers.UvicornWorker -b 0.0.0.0:8000
```

backendのDockerfile.prodで記載していますが、本番環境では上記のgunicornを使うように公式で推奨されています。
こういうの見落としがちなのでちゃんとみておきましょう。
>UvicornにはGunicornワーカークラスが含まれており、ASGIアプリケーションを実行できます。Uvicornのパフォーマンス上のすべての利点があり、Gunicornのフル機能のプロセス管理も提供されます。
>これにより、ワーカープロセスの数をオンザフライで増減したり、ワーカープロセスを正常に再起動したり、ダウンタイムなしでサーバーのアップグレードを実行したりできます。
>本番環境での展開では、uvicornワーカークラスでgunicornを使用することをお勧めします。
```python
gunicorn example:app -w 4 -k uvicorn.workers.UvicornWorker
```
https://www.uvicorn.org/#running-with-gunicorn

公式リファレンスは大事ですね、、

### project/frontend/Dockerfile.prod
```Dockerfile
###########
# BUILDER #
###########

#pull official base-image
FROM node:16-alpine as builder

WORKDIR /usr/src/app/

#add usr/src/app/node_modules/.bin to $PATH
ENV PATH /usr/src/app/node_modules.bin:$PATH

#install and cache app dependencies
COPY /app/package.json .
COPY /app/package-lock.json .
#npm install
RUN npm ci
RUN npm install react-scripts@4.0.3 -g --silent

#set environment
ARG REACT_APP_API_SERVICE_URL
ENV REACT_APP_API_SERVICE_URL $REACT_APP_API_SERVICE_URL
ARG NODE_ENV
ENV NODE_ENV $NODE_ENV

COPY /app/. .
RUN npm run build

#########
# FINAL #
#########

FROM nginx:stable-alpine

RUN rm -rf /etc/nginx/conf.d
COPY /app/conf /etc/nginx

COPY --from=builder /usr/src/app/build /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```
本番環境ではNginxをイメージとして使用するため、multistage buildを使用します。

さらにnginxの独自設定を構成に追加します。以下のコマンドを実行し、default.confを記載します。
```sh
# conf,conf.dディレクトリを作成
mkdir -p project/frontend/conf/conf.d
# default.confの作成
touch project/frontend/conf/conf.d/default.conf
```

### default.conf
```nginx
server {
  listen 80;
  location / {
    root   /usr/share/nginx/html;
    index  index.html index.htm;
    try_files $uri $uri/ /index.html;
  }
  error_page   500 502 503 504  /50x.html;
  location = /50x.html {
    root   /usr/share/nginx/html;
  }
}
```

さらにdocker-compose.prod.ymlを新規で作成します。
```yml
version: '3.8'

services:
  web:
    build:
      context: ./project/backend
      dockerfile: Dockerfile.prod
    ports:
      - 8004:8000
    environment:
      - ENVIRONMENT=production
      - DATABASE_URL=postgres://postgres:postgres@web-db:5432/web_prod
    depends_on:
      - web-db


  web-db:
    build:
      context: ./project/backend/db
      dockerfile: Dockerfile
    expose:
      - 5432
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres


  front:
    build:
      context: ./project/frontend
      dockerfile: Dockerfile.prod
      args:
        - NODE_ENV=production
        - REACT_APP_API_SERVICE_URL=${REACT_APP_API_SERVICE_URL}
    ports:
      - 3007:80
    depends_on:
      - web
```

上記のdocker-compose.prod.ymlで直しているので気づいたかもしれませんが、create.sqlも修正します。
### project/backend/db/create.sql
```sql
CREATE DATABASE web_prod
```
修正はこれで終了ですが、この環境をローカルで確認しておきます。
```sh
export REACT_APP_API_SERVICE_URL=http://localhost:8004
```
コンテナを立ち上げます。
```sh
docker-compose -f docker-compose.prod.yml up -d --build
```
コンテナが立ち上がったら、データベースを作成します。ここ結構大事なのでお忘れなく、、
```sh
docker-compose exec web python app/db.py
```
データベースを確認します.
```sh
docker-compose exec web-db psql -U postgres

#Try \? for help.
postgres= \c web_prod
#You are now connected to database "web_prod" as user "postgres".
web_prod= \dt
            List of relations
 Schema |    Name     | Type  |  Owner   
--------+-------------+-------+----------
 public | textsummary | table | postgres
(1 row)
```
ここまで確認できたらローカル環境の設定は終了です。

## ECR
次にECR（[Elastic Container Registry](https://docs.aws.amazon.com/ja_jp/AmazonECR/latest/userguide/what-is-ecr.html)）を使用します。
リポジトリ名をそれぞれ`frontend`,`backend`、リポジトリを2つ作成して下さい。
※可視性設定については「プライベート」にしておきます。
![](/images/aws-ecr-rep.png)

作成できたことが確認できたら、frontendとbackendのリポジトリにイメージをプッシュしましょう。
そのために、まずECR用のイメージをbuildします。
※ {AWS_ACCOUNT_ID}についてはご自身のAWS_ACCOUNT_IDを入力して下さい。

```sh
#frontend
docker build -f project/frontend/Dockerfile.prod \
-t 714232315359.dkr.ecr.ap-northeast-1.amazonaws.com/frontend:prod \
--build-arg NODE_ENV=production \
--build-arg REACT_APP_API_SERVICE_URL=${REACT_APP_API_SERVICE_URL} \
./project/frontend
```

```sh
#backend
docker build -f project/backend/Dockerfile.prod \
-t {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/backend:prod \
./project/backend
```

イメージのbuildが確認できたらプッシュします。
ですが、その前にDockerCLIを認証させる必要があるので、以下のコマンドを入力しましょう。


```sh
aws ecr get-login-password --region ap-northeast-1 | docker login --username AWS --password-stdin {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com
```
入力後に以下の内容が返ってきたら成功です。
```sh
Login Succeeded
```

以下のコマンドでECR用に作成したbuildイメージをプッシュします。
```sh
#frontend
docker push {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/frontend：prod
```

```sh
#backend
docker push {AWS_ACCOUNT_ID}.dkr.ecr.ap-northeast-1.amazonaws.com/backend：prod
```

ECRのコンソール上に各イメージが構築できていれば次の手順に移ります。
※この時点でGitHubにリポジトリを作成していなければ、現在のプロジェクトをpushしましょう。

## CodeBuild
GitHubにリポジトリを作成していれば、CodeBuildの手順にはいります。
「ビルドプロジェクトを作成」を押し、以下のように設定していきます。
※設定に触れてないところはデフォルトのままにしてます。

### プロジェクトの設定

`プロジェクト名：お好きな名前で`（記事ではecs-todo-appとしますので適宜読み替えて下さい）
`説明：任意`（記事ではbuild docker imagesとしますので適宜読み替えて下さい）
`ビルドバッジ：　有効にする`

### ソース

`ソースプロバイダ：GitHub`
`リポジトリ：GitHubアカウントのリポジトリ`
`リポジトリのURL：今回のプロジェクトのリポジトリを選択する`
>追加設定

`ビルドステータス：有効にする`

### プライマリソースのウェブフックイベント
`ウェブフック：有効にする`
`ビルドタイプ：単一ビルド`

### 環境
`環境イメージ：マネージド型イメージ`
`オペレーティングシステム：Ubuntu`
`ランタイム：Standard`
`イメージ：aws/codebuild/standard:5.0`※最新を選ぶ
`イメージのバージョン：このランタイムバージョンには常に最新のイメージを使用して下さい`
`特権付与：有効にする`
`サービスロール：新しいサービスロール`
`ロール名：自動で命名されてるのでそのまま`
>追加設定

タイムアウト
`時間：0`
`分：10`
環境変数
`名前：AWS_ACCOUNT_ID`
`値：自身のAWS_ACCOUNT_ID`
`Type：Plaintext`

「ビルドプロジェクトを作成する」を押す。
コンソール上でプロジェクトが確認できたらbuildspec.ymlファイルを作成します。
現在はプロジェクトにbuildspec.ymlがないのでこのまま「ビルドを開始」を押しても失敗します。
では、プロジェクト直下にvbuildspec.ymlを作成します。

## buildspec

### project/buildspec.yml
```yml
version: 0.2

env:
  variables:
    AWS_REGION: "ap-northeast-1"
    REACT_APP_API_SERVICE_URL: "http://localhost:8004"

  phases:
    pre_build:
      commands:
        - echo logging in to ecr
        - >
          aws ecr get-login-password --region $AWS_REGION \
            - docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
    build:
      commands:
        - echo building images app-backend, app-frontend
        - >
          docker build \
          -f project/backend/Dockerfile.prod \
          -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod \
          ./project/backend
        - >
          docker build \
          -f project/frontend/Dockerfile.prod \
          -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend:prod \
          --build-arg NODE_ENV=production \
          --build-arg REACT_APP_API_SERVICE_URL=${REACT_APP_API_SERVICE_URL} \
          ./project/frontend
    post_build:
      commands:
        - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod
        - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend:prod
```
作成後、リポジトリにプッシュします。プッシュしたら自動でbuildが始まります。
### 以下のエラーが、ビルドログが発生して成功しない場合
>An error occurred (AccessDeniedException) when calling the GetAuthorizationToken operation:
>User: <omitted> is not authorized to perform: ecr:GetAuthorizationToken on resource: *

このエラーはロールに権限が不足していることから発生しています。
作成したプロジェクトに付帯したロールへ`AmazonEC2ContainerRegistryPowerUser`権限を追加します。
追加後にもう一度新しいビルドを実行すれば成功するはずです。

## ELB
ELBの設定をします。ここまで長くてだれてきそうですがもうちょい頑張りましょう。
ELBはEC2のコンソール画面から設定画面に入っていきます。（最近また少し設定画面が変わった、、）


「ロードバランサーの作成」、「Application Load Balancer」の「作成」を押して以下のように設定していきます。

`Load balancer name：お好きな名前で`（記事ではecs-albとしますので適宜読み替えて下さい）
`Scheme：internet-facing`
`IP address type：ipv4`
`Listeners：HTTP / Port 80`
`VPC：ecs-vpc`
`Availability Zones：1-a, 1-c`両方のAZを選択し、public用に設定したサブネットを選択します。
- 1-a：`public-subnet-01`
- 1-c：`public-subnet-02`

`Security groups：新規作成`
- セキュリティグループ名：ecs-sec-gp
- インバウンドルールに`HTTP 80`、`ssh 22`を設定しておきます。

「セキュリティグループを作成」を押す。

`Listeners and routing：新規作成`(frontend用のターゲットグループの設定)
- Name：`ecs-target-g`
- Target type：`Instance`
- VPC：`ecs-vpc`
- Port：`80`
- Health check path：`/`

「ターゲットグループを作成」を押す。

Summaryで設定を確認したら、「ロードバランサーの作成」を押す。
作成を確認できたらbackend用のターゲットグループを作成します。
- Name：`ecs-target-bk-gp`
- Target type：`Instance`
- VPC：`ecs-vpc`
- Port：`8000`
- Health check path：`/docs`

「ターゲットグループを作成」を押す。

続けてロードバランサーの設定を追加します。
`ecs-alb`を選択し、画面下部のリスナータブをクリック。「ルールの表示/編集」をクリックして、ルールの挿入を選択します。
IF：条件にはパスを選択し、値は`/`
THEN：転送先のターゲットグループには`ecs-target-bk-gp`

![](/images/alb-listen.png)

ここまで終わったら、ロードバランサーのDNS名をコピーして、buildspec.ymlを再度編集します。
<LOAD_BALANCER_DNS_NAME>の箇所にDNS名を貼り付けて下さい。

```yml
version: 0.2

env:
  variables:
    AWS_REGION: "ap-northeast-1"
    REACT_APP_API_SERVICE_URL: "http://<LOAD_BALANCER_DNS_NAME>"

  phases:
    pre_build:
      commands:
        - echo logging in to ecr
        - >
          aws ecr get-login-password --region $AWS_REGION \
            - docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com
    build:
      commands:
        - echo building images app-backend, app-frontend
        - >
          docker build \
          -f project/backend/Dockerfile.prod \
          -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod \
          ./project/backend
        - >
          docker build \
          -f project/frontend/Dockerfile.prod \
          -t $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend:prod \
          --build-arg NODE_ENV=production \
          --build-arg REACT_APP_API_SERVICE_URL=${REACT_APP_API_SERVICE_URL} \
          ./project/frontend
    post_build:
      commands:
        - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod
        - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/frontend:prod

```

>example
```
REACT_APP_API_SERVICE_URL: "http://ecs-alb-hogehoge-1234568.ap-northeast-1.elb.amazonaws.com"
```
編集後はコミットして、再度GitHubへプッシュします。

## RDS
まずはサブネットグループを作成します。
「DBサブネットグループを作成」を押して以下の内容を設定していきます

### サブネットグループの詳細
`名前：お好きな名前で`（記事ではecs-db-subnetとしますので適宜読み替えて下さい）
`名前：任意の内容を記載`
`VPC：ecs-vpc`
`アベイラビリティゾーン：ap-northeast-1a、ap-northeast-1c`
`サブネット：private-subnet-01(10.0.2.0/24)、private-subnet-02(10.0.3.0/24)`

「作成」を押す。
サブネットが作成されたことが確認できたらデータベースを作成していきます。

「データベースを作成」を押し、以下のように設定していきます。
### データベース作成方法を選択
- 標準作成

### エンジンのオプション
`エンジンのタイプ：PostgresSQL`
`バージョン：PostgresSQL 12.7-R1`

### テンプレート
- 開発/テスト

### 設定
`DBインスタンス識別子：ecs-prod-db`

認証情報の設定
`マスターユーザー名：webapp`

※マスターパスワード、パスワードの確認はご自身で任意のものを設定して下さい。

### DB インスタンスクラス
`DB インスタンスクラス：バースト可能クラス（db.t3.microでOK）`

### 可用性と耐久性
`マルチAZ配置：スタンバイインスタンスを作成する (本稼働環境向けに推奨)`

### 接続
`Virtual Private Cloud (VPC)：ecs-vpc`
`サブネットグループ：ecs-db-subnet`
`パブリックアクセス：なし`
`VPC セキュリティグループ：既存の選択`
`既存の VPC セキュリティグループ：ecs-sec-gp`


### データベース認証
`データベース認証オプション：パスワード認証`


### 追加設定
`最初のデータベース名：prod`

設定を明記しないところに関してはデフォルトで大丈夫です。
最後に「データベースの作成」を押す。

RDSはちょっと時間かかるので、その間にセキュリティグループの設定を編集します。
セキュリティグループのインバウンドルールタブから「Edit inbound rules」を押して、インバウンドルールを追加します。
`タイプ：すべてのトラフィック`
`リソースタイプ：カスタム`
`ソース：セキュリティグループからecs-sec-gpを選択`

これを追加したらいよいよECSの作業に移ります。
（RDSが作成されるまで待ってても良いかも）

## ECS　-Task-
最後にようやくECSを使っていきます。
まずは、ECSコンソールの左ペインに表示されている「タスク定義」を選択し、「新しいタスク定義の作成」を押します。
以下の内容で設定していきます。

### 起動タイプの互換性の選択
`EC2`
### タスクとコンテナの定義の設定
`タスク定義名：ecs-front`
`タスクロール：なし`
`ネットワークモード：default`

### コンテナの定義
- コンテナの追加
`コンテナ名：front`
`イメージ：<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/frontend:prod`
`メモリ制限：ソフト制限　300`
`ポートマッピング：ホストポート 0, コンテナポート 80`
「作成」を押す。

2つ目も同様に作成します、
### 起動タイプの互換性の選択
`EC2`
### タスクとコンテナの定義の設定
`タスク定義名：ecs-back`
`タスクロール：なし`
`ネットワークモード：default`

### コンテナの定義
- コンテナの追加
`コンテナ名：back`
`イメージ：<AWS_ACCOUNT_ID>.dkr.ecr.ap-northeast-1.amazonaws.com/backend:prod`
`メモリ制限：ソフト制限　300`
`ポートマッピング：ホストポート 0, コンテナポート 8000`
環境変数
- `DATABASE_URL：postgres://<DB_MASTER_USER_NAME>:<DB_PASSWORD>@<DATABASE_END_POINT>:5432/<DATABASE_NAME>`
- `ENVIRONMENT：production`
「作成」を押す

タスク定義についてはここまでです。

## ECS -Cluster-
次にクラスターを作成します。
「クラスターの作成」を押して、以下のように設定します。

`クラスターテンプレートの選択：EC2 Linux + ネットワーキング`

### クラスターの設定
`クラスター名：お好きな名前で`（記事ではapp-clusterとしますので適宜読み替えて下さい）
`インスタンスの設定：オンデマンドインスタンス`
`EC2 インスタンスタイプ：t2.micro`
`インスタンス数：4`
`キーペア：任意のSSHキーを設定`（ない人は新規で作成して下さい）

### ネットワーキング
`VPC：ecs-vpc`
`サブネット：public-subnet-01, public-subnet-02`
`パブリック IP の自動割り当て：有効`
`セキュリティグループ：ecs-sec-gp`

「作成」を押す。
Clusterの作成後、最後にサービスを作成します。

## ECS -Service-
### サービスの設定
`起動タイプ：EC2`
`タスク定義：ファミリー｜ecs-front、リビジョン｜latest`
`クラスター：app-cluster`
`サービス名：front-service`
`タスクの数：1`
※他はデフォルトでOK

### ロードバランシング
`ロードバランサーの種類：Application Load Balancer`
`ロードバランサー名：ecs-alb`

### ロードバランス用のコンテナ
`コンテナ名 ポート：front:0:80`

「ロードバランサーに追加」を押して、続けて入力していきます。
`プロダクションリスナーポート： 80:HTTP`
`ターゲットグループ名：ecs-target-gp`

これらを設定後は、他に設定するものはないので「次のステップ」を押して、「サービスの作成」が押せる画面まで移動してください。
内容を確認したら「サービスの作成」を押す。

それでは続けてもう一つサービスも作成します。
### サービスの設定
`起動タイプ：EC2`
`タスク定義：ファミリー｜ecs-back、リビジョン｜latest`
`クラスター：app-cluster`
`サービス名：back-service`
`タスクの数：1`
※他はデフォルトでOK

### ロードバランシング
`ロードバランサーの種類：Application Load Balancer`
`ロードバランサー名：ecs-alb`

### ロードバランス用のコンテナ
`コンテナ名 ポート：back:0:8000`

「ロードバランサーに追加」を押して、続けて入力していきます。
`プロダクションリスナーポート： 80:HTTP`
`ターゲットグループ名：ecs-target-bk-gp`

これらを設定後は、他に設定するものはないので「次のステップ」を押して、「サービスの作成」が押せる画面まで移動してください。
内容を確認したら「サービスの作成」を押す。

これでECSでの設定も終了しました。
ターゲットグループでのヘルスチェックなども確認しつつ、以下のように自身のALBに置き換えてリンクに遷移してみます。
http://<LOAD_BALANCER_DNS_NAME>/todos
※POSTPOST

非常に長かったですが、これでECSまで終了になります。









