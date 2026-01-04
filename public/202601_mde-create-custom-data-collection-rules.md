---
title: Microsoft Defender for Endpoint カスタムデータ収集でDefenderログ（EDR）を拡張する
tags:
  - Security
  - Microsoft365
  - MicrosoftDefender
  - MicrosoftDefenderXDR
  - MicrosoftSecurity
private: true
updated_at: '2026-01-04T22:36:54+09:00'
id: ff082cacd6c2ddfc1292
organization_url_name: null
slide: false
ignorePublish: false
---

Microsoft Defender for Endpoint(MDE)をはじめとするEDRは、標準設定では膨大なエンドポイントのイベントの中から「重要そうなもの」だけをフィルタリングして記録する仕組みになっていることが多いです。そうでもしないと、端末のすべての操作を記録していたらログ量が飛んでもないことになります。これは帯域やストレージコストの観点では合理的ですが、組織固有の監視要件には対応しきれない場合があります。そんな中、MDEではカスタムデータ収集機能という、これまで取得していなかったログを取得する機能がプレビューで出てきたので、簡単にまとめておきます。

## カスタムデータ収集とは何か

簡単に言えば、「MDEが通常は捨ててしまうイベントを、自分で指定して強制的に記録させる」機能です。通常、MDEはカーネルレベルで大量のイベントを観測していますが、すべてをクラウドに送信するわけにはいきません。そこでMicrosoft側が「これは重要そう」と判断したイベントだけを選別して送信しています。このフィルタリングは一般的な脅威には有効ですが、例えば、組織固有のアプリケーションの挙動監視が必要な場合、標準のフィルタでは重要なイベントが捨てられてしまうことがあります。

また、正規ツールを悪用する「Living off the Land」攻撃（環境規制型攻撃）は、まさに標準設定の盲点を突いてきます。内部不正やシャドーITの調査では、一見「正常」に見える挙動こそが重要になりますし、長期的なコンプライアンス要件への対応には数年単位のデータ保持が求められます。カスタムデータ収集は、こうした「グローバルにはノイズだが、うちの環境ではシグナル」というイベントを拾うための仕組みです。

## 制約事項

- ライセンス要件として、MDE P2またはM365 E5が必須です。P1では使えません。
- Microsoft Sentinelとの連携が必須条件です。収集したデータは通常のDefenderポータルではなく、SentinelのLog Analyticsワークスペースに直接送られます。まあ、2026年6月には強制連携させられるので、大きな点ではない気がします。
- 制限値についても把握しておく必要があります。1ルールあたり、1デバイスあたり24時間で最大25,000イベントまでしか収集されません。フィルタが甘いとすぐ上限に達して収集が止まってしまいます。
- 対応OSはWindows 10/11およびWindows Server 2019/2022のみです。

## サポートされるイベントテーブルとスキーマ

カスタムデータ収集は、すべてのシステムイベントを無差別に収集するものではありません。現在、以下の5つの特定カテゴリがサポートされており、それぞれがSentinel内で固有のテーブルにマッピングされます。

| テーブル名 | 対応する標準テーブル | 収集内容 | 主な用途 |
|-----------|-------------------|---------|---------|
| DeviceCustomProcessEvents | DeviceProcessEvents | プロセス作成、終了、親子関係 | 標準ではフィルタされる正規プロセスの挙動や短命なプロセスの捕捉 |
| DeviceCustomNetworkEvents | DeviceNetworkEvents | TCP/UDP接続、IP、ポート、プロトコル | 失敗した接続や内部通信の詳細な監査ログ取得 |
| DeviceCustomFileEvents | DeviceFileEvents | ファイルの作成、変更、削除、アクセス | 機密ファイルへのアクセス監視、一時ファイルの痕跡捕捉 |
| DeviceCustomImageLoadEvents | DeviceImageLoadEvents | DLL/実行ファイルのメモリロード | DLLサイドローディング攻撃やハイジャックの検知 |
| DeviceCustomScriptEvents | 該当なし（独自機能） | スクリプト実行とプロセス詳細 | スクリプト監査 |

