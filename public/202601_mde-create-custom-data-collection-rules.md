---
title: Microsoft Defender for Endpoint カスタムデータ収集でEDRを拡張する
tags:
  - Security
  - Microsoft365
  - MicrosoftDefender
  - MicrosoftDefenderXDR
  - MicrosoftSecurity
private: true
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

Microsoft Defender for Endpoint(MDE)をはじめとするEDRは標準設定では膨大なエンドポイントのイベントの中から「重要そうなもの」だけをフィルタリングして記録する仕組みになっています。そうでもしないと、端末のすべての操作を記録していたらログ量が飛んでもないことになります。これは帯域やストレージコストの観点では合理的ですが、組織固有の監視要件には対応しきれない場合があります。そんな中、MDEではカスタムデータ収集機能がプレビューで出てきたので、簡単にまとめておきます。

## カスタムデータ収集とは何か

簡単に言えば、「MDEが通常は捨ててしまうイベントを、自分で指定して強制的に記録させる」機能です。通常、MDEはカーネルレベルで大量のイベントを観測していますが、すべてをクラウドに送信するわけにはいきません。そこでMicrosoft側が「これは重要そう」と判断したイベントだけを選別して送信しています。このフィルタリングは一般的な脅威には有効ですが、例えば、組織固有のレガシーアプリケーションの挙動監視が必要な場合、標準のフィルタでは重要なイベントが捨てられてしまうことがあります。

また、正規ツールを悪用する「Living off the Land」攻撃は、まさに標準設定の盲点を突いてきます。内部不正やシャドーITの調査では、一見「正常」に見える挙動こそが重要になりますし、長期的なコンプライアンス要件への対応には数年単位のデータ保持が求められます。カスタムデータ収集は、こうした「グローバルにはノイズだが、うちの環境ではシグナル」というイベントを拾うための仕組みです。

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
| DeviceCustomScriptEvents | 該当なし（独自機能） | スクリプト実行とプロセス詳細 | PowerShell以外のスクリプトエンジン完全監査 |

特に注目すべきは DeviceCustomScriptEventsです。このテーブルは標準の高度なハンティングスキーマには対応するものが存在しない、カスタムデータ収集固有の強力な機能です。標準のMDEでもPowerShellのログは充実していますが、その他のスクリプトエンジン（JScript、VBScript、Batch）や特定の非標準的なスクリプト実行は可視化されにくい状況があります。このテーブルを利用することで、特定のインタプリタやスクリプトファイルに対する完全な監査証跡を作成できます。

## 実践例：レガシースクリプトエンジンの完全監査

ここからは実際のユースケースを一つ取り上げて、設定から分析までの流れを解説します。今回はDeviceCustomScriptEventsを活用したレガシースクリプトの監視を取り上げます。

### 背景と課題

PowerShellは現代のWindows環境における標準的なスクリプト言語として広く使われており、MDEでも厳重に監視されています。しかし、攻撃者はこの「監視されている」事実を知っているため、あえて監視が手薄なwscript.exeやcscript.exe（VBScriptやJScriptのエンジン）を利用して活動することがあります。

これらのレガシースクリプトエンジンは、Windows環境に標準で存在し、多くの組織で無効化されていません。特に古い管理スクリプトやバッチ処理が残っている環境では、完全に無効化することが難しい場合もあります。標準のMDEではこれらの実行が完全にログ化されるとは限らず、攻撃者にとって「見えにくい経路」となってしまいます。

また、内部不正のシナリオでも、技術的に詳しいユーザーがVBScriptを使ってファイルを大量にコピーしたり、外部に送信したりする可能性があります。こうした挙動を検知するには、レガシースクリプトエンジンの実行を網羅的に記録する必要があります。

### 実装手順

実装は段階的に進めることをお勧めします。まずは小規模な範囲で開始し、ログ量や検出内容を確認してから全社展開するアプローチです。

#### ステップ1：動的タグの作成

まず監視対象デバイスを定義します。全社展開する前に、テスト用のタグを作成します。

Defender ポータルで「設定」→「エンドポイント」→「Asset rule management」を開き、「Add rule」で新規ルールを作成します。条件を設定する際は、例えば特定の部署や、VBScriptの使用が許可されていないはずの部門、あるいは重要資産を持つユーザーのデバイスなどを指定します。タグ名は分かりやすく、例えば「Legacy_Script_Monitor」のように設定します。

