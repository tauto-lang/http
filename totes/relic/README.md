# relic

HTTP サーバーライブラリ。ルーター・HTTP/1.1 の読み書きを証明の言語で書く。
ハンドラは普通の関数で、要求から読むもの・状態・認可はハンドラの引数ではなく effect にする。
ミドルウェアはその effect を handle して行から消す。

````
fn healthz(): Response { text_response(Status::ok(), "ok") }
fn missing(): Response { text_response(Status::not_found(), "not found") }

router!(```
Route ! {}
/healthz Healthz get: healthz
/* NotFound any: missing
```) => enum Route, fn route, fn handler
fn listen() ! {ServeIo, ∀ T. spawn(() -> T ! {TaskIo})} {
  serve("127.0.0.1:8080", route, handler)
}
````

`serve` の行は、サーバー自身が使う effect（`conn.taut` の `ServeIo`、タスクの中では `TaskIo`）に、ハンドラの行 `h` を足したもの。
`Host` 全体ではないので、環境変数・ファイル・プロセスの許可は求められない（`net` と、止まらないための `div` だけ。`server/Tauto.eure` の `trust.run`）。
`h` が `Host` の一部（ここでは `router!` の一行目が `! {}` なので空）なら、`serve` の行にそのまま加わる。

## `router!`

`router!` は `enum Route`、効果のない `fn route(request: &HttpRequest): Route`、
`fn handler(request: HttpRequest, route: Route): Response ! {…}` を出す。

````
router!(```
Route ! {}
middleware: trace, global_state, auth
/ Home get: home
/healthz Healthz get: healthz
/users Users get: users post: create {
  middleware: user_db
  /{user_id: UserId} User get: user delete: remove
}
/todos ?[title: Text]&page: Nat Todos get: todos
/* NotFound any: missing
```) => enum Route, fn route, fn handler
````

- 一行目は enum の名前と、`handler` に残る effect の行。ミドルウェアが handle したあとに残るものを書く。省略はできず、空なら `! {}`。
- 一行は「パス、クエリ、名前、メソッド: ハンドラ」。波括弧の中は子のルートとそのミドルウェアで、子のパスは親のパスに続く。メソッドのない行は子をまとめるだけで、`Route` の値にならない。
- `{user_id: UserId}` はパスの一節をその型で読む。`?page: Nat` は必須、`?[title: Text]` は `Option(Text)` で、無ければ `None`。
  読めないとそのルートは合わず、次を試す。最後は `/*` で、メソッドは `any` か七つ全部。404 は生成しない。
- 捕捉とクエリは名前順のレコードになり、ハンドラはそのレコードを受け取る（`fn user(u: { user_id: UserId })`）。
- `Text`・`Nat`・`u64`・`i64` は `src/path.taut` が読む。ほかの型 `T` は `T::from_path(segment: Text): Option(T)` で読む。
  別のパッケージの型は newtype で包む（`struct UserId { raw: Text }`）。
- ミドルウェアは名前だけを書き、左が外側。`fn trace<!e>(k: () -> Response ! e): Response ! e` の形で、
  effect を handle すれば行から消える。`middleware:` は書いた節とその子にかかる。
- 認可も effect にする。`examples/users.taut` では、ハンドラが `admin()` を起こし、root の `auth` ミドルウェアが
  そのときに初めて `authorization` を読む。認可のいらないハンドラは何も起こさない。
- `handler` は `get_headers`・`set_header`・`request_body` を常に handle する（`answer_request`）。
- パスの節は `/` で区切ったまま、percent 復号せずにハンドラへ渡す（`/users/a%20b` の `{name: Text}` は `a%20b`、`%2F` でも区切らない）。クエリの値は復号する。
- パスは合うがメソッドが合わないときは、その節のミドルウェアの中で 405 になり、`allow` を付ける。`HEAD` が無ければ `GET` が答え、本文は送らない。

## ファイル

HTTP の型と読み書き、percent エンコード（`bytes`・`http`・`wire`・`frame`・`percent`）は [`http`](../http) にある。

