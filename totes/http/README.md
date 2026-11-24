# http

HTTP/1.1 の型と読み書き。Rust の `http` と `httparse` に当たる。TCP にも effect にも触れない純粋な tote で、
サーバー（relic）とクライアントが同じ型と、同じ証明済みの読み書きを使う。

## ファイル

- `src/bytes.taut`：バイト列の走査（`take_until`・`strip_prefix`・`lines` など）と、その往復の補題。
- `src/status.taut`：状態コードの `Status`。三桁で、最初の桁は 1〜9 なので、100〜999 の外の状態は書けない。`Status::ok()`・`Status::not_found()` など RFC 9110 の名前の定数と、`Status::code`。`Status::read_bytes`：どの状態も三桁の数字に書いてから読むと元に戻る。
- `src/http.taut`：`Method`、`HttpRequest`、`HttpResponse`、ヘッダ（名前は ASCII の大文字小文字を区別しない。`Header::get` は最初の一つ、`Header::get_all` は同名のすべての値）、`plain`・`html`・`redirect` などの応答。`reason`：RFC 9110 の 15 節の状態すべてと、RFC 6585 の 428・429・431 の理由句（ほかの状態は空）。
- `src/wire.taut`：要求の読み取り `parse_request` と書き出し `print_request`、応答の書き出し `print_response` と読み取り `parse_response`。
- `src/percent.taut`：URL とフォームの percent エンコード（`%XX` と `+`、UTF-8）。`percent_encode`・`percent_decode` と、UTF-8 のどの文字列も encode してから decode すると元に戻ること（`Percent::decode_encode`）。`%XX` の読み書きと、その往復の定理は url tote（`Tauto.eure` の Git 依存）のものを使う。
- `src/frame.taut`：接続から届いたバイト列を少しずつ食べる読み手。ヘッダ（上限つき）、本文の区切り方の判断（要求と応答）、
  Content-Length の本文、chunked の本文（読み手と書き手）。本文はためずに届いた分だけ渡す。
- `src/at.taut`：ヘッダの各行を `Span` のまま、届いたバイト列の元の場所で `Bytes::get` を使って読む関数（`header_at`・`start_at`・`lines_at` など）と、`wire.taut` のリストを読む関数と同じ結果になることの証明（`request_at` など）。
- `src/tsplit.taut`：テキストを区切りのバイトで分ける `split_text`。区切りを `Text::byte`、各部分を `Text::slice` で元の場所から切り出し、`Text::split_on` と同じ結果になること（`split_text_eq`）を証明する。

## 証明していること

公理は使わない。

- `parse_print_request`・`parse_print_response`（wire）：区切りの文字を含まない要求・応答は、書いてから読むと元に戻る（応答は、利用者が付けた `content-length` と `transfer-encoding`（名前は大文字小文字を区別しない）をすべて落とし、本文の長さの `content-length` を 1 つだけ付けたものに戻る。`sent_headers_count`：どんなヘッダーでも出る `content-length` はちょうど 1 つ。`sent_transfer_encoding_values`（frame）：出る `transfer-encoding` は無い）。
- `feed_append`（frame）：読み手にバイト列を分けて渡しても、つないで渡しても同じ。TCP の区切りで読み方が変わらない。
- `Head::print_request`・`Head::print_response`（frame）：ヘッダの読み手はちょうどヘッダの終わりで止まり、本文とその後を残す。
- `length_body`・`chunked_read`・`chunked_bytes_read`（frame）：Content-Length と chunked の本文をちょうど読む。
- `HttpRequest::sized_lengths`・`HttpRequest::sized_no_coding`・`HttpResponse::sized_lengths`（frame）：本文の長さを決めるのは、
  Content-Length がすべて同じ値で、要求なら Transfer-Encoding がないときだけ。それ以外は 400 で断る（request smuggling を防ぐ）。

## まだないもの

- `Uri`（scheme・authority・path・query）、`Version`（HTTP/1.1 だけ）。

## 検査

```sh
for f in src/*.taut; do tauto check "$f"; tauto lint "$f"; done
tauto fmt --check src/*.taut
```