このタグは動的なので、条件に合致するデバイスが自動的に含まれ、デバイスの追加や削除が自動で反映されます。

#### ステップ2：カスタム収集ルールの作成

次にデータ収集ルールを定義します。「設定」→「エンドポイント」→「Custom data collection」を開き、「Create rule」を選択します。

基本情報では、Rule nameを「Monitor Legacy Script Engines」のように設定し、Descriptionには「wscript.exeとcscript.exeによるすべてのスクリプト実行を記録」と記入します。

スコープには先ほど作成した「Legacy_Script_Monitor」タグを選択します。これにより、タグが付与されたデバイスのみが監視対象となります。

データ収集設定では、Event typeとして「DeviceCustomScriptEvents」を選択します。この独自テーブルを使うことが今回のポイントです。Filter conditionsでは、InitiatingProcessFileNameが「wscript.exe」または「cscript.exe」であるイベントをすべて記録するように設定します。これにより、これらのエンジンで実行されたすべてのスクリプトの詳細が記録されます。

#### ステップ3：ルールの展開と確認

ルールを保存すると、約20分から1時間でターゲットデバイスに配布されます。展開状況は「Custom data collection」画面で確認できます。

実際にテストデバイスでVBScriptを実行してみて、Sentinelでデータが届いているか確認します。簡単なテストスクリプトとして、次のようなものを使えます。

```vbscript
' test.vbs
MsgBox "Test VBScript Execution"
```

これをcscript.exeまたはwscript.exeで実行し、数分後にSentinelでログが確認できれば成功です。

### データ分析とアラート作成

Sentinelに収集されたデータを活用して、実際の脅威検知と分析を行います。

#### 基本的な確認クエリ

まずは過去24時間にどのようなレガシースクリプトが実行されたかを確認するクエリです。

```kql
DeviceCustomScriptEvents_CL
| where TimeGenerated > ago(24h)
| where InitiatingProcessFileName in~ ("wscript.exe", "cscript.exe")
| summarize ExecutionCount = count() by 
    DeviceName, 
    InitiatingProcessCommandLine,
    InitiatingProcessAccountName
| order by ExecutionCount desc
```

このクエリにより、どのデバイスで、どのユーザーが、どのスクリプトを何回実行したかが一覧表示されます。

#### 詳細分析クエリ

スクリプトの実行元パスやコマンドライン引数を含む詳細情報を取得するクエリです。

```kql
DeviceCustomScriptEvents_CL
| where TimeGenerated > ago(7d)
| where InitiatingProcessFileName in~ ("wscript.exe", "cscript.exe")
| extend 
    ScriptPath = extract(@'"([^"]+)"', 1, InitiatingProcessCommandLine),
    ScriptFolder = extract(@'"([A-Z]:\\[^"]+)\\[^\\]+$', 1, InitiatingProcessCommandLine)
| project 
    TimeGenerated,
    DeviceName,
    InitiatingProcessAccountName,
    InitiatingProcessFileName,
    ScriptPath,
    ScriptFolder,
    InitiatingProcessCommandLine,
    InitiatingProcessParentFileName
| order by TimeGenerated desc
```

このクエリでは、実行されたスクリプトのパスを抽出し、どのフォルダから実行されたかを明確にします。また、親プロセスの情報も含めることで、スクリプトがどのようなコンテキストで実行されたかを把握できます。

#### 疑わしい実行パターンの検知

通常、正規の管理スクリプトは特定のパス（例：IT部門が管理する共有フォルダ）から実行されます。それ以外のパスからの実行は疑わしい可能性があります。

```kql
let ApprovedScriptPaths = dynamic([
    @"\\fileserver\scripts\",
    @"C:\IT\Scripts\"
]);
DeviceCustomScriptEvents_CL
| where TimeGenerated > ago(24h)
| where InitiatingProcessFileName in~ ("wscript.exe", "cscript.exe")
| extend ScriptPath = extract(@'"([^"]+)"', 1, InitiatingProcessCommandLine)
| where not(ScriptPath has_any (ApprovedScriptPaths))
| where ScriptPath has_any ("Downloads", "Temp", "Desktop", "AppData")
| project
    TimeGenerated,
    DeviceName,
    InitiatingProcessAccountName,
    ScriptPath,
    InitiatingProcessCommandLine,
    InitiatingProcessParentFileName
```

このクエリは、承認されたパス以外、特にユーザーのダウンロードフォルダや一時フォルダから実行されたスクリプトを検出します。

