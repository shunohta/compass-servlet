# 実験環境

観測は AWS EC2 上の Linux で行う。

| | |
|---|---|
| OS | Rocky Linux 9 (x86_64) |
| JDK | Azul Zulu 25 |
| Tomcat | 11.0.x を tarball で展開（パッケージマネージャは使わない） |

Tomcat 11 は Java 21+ を要求するので、Zulu 25 でそのまま動く。

- [jit-box-workflow.md](https://github.com/shunohta/compass-jit/blob/main/sharing/jit-box-workflow.md)（private）
- [aws-jit-env-spec.md](https://github.com/shunohta/compass-jit/blob/main/sharing/aws-jit-env-spec.md)（private）
