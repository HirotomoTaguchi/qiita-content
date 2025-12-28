---
title: デバイスコードフローを悪用して Microsoft 365 のアカウントを侵害するフィッシングキャンペーンが増えているらしいので軽くまとめる
tags:
  - "Security"
  - "Microsoft Security"
  - "SIEM & XDR"
  - "Entra ID"
private: false
updated_at: ""
id: null 
organization_url_name: null 
slide: false 
ignorePublish: false 
---

2024年ぐらいに流行っていたデバイスコードを悪用したフィッシング攻撃がまた増えているという話を小耳にはさみました。決して新しい攻撃手法とかではないですが、改めて対策等をまとめておきます。

:::note alert
本記事はあくまでメモです。対策についてのコメントを付記しているケースもありますが、いかなる網羅性や完全性も保証いたしかねます。
:::

## デバイスコードフローとは？

### デバイスコードフローが生まれた背景

仕組みを説明する前にこれが生まれた経緯から。Webブラウザを搭載していないデバイスや文字入力が困難なデバイスが、ユーザーの承認を通じてアクセストークンを取得するためのサインインフローです。[^1]。それらのデバイスをセットアップする時、キーボード入力を行うことは難しいので、画面に表示されたコードを別のデバイス（スマホやPCなど）で入力することでサインインを完了できる仕組みです。

### サインインの流れ

通常のデバイスコードフローは以下のように動作します。

1. デバイスが認証を要求し、短いコード（例「ABCD-1234」）を生成
2. ユーザーはスマホやPCで `https://microsoft.com/devicelogin` にアクセス
3. 表示されたコードを入力
4. 本人確認（通常はMFAを含む）を完了
5. 元のデバイスが自動的にログイン完了

<iframe width="560" height="315" src="https://www.youtube.com/embed/JU_-STABylw" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" loading="lazy" allowfullscreen></iframe>

## 悪用の流れ

### よくある手口

攻撃者はまず、WhatsApp、Signal、Microsoft Teamsなどの正規のメッセージングアプリを通じてターゲットに接触してくることが多いとのことです。この時、ターゲットに関連する信頼できる人物になりすまします。信頼関係ができたところで、攻撃者はオンライン会議やウェビナーへの招待を送ります。この招待メールやメッセージはMicrosoft Teamsの正規の招待状そっくりに作られていて、会議のタイトルと日時、参加方法の説明、そして「会議コード」として偽装されたデバイスコードが含まれます。被害者がリンクをクリックして「会議コード」を入力すると、正規のMicrosoft認証ページが表示されます。ユーザーは自分のアカウントでログイン（MFAも通過）しますが、攻撃者側で即座に有効なアクセストークンが生成され、攻撃者はパスワード不要でアカウントにアクセス可能になります。

