# 実験環境

観測は AWS EC2 上の Linux で行う。

| | |
|---|---|
| OS | Rocky Linux 9.7 (x86_64) |
| JDK | Azul Zulu 25.0.4 |
| Tomcat | 11.0.25 を `~/opt` に tarball で展開（パッケージマネージャは使わない） |

- [jit-box-workflow.md](https://github.com/shunohta/compass-jit/blob/main/sharing/jit-box-workflow.md)（private）
- [aws-jit-env-spec.md](https://github.com/shunohta/compass-jit/blob/main/sharing/aws-jit-env-spec.md)（private）

## Tomcat のログ保持

Tomcat が出すログは、保持の仕組みが3系統に分かれている。いずれも90日で揃えた。

| ログ | 設定場所 | 対応 |
|---|---|---|
| JULI の4種（`catalina.` / `localhost.` / `manager.` / `host-manager.`） | `conf/logging.properties` | 既定で `maxDays = 90` |
| `localhost_access_log.*.txt` | `conf/server.xml` の `AccessLogValve` | `maxDays="90"` を追加（既定は `-1` で無期限） |
| `catalina.out` | `/etc/logrotate.d/tomcat` | logrotate の設定を新規作成 |

`catalina.out` だけ Tomcat の管理外にある。`startup.sh` がシェルで標準出力をリダイレクトして
いるだけなので、JULI の `maxDays` が効かず、放置すると無限に伸びる。

logrotate の設定は次の通り。Tomcat がファイルを開いたままなので `copytruncate` が要る。

```
/home/rocky/opt/apache-tomcat-*/logs/catalina.out {
    daily
    rotate 90
    compress
    missingok
    notifempty
    copytruncate
    su rocky rocky
}
```
