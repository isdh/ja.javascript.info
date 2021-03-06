
# Promises チェーン

チャプター <info:callbacks> で言及した問題に戻りましょう。

- 私たちは次々に実行される一連の非同期タスクを持っています。例えば、スクリプトの読み込みです。
- 上手くコード化するにはどうすればよいでしょう？

Promise はそれをするためのいくつかの方法を提供します。

[cut]

このチャプターでは promise チェーンを説明します。

次のようになります:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000); // (*)

}).then(function(result) { // (**)

  alert(result); // 1
  return result * 2;

}).then(function(result) { // (***)

  alert(result); // 2
  return result * 2;

}).then(function(result) {

  alert(result); // 4
  return result * 2;

});
```

この考え方は、結果が `.then` ハンドラのチェーンを通じて渡されるということです。

ここでの流れは次の通りです:
1. 最初の promise は1秒で解決されます `(*)`,
2. その後、`.then` ハンドラが呼ばれます `(**)`,
3. 返却された値は次の `.then` ハンドラへ渡されます `(***)`,
4. ...同様に続きます。

結果がハンドラのチェーンに沿って渡されるので、一連の `alert` 呼び出しは `1` -> `2` -> `4` の順番で表示されます。

![](promise-then-chain.png)

`promise.then` の呼び出しは promise を返すので、続けて次の `.then` を呼び出すことができます。そのためすべてのコードが機能します。

ハンドラが値を返すとき、それは promise の結果になります。なので、次の `.then` はそれと一緒に呼ばれます。

これらの言葉をより明確にするために、ここではチェーンの始まりがあります:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result);
  return result * 2; // <-- (1)

}) // <-- (2)
// .then…
```

`.then` により返却される値は promise であるため、`(2)` で別の `.then` を追加することができます。`(1)` で値が返却されるとき、その promise は解決されるため、次のハンドラはその値で実行されます。

チェーンとは異なり、技術的には次のように1つの promise へ多くの `.then` を追加することも可能です。:

```js run
let promise = new Promise(function(resolve, reject) {
  setTimeout(() => resolve(1), 1000);
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});

promise.then(function(result) {
  alert(result); // 1
  return result * 2;
});
```

...しかし、これは完全に別物です。ここに図があります(上記のチェーンと比較してください):

![](promise-then-many.png)

同一の promise 上のすべての `.then` は同じ結果を得ます -- その promise の結果です。従って、上のコードでは、すべての `alert` は同じ `1` を表示します。それらの間での結果渡しはありません。

実際には、単一の promise に対し複数のハンドラが必要なケースはほとんどありません。チェーンの方がはるかに多く利用されます。

## promise の返却　

通常、`.then` ハンドラにより返却された値は、直ちに次のハンドラに渡されます。しかし例外もあります。

もし返却された値が promise である場合、それ以降の実行はその promise が解決するまで中断されます。その後、promise の結果が次の `.then` ハンドラに渡されます。

例:

```js run
new Promise(function(resolve, reject) {

  setTimeout(() => resolve(1), 1000);

}).then(function(result) {

  alert(result); // 1

*!*
  return new Promise((resolve, reject) => { // (*)
    setTimeout(() => resolve(result * 2), 1000);
  });
*/!*

}).then(function(result) { // (**)

  alert(result); // 2

  return new Promise((resolve, reject) => {
    setTimeout(() => resolve(result * 2), 1000);
  });

}).then(function(result) {

  alert(result); // 4

});
```

ここで最初の `.then` は `1` を表示し、行 `(*)` で `new Promise(…)` を返します。1秒後、それは解決され、結果(`resolve` の引数, ここでは `result*2`) は行 `(**)` にある2番目の `.then` のハンドラに渡されます。それは `2` を表示し、同じことをします。

したがって、出力は再び 1 -> 2 > 4 ですが、今は `alert` 呼び出しの間に 1秒の遅延があります。

promise を返却することで、非同期アクションのチェーンを組み立てることができます。

## 例: loadScript 

`loadScript` でこの機能を使って、スクリプトを1つずつ順番にロードしてみましょう。:

```js run
loadScript("/article/promise-chaining/one.js")
  .then(function(script) {
    return loadScript("/article/promise-chaining/two.js");
  })
  .then(function(script) {
    return loadScript("/article/promise-chaining/three.js");
  })
  .then(function(script) {
    // それらがロードされていることを表示するために、スクリプトで宣言されている関数を使用
    one();
    two();
    three();
  });
```

ここで、各 `loadScript` 呼び出しは promise を返し、次の `.then` はそれが解決されたときに実行されます。その後、次のスクリプトのロードを開始します。そのため、スクリプトは次々にロードされます。

私たちは、このチェーンにより多くの非同期アクションを追加することができます。ここで、このコードは依然として "フラット" であることに注目してください。それは大きくなっていますが右にではありません。"破滅のピラミッド" の兆候はありません。

技術的にはそれぞれの promise の後に、次のように promise を返却することなく直接 `.then` を書くことも可能であることに留意してください。:

```js run
loadScript("/article/promise-chaining/one.js").then(function(script1) {
  loadScript("/article/promise-chaining/two.js").then(function(script2) {
    loadScript("/article/promise-chaining/three.js").then(function(script3) {
      // この関数は変数 script1, script2 と script3 へアクセスすることができます
      one();
      two();
      three();
    });
  });
});
```

このコードは同じことをします: 順番に3つのスクリプトをロードします。しかし、"右に大きくなります"。そのため、コールバックと同じ問題があります。それを避けるためにチェーン(`.then` から promise を返す)を使用してください。

ネストされた関数が外側のスコープ(ここでは最もネストしているコールバックはすべての変数 `scriptX` へアクセスできます)にアクセスできるため、 `.then` を直接書くこともできますが、それはルールではなく例外です。


````smart header="Thenables"
正確には、`.then` は任意の "thenable" オブジェクトを返す可能性があり、それは promise として同じように扱われます。

"thenable" オブジェクトとは、メソッド `.then` を持つオブジェクトです。

この思想は、サードパーティライブラリが彼ら自身の "promise 互換な" オブジェクトを実装できるというものです。それらは拡張されたメソッドのセットを持つことができますが、`.then` を実装しているため、ネイティブの promise とも互換があります。

これは thenable オブジェクトの例です:

```js run
class Thenable {
  constructor(num) {
    this.num = num;
  }
  then(resolve, reject) {
    alert(resolve); // function() { native code }
    // 1秒後に this.num*2 で resolve する
    setTimeout(() => resolve(this.num * 2), 1000); // (**)
  }
}

new Promise(resolve => resolve(1))
  .then(result => {
    return new Thenable(result); // (*)
  })
  .then(alert); // 1000ms 後に 2　を表示
```

JavaScript は行 `(*)` で `.then` ハンドラによって返却されたオブジェクトをチェックします: もし `then` という名前のメソッドが呼び出し可能であれば、ネイティブ関数 `resolve`, `reject` を引数として(executor 似ています)それを呼び出し、それらのいずれかが呼び出されるまで待ちます。上の例では、`resolve(2)` が1秒後に `(**)` で呼ばれます。その後、結果はチェーンのさらに下に渡されます。

この特徴により、カスタムオブジェクトを `Promise` から継承することなく、promise チェーンで統合することができます。
````


## より大きな例: fetch 

フロントエンドのプログラミングでは、promise はネットワークリクエストの場合にしばしば使われます。なので、その拡張された例を見てみましょう。

私たちは、リモートサーバからユーザに関する情報をロードするために [fetch](mdn:api/WindowOrWorkerGlobalScope/fetch) メソッドを使います。メソッドは非常に複雑で、多くの任意パラメータがありますが、基本の使い方はとてもシンプルです:

```js
let promise = fetch(url);
```

これは、`url` へネットワークリクエストを行い、promise を返します。promise はリモートサーバがヘッダーで応答するとき、*完全なレスポンスがダウンロードされる前に* `response` オブジェクトで解決されます。