- `src/router.taut`：`router!`。
- `src/effects.taut`：`get_headers`、`set_header`、`request_body` と、それを答える `answer_request`。
- `src/response.taut`：`Response`（状態・ヘッダ・本文 `Bytes`）と、ソケットへ書くバイト列。
  ヘッダ名がトークンでない、値に CR・LF などの制御文字があるときは 500 になる。
  ハンドラが `content-length` のヘッダーを付けていても（名前は大文字小文字を区別しない）書き出さず、本文の長さの `content-length` を 1 つだけ付ける。
  本文は長さ付きで書くので、ハンドラが `transfer-encoding` を付けていても（同じく大文字小文字を区別しない）書き出さない。
  `connection` も relic が決める。ハンドラが `connection` を付けていても（同じく）すべて落とし（`finish_response`）、閉じる応答には `connection: close` を 1 つだけ、続ける応答には `connection` を付けない。
- `src/path.taut`：パスの節、クエリの値、`Text`・`Nat`・`u64`・`i64` の読み方。
- `src/app.taut`：テキストの要求を `route` と `handler` でテキストの応答にする `render`。
- `src/conn.taut`：TCP の標準エフェクト（`tcp_listen`・`tcp_accept`・`tcp_read`・`tcp_write` など）の上の HTTP/1.1。
  届いたバイトは `http` の `frame.taut` の読み手で要求にし（`take_all`。ネットワークなしで試せる）、関数が答える（`serve_response`）。
  ヘッダは 16 KiB まで（超えると 431）、本文は 1 MiB まで（413）、要求を送り終えるまで 10 秒（408）。
  接続ごとにタスクを立てて同時に扱い（最大 2048 接続）、遅いクライアントが他の接続を止めない。
  接続は HTTP/1.1 の持続的接続として次の要求を待ち、クライアントが `connection: close` を送るか（`connection` ヘッダーが複数あっても、どれかの値に `close` があれば閉じる。`asks_close`）、10 秒要求が来ないか、1000 要求に答えたら閉じる。
  accept の失敗は、許可がない（`Denied`）ときだけ止まる。ファイル記述子の不足などでは出力して 10 ミリ秒待ち（`accept_retry_ms`）、accept し直す。
- `src/serve.taut`：`route` と `handler` で待ち受ける `serve`。行は `{ServeIo, ∀ T. spawn(() -> T ! {TaskIo} ∪ h)} ∪ h`（`h` はハンドラの行）。同時接続は既定 2048（`connection_limit`）。
- `examples/todo.taut`：やることの一覧、追加（`?[title: Text]`）、削除（`{id: Nat}`）。
- `examples/guestbook.taut`：署名の一覧と、名前とメッセージの追加（必須のクエリ）。
- `examples/users.taut`：`trace`、`global_state`、`auth`、入れ子の `user_db`、newtype の `UserId`。
- `tests/`：ルーティング（examples を通して）、`http` の `frame.taut` の読み手、`src/conn.taut` が要求を少しずつ読むところ。
- `server/`：`/healthz` を待つサーバー。

## 証明していること

`http` の読み書きについて、公理は使わずに証明している。ルーターの証明はまだない。

- `parse_print_request`（wire）：区切りの文字を含まない（パスに `?`・空白・改行がなく、ヘッダ名に `:` と改行がなく、
  値に改行がない）すべてのリクエスト `r` について `parse_request(print_request(r)) = Some(r)`。
- `parse_print_response`（wire）：3 桁の状態と、区切りの文字を含まず送ってよいヘッダのすべての応答 `r` について、
  `parse_response(print_response(r))` は `sent_response(r)` を返す。これは `r` のヘッダーから `content-length` と `transfer-encoding`（名前は大文字小文字を区別しない。`without_framing`）をすべて除き、本文の長さの `content-length` を末尾に 1 つ足したもの（`sent_headers`）。
  `sent_headers_count`：`sent_headers(hs, b)` の `content-length` はどんな `hs` でもちょうど 1 つ。`without_content_length_count`：`hs` から来たものは残らない。`sent_content_length_values`（frame）：その値は本文の長さ。`sent_transfer_encoding_values`（frame）：`transfer-encoding` は出ない。
  relic の `response.taut` も同じ `without_framing` で落とすが、その書き出し自体は証明していない。`connection` を落とす `finish_response` も、テストで確かめているだけで証明していない。
- `feed_append`（frame）：読み手にバイト列を二つに分けて渡しても、つないで渡しても、状態・渡したバイト・残りは同じ。
  届き方（TCP の区切り）で読み方が変わらない。
