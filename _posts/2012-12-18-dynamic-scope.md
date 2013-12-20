---
layout: post
title: C# で Dynamic Scope
tags : [ASP.NET MVC]
---

この記事は [C# Advent Calendar 2012](http://atnd.org/events/33905) の18日目の記事です。

C# はとても良い言語ですが、しばしば他の言語のあれ(プログラマブルにメソッドを追加する機能、コンパイル時にコード走らせる機能、構造的部分型、パターンマッチ、Immutableで辞書のキーになれる型を簡単に作れる仕組み、返り値の型でのメソッドのディスパッチ、本物のマクロ、等々)が欲しいなぁと思うことはよくあると思います。今日はそういうものの中でも言語の枠組みの中でなんとかできそうなダイナミックスコープについて、実現する方法を考えてみます。

DynamicScope
===========


ごく稀に「この処理の実行中だけ、この変数の値を一時的に別の値に変更したい」ということがあります。たとえば、HttpContext.Current.Userを参照するMVC4というライブラリがいて、Aさんでログイン中なんだけどBさんのつもりで処理をして欲しいときなんかが、まさにそうです。コードで書くとこんな感じです。

{% highlight csharp linenos %}
var originalUser = HttpContext.Current.User;
HttpContext.Current.User = tempUser;

// tempUser さんがログインしてるつもりでとても重要な処理を実施する。
SomeImportantProcess();

HttpContext.Current.User = originalUser;
{% endhighlight %}

この一見動きそうなコードですが重要な問題があります。それはSomeImportantProcess()の中で例外が投げられた場合、HttpContext.Current.UserはtempUserのままになってしまうという点です。これを修正してみます。

{% highlight csharp linenos %}
var originalUser = HttpContext.Current.User;
HttpContext.Current.User = tempUser;

try
{
  // tempUser さんがログインしてるつもりでとても重要な処理を実施する。
  SomeImportantProcess();
}
finally
{
  HttpContext.Current.User = originalUser;
}
{% endhighlight %}

怠惰なプログラマの皆様におかれましてはもうやってらんない状況です。originalUserという変数が新たに出現するのもダサいですし、finallyを明示的に呼び出すなんて、よく訓練されたC#プログラマには我慢できません。

この手の「変数を一時的に別の値に置き換える」というのは広義のダイナミックスコープと言えます。であれば、ダイナミックスコープであると直接表現したいところです。理想的にはこんな感じです。

{% highlight csharp linenos %}
dynamicScope (HttpContext.Current.User = tempUser)
{
  // tempUser さんがログインしてるつもりでとても重要な処理を実施する。
  SomeImportantProcess();
}
{% endhighlight %}

残念ながらC#にはマクロが無いので、変わりにusing構文で妥協しましょう。丁度C#の世界にはTransactionScopeというものもありますし、そんな感じにできないものでしょうか。

{% highlight csharp linenos %}
using (DynamicScope(HttpContext.Current.User = tempUser))
{
  // tempUser さんがログインしてるつもりでとても重要な処理を実施する。
  SomeImportantProcess();
}
{% endhighlight %}

こんな感じですが、もうちょっとだけC#での実装に合わせて妥協します。

{% highlight csharp linenos %}
using (DynamicScope.Create(() => HttpContext.Current.User, tempUser))
{
  // tempUser さんがログインしてるつもりでとても重要な処理を実施する。
  SomeImportantProcess();
}
{% endhighlight %}

第一引数で変数を指定し、第二引数で一時的に上書きたい値を指定します。ギリギリ許せますね。

実装してみます。

{% highlight csharp linenos %}
public static class DynamicScope
{
    public static IDisposable Create<U>(Expression<Func<U>> expression, U value)
    {
        return new DynamicScope<U>(expression, value);
    }
}

/// <summary>
/// ダイナミックスコープをDisposableとして実現するクラス
/// </summary>
class DynamicScopeImpl<TValue> : IDisposable
{
    Action ResetAction;

    public DynamicScopeImpl(Expression<Func<TValue>> expression, TValue value)
    {
        var rhs = Expression.Parameter(typeof(TValue), "value");
        var body = expression.Body;
        var assign = Expression.Assign(body, rhs);
        var actionExp = Expression.Lambda<Action<TValue>>(assign, rhs);

        var action = actionExp.Compile();
        var original = expression.Compile()();

        action(value);

        ResetAction = () => action(original);
    }

    public void Dispose()
    {
        if (ResetAction != null)
        {
            ResetAction();
            ResetAction = null;
        }
    }
}
{% endhighlight %}

第一引数はrefでも良さそうに見えますが、refはクロージャで持ち回すことができなかったので、Expressionになっています。もしかしたらILをどうこうしたりExpressionを頑張ればrefでもいけるかもしれません。識者の登場が待たれます。

荒削りではありますがこれである程度安心してグローバル変数を上書きできるようになりました! エラー処理は読者への宿題とします!

マルチスレッド対応は……スレッドコンテキストスイッチをフックできればなんとかなりそうですが、そんなことできるのかしら？識者の方教えてください。

テストも書いてみます。

{% highlight csharp linenos %}
[TestMethod]
public void DynamicScopeTest_Object()
{
    var x = Utility.CreateDefault<DefaultTestClass>();

    Assert.AreEqual("hoge", x.Name);

    using (Utility.DynamicScope(() => x.Name, "name"))
    {
        Assert.AreEqual("name", x.Name);
    }

    Assert.AreEqual("hoge", x.Name);
}

static int GlobalVariable = 5;

[TestMethod]
public void DynamicScopeTest_Global()
{
    Assert.AreEqual(5, GlobalVariable);

    using (Utility.DynamicScope(() => GlobalVariable, 6))
    {
        Assert.AreEqual(6, GlobalVariable);
    }

    Assert.AreEqual(5, GlobalVariable);
}

[TestMethod]
public void DynamicScopeTest_Local()
{
    var now = DateTime.Now;
    var localVar = now;

    using (Utility.DynamicScope(() => localVar, now.AddMinutes(1)))
    {
        Assert.AreEqual(now.AddMinutes(1), localVar);
    }

    Assert.AreEqual(now, localVar);
}

[TestMethod]
public void DynamicScopeTest_MultipleDispose()
{
    var now = DateTime.Now;
    var localVar = now;

    using (var scope = Utility.DynamicScope(() => localVar, now.AddMinutes(1)))
    {
        Assert.AreEqual(now.AddMinutes(1), localVar);
        scope.Dispose();
        Assert.AreEqual(now, localVar);
        localVar = now.AddSeconds(5);
    }

    Assert.AreEqual(now.AddSeconds(5), localVar);
}
{% endhighlight %}

このテストを書いてみてわかることは、Expressionは無名関数と同様に環境を保持しているように見えるということです。どうなっているんでしょうか？謎です。
