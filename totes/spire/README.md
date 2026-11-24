# spire

Rust の tower に当たる、サービスとミドルウェアの合成。合成の法則とミドルウェアの振る舞いを証明付きで。外部の依存はない。

サービスは状態付きの純粋な関数 `(S, Req) -> @outcome Outcome(Res, E) × @state S` として表す。
回数・残りトークンのような、tower ではミドルウェアの中に隠れる状態も `S` に出てくるので、
ミドルウェアは普通の関数になり、振る舞いを定理として書ける。時間は状態を変える関数
（`refill` など）を呼ぶ側が進める。

- `src/service.taut`: `Outcome(Res, E)`（応答かエラー）、`Service(S, Req, Res, E)`、`service_call`、状態を使わない `service_fn`。
- `src/layer.taut`: `Layer(A, B)` はサービスを別のサービスで包む。`Layer::then(inner, outer)` で合成し、
  結合律 `Layer::then_assoc` と単位律 `Layer::identity_left` / `Layer::identity_right` を証明する。
- `src/middleware.taut`: `map_request`、`map_response`、`map_err`、`filter`。
  `map_request` を二回かけると合成した関数で一回かけたのと同じこと、`filter` が拒んだ要求は
  サービスに届かず状態も変えないこと、通した要求はサービスそのものの呼び出しになることを証明する。
- `src/retry.taut`: `retry(should, n)` は `should` が認めたエラーのとき、失敗した呼び出しが残した状態で
  最大 `n` 回呼び直す。`retry(_, 0)` がサービスそのものであること、応答と `should` が拒むエラーは
  呼び直さないことを証明する。
- `src/rate_limit.taut`: `rate_limit()` はトークンの数をサービスの状態の隣に持ち、要求ごとに一つ使う。
  トークンがないと `overloaded` で拒み、サービスを呼ばない。`refill(n, st)` で `n` に戻す。
  `run_all` で要求の列を順に流したとき、通った要求の数と残りのトークンの和が最初のトークンの数に
  等しいこと（`rate_limit_budget`、つまり refill の間に通るのは高々 `n` 件）を証明する。
- `examples/basics.taut`: `service_fn`、`map_request`、`map_response`、`map_err`、`filter`、`refill` を別のファイルから
  import して使い、結果を定理で確かめる。
- `examples/stack.taut`: 二回に一回しか答えないサービスを `retry` と `rate_limit` で包み、
  三つの要求への答えを計算で確かめる。

## まだないもの

tower の `timeout`、`buffer`、`load_shed`、`concurrency_limit` は、非同期の実行と時計を前提にするので
まだない。effect（`fail` など）を使うコードとの橋渡しも、このライブラリの帰納型を実行時に扱えるように
なってから足す。

## 検査

```sh
for f in src/*.taut examples/*.taut; do tauto check "$f"; tauto lint "$f"; done
tauto fmt --check src/*.taut examples/*.taut
```