完全なレスポンスを見るためには、`response.text()` メソッドを呼ぶ必要があります: これは完全なテキストがリモートサーバからダウンロードされたときに解決され、そのテキストを結果とする promise を返します。

以下のコードは `user.json` へリクエストを行い、サーバからそのテキストをロードします:

```js run
fetch('/article/promise-chaining/user.json')
  // この .then はリモートサーバが応答したときに実行されます
  .then(function(response) {
    // response.text() は、レスポンスのダウンロードが完了した際に
    // 完全なレスポンステキストで解決される新たな promise を返します
    return response.text();
  })
  .then(function(text) {
    // ...そして、ここではリモートファイルの中身が参照できます
    alert(text); // {"name": "iliakan", isAdmin: true}
  });
```

リモートデータを読んで、JSON としてパースするメソッド `response.json()` もあります。我々のケースでは、より一層便利なのでそれに置き換えてみます。

わかりやすくするために、アロー関数も使います:

```js run
// 上と同じですが、response.json() はリモートコンテンツを JSON としてパースします
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => alert(user.name)); // iliakan
```

次に、ロードしたユーザで何かしてみましょう。

例えば、github へもう1つリクエストを行い、ユーザプロフィールを読み込みアバターを表示させてみます。:

```js run
// user.json へのリクエスト
fetch('/article/promise-chaining/user.json')
  // json としてロード
  .then(response => response.json())
  // github へのリクエスト
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  // json としてロード
  .then(response => response.json())
  // ３秒間アバター画像を表示 (githubUser.avatar_url) 
  .then(githubUser => {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => img.remove(), 3000); // (*)
  });
```

このコードは動作します(コードの詳細についてはコメントをみてください)が、完全に自己記述的であるべきです。ここには promise を使い始める人が行う典型的な問題があります。

行 `(*)` を見てください: アバターの表示が終了して削除された *後* に何かをするにはどうすればいいでしょうか？例えば、ユーザ情報を編集するためのフォームを表示したいとします。今のところ、方法はありません。

チェーンを拡張可能にするには、アバターの表示が終了したときに resolve を行う promise を返す必要があります。

次のようになります:

```js run
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
*!*
  .then(githubUser => new Promise(function(resolve, reject) {
*/!*
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
*!*
      resolve(githubUser);
*/!*
    }, 3000);
  }))
  // 3秒後にトリガされます
  .then(githubUser => alert(`Finished showing ${githubUser.name}`));
```

今、`setTimeout` は `img.resolve()` を実行した直後に `resolve(githubUser)` を呼び出します。なので、チェーン内の次の `.then` に制御を渡し、ユーザデータを転送します。

ルールとして、非同期アクションは常に promise を返すべきです。

これは、あるアクションの後に別のアクションを実行させることができます。たとえ現時点ではチェーンの拡張予定はなくても、後で必要になるかもしれません。

最後に、先程のコードは再利用可能な関数に分割できます:

```js run
function loadJson(url) {
  return fetch(url)
    .then(response => response.json());
}

function loadGithubUser(name) {
  return fetch(`https://api.github.com/users/${name}`)
    .then(response => response.json());
}

function showAvatar(githubUser) {
  return new Promise(function(resolve, reject) {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
      resolve(githubUser);
    }, 3000);
  });
}

// 上記を使う:
loadJson('/article/promise-chaining/user.json')
  .then(user => loadGithubUser(user.name))
  .then(showAvatar)
  .then(githubUser => alert(`Finished showing ${githubUser.name}`));
  // ...
```

## エラーハンドリング 

非同期アクションは失敗する可能性があります: エラーの場合、対応する promise は reject されます。例えば、リモートサーバが利用不可で `fetch` が失敗する場合です。エラー(拒否/reject)を扱うには `.catch` を使います。

promise のチェーンはその点で優れています。promise が reject されると、コントロールはチェーンに沿って最も近い reject ハンドラにジャンプします。それは実際に非常に便利です。

例えば、下のコードでは URL が誤っており(存在しないサーバ)、`.catch` がエラーをハンドリングします:

```js run
*!*
fetch('https://no-such-server.blabla') // rejects
*/!*
  .then(response => response.json())
  .catch(err => alert(err)) // TypeError: failed to fetch (エラーメッセージ内容は異なる場合があります)
