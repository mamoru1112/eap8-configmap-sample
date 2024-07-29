# EAP 8 コンテナにて環境依存のパラメータを ConfigMap を用いて注入するサンプル

## はじめに
本サンプルでは、環境依存のパラメータを EAP 8 コンテナイメージから外出しして、ConfigMap 及び環境変数を用いてコンテナに注入するサンプルです。EAP 設定ファイル (standalone.xml) を ConfigMap を用いて注入しています。

## 利用方法
### EAP 8 設定ファイル (standalone.xml) 内容確認
本サンプルでは、ConfigMap を用いて設定ファイルがマウントされたことを確認するため、2行目に「`<!-- ConfigMap Mounted  -->`」が記載されています。
また、環境変数にて設定したパラメータを読み込めるように、root-logger のログレベルに変数 `ROOT_LOGGER_LEVEL` を設定しています。
```
            <root-logger>
                <level name="${ROOT_LOGGER_LEVEL}"/>
                <handlers>
                    <handler name="CONSOLE"/>
                </handlers>
            </root-logger>
```

### 設定ファイルを ConfigMap に格納
```
$ oc create cm standalone --from-file=standalone.xml 
configmap/standalone created
```

### Deployment マニフェスト (deploy.yaml) 設定内容の確認
/opt/eap/standalone/configuration ディレクトリ配下に複数の設定ファイルが存在するため、standalone.xml が ConfigMap に格納された設定ファイルを参照するように subPath を設定しています。
また、環境変数 ROOT_LOGGER_LEVEL に "INFO" を代入しています。
```
    spec:
      volumes:
      - name: standalone
        configMap:
          name: standalone
      containers:
      - volumeMounts:
        - name: standalone
          mountPath: /opt/eap/standalone/configuration/standalone.xml
          subPath: standalone.xml
        env:
        - name: ROOT_LOGGER_LEVEL
          value: "INFO"
```

### Deployment マニフェスト (deploy.yaml) のデプロイ
```
$ oc apply -f deploy.yaml 
deployment.apps/helloworld created
```

### 環境依存のパラメータが ConfigMap を用いて注入されたことの確認
```
$ oc rsh <Pod名> head -n3 /opt/eap/standalone/configuration/standalone.xml
<?xml version="1.0" encoding="UTF-8"?>
<!-- ConfigMap Mounted  -->
<server xmlns="urn:jboss:domain:20.0">

$ oc rsh <Pod名> env | grep ROOT_LOGGER_LEVEL
ROOT_LOGGER_LEVEL=INFO
```

## 補足：　標準出力にログ出力する設定
本サンプルに含まれる EAP 設定ファイル (standalone.xml) では、標準出力へログ出力する設定が含まれています。標準出力へログ出力するための standalone.xml の該当の設定は以下の箇所です。
```
            <console-handler name="CONSOLE">
                <formatter>
                    <named-formatter name="COLOR-PATTERN"/>
                </formatter>
...
		        <level name="${ROOT_LOGGER_LEVEL}"/>
                <handlers>
                    <handler name="CONSOLE"/>
                </handlers>
            </root-logger>
```

EAP においてログ出力させるためには、ロガー・ハンドラー・フォーマッターの機能を組み合わせて設定する必要があります。
* ロガー：  ログカテゴリ・ログ出力レベルを設定
* ハンドラー：  ログ出力場所を設定
* フォーマッター：  ログ出力フォーマットを設定

上記の設定例では、ログカテゴリーによってキャプチャーされないサーバーへ送信されたすべてのログメッセージをキャプチャーするルートロガー [2] から、Console ログハンドラー [3] を用いて標準出力へログ出力させています。機能要件からロガー (ログカテゴリ) を追加する場合などには、ハンドラーとして Console ログハンドラーを設定することにより標準出力へログ出力させてください。

また、以下のドキュメントのリンクでは EAP 7.4 のリンクを記載していますが、現在 EAP 8 ではドキュメントのモダナイゼーションを実施 [4] しており、EAP 8 においてもログ出力の設定方法は変わっておりません。

[1] https://github.com/mamoru1112/eap8-configmap-sample
[2] https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/7.4/html/configuration_guide/configure_root_logger
[3] https://docs.redhat.com/ja/documentation/red_hat_jboss_enterprise_application_platform/7.4/html/configuration_guide/configuring_log_handlers#configure_console_log_handler
[4] https://access.redhat.com/ja/articles/7063141?extIdCarryOver=true&sc_cid=701f2000001OH74AAG
