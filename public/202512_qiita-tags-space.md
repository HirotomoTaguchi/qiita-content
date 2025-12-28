---
title: Qiitaってタグにスペース使えないんだね。知らなくてCLIで時間溶けた
tags:
  - Qiita
  - QiitaCLI
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

元々、別のプラットフォームでブログを書いていたのですが、「Qiitaの方が伸びるよ」という噂を耳にしまして。正直、よこしまな気持ち全開でQiitaにやってきました。

## CLIで謎のエラー地獄

せっかくだから Qiita CLI を使って、GitHubで記事管理しようと思い、投稿してみたのですが、何度やってもエラー。。。

![](https://github.com/user-attachments/assets/67d4d986-5ae3-45a1-b0f5-2a82d8b3a3ff)

```エラーメッセージ
Qiita APIへのリクエストに失敗しました
  記事ファイルに不備がないかご確認ください
  または、Qiitaのアクセストークンが正しいかご確認ください
```

アクセストークンの設定を見直す。ファイル名を見直す。YAMLの構文も確認する。インデントも疑う。気づいたら2時間経ってました。

## 犯人はtagsのスペースでした

原因はこれでした。Qiitaのtagsにはスペースが使えないんですね。GUIでやればすぐに気が付くのですが、、、初記事からCLI使ってしまったので気が付くのが遅れました。

```
tags:
  - Qiita CLI  # ← これがダメ
  - Microsoft Defender     # ← これもダメ
```


正解はこうです。

```
tags:
  - QiitaCLI
  - MicrosoftDefender
```

## エラーメッセージ、もうちょっと親切にして...

エラーメッセージが「タグにスペースは使えません」みたいに直球で教えてくれればよかったんですが、そういう感じじゃなかったんですよね。まさかタグが原因だとは思わず。


## 教訓

- Qiitaのtagsにスペースは使えない

同じところでハマる人が減りますように。それでは、Qiitaライフ楽しんでいきます。