```

または、おそらくサーバはすべて正しく動作したが、応答が有効な JSON でない場合:

```js run
fetch('/') // fetch はうまく動作し、サーバは成功を応答します
*!*
  .then(response => response.json()) // rejects: ページが HTML で有効な json ではなかった場合
*/!*
  .catch(err => alert(err)) // SyntaxError: Unexpected token < in JSON at position 0
```

下の例では、アバターの読み込みと表示のチェーンでのすべてのエラーを処理する `.catch` を追加しています:

```js run
fetch('/article/promise-chaining/user.json')
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`))
  .then(response => response.json())
  .then(githubUser => new Promise(function(resolve, reject) {
    let img = document.createElement('img');
    img.src = githubUser.avatar_url;
    img.className = "promise-avatar-example";
    document.body.append(img);

    setTimeout(() => {
      img.remove();
      resolve(githubUser);
    }, 3000);
  }))
  .catch(error => alert(error.message));
```

ここでは、`.catch` はまったく呼ばれません。なぜならエラーが起きていないからです。しかし、上の promise のいずれかが reject となった場合、catch は実行されます。

## 暗黙の try..catch 

executor と promise ハンドラのコードは "見えない `try..catch`" を持っています。エラーが起きた場合、キャッチして reject として扱います。

例えば、このコードを見てください:

```js run
new Promise(function(resolve, reject) {
*!*
  throw new Error("Whoops!");
*/!*
}).catch(alert); // Error: Whoops!
```

...これは次のと同じように動作します:

```js run
new Promise(function(resolve, reject) {
*!*
  reject(new Error("Whoops!"));
*/!*  
}).catch(alert); // Error: Whoops!
```

executor にある "見えない `try..catch` はエラーを自動的にキャッチし reject として扱っています。

これは executor だけでなくハンドラの中でも同様です。`.then` ハンドラの中で `throw` した場合、promise の reject を意味するので、コントロールは最も近いエラーハンドラにジャンプします。

ここにその例があります:

```js run
new Promise(function(resolve, reject) {
  resolve("ok");
}).then(function(result) {
*!*
  throw new Error("Whoops!"); // promise を rejects
*/!*
}).catch(alert); // Error: Whoops!
```

また、これは `throw` だけでなく同様にプログラムエラーを含む任意のエラーに対してです:

```js run
new Promise(function(resolve, reject) {
  resolve("ok");
}).then(function(result) {
*!*
  blabla(); // このような関数はありません
*/!*
}).catch(alert); // ReferenceError: blabla is not defined
```

副作用として、最後の `.catch` は明示的な reject だけでなく、上記のハンドラのような偶発的なエラーもキャッチします。

## 再スロー 

すでにお気づきのように、`.catch` は　`try..catch` のように振る舞います。私たちは必要な数の `.then` を持ち、最後に単一の `.catch` を使用してすべてのエラーを処理します。

通常の `try..catch` では、エラーを解析し、処理できない場合は再スローする場合があります。promise でも同じことが可能です。`.catch`　の中で `throw` する場合、コントロールは次の最も近いエラーハンドラにジャンプします。そして、エラーを処理して正常に終了すると、次に最も近い成功した `.then` ハンドラに続きます。

下の例では、`.catch` がエラーを正常に処理しています:

```js run
// 実行: catch -> then
new Promise(function(resolve, reject) {

  throw new Error("Whoops!");

}).catch(function(error) {

  alert("The error is handled, continue normally");

}).then(() => alert("Next successful handler runs"));
```

ここでは、`.catch` ブロックが正常に終了しています。なので、次の成功ハンドラが呼ばれます。また何かを返すこともでき、その場合も同じです(値が渡されます)。

...そしてここでは `.catch` ブロックはエラーを解析し、再度スローしています:

