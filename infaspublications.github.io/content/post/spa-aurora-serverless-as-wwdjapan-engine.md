+++
draft = false
date = "2017-07-07T20:12+09:00"
slug = "spa-aurora-serverless-as-wwdjapan-engine"
tags = ["AWS","SPA","Aurora","Serverless","CloudFront","S3","Lambda","API Gateway","SSR","Continuous Delivery"]
image = "/images/spa-aurora-serverless-as-wwdjapan-engine/cover.png"
comments = true	# set false to hide Disqus
share = true	# set false to hide share buttons
menu= ""		# set "main" to add this content to the main menu
author = "aggre"
title = "ニュースメディアWWD JAPAN.comを支えるSPA+Aurora+サーバーレス"
+++

こんにちは。INFASパブリケーションズでチーフエンジニアをやっている aggre です。主にWWD JAPAN.comというニュースメディアをチームで開発しています。

INFASパブリケーションズは、ファッション専門業界紙「WWD JAPAN」や、そのウェブ版として2012年にローンチした「WWD JAPAN.com」を運営している会社です。ほかにカルチャー誌「STUDIO VOICE」も発行しています。ちなみに「WWD JAPAN」の母体である「WWD」はアメリカで1910年に創刊した結構歴史のある業界紙だったりします。

## SPAとAuroraとサーバーレス

現在、WWD JAPAN.comではフロントエンドをSPA( Single Page Application )、バックエンドをDB以外はサーバーレスアーキテクチャにしています。