#### 夜間・休日の実行検知

業務時間外のスクリプト実行は、自動化されたタスクでない限り疑わしい可能性があります。

```kql
DeviceCustomScriptEvents_CL
| where TimeGenerated > ago(7d)
| where InitiatingProcessFileName in~ ("wscript.exe", "cscript.exe")
| extend 
    Hour = datetime_part("Hour", TimeGenerated),
    DayOfWeek = dayofweek(TimeGenerated)
| where Hour < 7 or Hour > 19 or DayOfWeek in (0d, 6d)  // 夜間または週末
| project
    TimeGenerated,
    DeviceName,
    InitiatingProcessAccountName,
    InitiatingProcessCommandLine,
    Hour,
    DayOfWeek
| order by TimeGenerated desc
```

#### アラートルールの作成

検知ロジックが固まったら、Sentinelの分析ルールとして設定します。例えば、「承認されていないパスからのVBScript実行」を検知するルールを作成します。

Sentinelで「分析」→「作成」→「スケジュール済みクエリルール」を選択し、先ほどのクエリを設定します。重要度は環境に応じて設定しますが、最初は「情報」レベルで開始し、誤検知の状況を見ながら調整することをお勧めします。

クエリの実行頻度は5分から15分程度が適切です。あまり頻繁に実行するとSentinelのコストが上がりますし、緊急性の低い検知であれば1時間おきでも十分な場合があります。

## 他の活用シーン

今回はDeviceCustomScriptEventsを詳しく解説しましたが、他のイベントタイプも強力です。

DeviceCustomImageLoadEventsを使えば、System32以外からロードされるDLLを監視することで、DLLサイドローディング攻撃の早期検知が可能になります。攻撃者は正規のアプリケーションと同じディレクトリに悪意のあるDLLを配置し、アプリケーション起動時にそのDLLを読み込ませる手法を使います。標準設定では署名されたDLLのロードはログ量が膨大になるため除外されることが多いですが、カスタム収集により特定のパスからのDLL読み込みを記録できます。

DeviceCustomFileEventsは、ハニートークン（おとりファイル）の監視に最適です。ファイルサーバーに「機密情報.xlsx」のような名前のおとりファイルを配置し、そのファイルへのアクセスをすべて記録します。標準設定ではファイルの読み取りはノイズとして除外されることがありますが、カスタム収集により、攻撃者や内部不正者がファイルを開こうとした瞬間を捉えることができます。

DeviceCustomNetworkEventsは、特定の業務アプリケーションのネットワーク通信をデバッグする際に役立ちます。レガシーシステムが通信エラーを起こしている場合、そのアプリケーションに関するすべてのネットワーク接続試行（成功・失敗問わず）を記録することで、問題の切り分けが容易になります。

DeviceCustomProcessEventsは、ポータブルアプリや未承認ソフトウェアの検出に使えます。ユーザー領域から直接実行される実行ファイルを網羅的に収集することで、管理外のソフトウェアの使用状況を可視化できます。

## コスト管理とスケーリング

カスタムデータ収集を本格的に展開する際、コスト管理は避けて通れない課題です。Sentinelはデータ取り込み量に基づいて課金されるため、無計画な収集は予算を殺します。

まず重要なのは、ルールを作成する前に予想されるログ量を見積もることが重要そうです。少数のデバイス（5台から10台程度）に対して1週間ほどテスト運用し、1デバイスあたりの1日のイベント数を測定します。

フィルタ条件は可能な限り具体的にすることをお勧めします。例えば、「すべてのファイル操作」ではなく「特定のフォルダ配下のファイル操作」に限定する、「すべてのネットワーク接続」ではなく「特定のポートへの接続」に限定する、といった工夫が必要です。

25,000イベントの上限についても意識する必要があります。この上限に頻繁に達するようであれば、フィルタ条件を見直すか、複数のルールに分割することを検討します。Sentinel側でイベント数を監視し、上限に近づいているルールを検知するアラートを設定しておくと、問題を早期に発見できます。

また、動的タグを活用したスコープ管理も重要です。全社展開が必要でない場合は、特定の部署や重要資産のみをターゲットにすることで、コストを抑えつつ必要な可視性を確保できます。インシデント発生時には、対象デバイスに特定のタグを付与することで、オンデマンドで詳細な監視を開始する「インシデントドリブン」な運用も効果的です。

## AMAとのすみわけ

TBA

## まとめ

TBA
