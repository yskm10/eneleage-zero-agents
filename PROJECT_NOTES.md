# PROJECT_NOTES — ENELEAGE Zero 営業パートナー管理(eneleage-zero-agents)

このファイルは、新しいチャットでClaudeに作業を頼むときに最初に渡す「引き継ぎ資料」です。
最新版の確認先: `https://raw.githubusercontent.com/yskm10/eneleage-zero-agents/main/PROJECT_NOTES.md`。作業前に必ず最新を確認し、差分があればそれを土台にしてから編集すること。

## Claudeへのお願い

このリポジトリは、非エンジニアの担当者(森)がClaudeに依頼しながら運用している静的サイトです。変更を依頼された際は、専門用語を避けて具体的に説明し、大きな変更は小さいステップに分けて進めてください。可能であれば変更前にコミットしておき、うまくいかなければ無理に直そうとせず元に戻して再度試してください。

## このリポジトリの役割

ENELEAGE Zero(太陽光パネル不要のスマート蓄電池、メーカー:株式会社ボルタ)の営業パートナー(業務委託)の募集・管理システム。GitHub Pagesで静的ホスティングし、Supabaseをバックエンドに使っている。カスタムドメイン `agent.medicom.co.jp` で公開(DNSはバリュードメイン/コアサーバーで管理、CNAMEレコードで設定済み)。

## ファイル構成

- `index.html`(旧 agent-registration.html を改名):公開の応募フォーム。
  - 種別(個人/法人)トグルで表示項目が切り替わる
  - 氏名・住所・電話番号・LINE ID(必須)・メールアドレス・営業エリア
  - インボイス登録区分トグル(登録ありなら登録番号欄が出る)
  - 振込先(銀行名・支店名・口座種別・口座番号・口座名義カナ)
  - 身分証明書アップロード(表面必須・裏面任意)、Supabase Storageの `id-documents` バケットへ保存
  - 主要営業品目チェックボックス(太陽光/リフォーム・外壁塗装/注文住宅/その他+自由記述)
  - Supabaseクライアントは `persistSession: false` にしている(下記「注意点」を参照)
- `agent-admin.html`:社内管理者用ダッシュボード。Supabase Authのemail/passwordでログイン。3タブ構成:
  - **計算タブ**:歩合計算機(下記「歩合計算の仕様」を参照)
  - **応募者タブ**:応募者一覧、ステータス変更(pending/active/inactive/rejected)、身分証明書の閲覧(`createSignedUrl`で署名付きURLを発行)、詳細表示のトグル
  - **案件タブ**:成約記録の入力(担当者・金額・日付・メモ)、歩合の自動計算プレビュー、支払いステータス管理(成約/施工完了/歩合支払済)、集計サマリー

## Supabase

- プロジェクト名:`eneleage-zero-agents`(project ref: `escdzmgtgqssbqhqlubw`、リージョン: ap-northeast-1東京)
- テーブル:
  - `public.sales_agents`:応募者情報。主なカラムは `entity_type`, `company_name`, `name`, `address`, `phone`, `line_id`, `email`, `staff_mobile`, `sales_area`, `invoice_registered`, `invoice_number`, `bank_name`, `bank_branch`, `bank_account_type`, `bank_account_number`, `bank_account_holder`, `id_document_path_front`, `id_document_path_back`, `main_products`(配列), `status`, `note`
  - `public.deals`:成約記録。`agent_id`, `agent_name_snapshot`, `price`, `invoice_registered_at_sale`, `commission`, `status`, `deal_date`, `note`
- Storageバケット:`id-documents`(非公開)
- RLSの注意点(ここを崩すと応募フォームか管理画面のどちらかが壊れる):
  - `sales_agents`と`storage.objects`は `anon`(応募フォーム用)と `authenticated`(管理画面用)の両方にinsert/selectポリシーが必要
  - `storage.buckets` 自体にも `anon`/`authenticated` 向けのselectポリシーが要る(バケットの存在確認ができないとアップロードが失敗する)
  - `index.html` 側のSupabaseクライアントは `persistSession: false, autoRefreshToken: false, detectSessionInUrl: false` にしている。これを外すと、同じブラウザで管理画面にログイン済みの状態で応募フォームを開いたときに `authenticated` ロールとして送信されてしまい、ポリシー次第で失敗する

## 歩合計算の仕様(2026年9月時点)

- 基準価格:200万円(税別・本体価格のみ、施工費は含まない)
- インボイス登録事業者:基準委託料40万円、200万円超過分の60%を加算
- インボイス未登録事業者:基準委託料37.5万円、200万円超過分の58%を加算
- 200万円を下回る値引きは、金額の大小にかかわらず事前承認が必要(社内ルール。以前は160万円まではノーチェックだったが、現在は全て承認必須に変更済み)
- **この計算式は3箇所に重複している**:`agent-admin.html`の計算タブ、単体配布しているコミッション計算アーティファクト(Claude Artifact)、営業資料の記載。仕様変更時はこの3つを揃えて直すこと

## 既知の未対応事項

- `agent-admin.html`の応募者詳細ビューに法人電話番号(`staff_mobile`)の表示が抜けている。次に何か機能を触るタイミングで一緒に直す予定

## デプロイ方法

GitHub Pagesで公開(ビルドステップなし、HTMLファイルを直接配置)。`index.html`と`agent-admin.html`を更新してpushすれば反映される。カスタムドメイン・HTTPS強制は設定済み。
