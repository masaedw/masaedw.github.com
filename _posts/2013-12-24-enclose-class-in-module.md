---
layout: post
title: "クラス定義はモジュールで囲む"
description: ""
category: ""
tags: [Ruby]
---
{% include JB/setup %}

モジュール内のクラスを定義をするときに、class MyMod::MyClass; endとするか、module MyMod; class MyClass; end; endとするか、どっちがいいのかなぁと思っていろんなgemのコードを読むと、圧倒的に後者であることが多い。

一見前者の方が簡潔に書けて良さそうな気がするが、定数を探すスコープが変わるという実用上の問題があるため、みんな後者を使う。具体的に言うと、内部のクラスで親モジュールの別のクラスを参照しようとする際、クラス名だけで参照可能にしたいなら、モジュールで囲む必要がある。

{% gist 8099033 %}