特に注目すべきは DeviceCustomScriptEventsだと思います。このテーブルは標準の Advanced Hunting スキーマには対応するものが存在しない、カスタムデータ収集固有の強力な機能です。標準のMDEでもPowerShellのログは充実していますが、その他のスクリプトエンジン（JScript、VBScript、Batch）や特定の非標準的なスクリプト実行は可視化されにくい状況があります。このテーブルを利用することで、特定のインタプリタやスクリプトファイルに対する完全な監査証跡を作成できます。

## 検証：PowerShell 難読化の可視化

ここからは、実際に攻撃者がよく使用する手口をシミュレートし、それを「標準機能」と「カスタムデータ収集」でどう見えるかが異なるかを見ていきます。題材として、攻撃者が検知回避のために行う「PowerShell の Base64 難読化（EncodedCommand）」を取り上げます。

注意：

### 背景と課題

攻撃者は、セキュリティ製品の監視の目を逃れるために、実行するコマンドを隠そうとします。その最もポピュラーな手法が、PowerShell の `-EncodedCommand` オプションを使用した Base64 エンコードです。

標準の MDE ログ（`DeviceProcessEvents`）では、プロセスが起動された事実は記録されますが、その引数（コマンドライン）は攻撃者が隠ぺいした「意味不明な英数字の羅列」のまま記録されます。これでは、SOCアナリストが「何か怪しいコマンドが走った」ことまでは分かっても、具体的に「Mimikatz が実行されたのか」「ランサムウェアがダウンロードされたのか」を即座に判断することができません。解読（デコード）の手間が発生し、初動対応の遅れに直結します。

### 今回のデモシナリオ

実際に以下の PowerShell コマンドを実行して、ログの見え方を比較してみます。これは、「Mimikatz の実行」を模した無害なメッセージを表示するだけのコマンドを、Base64 で隠ぺいして実行するシナリオです。

＜難読化する前のコマンド＞

```powershell
# ただのメッセージを表示するだけのコマンドです
powershell.exe -Command "Write-Host '【DEMO】Critical Alert: Mimikatz Execution Detected!' -ForegroundColor Red; Start-Sleep -Seconds 10"
```


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

| 項目 | 記録内容 |
| --- | --- |
| **ActionType** | `ProcessCreated` |
| **FileName** | `powershell.exe` |
| **ProcessCommandLine** | `powershell.exe -EncodedCommand VwByAGkAdABlAC0ASABvAHMAdAAg...` |
| **評価** | ❌ **中身不明**。「何か長いコマンド」であることしか分からず、危険性の判断にはデコード作業が必須です。 |

![](https://github.com/user-attachments/assets/5407ecc7-a8f7-4b63-93be-f9db427ac63b)

### 事前設定


### カスタムデータ収集 (DeviceCustomScriptEvents) での見え方

一方、今回紹介するカスタムデータ収集（`DeviceCustomScriptEvents`）を有効にすると、Windows の AMSI (Antimalware Scan Interface) と連携し、メモリ上で展開された**実行直前のコード**が記録されます。

| 項目 | 記録内容 |
| --- | --- |
| **ActionType** | `AmsiScriptContent` |
| **ScriptContent** | `Write-Host '【DEMO】Critical Alert: Mimikatz Execution Detected!' -ForegroundColor Red; Start-Sleep -Seconds 10` |
| **評価** | ✅ **完全可視化**。隠されていた「Mimikatz」の文字列が平文ではっきり見えています。デコード不要で即座に悪性と断定可能です。 |

## 全部のログを取り巻くっていたら、破産まっしぐら

- TBA

## AMAとのすみわけ

- TBA

## 他の活用シーン

今回は PowerShell の難読化解除に焦点を当てましたが、`DeviceCustomScriptEvents` は他にも多様なシナリオで威力を発揮しそうです。まだ検証が追いついていない部分があるので、随時アップデートするか他のブログにします。

- TBA

## まとめ

Microsoft Defender for Endpoint のカスタムデータ収集機能は、これまで「ブラックボックス」だった領域に光を当てる強力な機能です。コスト（Sentinel ログ取り込み量）とのバランスを考慮する必要はありますが、興味がある方は、まずは重要資産や管理者端末など、範囲を限定してスモールスタートとしてみるといいと思います。
