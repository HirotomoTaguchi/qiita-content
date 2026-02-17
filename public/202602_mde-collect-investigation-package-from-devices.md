---
title: 今更ながら、Defender for Endpoint の調査パッケージがとても便利
tags:
  - Security
  - Defender
  - Microsoft365
  - MicrosoftDefenderXDR
  - MicrosoftSecurity
private: false
updated_at: '2026-02-18T06:17:57+09:00'
id: 71958f93e9e80073efa7
organization_url_name: null
slide: false
ignorePublish: false
---

セキュリティインシデントが発生したとき、「あのマシンで何が起きていたのか？」を迅速に把握することが対応のポイントになることがありますが、Microsoft Defender for Endpoint（MDE）には 調査パッケージ（Investigation Package） という機能があり、これが思いのほか強力です。今回は改めてその魅力をご紹介します。（新しい機能でもなんでもなく、今更感ありますが・・・）

## 注意事項

- 本記事は2026年2月6日時点の情報を元に執筆しています。

## 調査パッケージとは？

調査パッケージとは、インシデント調査に役立つフォレンジック情報をデバイスから一括収集し、ZIP ファイルとして取得できる機能です。エンドポイントに直接ログインしなくても、Microsoft 365 Defender ポータルからリモートで収集できる点が最大の強みです。

## 調査パッケージの取り方（Microsoft 365 Defender ポータルから手動取得）

[Microsoft 365 Defender ポータル](https://security.microsoft.com) にアクセスします。そして、左メニューから 「デバイス」 を選択し、調査対象のデバイスを開きます。デバイス詳細ページ右上の 「...」（その他のアクション） をクリックします。「調査パッケージの収集」 を選択します。

![](https://github.com/user-attachments/assets/32a5eaa1-41a4-4a50-9f89-8672b36bb728)


収集が完了すると、「アクション センター」 から ZIP ファイルをダウンロードできます。簡単ですね。収集には数分〜十数分かかる場合があります。デバイスがオンラインであることが前提です。

![](https://github.com/user-attachments/assets/74d28321-5d36-4f8c-b33a-c6f7c814b7ac)

## 調査パッケージに含まれる項目

取得した ZIP を展開すると、以下のようなデータが含まれています。

![](https://github.com/user-attachments/assets/cd85d776-e1f9-4ce3-b0b2-4cefc9e005f8)

## セキュリティイベントログが .evtx 形式で取れるのが個人的なポイント

個人的に一番感動したのが、セキュリティイベントログ（Security.evtx）が、そのまま `.evtx` 形式で出力される点です。特に、テキストやCSVに変換されたログではなく、ネイティブの .evtx 形式で取得できるのがいいですね。その後の解析のフローを考えたときにとても嬉しいです。

## まとめ

調査パッケージは、インシデント対応において「まず何を確認すべきか」という問いに対する強力な回答を一括で提供してくれるツールです。特に `.evtx` 形式でのイベントログ取得は、既存の解析ワークフローにそのまま組み込めるため非常に実用的です。また、API や Logic Apps を活用した自動収集を組み込んでおくことで、インシデント発生直後の貴重な揮発性情報を取り逃がすリスクを大幅に低減できます。まだ使ったことがない方は、ぜひ次のインシデント対応演習で試してみてください。


*参考: [Microsoft Docs - Collect investigation package](https://learn.microsoft.com/en-us/defender-endpoint/respond-machine-alerts#collect-investigation-package-from-devices)*
