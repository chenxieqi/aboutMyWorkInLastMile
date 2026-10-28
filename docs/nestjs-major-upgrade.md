# NestJS / TypeORM / AWS SDK 大規模アップグレード

> プロジェクト A で担当した PR のうち、コード変更量が最も大きかったものをピックアップして紹介します。

## 概要

| 項目 | 内容 |
| --- | --- |
| 時期 | 2024年11月〜12月（約1ヶ月） |
| 規模 | 約300ファイル、+10k / -8.5k 行、60コミット超 |

単純なマイナーバージョンアップではなく、次の3つの技術的負債を一度に解消したプロジェクトです。

1. NestJS 本体の **3メジャーバージョン跨ぎ**（v7 → v10）
2. TypeORM のメジャーアップグレード（0.2 → 0.3、API の全面的な breaking change）
3. AWS SDK のアーキテクチャ刷新（v2 → v3）

## 1. 主要パッケージのアップグレード

| パッケージ | 前 | 後 | 備考 |
| --- | --- | --- | --- |
| `@nestjs/*`（common / core / jwt / passport / typeorm 等） | v7.x | v10.4 | 7→8→9→10 の3メジャー跨ぎ |
| `typeorm` | 0.2.x | 0.3.x | API の全面的な breaking change |
| `aws-sdk` | v2（単一パッケージ） | `@aws-sdk/client-s3` / `client-sqs` / `client-ses` / `s3-request-presigner` | モジュール分割・書き方の刷新 |
| `rxjs` | 6.6 | 7.8 | |
| `@ssut/nestjs-sqs` | 1.3 | 3.0 | 内部が AWS SDK v3 に |
| `nestjs-i18n` | 8.0 | 10.5 | パーサー方式 → リゾルバー方式 |
| `multer-s3` | 2.9 | 3.0 | |
| `@nestjs-modules/mailer` | 1.5 | 2.0 | |

## 2. TypeORM 0.2 → 0.3：Repository Module パターンへの再構築

改修範囲が最も大きかった部分です。

TypeORM 0.3 では `getCustomRepository()` が削除されました。従来は多数のファイルがこの API でカスタム Repository を取得していたため、取得方法そのものを設計し直す必要がありました。

**対応方針**

- 共通ディレクトリに業務ドメイン単位の **Repository Module** を新設（12モジュール）
- 各モジュールは `useFactory` で `DataSource` を注入し、カスタム Repository を組み立てて provide する

```ts
@Module({
  providers: [
    {
      provide: XxxRepository,
      inject: [DataSource],
      useFactory: (dataSource: DataSource) =>
        dataSource.getRepository(XxxEntity).extend(customMethods),
    },
  ],
  exports: [XxxRepository],
})
export class XxxRepositoryModule {}
```

- ルートモジュール（`app.module.ts`）を整理し、Repository Module 群と、日付・S3・トークン・メール・決済などの横断的サービスモジュールを統一的に読み込む構成に変更

**付随する変更**

- `ormconfig` を `ConnectionOptions` のリテラルから `new DataSource({...})` に変更し、廃止された `cli` 設定を削除
- TypeORM CLI の引数変更に合わせて migration 系スクリプトを `-f` → `-d` に変更
- 起動時のマイグレーション実行を独自実装から標準の `dataSource.runMigrations()` に置き換え

## 3. AWS SDK v2 → v3：Command パターンへの全面書き換え

v3 では v2 の「チェーン形式 + `.promise()`」が廃止され、「Command を組み立てて `client.send(command)`」という形に変わりました。

**S3**

- `PutObject` / `GetObject` / `HeadObject` / `DeleteObject` / `CopyObject` / マルチパートアップロード / `waitUntilObjectExists` 等をすべて Command 形式へ移行
- `getSignedUrl` は独立パッケージ `@aws-sdk/s3-request-presigner` の関数に変更
- 呼び出しごとに `new S3()` していた実装を、コンストラクタで `S3Client` を一度だけ生成・注入する形に変更（設定は `ConfigService` から読み込み、グローバル設定に依存しない）
- v3 ではアップロード結果に `Location` が含まれないため、オブジェクト URL を自前で組み立てる処理を追加
- 複数ファイルの一括ダウンロード / zip 化など、v2 の高レベル API に依存していた処理を書き換え

**SQS / SES**

- SQS の Producer / Consumer（Module・Service 計4ファイル）を `@ssut/nestjs-sqs` v3 に対応
- メール送信を `@aws-sdk/client-ses` に移行

**その他**

- 起動時に行っていた `aws-sdk` のグローバル `config.update()` を削除（v3 にはグローバル設定の仕組みがないため）

## 4. i18n の API 変更対応

`nestjs-i18n` v8 → v10 で `I18nJsonParser` が廃止されたため、`AcceptLanguageResolver` / `QueryResolver` / `HeaderResolver` を組み合わせるリゾルバー方式に書き換えました。

## 5. 長期ブランチの運用

約1ヶ月の長期ブランチとなったため、本流（develop）を継続的に取り込みながら進めました。

- 本流側で並行して入った機能追加・修正との衝突解消
- インフラ設定（CORS 制御等）やキャッシュ関連 migration の調整
- 仕上げとして、migration 漏れ・型エラー・テストコード（当時は Jest）の修正

## まとめ

- フレームワーク本体・ORM・クラウド SDK という **3層の依存を同時に** 刷新し、ほぼ全モジュールに及ぶ改修を約1ヶ月で完了
- 作業の多くは API 置き換えという定型作業だが、設計判断として最も重要だったのは、廃止された `getCustomRepository()` の代替として導入した **Repository Module パターン**
- このパターンはその後もチームで継続的に使われており、現在の「横断的サービスを共通モジュールとして切り出す」構成の起点になった