サーバーレスアーキテクチャの定義については [AWSによる説明](https://aws.amazon.com/jp/serverless/) が分かりやすいです。

データベースはAuroraを使っているのでサーバーレスではありません。ここ、詳しくは後述します！

![Architecture](/images/spa-aurora-serverless-as-wwdjapan-engine/architecture.png)

この図はWWD JAPAN.comのアーキテクチャを俯瞰したものです。

管理者しかアクセスしないエリアを除くと、CloudFront+S3+API Gateway+Lambdaによる構成で、これはウェブサービスにおける最小構成のサーバーレスアーキテクチャではないかと思います。

記者などの管理者専用画面として、WordPressをEC2で運用していますが、フロントエンドはSPAになっているので管理者以外がEC2にアクセスすることはありません。

これからの計画として、SPAのPWA( Progressive Web App )への移行や、Auroraを代替するデータベースへの移行を進めたいと考えています。

現時点においてそれぞれのサービスをWWD JAPAN.comがどのように使っているのか見ていきます。

## CloudFront

`https://www.wwdjapan.com` 以下すべてのリクエストをはじめに受け取ります。そのためにRoute53のALIASレコードをCloudFrontに設定してあります。

CloudFrontではSSL暗号化と、リクエストに合ったオリジンへの転送をします。基本的にはS3かAPI Gatewayへの転送をすることになります。

`/dist/*` だったらS3へ、それ以外はAPI Gatewayへ、という具合です。API Gatewayへの転送はSSR( Server Side Rendering )のためですね。

![CloudFront](/images/spa-aurora-serverless-as-wwdjapan-engine/cloudfront.png)

## S3

フロントエンドで動作するSPAのソースコードと、画像などのアセットを保管しています。

フロントエンドがSPAになることによって、アプリケーションは静的なHTML, JavaScript, CSS, manifest.json などのテキストファイル **だけ** で成立します。WWD JAPAN.comではソースコード管理にGitLabのクラウドを使っていますが、デプロイはビルドプロセスを経た成果物をGitLab CIの中からS3に送るだけです。デプロイが高速になって継続的デリバリーにも貢献しています。GitLab CIのパフォーマンス次第ではありますが平均して2分くらいでデプロイができます。

![S3](/images/spa-aurora-serverless-as-wwdjapan-engine/s3.png)

## Lambda

SPAからの要求にJSONをレスポンスしたり、SSRを実行します。

認証の必要なAPIは認証用のLambdaにJWT( JSON Web Token )で生成したトークンを渡すことでLambdaだけで完結させています。認証にJWTを使うことでデータベースへのアクセスが不要となりました。

WWD JAPAN.comではLambdaに2つのエイリアスを設定しています。 `production` と `dev` というエイリアスです。

また、大きく2つのタイプのLambda関数群があります。単体で動作する標準的なAPIのためのLambda関数と、特定のユーザーエクスペリエンスのためのAPI（いわゆるBackend for Frontend, BFF）のためのLambda関数です。BFFタイプのLambda関数は独立タイプのLambda関数のラッパーになっています。もちろん、標準的なケースなら独立タイプのLambda関数だけを使用することもできます。

![Lambda](/images/spa-aurora-serverless-as-wwdjapan-engine/lambda.png)

## API Gateway

HTTPリクエストとLambdaに渡すオブジェクトをマッピングして、Lambdaからのレスポンスをクライアントに返します。

Lambdaのタイプ毎にAPI（これはAWSコンソールにおける”API”です！）も分けています。例えば、`api` というAPIと、`api-for-wwd` のようなAPIに分かれます。

Lambda関数のエイリアスに合わせて、API毎のステージも2つあります。WWD JAPAN.comでは、`production` を呼び出す `stable` と、`dev` を呼び出す `unstable` という具合です。

また、常に最新の情報が担保されなくても良い結果整合性のAPIが多いので、API GatewayをオリジンとしたCloudFrontでキャッシュしています。API Gatewayにもキャッシュ機能はありますが、CloudFrontと異なり時間課金になるため注意が必要です。

![API Gateway](/images/spa-aurora-serverless-as-wwdjapan-engine/api-gateway.png)

## Aurora

AuroraはMySQL互換のリレーショナルデータベースで、サーバーレスとは異なりインスタンスが常時起動しています。

ここだけサーバーレスではないのですが、複雑で未知なクエリを数ms〜数十msでレスポンスするためにはリレーショナルデータベースが必要だったのです。そして私はサーバーレスなリレーショナルデータベースというソリューションをまだ発見できずにいます。その答えはBigQueryでしょうか？もしも知っていたらコメント欄などで教えてください！

完全なサーバーレスにするためには、いまのところデータベースはデータウェアハウス系のサービスかNoSQLという選択肢になります。平常時のレスポンスを最優先と考えた上でAuroraを代替できるものがあれば、そちらへの移行を実施したいと思ってます。

### Aurora のパフォーマンス

Auroraはサーバーレスではありませんが、それでも十分な可用性を備えています。WWD JAPAN.comでは1つのWriterインスタンスのほかマルチAZにある2つのReaderレプリカを使っています。リクエストは自動的に分散され、自動フェイルオーバーによってインスタンスの変更/削除も無停止で実施できます。

AuroraはAPIのバックエンドで接続されていて、APIは毎日平均100万前後のリクエストがあります。そのうち60%前後がキャッシュされているので、再計算されるAPIリクエストは毎日40万くらいです。 Auroraはすべて `r3.xlarge` クラスのインスタンスで構成していて、平均CPU使用率が16%前後、もっともCPU使用率の高いインスタンスでもCPU使用率が平均30%前後に収まっています。セレクトのレイテンシーは平均11.7msです。

## SSR

SSR( Server Side Rendering )の必要性について賛否はありますが、ニュースメディアでは必須要件になるのではないでしょうか。

JavaScriptによるウェブアプリケーションは、Googleでなら正当に評価されますが、FacebookやTwitterにおいては意図しない評価を受けてしまいます。ニュースメディアをクロールして自動的にコンテンツを収集する類のサービスでも同様です。

つまり、SSRの賛否についてしばしばトピックになる応答性などよりも、まずコンテンツが正当な評価を受けるためにSSRが必要になるということです。

すべてのリクエストをひとつのLambdaで処理する

WWD JAPAN.comではLambdaでSSRを実行しています。

そのためには、 `/*` のような曖昧なリクエストをひとつのLambdaに送る必要があります。このようなケースで有効なのがAPI Gatewayのプロキシ統合機能です。

API Gatewayで作成したAPIに `{proxy+}` というパスのリソースを追加することで、そのAPIへのすべてのリクエストをひとつのLambdaに転送できます。マッピングテンプレートやレスポンスのテンプレートも作る必要はなくなり、すべてのリクエスト情報をLambdaに転送して、Lambdaから返ってきた値をそのまま返却するという機能です。詳しくは [AWSのドキュメント](http://docs.aws.amazon.com/ja_jp/apigateway/latest/developerguide/api-gateway-set-up-simple-proxy.html) を参照ください。

### クライアントサイドだけを担保する

WWD JAPAN.comではクライアントサイドで実行するHTML+JavaScriptをLambdaでそのまま実行しています。( 厳密にはベンダーのスクリプトを除外したりしてますが )

つまりUniversalでは開発していません。これにはいくつかベネフィットがあります。

#### 集中

クライアントサイドのために加えた変更は、クライアントサイドでのテストをパスできればリリースできます。クライアントサイドで動くのにサーバサイドで失敗するようなことがなくなり、エンジニアがクライアントサイドの開発に関心を集中させることができます。

#### 継続的デリバリー

先にも書きましたが、ピュアなSPAならS3の静的ファイルを更新すればデプロイ完了です！

#### リーン

顧客に早く価値を届けて、チームが早く学習していく上では、回避可能な開発をなるべく避けるのが得策です。

Node.jsの中にクライアントサイドの環境を再現するためにはPhantomJSを利用しています。単純なスクレイピングならChromeのヘッドレスモードよりもPhantomJSの方が高速なことを確認しました。…と書きながら思い出したのですが、ちょっとアンフェアな条件下で比較していたかも知れません。ここはまた改めて比較してみたいと思います。

ではまた！

fyi, INFASパブリケーションズでは [サーバーレスで一緒に開発してくれるエンジニア](https://jobs.forkwell.com/infas-publications/jobs/1255) を募集しています！