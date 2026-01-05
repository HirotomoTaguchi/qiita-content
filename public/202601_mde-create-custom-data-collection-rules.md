---
title: Microsoft Defender for Endpoint カスタムデータ収集でDefenderログ（EDR）を拡張する
tags:
  - Security
  - Microsoft365
  - MicrosoftDefender
  - MicrosoftDefenderXDR
  - MicrosoftSecurity
private: true
updated_at: '2026-01-05T20:36:26+09:00'
id: ff082cacd6c2ddfc1292
organization_url_name: null
slide: false
ignorePublish: false
---

Microsoft Defender for Endpoint(MDE)をはじめとするEDRは、膨大なエンドポイントのイベントを分析しますが、ログの保存においては「重要そうなもの」だけをフィルタリングして保存する仕組みになっていることが多いです。そうでもしないと、端末のすべてのログを記録していたらログ量が飛んでもないことになります。これは帯域やストレージコストの観点では合理的ですが、組織固有の監視要件には対応しきれない場合があります。そんな中、MDEではカスタムデータ収集機能という、これまで取得していなかったログを取得する機能がプレビューで出てきたので、簡単にまとめておきます。

## 注意事項

- 本記事は2026年1月6日時点の情報を元に執筆しています。
- この記事の一部の情報は、GAされる前に大幅に変更される可能性があるプレリリース製品に関するものが含まれています。

## カスタムデータ収集とは何か[^1]

簡単に言えば、「MDEが通常は捨ててしまうイベント指定して記録する」機能です。通常、MDEはカーネルレベルで大量のイベントを観測し、分析していますが、すべてをクラウドに保存するわけにはいきません。そこでMicrosoft側が「これは重要そう」と判断したイベントだけを選別・整形して保存しています。これはどのEDRでも仕組みは大体同じです。

このフィルタリングは一般的な脅威には有効ですが、例えば、組織固有のアプリケーションの挙動監視が必要な場合、標準のフィルタでは必要なイベントが捨てられてしまうことがあります。また、正規ツールを悪用する「Living off the Land」攻撃（環境規制型攻撃）は標準設定の盲点を突いてきます。カスタムデータ収集は、こうした「一般的にはノイズだが、うちの環境ではシグナル」というイベントを拾うための仕組みです。

## 制約事項

- ライセンス要件として、MDE P2またはM365 E5が必須です。P1では使えません。
- Microsoft Sentinelとの連携が必須条件です。収集したデータは通常のDefenderポータルではなく、SentinelのLog Analyticsワークスペースに直接送られます。まあ、2026年6月には強制連携させられるので、大きな点ではない気がします。
- 制限値についても把握しておく必要があります。1ルール及び1デバイスあたり24時間で最大25,000イベントまでしか収集されません。フィルタが甘いとすぐ上限に達して収集が止まってしまいます。
- 対応OSはWindows 10/11およびWindows Server 2019/2022のみです。

## サポートされるイベントテーブルとスキーマ

カスタムデータ収集は、すべてのシステムイベントを無差別に収集するものではありません。現在、以下の5つの特定カテゴリがサポートされており、それぞれがSentinelでの固有のテーブルにマッピングされます。[^1]

| テーブル名 | 対応する標準テーブル | 収集内容 | 主な用途 |
|-----------|-------------------|---------|---------|
| DeviceCustomProcessEvents | DeviceProcessEvents | プロセス作成、終了、親子関係 | 標準ではフィルタされる正規プロセスの挙動や短命なプロセスの捕捉 |
| DeviceCustomNetworkEvents | DeviceNetworkEvents | TCP/UDP接続、IP、ポート、プロトコル | 失敗した接続や内部通信の詳細な監査ログ取得 |
| DeviceCustomFileEvents | DeviceFileEvents | ファイルの作成、変更、削除、アクセス | 機密ファイルへのアクセス監視、一時ファイルの痕跡捕捉 |
| DeviceCustomImageLoadEvents | DeviceImageLoadEvents | DLL/実行ファイルのメモリロード | DLLサイドローディング攻撃やハイジャックの検知 |
| DeviceCustomScriptEvents | 該当なし（独自機能） | スクリプト実行とプロセス詳細 | スクリプト監査 |