- `Head::print_request`（frame）：印字した要求のあとに何が続いても、ヘッダの読み手はちょうどヘッダの終わりで止まり、
  本文とその後を残す（予算がヘッダの長さ以上のとき。`HttpRequest::head_budget_length`）。`parse_request_head` で
  そのヘッダは本文を除いた元の要求に読める。
- `Head::print_response`（frame）：印字した応答のあとに何が続いても、ヘッダの読み手はちょうどヘッダの終わりで止まり、
  本文とその後を残す（予算がヘッダの長さ以上のとき。`HttpResponse::head_budget_length`）。`parse_response_head` で
  そのヘッダは `sent_headers` にした元の応答（本文を除く）に読める。`HttpResponse::framing_sent`：本文のある応答で、
  ハンドラが Content-Length と Transfer-Encoding を何を付けていても（どちらも落とされる）、クライアントは本文をちょうどその長さと
  読む（あとは `length_body`）。`decimal_text_of_nat`：`Text::of_nat` で書いたどの数も `decimal` がその数に読む。
- `length_body`（frame）：Content-Length が `n` の本文の読み手は、ちょうど `n` バイトを渡して残りを次に回す。
- `HttpRequest::sized_lengths`・`HttpRequest::sized_no_coding`・`HttpResponse::sized_lengths`（frame）：本文を `n` バイトと読むのは、
  Content-Length がすべて `n` と読めて、要求なら Transfer-Encoding がないときだけ。値の違う Content-Length や
  Transfer-Encoding との併用は 400 で断る（request smuggling を防ぐ）。
- `chunked_read`（frame）：16 進の大きさがデータの長さを言い、大きさの行が上限に収まる chunk の並びを chunked で送ると、
  読み手はデータをつないだものをちょうど渡し、最後の chunk と空行の直後で止まって残りを次に回す。
- `chunked_bytes_read`・`chunked_bytes_read_below`（frame）：書き手 `chunked_bytes` が出す本文は、読み手が受け取って
  部分をつないだものをちょうど渡し、直後で止まる。大きさの行が `c + 2` バイトなら、各部分が `16^(c+1)` バイト未満で
  あればよい。`Hex::text_value` で書き手の 16 進は読み手の読む値に戻る。

## 検査

```sh
for f in src/*.taut examples/*.taut server/src/main.taut; do tauto check "$f"; tauto lint "$f"; done
tauto fmt --check src/*.taut examples/*.taut server/src/*.taut tests/src/*.taut
tauto test tests
tauto run tests/src/main.taut
tauto build --backend rust --release server   # 出来たサーバを起動して curl http://127.0.0.1:8080/healthz
```

## まだないもの

- ストリーミングの本文、WebSocket、TLS。
- フォーム・Cookie・JSON の読み書き、CORS などのミドルウェア。
- `Route` からパスを書き戻す関数。

## バイナリのアセットサーバー

`src/binary.taut` は `BinaryRequest`、`BinaryResponse`、`binary_wire`、`serve_binary` を提供する。
コールバックは要求を受け取り、不変の `Bytes` の本文を返す。
GET は任意のバイト列をそのまま返し、HEAD は本文なしで同じ Content-Length を返す。
サーバーは要求ヘッダを 16 KiB までに制限し、読み書きに 10 秒の期限を付け、接続ごとに閉じ、
対応しないメソッドと content type の改行を拒む。`serve_binary` はコールバックの行を `h` として受け取り、
ホストには `BinaryIo`・`stop`・`tcp_accept`・`tcp_listen` と `h` だけを求め、`Host` 全体は求めない。
`serve_binary` は接続を一つずつ順に処理する（`accept_each` は `answer(c, f)` を終えてから次の接続を受ける）。
理由句は状態コードから `http` の `reason` で付け、一覧にない状態は空にする。
accept の失敗は、許可がない（`Denied`）ときだけ止まる。ほかの失敗は出力して `accept_retry_ms`（10 ミリ秒）待ち、accept し直す。
そのため `BinaryIo` に `sleep` が加わった（`sleep` は `tauto run` の許可を要らない）。
アセットはリスナーを開く前に読み込んでおく。ローカルの静的アセット向けで、
一般的な要求本文や keep-alive のサーバーではない。

`tauto test tests --filter binary` でこの部分のテストだけを実行できる。
