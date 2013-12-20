---
layout: post
title: アクティブパターンでエラー処理をする
tags : [F#]
---

この記事は [F# Advent Calendar 2011](http://partake.in/events/1c24993a-c475-4fc2-bca4-7a1cd2f81869) の参加記事です。

JavaやC#のような言語でプログラムを書いていると、処理本体は短いのに、なんでこんなに長くなっちゃうんだろう……ということがよくあります。関数の頭でnullチェックをして、Exceptionをキャッチするためにtry...finallyで囲んで、等とやっていると本体がどこにあるのかわからなくなるということは日常茶飯事です。こういったコードはどこが重要な場所なのかがわかりにくく、メンテナンス性が低いものです。

現代的なライブラリや言語ではできるだけ本質的なところだけを書けるようにいろいろな仕組みが導入されています。
今日はそういった仕組みの一つとして F# のアクティブパターンを紹介します。

社内勉強会のプログラミング添削会(?)で書いたじゃんけんプログラムを例にとって説明します。

まずアクティブパターンを使わないで書いた場合のプログラムがこれです。
Rock,Paper,Scissorsのうちどれかを二つ選んで引数として渡すと、左か右のどちらがじゃんけんに勝っているかを判定して表示するものです。

{% highlight ocaml %}
type JankenHand =
    | Rock
    | Paper
    | Scissors

type State =
    | Draw
    | Right
    | Left

let Janken l r =
    match l, r with
    | Rock, Rock      -> Draw
    | Rock, Paper     -> Right
    | Rock, Scissors  -> Left
    | Paper, Rock     -> Left
    | Paper, Paper    -> Draw
    | Paper, Scissors -> Right
    | Scissors, Rock  -> Right
    | Scissors, Paper -> Left
    | Scissors, Scissors -> Draw

let parse s =
    match (s:string).ToLower() with
    | x when x.StartsWith("r") -> Rock
    | x when x.StartsWith("p") -> Paper
    | x when x.StartsWith("s") -> Scissors
    | _ -> failwith "argument is not in (r,p,s)"

let main = function
    | [|_; l; r |] ->
        match Janken (parse l) (parse r) with
        | Draw -> "draw"
        | Left -> "Left wins"
        | Right -> "Right wins"
        |> printfn "%s"
        0
    | _ ->
        printfn "fsi Janken.fsx {R,P,S} {R,P,S}"
        1

main fsi.CommandLineArgs
{% endhighlight %}

アクティブパターンを使った版がこれです。

{% highlight ocaml %}
type JankenHand =
    | Rock
    | Paper
    | Scissors

type State =
    | Draw
    | Right
    | Left

let Janken l r =
    match l, r with
    | Rock, Rock      -> Draw
    | Rock, Paper     -> Right
    | Rock, Scissors  -> Left
    | Paper, Rock     -> Left
    | Paper, Paper    -> Draw
    | Paper, Scissors -> Right
    | Scissors, Rock  -> Right
    | Scissors, Paper -> Left
    | Scissors, Scissors -> Draw

let (|JankenHand|_|) j =
    match (j:string).ToLower() with
    | x when x.StartsWith("r") -> Some Rock
    | x when x.StartsWith("p") -> Some Paper
    | x when x.StartsWith("s") -> Some Scissors
    | _ -> None

let main = function
    | [|_; JankenHand l; JankenHand r|] ->
        match Janken l r with
        | Draw -> "draw"
        | Left -> "Left wins"
        | Right -> "Right wins"
        |> printfn "%s"
        0
    | _ ->
        printfn "fsi Janken.fsx {R,P,S} {R,P,S}"
        1

main fsi.CommandLineArgs
{% endhighlight %}

元々 parse  という関数だったのが (|JankenHand|_|) という関数に変わり、main関数の中のパターンマッチがそれに伴って変化しています。

最初の実装では main 関数の中で引数のパースに失敗したとき、例外が飛んでプログラムがスタックトレースを吐いて終了するようになってしました 。パースに失敗したとき例外が飛ぶようになっていると、スタックトレースを吐いて死なないようにするためには、parse部分を try...catch で囲む必要があります。しかも、引数のパースが失敗したときにプログラムの正しい使い方を表示するのであれば、catch 説の中に使い方の表示をするコードを書かねばならず、すぐ外側に既に存在している使い方表示とコードが重複してしまいます。

アクティブパターンをつかって処理する場合、パースが正常に完了したときにのみパターンにマッチするため、それ以上特別な例外処理を追加することなく、通常の条件分岐(match式、function式)の中で自然に例外的状況を取り扱うことができるようになっています。変更後のコードではおかしな入力があった場合にはスタックトレースを吐いて死ぬのではなく、使い方を表示して終了します。

このようにアクティブパターンを使って例外処理を書くことは、単に見た目に簡潔になるだけでなく、網羅的でもあります。あらゆるパターンを網羅していない場合にはコンパイルエラーになるため、XXの場合のケアを忘れていたためプログラムが落ちた、ということが起こり得ないのです。

パターンマッチとアクティブパターンはかっこいいif文以上の価値があるので積極的に使っていきたいですね。
