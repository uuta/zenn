---
title: "BigQuery×Golang開発の裏側：エミュレータ活用からAPI統合まで"
emoji: "🍉"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["bigquery", "go"]
published: false
---

SocialDog Analyticsチームエンジニアのuutaです。
SocialDog Analyticsチームでは主にSocialDogの分析機能の開発を行っています。

## 前提・問題意識

SocialDogの投稿パフォーマンス機能開発において、大量のデータを効率的に処理する必要性から、BigQueryの採用を検討しました。その過程で、既存のバッチ処理やAPIで使用しているGolangとBigQueryを連携させる環境の構築に取り組むことになりました。

しかし、GolangでBigQueryを扱う環境の構築に関する情報が非常に限られていたため、多くの課題に直面しました。この記事では、私たちが経験した問題と、その解決策を共有します。

## GolangとBigQueryを統合したい

SocialDogのバックエンドの新機能開発では、Golangを主に使用しています。投稿パフォーマンス機能のAPI、バッチ機能開発でもGolangを利用するため、どのようにBigQueryと統合するかが問題になりました。当初は、ローカル開発環境をbigquery-emulator（後述）のみで完結させることを目指していましたが、現在は一部開発用のBigQueryを使用して実装を進める形に落ち着いています。

- **本番環境**: BigQueryを使用
- **開発環境**: bigquery-emulatorの使用、開発用に割り当てたBigQueryを一部テストで使用

## Golangで利用できるパッケージについて

また、BigQueryを取り扱うためのGolangのパッケージとして、下記を採用しています。