```js run
// 実行: catch -> catch -> then
new Promise(function(resolve, reject) {

  throw new Error("Whoops!");

}).catch(function(error) { // (*)

  if (error instanceof URIError) {
    // エラー処理
  } else {
    alert("Can't handle such error");

*!*
    throw error; // ここで投げられたエラーは次の catch へジャンプします
*/!*
  }

}).then(function() {
  /* 実行されません */
}).catch(error => { // (**)

  alert(`The unknown error has occurred: ${error}`);
  // 何も返しません => 実行は通常通りに進みます

});
```

ハンドラ `(*)` はエラーをキャッチしましたが処理していません。なぜなら、`URIError` ではないからです。なので、再びスローします。その後、実行は次の `.catch` へ移ります。

下のセクションでは、再スローの実際的な例を見ていきます。

## Fetch エラー処理の例 

ユーザ読み込みの例のエラー処理を改善しましょう。

[fetch](mdn:api/WindowOrWorkerGlobalScope/fetch) により返却された promise は、要求を行うことができない場合に reject します。例えば、リモートサーバが利用不可の場合、または URL が不正な場合です。しかし、リモートサーバが 404 や 500 エラーといったの応答の場合、有効な応答とみなします。

仮にサーバが行 `(*)` で 500 エラーの非JSONページを返したらどうなるでしょう？そのようなユーザはおらず、github が `(**)` で 404 エラーを返すとどうなるでしょう？

```js run
fetch('no-such-user.json') // (*)
  .then(response => response.json())
  .then(user => fetch(`https://api.github.com/users/${user.name}`)) // (**)
  .then(response => response.json())
  .catch(alert); // SyntaxError: Unexpected token < in JSON at position 0
  // ...
```

現時点では、コードは何が起きても応答を JSON として読み込もうとし、構文エラーで死にます。`no-such-user.json` は存在しないので、上の例を実行することでそれを見ることができます。

このエラーはチェーンを通じて失敗したものであり、詳細(何が失敗したのか、どこで失敗したのか)を含まないため良くありません。

したがって、もう1ステップ追加しましょう: HTTP ステータスが持っている `response.status` をチェックします。それが 200 でなければエラーをスローします。

```js run
class HttpError extends Error { // (1)
  constructor(response) {
    super(`${response.status} for ${response.url}`);
    this.name = 'HttpError';
    this.response = response;
  }
}

function loadJson(url) { // (2)
  return fetch(url)
    .then(response => {
      if (response.status == 200) {
        return response.json();
      } else {
        throw new HttpError(response);
      }
    })
}

loadJson('no-such-user.json') // (3)
  .catch(alert); // HttpError: 404 for .../no-such-user.json
```

1. 他のエラータイプと区別するために HTTP エラーのためのカスタムクラスを作ります。さらに、新しいクラスは `response` オブジェクトを受け取り、エラーに保存するコンストラクタを持ちます。これにより、エラー処理のコードがアクセスできるようになります。
2. 次に、リクエストとエラー処理のコードを `url` へ fetch する関数に置き、 *加えて* 任意の非 200 ステータスをエラーとして扱います。
3. これで `alert` はより良いメッセージが表示されます。

エラーに対する独自のクラスを持つことの素晴らしい点は、エラー処理コードで簡単にチェックできることです。

例えば、リクエストを行い 404 となった場合 -- ユーザに情報を変更するよう依頼します。


下のコードは github から指定された名前のユーザを読み込みます。もし存在しないユーザであれば、正しい名前を訪ねます。:

```js run
function demoGithubUser() {
  let name = prompt("Enter a name?", "iliakan");

  return loadJson(`https://api.github.com/users/${name}`)
    .then(user => {
      alert(`Full name: ${user.name}.`); // (1)
      return user;
    })
    .catch(err => {
*!*
      if (err instanceof HttpError && err.response.status == 404) { // (2)
*/!*
        alert("No such user, please reenter.");
        return demoGithubUser();
      } else {
        throw err;
      }
    });
}