特に注目すべきは DeviceCustomScriptEvents ですかね。このテーブルは標準の Advanced Hunting スキーマには対応するものが存在しない、カスタムデータ収集固有の強力な機能です。標準の Advanced Hunting でもコマンドラインのログは取得していますし、私の認識では、標準の Advanced Hunting において、MDEが脅威スコアが高いと判断した場合やアラートに関連する場合など、フィルタリングされた一部のログのみが保存されます。すべてのスクリプト実行が記録されるわけではなく、特に検知に至らないレベルの挙動や、完全に無害と判定されたスクリプトの内容は、容量節約のためにクラウドに保存されない（ドロップされる）傾向にあります。（注）

![](https://github.com/user-attachments/assets/4a928a8d-818b-4c3e-afc7-ea6dett)

:::note alert
注：私の認識では、実行されたスクリプトの情報はMDEで分析され、Defenderがアラートと判定している場合のみ確認でき、Advanced Huntingで参照できるログとしては保存されない認識ですが、公式ドキュメントにおいて、直接的にそれを裏付ける情報を見つけることができませんでした。見つけられた方、もしくは認識違いだと知っている方は優しく教えていただけたら幸いです。
:::

そこで、DeviceCustomScriptEvents テーブルを利用することで、特定のインタプリタやスクリプトファイルに対する監査証跡を明示的に保存できます。これは内部的に AMSI (Antimalware Scan Interface) のような機構を通じてデコードされたスクリプト内容を取得していると推測されますが、後述するように難読化解除後のコードを確認できるという強力な利点があります。

## 検証：PowerShell 難読化の可視化

ここからは、実際に攻撃者がよく使用する手口を簡易的にシミュレートし、それを標準機能である「DeviceProcessEvents」と「カスタムデータ収集(DeviceCustomScriptEvents)」でどう見えるかが異なるかを見ていきます。題材として、攻撃者が検知回避のために行う「PowerShell の Base64 難読化（EncodedCommand）」を取り上げます。

### 背景と課題

攻撃者は、セキュリティ製品の監視の目を逃れるために、実行するコマンドを隠そうとします。その最もポピュラーな手法が、PowerShell の `-EncodedCommand` オプションを使用した Base64 エンコードです。

標準の MDE ログ（`DeviceProcessEvents`）では、プロセスが起動された事実は記録されますが、その引数（コマンドライン）は攻撃者が隠ぺいした「意味不明な英数字の羅列」のまま記録されます。Defenderはこれらをバックエンドでデコードされたものを分析し[^2]、危険だと判定したら証拠を残しますが、Defenderのアラートの網にかからなかったものは捨てられてしまい、難読化されたものだけが残ります。

:::alert
何度も言うようですが、バックエンドではデコードされてMDEで分析されています。あくまで、記録として残るかという点が論点です。
:::

そこで、DeviceCustomScriptEvents テーブルを利用することで、特定のインタプリタやスクリプトファイルに対する監査証跡を明示的に保存できます。これは内部的に AMSI (Antimalware Scan Interface) のバッファを参照していると考えられ、後述するように難読化解除後のコードを確認できるという強力な利点があります。

### 今回のデモシナリオ

実際に以下の PowerShell コマンドを実行して、ログの見え方を比較してみます。これは、「Mimikatz の実行」を模した無害なメッセージを表示するだけのコマンドを、Base64 で隠ぺいして実行するシナリオです。攻撃っぽくはありませんが、わかりやすさの観点ではいいシナリオかと思いました。

＜難読化する前のコマンド＞

```powershell
# ただのメッセージを表示するだけのコマンドです
powershell.exe -Command "Write-Host '【DEMO】Critical Alert: Mimikatz Execution Detected!' -ForegroundColor Red; Start-Sleep -Seconds 10"
```
＜難読化する前のコマンドのDefenderポータルでの見え方＞

![](https://github.com/user-attachments/assets/73c8b0d6-2eef-4c73-8621-6625af84c2fe)


＜実行するテストコマンド（難読化済み）＞

```powershell
# 隠したいコマンド（ペイロード）
$OriginalCommand = "Write-Host '【DEMO】Critical Alert: Mimikatz Execution Detected!' -ForegroundColor Red; Start-Sleep -Seconds 10"

# コマンドをBase64に変換（難読化）
$Bytes = [System.Text.Encoding]::Unicode.GetBytes($OriginalCommand)
$EncodedCommand = [System.Convert]::ToBase64String($Bytes)

# 難読化された状態で実行（これがログに残る攻撃コマンドです）
Write-Host "実行コマンド: powershell.exe -EncodedCommand $EncodedCommand" -ForegroundColor Yellow
powershell.exe -EncodedCommand $EncodedCommand
```

＜標準ログ (DeviceProcessEvents) での見え方＞

既存のプロセス監視ログでは、以下のように記録されます。

![](https://github.com/user-attachments/assets/5407ecc7-a8f7-4b63-93be-f9db427ac63b)

`powershell.exe -EncodedCommand VwByAGkAdABlAC0ASABvAHMAdAAg...` となっており、内容を監査することはできません。なお、DeviceProcessEvents は新しいプロセスの起動のみを記録するため、エンコード処理を行った前段の PowerShell セッション内でのコマンド（$Bytes の代入や Base64 変換など）は記録されていません。これらはプロセス起動を伴わない、既存プロセス内での処理だからです。

### カスタムデータ収取の設定[^3]

事前設定として、SentinelとDefenderポータルを統合しておきます。

https://blog.cloudnative.co.jp/24112/

その後、[Defenderポータル] > [Settings] > [Endpoints] > [Custom Data Collection] から収集の条件を設定し、保存するだけです。今回は検証なので、非常に幅広にログ取得の設定をしました。

![](https://github.com/user-attachments/assets/7aa61bd8-a0c7-4b74-9bc0-b43d854cc1f2)


### カスタムデータ収集 (DeviceCustomScriptEvents) での見え方

有効にした後、ログを見ると、実行直前のスクリプトが Advanced Hunting で記録されてました。また、興味深い点として、DeviceProcessEvents では記録されなかったセッション内でのコマンド（`$Bytes` の代入や Base64 変換処理など）も DeviceCustomScriptEvents には記録されていました。これは、DeviceCustomScriptEvents がプロセス起動イベントではなく、AMSI のようなスクリプト実行監視の仕組みを通じて、PowerShell セッション内で実行されたコード全体を捕捉しているためと考えられます。

![](https://github.com/user-attachments/assets/1332b941-248f-441d-ae1d-5f7e0fa5651f)

## 全部のログを取り巻くっていたら、破産まっしぐら

カスタムデータ収集は無計画に使うとコストが跳ね上がります。定額性のAdvanced Huntingと違い、Sentinel のログ取り込み料金は従量課金なので「とりあえず全部取る」という発想は危険です。（制限があるので全部はとれないですが・・・）加えて、Advanced Huntingでもすごいログの量ですが、これまで保存をしてこなかったログの量を舐めてはいけません。ご利用は計画的に。

## AMAとのすみわけ

- TBU

## 他の活用シーン

今回は PowerShell の難読化解除に焦点を当てましたが、`DeviceCustomScriptEvents` は他にも多様なシナリオで威力を発揮しそうです。まだ検証が追いついていない部分があるので、随時アップデートするか他のブログにします。

## まとめ

Microsoft Defender for Endpoint のカスタムデータ収集機能は、これまで「ブラックボックス」だった領域に光を当てる強力な機能です。コスト（Sentinel ログ取り込み量）とのバランスを考慮する必要はありますが、興味がある方は、まずは重要資産や管理者端末など、範囲を限定してスモールスタートとしてみるといいと思います。

[^1]: [Microsoft Defender for Endpointでのカスタム データ収集 (プレビュー)](https://learn.microsoft.com/ja-jp/defender-endpoint/custom-data-collection)
[^2]: [Antimalware Scan Interface (AMSI)](https://learn.microsoft.com/en-us/windows/win32/amsi/antimalware-scan-interface-portal)
[^3]: [https://learn.microsoft.com/ja-jp/defender-endpoint/create-custom-data-collection-rules](https://learn.microsoft.com/ja-jp/defender-endpoint/create-custom-data-collection-rules)
