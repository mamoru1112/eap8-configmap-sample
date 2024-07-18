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