![デバイスコードフロー](https://github.com/user-attachments/assets/9362b2d1-9549-4d76-85bd-d7f5ff8c5d62 "デバイスコードフロー")

重要なのは、リンク先が `https://microsoft.com/devicelogin` というMicrosoftの正規のURLであることです。そのため、URLフィルタリングやセキュリティソフトでは検出できません。ここでの問題は、ユーザーが正しく認証したにもかかわらず、その認証が攻撃者のために使われることです。アクセス権を得た攻撃者は、メールの検索と窃取を行います。「username」「password」「admin」「credentials」「secret」「ministry」などのキーワードでメールを検索し、侵害したアカウントから組織内の他のユーザーにフィッシングメールを送信して横展開を図ります。トークンが有効な限り（場合によっては数週間から数ヶ月）、継続的にアクセスし続けます。

デバイスコードフィッシングが厄介なのは、多要素認証（MFA）を突破します。ユーザー本人がMFAを完了するため、MFAだけでは防げません。また正規のURLを使用するため、CASBやSWG、FWなどでは検出するのが難しいです。トークンベースのアクセスなので、パスワード変更だけでは防げませんし、トークンの有効期限まで（場合によっては数ヶ月）アクセス可能で、デバイスの状態チェックなど、一部のセキュリティコントロールを迂回します。

### Storm-2372による攻撃の実態

Microsoftの調査によると、2024年時点では、Storm-2372というロシアの国家利益に関連する攻撃グループなどに積極的に悪用されているとのことです。[^2] 彼らが標的にしているのはヨーロッパ、北米、アフリカ、中東の政府機関、NGO（非政府組織）、ITサービスおよび技術企業、防衛産業といった重要インフラや機密情報を持つ組織です。

### 2025年には更に拡大

加えて、ProofpointはOAuthデバイスコードフィッシングを利用した複数のキャンペーンを自社のブログで報告しています。そこでは、脅威アクターがOAuthデバイス付与承認フローとQRコードを組み合わせてMicrosoftアカウントを侵害することを狙うフィッシングツールの存在や、2025年内で複数回にわたって、攻撃キャンペーンで悪用されていたことなどを報告しています。[^3]

## 対策

### 基本姿勢

最も効果的な対策は、条件付きアクセスポリシーでデバイスコードフローそのものをブロックすることです。マイクロソフト社も公式ドキュメントにおいてブロックを推奨しています。[^5]

![](https://github.com/user-attachments/assets/8e0d2485-e76f-4496-a066-d7f883c9327a)

:::note info
ブロックを推奨するぐらいならデフォルトオフにしてくれと文句を言いたいところですね。一応2025年2月ぐらいから、利用していないテナントに対してブロックのポリシーを配り始めてはいる[^3]ようなので、彼らの言い分としては放置しているということではないということですが、悪用されそうな機能は最初から手を打っておいてもいいと思うの。MSに強くFBしましょう。
:::

まずは影響を確認するため、レポート専用モードでポリシーを作成します。Microsoft Entra管理センターにサインインし、「保護 > 条件付きアクセス > ポリシー」に移動して、「新しいポリシー」を選択します。

- 分かりやすい名前を付けます（例「デバイスコードフローのブロック - 評価中」）。
- 割り当て設定では、ユーザーは「すべてのユーザー」を選択しますが、緊急アクセス用アカウント（Break Glassアカウント）を除外することが推奨です。
- ターゲットリソースは「すべてのクラウドアプリ」を選択します。
- 条件では「認証フロー」を選択し、「構成」を「はい」に設定して、「デバイスコードフロー」にチェックします。
- アクセス制御では「アクセスをブロック」を選択します。
- ポリシーの有効化では「レポート専用」を選択します（これが重要です）。

このポリシーをしばらく運用し、「サインインログ」で影響を確認します。


問題ないことを確認次第、レポートモード専用を辞めて本番適用します。



### どうしても使うアカウントはカスタム検出クエリで検出する

どうしても利用せざるを得ないアカウントは Microsoft Sentinelで以下のクエリを定期実行し、悪用をモニタリングするのが無難です。（ライセンスがある場合）下記はデバイスコードフィッシングの疑いのある活動を検出するクエリです。

```kusto
SigninLogs
| where TimeGenerated > ago(90d)
| where AuthenticationProtocol == "deviceCode"
| summarize Count = count() by AppDisplayName, UserPrincipalName, IPAddress
| order by Count desc
```

また、マイクロソフト社のブログでは、より不審なアクティビティに絞った検出を行うクエリが紹介されていました。

```kusto
// 疑わしいデバイスコード認証を検出
let suspiciousUserClicks = materialize(
    UrlClickEvents
    | where ActionType in ("ClickAllowed", "UrlScanInProgress", "UrlErrorPage") 
           or IsClickedThrough != "0"
    | where UrlChain has_any ("microsoft.com/devicelogin", 
                               "login.microsoftonline.com/common/oauth2/deviceauth")
    | extend AccountUpn = tolower(AccountUpn)
    | project ClickTime = Timestamp, ActionType, UrlChain, 
              NetworkMessageId, Url, AccountUpn
);
// 短時間内のリスクの高いサインインをチェック
let interestedUsersUpn = suspiciousUserClicks
    | where isnotempty(AccountUpn)
    | distinct AccountUpn;
let suspiciousSignIns = materialize(
    AADSignInEventsBeta
    | where ErrorCode == 0
    | where AccountUpn in~ (interestedUsersUpn)
    | where RiskLevelDuringSignIn in (10, 50, 100)
    | extend AccountUpn = tolower(AccountUpn)
    | join kind=inner suspiciousUserClicks on AccountUpn
    | where (Timestamp - ClickTime) between (-2min .. 7min)
    | project Timestamp, ReportId, ClickTime, AccountUpn, 
              RiskLevelDuringSignIn, SessionId, IPAddress, Url
);
suspiciousSignIns
```

また、新規デバイス登録の監視用クエリです。

```kusto
CloudAppEvents
| where AccountDisplayName == "Device Registration Service"
| extend DeviceName = tostring(parse_json(tostring(RawEventData.ModifiedProperties))[1].NewValue)
| extend DeviceId = tostring(parse_json(tostring(parse_json(tostring(RawEventData.ModifiedProperties))[6].NewValue))[0])
| extend UserPrincipalName = tostring(RawEventData.ObjectId)
| where TimeGenerated > ago(24h)
| project TimeGenerated, DeviceName, DeviceId, UserPrincipalName
```

### やられちゃった場合にどうするか？（メモ）

やられちゃった場合にどうするかの対応の一例をメモを書いておきます。

:::note alert
あくまでメモです。網羅性は保証していません。
:::

#### 応急処置

最優先でユーザーアカウントの無効化を行います。Microsoft Entra管理センター > ユーザー > 対象ユーザーと進み、「サインインをブロック」を有効化します。

また、すべてのリフレッシュトークンの取り消しを行います。[^4]

```powershell
Revoke-MgUserSignInSession -UserId <UserId>
```

不正なデバイスが登録されてしまっている場合は登録デバイスの確認と削除も必要です。Microsoft Entra管理センター > デバイスと進み、疑わしいデバイスを検索して削除します。

#### 侵害範囲の調査

攻撃者が侵害したアカウントから他のユーザーにフィッシングメールを送信した可能性があるため、送信メールの確認も必要です。

```kusto
EmailEvents
| where SenderFromAddress == "suspect@contoso.com"
| where TimeGenerated > ago(30d)
// 攻撃アクターの手口として、ミーティングの招待をよく使うのでその条件で調べてみる
| where Subject contains "meeting" or Subject contains "invitation"
| project TimeGenerated, RecipientEmailAddress, Subject, Url
```

TBA

## まとめ

デバイスコードフィッシングは2024年8月から急増し、2025年に入ってさらに進化を続けています。この攻撃は正規のMicrosoft認証システムを悪用するため厄介ですが、ちゃんと対策しておきましょう。

[^1]: [Microsoft identity platform and the OAuth 2.0 device authorization grant flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-device-code)
[^2]: [Storm-2372がデバイスコードフィッシングキャンペーンを実施しています](https://www.microsoft.com/en-us/security/blog/2025/02/13/storm-2372-conducts-device-code-phishing-campaign/)
[^3]: [Access granted: phishing with device code authorization for account takeover](https://www.proofpoint.com/us/blog/threat-insight/access-granted-phishing-device-code-authorization-account-takeover)
[^4]: [user: revokeSignInSessions](https://learn.microsoft.com/ja-jp/graph/api/user-revokesigninsessions?view=graph-rest-1.0&tabs=http)
[^5]: [条件付きアクセス ポリシーを使用して認証フローをブロックする](https://learn.microsoft.com/ja-jp/entra/identity/conditional-access/policy-block-authentication-flows)
