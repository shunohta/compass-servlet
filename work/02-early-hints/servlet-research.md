# early hints の調査

## 関連箇所の洗い出し

`jakartaee/servlet` の `main`（`97a5c65`）で実行した。以下の行番号はこの時点のもの。

```bash
git grep -niE 'early[ _-]?hints?|sendEarlyHints|SC_EARLY_HINTS|RFC ?8297'
```

```text
api/src/main/java/jakarta/servlet/http/HttpServletRequest.java:297:     * @deprecated In favor of 103 early hints
api/src/main/java/jakarta/servlet/http/HttpServletRequestWrapper.java:335:     * @deprecated In favor of 103 early hints
api/src/main/java/jakarta/servlet/http/HttpServletResponse.java:263:    void sendEarlyHints();
api/src/main/java/jakarta/servlet/http/HttpServletResponse.java:528:    int SC_EARLY_HINTS = 103;
api/src/main/java/jakarta/servlet/http/HttpServletResponseWrapper.java:145:     * The default behavior of this method is to call sendEarlyHints() on the wrapped response object.
api/src/main/java/jakarta/servlet/http/HttpServletResponseWrapper.java:150:    public void sendEarlyHints() {
api/src/main/java/jakarta/servlet/http/HttpServletResponseWrapper.java:151:        this._getHttpServletResponse().sendEarlyHints();
api/src/main/java/jakarta/servlet/http/PushBuilder.java:87: * @deprecated In favor of 103 early hints
api/src/test/java/ee/jakarta/servlet/http/MockHttpServletResponse.java:77:    public void sendEarlyHints() {
spec/src/main/asciidoc/servlet-spec-body.adoc:144:* RFC 8297 An HTTP Status Code for Indicating Hints
spec/src/main/asciidoc/servlet-spec-body.adoc:1678:client already has. Server push has essentially been replaced by RFC 8297 (Early
spec/src/main/asciidoc/servlet-spec-body.adoc:8571:Add support for RFC 8297 (Early Hints) via a new method `sendEarlyHints()` on
tck/tck-runtime/src/main/resources/servlet/tck/signature/jakarta.servlet.sig_6.2:864:fld public final static int SC_EARLY_HINTS = 103
tck/tck-runtime/src/main/resources/servlet/tck/signature/jakarta.servlet.sig_6.2:920:meth public abstract void sendEarlyHints()
tck/tck-runtime/src/main/resources/servlet/tck/signature/jakarta.servlet.sig_6.2:949:meth public void sendEarlyHints()
```

## `sendEarlyHints()` の javadoc

`api/src/main/java/jakarta/servlet/http/HttpServletResponse.java:254-263`

- その時点のレスポンスヘッダをそのまま載せた 103 を送る
- そのヘッダにはコンテナが自動追加したものが含まれることがある
- レスポンスをコミットしない
- コミット前なら複数回呼んでよい
- コミット後に呼んでも何も起きない
- `@since Servlet 6.2`

推測: 引数が無いので、載せるヘッダを個別に指定する手段は無い。

## 分かったこと

- early hints の API は `HttpServletResponse#sendEarlyHints()` の1つだけ
  - 仕様書の変更履歴（`servlet-spec-body.adoc:8571`、Issue 542）に、RFC 8297 のサポートを
    このメソッドで追加したと書かれている
- ラッパー（`HttpServletResponseWrapper.java:150`）は包んでいる相手に呼び出しを渡すだけで、
  Servlet 6.2 で追加された
- TCK が確かめているのは、メソッドが存在するかどうかだけ
  - `jakarta.servlet.sig_6.2` に名前が登録されている（864 / 920 / 949）
  - 呼んだときに 103 が飛ぶか、といった振る舞いは確かめていない
- `servlet-spec-body.adoc` にルールの記載は無い
  - 追加したことと、server push の代わりという記載のみ

## 実装での確認

Tomcat 11.0.25 を立てて動かし、リクエストとレスポンスの流れをデバッガで追った。
内容は `tomcat-research.md`。