demoGithubUser();
```

ここでは:

1. `loadJson` が有効なユーザオブジェクトを返した場合、名前は `(1)` で表示されユーザが返されます。これによりユーザ関連のアクションをチェーンに追加できます。その場合、以下の `.catch`は無視され、すべてが非常にシンプルで問題ありません。
2. それ以外の場合は、エラーの場合は行 `(2)` でチェックします。確かに HTTP エラーでステータスが 404(見つからない) の場合、ユーザに再入力を依頼します。他のエラーの場合は、処理の仕方を知らないため再スローします。 

## 未処理の reject 

エラーが処理されない場合何がおきるでしょう？例えば、上の例のように再スローした後。もしくは次のようにチェーンの終わりにエラーハンドラを追加し忘れている場合です。:

```js untrusted run refresh
new Promise(function() {
  noSuchFunction(); // ここでエラー(このような関数はない)
}); // .catch は未アタッチ
```

もしくは:

```js untrusted run refresh
// 末尾に .catch のない promise のチェーン
new Promise(function() {
  throw new Error("Whoops!");
}).then(function() {
  // ...何か...
}).then(function() {
  // ...何か...
}).then(function() {
  // ...が、この後に catch はありません!
});
```

エラーの場合、promise のステータスは "rejected" になり、実行は最も近い reject ハンドラにジャンプするはずです。しかし上の例ではそのようなハンドラはありません。そのため、エラーは "スタック" します(行き詰まります)。

実際には、それは通常悪いコードによるものです。 確かになぜエラー処理がないのでしょうか？

多くの JavaScript エンジンはこのような状況を追跡し、その場合にはグローバルエラーを生成します。コンソールで見ることができます。

ブラウザでは、イベント `unhandledrejection` を使ってキャッチできます。:

```js run
*!*
window.addEventListener('unhandledrejection', function(event) {
  // イベントオブジェクトは2つの特別なプロパティを持っています:
  alert(event.promise); // [object Promise] - エラーを生成した promise 
  alert(event.reason); // Error: Whoops! - 未処理のエラーオブジェクト
});
*/!*

new Promise(function() {
  throw new Error("Whoops!");
}); // エラーを処理する catch がない
```

そのイベントは [HTML 標準](https://html.spec.whatwg.org/multipage/webappapis.html#unhandled-promise-rejections) の一部です。今、エラーが発生し `.catch` がない場合、`unhandledrejection` ハンドラがトリガします: `event` オブジェクトはエラーに関する情報を持っているので、何かをすることができます。

通常、このようなエラーはリカバリ不可なので、最善の方法はユーザにその問題を知らせ、サーバへそのインシデントについて報告することです。

Node.JSのようなブラウザ以外の環境では、未処理のエラーを追跡する他の同様の方法があります。

## サマリ 

要約すると、`.then/catch(handler)` はハンドラが何をするかによって変化する新しい promise を返します:

1. もし値を返したり、`return` (`return undefined` と同じ) なしで終了した場合、新しい promise が resolve になり、最も近い resolve ハンドラ(`.then` の最初の引数)がその値で呼ばれます。
2. もしエラーをスローした場合、新しい promise は reject になり、最も近い reject ハンドラ(`.then` または `.catch` の2つ目の引数)がそれと一緒に呼ばれます。
3. もし promise を返す場合、JavaScript はそれが完了するまで待機し、同じ方法で結果に作用します。

`.then/catch` によって返される promise がどのように変化するかの図です:

![](promise-handler-variants.png)

どのようなハンドラが呼ばれるかの小さな図です:

![](promise-handler-variants-2.png)

上のエラー処理の例では、`.catch` は常にチェーンの最後でした。実際には、すべての promise チェーンが `.catch` を持っている訳ではありません。通常のコードと同じように、常に `try..catch` でラップされている訳ではありません。

エラーを処理したい/エラーを処理する方法を知りたい場所に `.catch` を置くべきです。カスタムエラークラスを使用すると、エラーを分析し、処理できないエラー再スローすることができます。

範囲外のエラーについては、 `unhandledrejection` イベントハンドラ（ブラウザ用、および他の環境用の類似物）を用意する必要があります。 このような不明なエラーは通常は回復不可能なので、ユーザーに知らせて、おそらくサーバーにそのインシデントについて報告するだけです。
