---
layout: post
title: "Sequel::Modelを使うときはautoloadする"
description: ""
category: ""
tags: [Ruby]
---
{% include JB/setup %}

最近[Sequel](http://sequel.jeremyevans.net/)を使いはじめて、ハマったことについて。

[Sequel::Model](https://github.com/jeremyevans/sequel#sequel-models)を使うと、ARパターンを実現できるので、やってみたいなーと思って、こういうコードを書く。

{% gist 8056050 post.rb %}

ヤッターモデルの定義ができたよー。さて、アプリケーションクラスからこれを使おうっと。

{% gist 8056050 app.rb %}

さてさて実行……

{% gist 8056050 gistfile1.txt %}

!!突然の死!!
 
ググると[こんなの](http://stackoverflow.com/questions/13115550/defining-sequel-models-before-connecting)が出てくる。

>By design, Sequel requires the database connection be set up before model class definition, since it parses the database schema on model class creation. So you should set up your initialization code to connect to the database first.

Sinatra::Modelが設計上の選択としてコネクションを必要とするというのは理由の説明ではあるが、requireしたいんだけど、という質問の答えにはなっていない。問題なのは、requireされたコードがその場で実行されるということなので、解決策としては、requireを遅延させてやることである。かといって、例えばsinatraのconfigureブロックの中でrequireを行うなんてバカなことはやりたくない。

結論としてはautoloadを使う。これで、ファイル管理上の問題を解決できる。

{% gist 8056050 app-autoload.rb %}

