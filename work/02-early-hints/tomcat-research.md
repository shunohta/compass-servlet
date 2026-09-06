# Tomcat の動きを見る

- Repo: [apache/tomcat](https://github.com/apache/tomcat)
- Tag: [11.0.25](https://github.com/apache/tomcat/tree/11.0.25)

以下の行番号はこのタグの時点のもの。

## 目的

リクエストが届いてからレスポンスが返るまで、Tomcat の中で何を通るのかを見る。
`sendEarlyHints()` の周辺を読む前に、その土台を押さえておく。

## 前提となる用語

**サーブレット** — リクエストを1本受け取ってレスポンスを作る Java のクラス。
`HttpServlet` を継承して `doGet` などを書き、Tomcat がそれを呼ぶ。
この `HttpServlet` や `HttpServletRequest` の仕様を決めているのが jakartaee/servlet で、
Spring では `DispatcherServlet` という1本のサーブレットの中から `@RestController` が呼ばれる。

**JSP** — HTML の中に Java を埋め込んで書く仕組み。
実行時に Java のサーブレットへ変換してコンパイルされるので、JSP もサーブレットの一種になる。
変換を担当するのが Jasper で、その取次ぎ役が `JspServlet`。

**webapp** — Web アプリケーション1つ。
`webapps/` のディレクトリ1つに対応し、ディレクトリ名がそのまま URL の先頭になる。
`ROOT` だけは予約名で `/` に対応するので、`webapps/ROOT/` が `http://localhost:8080/` の中身になる。

## 環境

EC2 に Tomcat 11.0.25 の tarball を展開して起動し、VS Code からリモートデバッグで繋いだ。

- デバッガから繋げるようにするには `bin/catalina.sh jpda start` で起動する
- この待ち受けは認証が無く、繋がれば任意のコードを実行できる
- そのため外には開けず、サーバ自身からしか繋げない設定にする
- 手元からは SSH のポート転送を通して繋ぐ
- clone はタグ 11.0.25 の detached HEAD にして、動いている jar と行番号を揃える

```bash
# サーバ側
JPDA_TRANSPORT=dt_socket JPDA_ADDRESS=localhost:8000 bin/catalina.sh jpda start

# 手元
ssh -N -L 8000:localhost:8000 <host>
```

VS Code 側は `.vscode/launch.json` に `request: attach` の構成を置いて繋ぐ。

## ブレークポイント

3箇所で足りる。

| | 場所 | 見えるもの |
|---|---|---|
| 入口 | `CoyoteAdapter.java:307` | リクエストがコンテナに渡る境界 |
| 間 | `HttpServlet.java:705` | サーブレットに入る手前 |
| 出口 | `Http11Processor.java:888` | レスポンスのヘッダが確定する場所 |

## 入口の CallStack

`CoyoteAdapter.java:307` で止めたところ。下から上へ読むと、リクエストが辿った順になる。

```text
CoyoteAdapter.service(Request,Response) (/tomcat/java/org/apache/catalina/connector/CoyoteAdapter.java:307)
Http11Processor.service(SocketWrapperBase) (/tomcat/java/org/apache/coyote/http11/Http11Processor.java:408)
AbstractProcessorLight.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProcessorLight.java:71)
AbstractProtocol$ConnectionHandler.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProtocol.java:1307)
NioEndpoint$SocketProcessor.doRun() (/tomcat/java/org/apache/tomcat/util/net/NioEndpoint.java:2201)
SocketProcessorBase.run() (/tomcat/java/org/apache/tomcat/util/net/SocketProcessorBase.java:74)
ThreadPoolExecutor.runWorker(ThreadPoolExecutor$Worker) (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:949)
ThreadPoolExecutor$Worker.run() (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:483)
TaskThread$WrappingRunnable.run() (/tomcat/java/org/apache/tomcat/util/threads/TaskThread.java:74)
Thread.runWith(Object,Runnable) (/java.base/java.lang/Thread.java:1487)
Thread.run() (/java.base/java.lang/Thread.java:1474)
```

リクエストが処理されるまでに3つの係が動く。デバッガのスレッド一覧にそのまま出ている。

| 係 | スレッド名 | 仕事 | 本数 |
|---|---|---|---|
| Acceptor | `http-nio-8080-Acceptor` | 新しい接続を受け付ける | 1 |
| Poller | `http-nio-8080-Poller` | 接続を見張り、データが届いたものを見つける | 1 |
| Worker | `http-nio-8080-exec-1`〜 | 見つかったものを実際に処理する | 10 |

Acceptor と Poller は気づくだけなので1本で足りる。
重い処理は全部 Worker に渡すので、Worker だけ本数が多い。
上の CallStack は、その Worker 1本の中身。
なお Worker だけクラス名（`ThreadPoolExecutor$Worker`）とスレッド名（`exec`）が一致しない。

下から読むとこうなる。

1. プールのスレッドが、キューからタスクを1つ取り出す
   `Thread.run` / `TaskThread$WrappingRunnable.run` / `ThreadPoolExecutor.runWorker`

   - Worker＝プールで待機していて、タスクを取り出して実行する係
   - タスクはスレッドではなく、やることが書かれたオブジェクト
   - スレッドを作るのは高いので、起動時に10本用意して使い回す（上限200本）

2. タスクの中身は「この接続を処理せよ」という指示
   `SocketProcessorBase.run` / `NioEndpoint$SocketProcessor.doRun`

   - ソケット＝通信の口で、接続1本につき1つできる
   - タスクは対象のソケットを持っていて、それが上の層へ引数として渡っていく
   - このタスクを作ってキューに積んだのは Poller で、Worker は拾っただけ

3. その接続を担当する係（Processor）を用意する
   `AbstractProtocol$ConnectionHandler.process`

   - Processor は「どこまで読んだか」を覚えているので、接続ごとに必要になる
   - 処理していない間は返却されて、他の接続に回る

4. 1本の接続に複数のリクエストが流れるので、1本ずつ処理する
   `AbstractProcessorLight.process`

   - HTML の後に CSS や画像を同じ接続で取るため、繋ぎ直さずに使い回す
   - 次が届いていなければ抜けて、スレッドと Processor を返す

5. 届いたバイト列を切り分けて、HTTP のリクエストとして組み立てる
   `Http11Processor.service`

   - バイトはソケットではなく OS 側に溜まっていて、読むとは自分のバッファへ写すこと
   - 解析は空白と改行で区切って切り出すだけ
   - 一度に全部届くとは限らないので、3 の「どこまで読んだか」が要る

6. サーブレットが受け取れる形に詰め替えて、コンテナへ渡す
   `CoyoteAdapter.service`

   - coyote は HTTP 担当、catalina はサーブレット担当で、パッケージ名で分かれる
   - coyote の Request と catalina の Request は別のクラスで、後者が `HttpServletRequest` を実装する

## 間の層の CallStack

入口と出口はサーブレットを挟んで前後に位置しているので、
どちらの CallStack にもサーブレットが現れない。
その間を見るには `HttpServlet.java:705` に張る。
Tomcat はサーブレット API のソースも同梱しているので、ここにも張れる。

```text
HttpServlet.service(ServletRequest,ServletResponse) (/tomcat/java/jakarta/servlet/http/HttpServlet.java:705)
ApplicationFilterChain.doFilter(ServletRequest,ServletResponse) (/tomcat/java/org/apache/catalina/core/ApplicationFilterChain.java:132)
WsFilter.doFilter(ServletRequest,ServletResponse,FilterChain) (/tomcat/java/org/apache/tomcat/websocket/server/WsFilter.java:59)
ApplicationFilterChain.doFilter(ServletRequest,ServletResponse) (/tomcat/java/org/apache/catalina/core/ApplicationFilterChain.java:111)
StandardWrapperValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/core/StandardWrapperValve.java:165)
StandardContextValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/core/StandardContextValve.java:77)
AuthenticatorBase.invoke(Request,Response) (/tomcat/java/org/apache/catalina/authenticator/AuthenticatorBase.java:535)
StandardHostValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/core/StandardHostValve.java:115)
ErrorReportValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/valves/ErrorReportValve.java:86)
AbstractAccessLogValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/valves/AbstractAccessLogValve.java:782)
StandardEngineValve.invoke(Request,Response) (/tomcat/java/org/apache/catalina/core/StandardEngineValve.java:71)
CoyoteAdapter.service(Request,Response) (/tomcat/java/org/apache/catalina/connector/CoyoteAdapter.java:347)
Http11Processor.service(SocketWrapperBase) (/tomcat/java/org/apache/coyote/http11/Http11Processor.java:408)
AbstractProcessorLight.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProcessorLight.java:71)
AbstractProtocol$ConnectionHandler.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProtocol.java:1307)
NioEndpoint$SocketProcessor.doRun() (/tomcat/java/org/apache/tomcat/util/net/NioEndpoint.java:2201)
SocketProcessorBase.run() (/tomcat/java/org/apache/tomcat/util/net/SocketProcessorBase.java:74)
ThreadPoolExecutor.runWorker(ThreadPoolExecutor$Worker) (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:949)
ThreadPoolExecutor$Worker.run() (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:483)
TaskThread$WrappingRunnable.run() (/tomcat/java/org/apache/tomcat/util/threads/TaskThread.java:74)
```

下半分は入口の CallStack と同じ。ここから先が違う。

1. コンテナのパイプラインに入る
   `CoyoteAdapter.service:347`

   - 入口の 6 の続きで、ここから先がサーブレットの世界
   - バルブ＝パイプラインに差し込む処理の部品で、Tomcat 独自の呼び名

2. バルブを外側から内側へ順に通る

   ```text
   上（内側）  StandardWrapperValve     サーブレット1つ
               StandardContextValve     webapp（Web アプリケーション）1つ
               AuthenticatorBase        認証
               StandardHostValve        ホスト名
               ErrorReportValve         エラーページの生成
               AbstractAccessLogValve   アクセスログ
   下（外側）  StandardEngineValve      サーバ全体
   ```

   - 先に呼ばれた方が下に積まれるので、外側の Engine が一番下に来る
   - Engine / Host / Context / Wrapper が骨格で、その間に横断的な処理が挟まる
   - Engine と Host は `server.xml` に書かれていて、Context と Wrapper は `webapps/` の中から自動で作られる

3. 挟まっているバルブの位置には意味がある

   - 各バルブは内側を呼び出して終わったら戻ってくるので、内側で起きたことを拾える
   - アクセスログは Engine のすぐ内側なので、全リクエストが必ず通る
   - エラーページの生成は Host より外側なので、内側で起きた例外を受け止められる
   - 認証は Context の直前で、認証の要否が webapp ごとに違うのでこの位置になる

4. フィルタを順に通す

   ```text
   上  ApplicationFilterChain.doFilter:132   残り0本 → サーブレットを呼ぶ
       WsFilter.doFilter:59                  WebSocket 用のフィルタ
   下  ApplicationFilterChain.doFilter:111   残り1本 → 次のフィルタを呼ぶ
   ```

   - フィルタ＝サーブレットの前後に差し込む処理で、バルブと形は同じ
   - 違いは持ち主で、バルブは Tomcat 独自、フィルタはサーブレット仕様の標準なのでアプリが書ける
   - 各フィルタは `chain.doFilter()` を呼べば次へ進み、呼ばなければそこで止まる（認証で弾けるのはこれ）
   - `ApplicationFilterChain` は今何本目かを覚えていて、呼ばれるたびに1つ進む
   - `:111` と `:132` は同じメソッドの別の分岐で、残りがあるかどうかで行き先が変わる
   - `WsFilter` は WebSocket 用で、何も設定しなくても入っている

5. サーブレットに入る
   `HttpServlet.service:705`

   - ここから先が `jakarta.servlet` の世界
   - `req` / `res` の実体は `RequestFacade` / `ResponseFacade`
   - facade は Tomcat 内部用のメソッドを隠して、サーブレット API の分だけを見せる包み
   - `this` は今動いているサーブレット自身で、今回は `JspServlet` だった
   - 静的な HTML なら `DefaultServlet` になるので、`/` の中身が JSP だと分かる

Spring MVC なら、`HttpServlet.service` の位置に `DispatcherServlet` が座り、
そのフィルタは `ApplicationFilterChain` の中に並ぶ。

## 出口の CallStack

`Http11Processor.java:888` で止めたところ。サーブレットが終わった後の続き。

```text
Http11Processor.prepareResponse() (/tomcat/java/org/apache/coyote/http11/Http11Processor.java:888)
AbstractProcessor.action(ActionCode,Object) (/tomcat/java/org/apache/coyote/AbstractProcessor.java:414)
Response.action(ActionCode,Object) (/tomcat/java/org/apache/coyote/Response.java:240)
Http11OutputBuffer.doWrite(ByteBuffer) (/tomcat/java/org/apache/coyote/http11/Http11OutputBuffer.java:193)
Response.doWrite(ByteBuffer) (/tomcat/java/org/apache/coyote/Response.java:780)
OutputBuffer.realWriteBytes(ByteBuffer) (/tomcat/java/org/apache/catalina/connector/OutputBuffer.java:319)
OutputBuffer.flushByteBuffer() (/tomcat/java/org/apache/catalina/connector/OutputBuffer.java:844)
OutputBuffer.realWriteChars(CharBuffer) (/tomcat/java/org/apache/catalina/connector/OutputBuffer.java:472)
OutputBuffer.flushCharBuffer() (/tomcat/java/org/apache/catalina/connector/OutputBuffer.java:849)
OutputBuffer.close() (/tomcat/java/org/apache/catalina/connector/OutputBuffer.java:221)
Response.finishResponse() (/tomcat/java/org/apache/catalina/connector/Response.java:426)
CoyoteAdapter.service(Request,Response) (/tomcat/java/org/apache/catalina/connector/CoyoteAdapter.java:376)
Http11Processor.service(SocketWrapperBase) (/tomcat/java/org/apache/coyote/http11/Http11Processor.java:408)
AbstractProcessorLight.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProcessorLight.java:71)
AbstractProtocol$ConnectionHandler.process(SocketWrapperBase,SocketEvent) (/tomcat/java/org/apache/coyote/AbstractProtocol.java:1307)
NioEndpoint$SocketProcessor.doRun() (/tomcat/java/org/apache/tomcat/util/net/NioEndpoint.java:2201)
SocketProcessorBase.run() (/tomcat/java/org/apache/tomcat/util/net/SocketProcessorBase.java:74)
ThreadPoolExecutor.runWorker(ThreadPoolExecutor$Worker) (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:949)
ThreadPoolExecutor$Worker.run() (/tomcat/java/org/apache/tomcat/util/threads/ThreadPoolExecutor.java:483)
TaskThread$WrappingRunnable.run() (/tomcat/java/org/apache/tomcat/util/threads/TaskThread.java:74)
```

下半分は入口の CallStack と同じ。ここから先が違う。

1. サーブレットが終わり、コンテナから戻ってきた
   `CoyoteAdapter.service:376` → `Response.finishResponse`

   - サーブレットはもう終わっているので、スタックに残っていない
   - 本文はバッファに溜まっているだけで、まだ1バイトも送られていない

2. 溜まっていた本文を吐き出しにかかる
   `OutputBuffer.close` から4段

   ```text
   OutputBuffer.flushCharBuffer()    文字バッファを空にする
   OutputBuffer.realWriteChars()     文字をバイトに変換する
   OutputBuffer.flushByteBuffer()    バイトバッファを空にする
   OutputBuffer.realWriteBytes()     バイトを下の層へ渡す
   ```

   - サーブレットは文字でもバイトでも書けるが、HTTP の本文はバイト列なので変換が要る
   - 今回は `getWriter()` で書かれたため、文字バッファ経由でバイトに変換される
   - `getOutputStream()` で書かれていれば、文字側の2段は出てこない

3. coyote 側へ渡す
   `Response.doWrite` → `Http11OutputBuffer.doWrite`

   - ここから先が、HTTP の形に整えてソケットへ書き出す担当

4. 書こうとした瞬間に「まだヘッダを送っていない」と気づく
   `Http11OutputBuffer.java:193` の `response.action(ActionCode.COMMIT, null)`

   - これがコミット
   - 送る順番はヘッダが先だが、作る順番は本文が先
   - 本文の長さやサーブレットの最後の変更を待つため、ぎりぎりまで作らない

5. ヘッダを組み立てる
   `AbstractProcessor.action:414` → `Http11Processor.prepareResponse:888`

   - Content-Type / Transfer-Encoding / Date がここで表に入る
   - デバッガで件数が 0 から 3 に増えるのを確認した

## つまずき: 別バージョンのソースが表示される

CallStack の一部が `tomcat-coyote-11.0.5.jar` から解決されていた。
動いているのは 11.0.25 なので、表示だけが別バージョンという状態になる。

原因は clone の中にあった。

- clone の `modules/*/pom.xml` が、リリース済みの Tomcat 11.0.5 に依存している
- Tomcat 本体は Ant でビルドされるので、Maven の pom を持つのは `modules/` だけ
- VS Code の Java 拡張はそこだけを Maven プロジェクトとして認識し、依存をソース付きで
  Maven Central からダウンロードして `~/.m2/repository` に保存する
- ソースの解決で、フォルダに指定したソースルートより、そちらが優先される

判別は `~/.m2/repository` のタイムスタンプでついた。
jar 自体の日付は公開日のままだが、Maven が取得時に作る `_remote.repositories` と
`m2e-lastUpdated.properties` が作業当日の日時になっていた。

`.vscode/settings.json` で `modules` を取り込みから外して解消した。

```json
"java.import.exclusions": [
  "**/node_modules/**",
  "**/.metadata/**",
  "**/archetype-resources/**",
  "**/META-INF/maven/**",
  "**/modules/**"
]
```

既定値を上書きする設定なので、既定の4つを残したうえで `**/modules/**` を足す。
適用にはコマンドパレットの `Java: Clean Java Language Server Workspace` が要る。

今回は該当ファイルが 11.0.5 と 11.0.25 で同じだったため、実害は出ていなかった。
ただし版が違えば行がずれて、見ているコードと止まっている場所が食い違う。
