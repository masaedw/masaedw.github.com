---
layout: post
title: "Swift2: Control Flow"
description: ""
category: ""
tags: []
---
{% include JB/setup %}

iBooksのSwift本を読む続き。

* 辞書からデータが取得できたかできないかはnilが帰ってきたかどうかで判断できるので、 `if let value = dict[key] { ...` と書ける。

* for-in はまあ、はい。
* for もはい。

whileループの説明で突然 Snakes and Ladders というゲームが出てきた。

* 他の言語のdo-whileに相当するものがrepeat-while。repeat-whileはwhileより Snakes and Ladders には向いてる。

* switch-caseはexhaustiveでなければならない。(全パターン網羅してるか、default:が必要)
* フォールスルーはしない(ので、各ケース中にbreakは必要ない。やりたければfallthroughキーワードを使う)
* caseの条件ではletをつかって値の束縛(value binding)ができる。whereをつかうと値に対する条件も書ける。
* ネストしたループを抜けるためにループラベルをつけられる。`label: while ...` とかくと、`break label` でそのラベルのwhileを脱出できる。(またSnakes and Laddersが登場)