- 本番環境
  - BigQuery REST APIを取り扱う
    - [bigquery package - cloud.google.com/go/bigquery - Go Packages](https://pkg.go.dev/cloud.google.com/go/bigquery)
  - BigQuery Storage Write APIを取り扱う
    - [managedwriter package - cloud.google.com/go/bigquery/storage/managedwriter - Go Packages](https://pkg.go.dev/cloud.google.com/go/bigquery/storage/managedwriter)
- 開発環境
  - BigQueryサーバーをローカル環境でエミュレートする
    - https://github.com/goccy/bigquery-emulator

## ローカル環境の構築

### bigquery-emulatorを用いたローカル・テスト環境の構築

開発効率を上げるため、まずはbigquery-emulatorを使用したローカル環境の構築に取り組みました。bigquery-emulatorはBigQueryサーバーをローカル環境でエミュレートするためのGo製のオープンソースソフトウェアです。

- [https://github.com/goccy/bigquery-emulator](https://github.com/goccy/bigquery-emulator)

goccyさんという方が開発されています。

- [https://github.com/goccy](https://github.com/goccy)

以下は、その手順の概要です：

1. Docker環境の準備
2. bigquery-emulatorのコンテナ設定
3. Golang環境とのインテグレーション

SocialDogの開発ではDocker Composeを採用しており、比較的簡単にコンテナを作成することができました。

```yaml
bigquery-emulator:
  image: ghcr.io/goccy/bigquery-emulator:0.6
  platform: linux/amd64
  ports:
    - "9050:9050"
    - "9060:9060"
  volumes:
    - ../../:/var/www/html
```

imageは下記から選択することができます。

- [https://github.com/goccy/bigquery-emulator/pkgs/container/bigquery-emulator](https://github.com/goccy/bigquery-emulator/pkgs/container/bigquery-emulator)

## 開発中のトラブルと解決策のtips

BigQueryとGolangをベースにした開発環境との統合に関する情報が少なく、開発環境構築には苦慮しました。今も課題はありますが、開発中に直面した課題とチームで採用した解決策について紹介したいと思います。

### 1. bigquery-emulatorで関数が動作しない問題

bigquery-emulatorを使用する中で、いくつかの関数が動作しないという問題に直面しました。具体的には、ANY_VALUE、DATETIME_BUCKET、MAX_BY、MIN_BYなどの関数が使用できませんでした。

理由は、bigquery-emulatorが依存しているhttps://github.com/goccy/go-zetasqliteで、上記の関数が現状サポートされていなかったためです。MAX_BYやMIN_BYなどの関数は、BigQueryが提供する比較的新しいクエリであり、サポート外の関数が出るのは避けられない印象です。

今回は、下記の方法を用いてBigQueryへのリクエストを表現することにしました。

1. CREATE FUNCTIONを使用して、必要な関数を擬似的に作成
2. BigQuery本体でクエリを実行し、結果を確認する方法を併用

まず、1に関して、`udfs`というstring型のスライスの中にCREATE FUNCTIONを入れ、擬似的にTIMESTAMP_BUCKET、DATETIME_BUCKETが利用できる形にしました。以下はCREATE FUNCTIONをGoで実装する例です。

```go
func InitEmulator(f NewClientFunc) error {
	ctx := context.Background()
	c, err := f(ctx)
	if err != nil {
		return fmt.Errorf("failed init emulator: %w", err)
	}
	defer c.Close()
	udfs := []string{
		`
			CREATE FUNCTION TIMESTAMP_BUCKET(datetime TIMESTAMP, int INTERVAL) AS (
				CASE int
					WHEN INTERVAL 1 HOUR
						THEN datetime
					WHEN INTERVAL 1 DAY
						THEN TIMESTAMP(EXTRACT(DATE FROM datetime))
					WHEN INTERVAL 7 DAY
						THEN TIMESTAMP_SUB(TIMESTAMP(EXTRACT(DATE FROM datetime)), INTERVAL EXTRACT(DAYOFWEEK FROM datetime) - 1 DAY)
				END
			)
		`,
		`
			CREATE FUNCTION DATETIME_BUCKET(datetime DATETIME, int INTERVAL) AS (
				CASE int
					WHEN INTERVAL 1 HOUR
						THEN datetime
					WHEN INTERVAL 1 DAY
						THEN EXTRACT(DATE FROM datetime)
					WHEN INTERVAL 7 DAY
						THEN DATETIME_SUB(EXTRACT(DATE FROM datetime), INTERVAL EXTRACT(DAYOFWEEK FROM datetime) - 1 DAY)
				END
			)
		`,
	}
	for _, udf := range udfs {
		q := c.c.Query(udf)
		j, err := q.Run(ctx)
		if err != nil {
			return fmt.Errorf("failed to create udf: %w", err)
		}

		status, err := j.Wait(ctx)
		if err != nil {
			return fmt.Errorf("failed to create udf: %w", err)
		}

		if err := status.Err(); err != nil {
			return fmt.Errorf("failed to create udf: %w", err)
		}
	}
	return nil
}

```

また、2に関して、BigQueryに用意した開発用のデータを用いて直接クエリを叩いて結果を確認しながら開発を進める方法を部分的に採用しています。

### 2. テストデータの永続化の問題

次に直面したのは、開発で使用するテストデータをどのように挿入・維持するかという問題です。最初は、bigquery-emulatorに存在する`—database`と `—data-from-yaml` というコマンドでデータの挿入と永続化を試みましたが、下記のエラーにより期待する動作が得られませんでした。

```yaml
BigQuery error in query operation: Error processing job '<job_id>': failed to
scan rows: failed to decode value: illegal base64 data at input byte 4

```

一旦、compose.ymlファイルに`—data-from-yaml`を追加して、ymlファイルからDocker Composeを起動するたびに、bigquery-emulatorに特定のデータを挿入するという形にしています。

```yaml
bigquery-emulator:
  image: ghcr.io/goccy/bigquery-emulator:0.6
  platform: linux/amd64
  ports:
    - "9050:9050"
    - "9060:9060"
  volumes:
    - ../../:/var/www/html
  command: bigquery-emulator --project=<project-name> --data-from-yaml=<data.yaml>
```

暫定的な策ですが、開発で使用するデータをymlファイルを通して入れることができました。

### 3. BigQuery Storage Write APIとの統合

データの書き込みは、BigQuery Storage Write APIを利用する形にしました。Google公式のドキュメントにも記載がある通り、コストの面でメリットがあるように思えたからです。

> Storage Write API は、古い insertAll ストリーミング API よりも大幅に低コストです。さらに、1 か月あたり最大 2 TiB を無料で取り込むことができます。- BigQuery Storage Write API の概要  |  Google Cloud

問題は、BigQuery Storage Write APIをGolangに組み込むための情報が不足していたことです。　体系的に解説する記事が当時は存在しなかったため、ここら辺を参考に実装を進めました。

- https://github.com/bucketeer-io/bucketeer/blob/19548c4f783869040b51fb0eddde5d9b25adcdab/pkg/storage/v2/bigquery/writer/client.go
- https://github.com/luci/luci-go/blob/3f7339cdf0b0747619d469d92b84725a9006e050/resultdb/bqutil/storagewrite.go

BigQuery書き込み用のNewWriter関数とbigquery-emulator書き込み用のNewEmulatorWriter関数の2種類を用意し、環境に応じて接続先を変えられるように実装しました。下記は実装の例です。

```go
// BigQueryのManagedWriter用（主に書き込み）のクライアント
type WriteClient struct {
	c         *managedwriter.Client
	projectID string
	datasetID string
	Option
}

type NewWriteClientFunc func(context.Context) (*WriteClient, error)

func newWriteClient(c *managedwriter.Client, projectID, datasetID string, opts ...CreateOption) *WriteClient {
	o := Option{
		logger: logr.Discard(),
	}
	for _, opt := range opts {
		opt(&o)
	}
	return &WriteClient{c: c, projectID: projectID, datasetID: datasetID, Option: o}
}

func NewWriter(projectID, datasetID string, opts ...CreateOption) NewWriteClientFunc {
	return func(ctx context.Context) (*WriteClient, error) {
		c, err := managedwriter.NewClient(
			ctx,
			projectID,
		)
		if err != nil {
			return nil, fmt.Errorf("failed to create new bq client: %w", err)
		}
		return newWriteClient(c, projectID, datasetID, opts...), nil
	}
}

func NewEmulatorWriter(projectID, datasetID, serverURL string, opts ...CreateOption) NewWriteClientFunc {
	return func(ctx context.Context) (*WriteClient, error) {
		c, err := managedwriter.NewClient(
			ctx,
			projectID,
			option.WithoutAuthentication(),
			option.WithEndpoint(serverURL),
			// bigquery-emulator接続時はtlsを無効にする。
			option.WithGRPCDialOption(grpc.WithTransportCredentials(insecure.NewCredentials())),
		)
		if err != nil {
			return nil, fmt.Errorf("failed to create new bq emulator client: %w", err)
		}
		return newWriteClient(c, projectID, datasetID, opts...), nil
	}
}
```

ここで苦慮したのが、BigQuery Storage Write APIのどのストリームを使用して書き込みを行うかです。書き込み方法には下記の4種類があります。

- **デフォルトタイプ**: デフォルトの方法。書き込んだ情報をすぐに利用できる
- **保留タイプ**: ストリームをcommitするまでレコードが保留状態になる方法。commit次第、保留中のデータを全て読み取ることができる
- **コミットタイプ**: ストリームにレコードを書き込み次第、レコードをすぐに読み取ることができる方法
- **バッファタイプ**: 行レベルでcommitが行われ、ストリームがフラッシュされるまでレコードがバッファされる方法。通常は使用しない

![BigQuery Storage Write APIの書き込み方法](/images/bigquery-local-dev/data-ingestion-decision-map.png)

- 参考: [BigQuery Storage Write API の概要  |  Google Cloud](https://cloud.google.com/bigquery/docs/write-api?hl=ja)

当初はデフォルトストリームを用いて書き込みを行うつもりでした。しかし、bigquery-emulatorで対応しておらず、保留タイプを使用して書き込みを行う形に変更しました。

```go
pendingStream, err := wc.c.CreateWriteStream(ctx, &storagepb.CreateWriteStreamRequest{
	Parent:      tableName,
	WriteStream: &storagepb.WriteStream{Type: storagepb.WriteStream_PENDING},
})
```

※試してはいないですが、下記のPRでデフォルトストリームに対応したようです🎉

- https://github.com/goccy/bigquery-emulator/pull/226

### 4. 開発環境と本番環境の出し分け

開発環境ではbigquery-emulatorに、本番環境ではBigQueryに接続するため、main.go内で環境変数により接続先を変える実装にしています。

```go
	var bqNewClientFunc bq.NewClientFunc
	var bqNewWriteClientFunc bq.NewWriteClientFunc
	if cfg.IsDevelopment() {
		bqNewClientFunc = bq.NewEmulatorClient(
			cfg.BqProjectID,
			cfg.BqDatasetID,
			"http://bigquery-emulator:9050",
			bq.WithLogger(l),
		)
		bqNewWriteClientFunc = bq.NewEmulatorWriter(
			cfg.BqProjectID,
			cfg.BqDatasetID,
			"bigquery-emulator:9060",
			bq.WithLogger(l),
		)
		if err := bq.InitEmulator(bqNewClientFunc); err != nil {
			return fmt.Errorf("failed to initializing bigquery-emulator: %w", err)
		}
	} else {
		bqNewClientFunc = bq.NewClient(cfg.BqProjectID, cfg.BqDatasetID, bq.WithLogger(logger))
		bqNewWriteClientFunc = bq.NewWriter(cfg.BqProjectID, cfg.BqDatasetID, bq.WithLogger(logger))
	}

```

開発向けに作成した`NewEmulatorClient`、`NewEmulatorWriter` の3つ目の引数でserverURLを指定しているのですが、これらはそれぞれ下記のように設定する必要がありました。

- http://bigquery-emulator:9050
- bigquery-emulator:9060

### 5. レコードの読み込み、書き込み

レコードの読み込みはBigQuery REST APIを使用したため、下記のドキュメントのコードのようにクエリを定義し、パラメータを付与してReedメソッドを呼ぶという実装にしています。

```go
q := client.Query(`
    SELECT year, SUM(number) as num
    FROM bigquery-public-data.usa_names.usa_1910_2013
    WHERE name = @name
    GROUP BY year
    ORDER BY year
`)
q.Parameters = []bigquery.QueryParameter{
	{Name: "name", Value: "William"},
}
it, err := q.Read(ctx)
if err != nil {
    // TODO: Handle error.
}
```

- [bigquery package - cloud.google.com/go/bigquery - Go Packages](https://pkg.go.dev/cloud.google.com/go/bigquery)

レコードの書き込みは、BigQuery Storage Write API用のクライアントを使用しています。下記は実際に使用しているコードを少し修正したものです。Addメソッドに渡される`[]FooAdd` をJSONに変換し、BigQuery Storage Write APIに書き込みを行うメソッドに値を渡すことで、書き込みを実現しています。

```go
type FooAdd struct {
	Foo            string        `json:"foo"`
}

func (repo *FooRepository) Add(ctx context.Context, data []FooAdd) error {
	w, err := repo.w(ctx)
	if err != nil {
		return fmt.Errorf("failed to create write client: %w", err)
	}
	defer w.Close()
	jsonData, err := ConvertToJSON(data)
	if err != nil {
		return fmt.Errorf("failed to convert to json: %w", err)
	}
	if err := w.WriteBatch(ctx, bq.FooSchema, bq.Foo, jsonData); err != nil {
		return fmt.Errorf("failed to add: %w", err)
	}
	return nil
}

```

## まとめ

BigQueryとGolangを用いた開発環境の統合は、数多くの課題に直面しました。Analyticsチームで環境を構築し始めた当時、ベストプラクティス的な情報が少なく、暫定的な解決策を見つけるのに試行錯誤した記憶があります。

この記事がGolangでBigQueryを導入しようとしている皆さんの参考になれば幸いです。

## SocialDogについて

株式会社SocialDogは、「あらゆる人がSNSを活用できる世界を創る」をミッションとし、SNSマーケティング運用担当者のためのオールインワンツールを提供しています。

https://portal.socialdog.jp/

今後はX以外のSNSのデータも集め、より高度な分析機能を開発していきます。
超大規模なデータを使ったり、モダンな技術を取り入れながら新機能開発をしたいエンジニアを募集中ですので、興味があればぜひお話ししましょう！

## 参考

- [https://techblog.enechain.com/entry/escan-with-bigquery-emulator](https://techblog.enechain.com/entry/escan-with-bigquery-emulator)
- https://docs.google.com/presentation/d/1j5TPCpXiE9CvBjq78W8BWz-cGxU8djW1qy9Y6eBHso8/edit#slide=id.p
