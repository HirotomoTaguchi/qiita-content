---
title: Teams の「外部アクセス」のログを Purview で監査する（メモ）
tags:
  - Security
  - MicrosoftTeams
  - Microsoft365
  - Purview
  - MicrosoftSecurity
private: false
updated_at: '2026-02-01T22:03:42+09:00'
id: ce4a8cf48f201f5c5671
organization_url_name: null
slide: false
ignorePublish: false
---

Microsoft Teamsで自社組織以外の外部組織とのコラボレーションする方法は、僕の理解だと「外部アクセス[^1]」「ゲストユーザー[^2]」「直接接続（Teams Connect）[^3]」がある認識ですが、今回は外部アクセスのチャットのログをPurviewでとることを簡単にメモがてらまとめます。

## 前提条件

監査ログをエクスポートするには、Purview の「監査 (プレミアム)」 に相当するライセンスが必要です。監査ログは「監査（スタンダード）」でも保持されますが、本ユースケースにおいては項目が足りません。また、グローバル管理者、コンプライアンス管理者、または監査ログの役割を持っている必要があります。統合監査ログは既定で有効化されているため、特別な設定は不要です。

:::alert
本記事は2026年2月1日時点での情報を元に作成しています。また、本記事は筆者が検証や調べた内容を元に執筆していますが、何かしら保証を与えるものではありません。
:::

## Microsoft Purviewでの監査ログエクスポート手順

まず、[Microsoft Purviewコンプライアンスポータル](https://compliance.microsoft.com/)にサインインします。左側のナビゲーションメニューから「監査」を選択すると、監査ログの検索画面が表示されます。

### 検索条件の設定

監査ログ検索画面では、Teamsの外部アクセスログを取得するための条件を設定します。

「アクティビティ」では「Microsoft Teamsのアクティビティ」の以下を選択します。[^4]

| 管理画面での選択肢（英語） | ログ上の表記 | 説明 |
|---|---|---|
| Posted a new message | MessageSent | メッセージ送信 |
| Created a chat | ChatCreated | チャット作成（外部アクセスの場合は最初のメッセージの送信と同時刻に出るように見える） |
| Add Members | MemberAdded |メンバー追加（外部アクセスが送付される場合は、ユーザーが外部アクセスを許可した際に出ると思われる。） |

![](https://github.com/user-attachments/assets/fe999f3b-f8cc-481d-bef7-f56cd8e6f1f1)

:::alert
Purview のポータルでの選択画面の項目と、実際のログの項目の表示形式は異なるので注意してください。
:::

「日付と時刻の範囲」では調査対象の期間を指定します。過去30日間や特定の期間など、必要に応じて設定してください。

「ユーザー」フィールドは、特定ユーザーのログのみを取得したい場合に入力します。空白のままにすると全ユーザーが対象となります。「ファイル、フォルダー、またはサイト」フィールドは空白のままで構いません。

### 検索の実行

条件を設定したら「検索」ボタンをクリックします。検索結果が表示されるまで数秒から数分かかります。結果が表示されたら、検索結果の件数を確認してください。1回の検索で最大50,000件まで表示可能です。それ以上のレコードがある場合は、期間を分割して複数回検索する必要があります。

### 結果のエクスポート

検索結果画面の上部にある「エクスポート」をクリックし、「すべての結果をダウンロード」を選択します（表示されている結果のみダウンロードすることもできますが、通常は全件ダウンロードを推奨します）。エクスポートジョブが開始されたら、「エクスポート」タブに移動してダウンロードの進行状況を確認できます。

### CSVファイルのダウンロード

エクスポートが完了したら、「結果のダウンロード」をクリックするとCSVファイルが自動的にダウンロードされます。ファイル名は `AuditLog_YYYYMMDD_HHMMSS.csv` のような形式になります。

![](https://github.com/user-attachments/assets/3250c688-07a0-4dea-8301-7ac749efeeb3)


## エクスポートされた監査ログの構造

Microsoft PurviewからエクスポートされたTeamsの監査ログは、CSVファイルとして提供されますが、エンベロープ形式で重要な情報がJSON形式で埋め込まれているため、ぱっと見わかりにくいのが難点です。エクスポートされたCSVの構造は以下のようになっています。

```csv
RecordId,CreationDate,RecordType,Operation,UserId,AuditData,AssociatedAdminUnits,AssociatedAdminUnitsNames
1e0607b5-20e7-55a8-bf04-bb94eaf44e51,"2026-01-24T06:54:24.0000000Z ",25,MessageSent,.cid.2640d6262885f83e,"{""AppAccessContext"":{...}}",,
```

RecordIdは監査レコードの一意識別子、CreationDateはイベント発生日時、RecordTypeはレコードタイプ（Teamsの場合は25）、OperationはMessageSentなどの実行された操作、UserIdは操作を実行したユーザーを示します。しかし、`HasForeignTenantUsers` や `HasGuestUsers` といった外部アクセス監査で最も重要なフィールドは、すべて `AuditData` 列内のJSON文字列に埋め込まれています。

### AuditData内のJSON構造

`AuditData` 列には以下のような複雑なJSON構造が含まれています。（サンプル）

```json
{
  "CreationTime": "2026-MM-DDT00:00:00",
  "Operation": "MessageSent",
  "UserId": "*",
  "ClientIP": "219[.]59[.]*[.]*",
  "ParticipantInfo": {
    "HasForeignTenantUsers": true,
    "HasGuestUsers": false,
    "HasOtherGuestUsers": false,
    "HasUnauthenticatedUsers": false,
    "ParticipatingDomains": ["yourdomain[.]com"],
    "ParticipatingTenantIds": [
      "00000000-0000-0000-0000-000000000000",
      "00000000-0000-0000-0000-000000000000"
    ]
  },
  "ExtraProperties": [
    {"Key": "Country", "Value": "jp"},
    {"Key": "ClientName", "Value": "skypeteams"},
    {"Key": "OsName", "Value": "windows"}
  ]
}
```

このJSON構造の中で、監査において特に重要なのは `ParticipantInfo` セクションです。`HasForeignTenantUsers` は、外部テナント（別のMicrosoft 365組織または一般消費者向けのTeamsのテナント）のユーザーが参加しているかを示します。これが `true` の場合、組織外のユーザーとの通信が行われていることを意味し、セキュリティリスクの観点から注意が必要です。

ちなみに、`HasGuestUsers` は、自組織のテナントに正式に招待されたゲストユーザーが参加しているかを示します。ゲストユーザーはAzure ADのB2Bゲストとして組織のディレクトリに存在するため、外部テナントユーザーよりも管理された形でのアクセスとなります。


## おわりに

レコードの数が増えてくると、JSON内の重要フィールドを正規化しないと見にくいですね。。。まずは監査ログのエクスポートから始め、自組織の外部アクセスの状況を把握することから始めると良いなと思いました。

[^1]: [ゲスト アクセスと外部アクセスを使用して、組織外の人々とコラボレーションする](https://learn.microsoft.com/ja-jp/microsoftteams/communicate-with-users-from-other-organizations)
[^2]: [ゲスト アクセスと外部アクセスを使用して、組織外の人々とコラボレーションする](https://learn.microsoft.com/ja-jp/microsoftteams/communicate-with-users-from-other-organizations)
[^3]: [共有チャンネルでTeamsのユーザー体験を変える！Microsoft Teams Connect](https://blog.cloudnative.co.jp/11750/) 
[^4]: [監査ログ アクティビティ](https://learn.microsoft.com/ja-jp/purview/audit-log-activities)
