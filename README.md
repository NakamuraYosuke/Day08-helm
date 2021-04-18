# Day08-helm
## Helmとは
Kubernetes(以下、k8sと表記します)のパッケージマネージャです。

k8sリソースのインストールと管理を効率化します。apt/yum/homebrewのk8s版みたいなイメージになります。

リポジトリによってチャートが管理されており、チャートがk8sのマニフェストをまとめているという構成になっています。

Helm CLIを使ってローカルにリポジトリを追加し、そのリポジトリ内で管理されているチャートをインストールすることでk8sクラスタにリソースをデプロイすることができます。

## 環境構成
MinikubeにHelmを使ってLoki環境の構築を行います。

- デプロイ先

    Minikube v1.5.2（ローカルマシンはMacを使用）
- デプロイツール

    Helm v3.1.2
    (補足) 以下のロギングスタックの構成を構築するために、loki/loki-stackの version: 0.32.1 のチャートを使用します。

- ロギングスタックの構成（Minikubeにデプロイするもの）
    - Promtail : ログを収集してLokiに送信するエージェント
    - Loki : ログの保存とクエリの処理を行うメインサーバー
    - Grafana : ログの照会、表示を行う監視ツール

## Grafana Loki
Lokiは、Prometheusにインスパイアされた、水平方向にスケーラブルで可用性の高いマルチテナントログ集約システムです。

https://github.com/grafana/loki

Prometheusがメトリックを対象としていることに対して、Lokiはログを対象にしています。

また、ログの配信方法がPrometheusがプル型なのに対し、Lokiではプッシュ型です。

Datadogのように監視対象のサーバーにエージェントをインストールすることでそのエージェントからログが配信されるようになります。

Lokiのロギングスタックは以下の3つのコンポーネントで構成されています。

- ログを収集してLokiに送信するエージェント
- ログの保存とクエリの処理を行うメインサーバー
- ログの照会、表示を行う監視ツール

## Loki構築
### Helmインストール
brewを使ってHelmをインストールします。
```
$ brew install helm
```
以下のバージョンがインストールされたことを確認します。
```
$ helm version
version.BuildInfo{Version:"v3.5.4", GitCommit:"1b5edb69df3d3a08df77c9902dc17af864ff05d1", GitTreeState:"dirty", GoVersion:"go1.16.3"}
```

### チャートのリポジトリ追加
lokiのチャートがあるリポジトリを追加します。
```
$ helm repo add loki https://grafana.github.io/loki/charts
"loki" has been added to your repositories
```

lokiという名前でチャートが登録されていることを確認します。
```
$ helm repo list
NAME    URL
loki    https://grafana.github.io/loki/charts
```

チャートのリポジトリが登録済みである場合は古くなっているかもしれないので更新します。
```
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "loki" chart repository
Update Complete. ⎈ Happy Helming!⎈
```

### Namespace作成
`loki`という名前のnamespaceを作成します。
```
$ kubectl create ns loki
namespace/loki created
```

### loki, grafana, promtailをk8sクラスタにデプロイ
今回使用するチャートはlokiリポジトリのloki-stackになります。

以下コマンドを実行してチャートの情報を確認します。
```
$ helm inspect chart loki/loki-stack
apiVersion: v1
appVersion: v2.0.0
dependencies:
- condition: loki.enabled
  name: loki
  repository: file://../loki
  version: ^2.0.0
- condition: promtail.enabled
  name: promtail
  repository: file://../promtail
  version: ^2.0.0
- condition: fluent-bit.enabled
  name: fluent-bit
  repository: file://../fluent-bit
  version: ^2.0.0
- condition: grafana.enabled
  name: grafana
  repository: https://grafana.github.io/helm-charts
  version: ~5.7.0
- condition: prometheus.enabled
  name: prometheus
  repository: https://prometheus-community.github.io/helm-charts
  version: ~11.16.0
- condition: filebeat.enabled
  name: filebeat
  repository: https://helm.elastic.co
  version: ~7.8.0
- condition: logstash.enabled
  name: logstash
  repository: https://helm.elastic.co
  version: ~7.8.0
deprecated: true
description: 'DEPRECATED Loki: like Prometheus, but for logs.'
home: https://grafana.com/loki
icon: https://raw.githubusercontent.com/grafana/loki/master/docs/sources/logo.png
kubeVersion: ^1.10.0-0
name: loki-stack
sources:
- https://github.com/grafana/loki
version: 2.1.2
```
このチャートは複数のチャート（上記でいうとloki, promtail, fluent-bit, grafana, prometheus）と依存関係を持つ構成だということがわかります。

さらにconditionを使うことでこれらのチャートを使用するかどうかの選択ができるようになっています。

values.yaml で変数定義しており、condition で指定している変数は以下のようになっていることを確認できます。

```
$ helm inspect values loki/loki-stack
loki:
  enabled: true

promtail:
  enabled: true

fluent-bit:
  enabled: false

grafana:
  enabled: false
  sidecar:
    datasources:
      enabled: true
  image:
    tag: 6.7.0

prometheus:
  enabled: false

filebeat:
  enabled: false
  filebeatConfig:
    filebeat.yml: |
      # logging.level: debug
      filebeat.inputs:
      - type: container
        paths:
          - /var/log/containers/*.log
        processors:
        - add_kubernetes_metadata:
            host: ${NODE_NAME}
            matchers:
            - logs_path:
                logs_path: "/var/log/containers/"
      output.logstash:
        hosts: ["logstash-loki:5044"]

logstash:
  enabled: false
  image: grafana/logstash-output-loki
  imageTag: 1.0.1
  filters:
    main: |-
      filter {
        if [kubernetes] {
          mutate {
            add_field => {
              "container_name" => "%{[kubernetes][container][name]}"
              "namespace" => "%{[kubernetes][namespace]}"
              "pod" => "%{[kubernetes][pod][name]}"
            }
            replace => { "host" => "%{[kubernetes][node][name]}"}
          }
        }
        mutate {
          remove_field => ["tags"]
        }
      }
  outputs:
    main: |-
      output {
        loki {
          url => "http://loki:3100/loki/api/v1/push"
          #username => "test"
          #password => "test"
        }
        # stdout { codec => rubydebug }
      }
```

granfana.enabledの値がfalse になっているため、以下のインストールコマンド実行時に--set grafana.enabled=trueというオプションを追加して有効化します。

```
$ helm install -n loki loki/loki-stack --generate-name --set grafana.enabled=true
WARNING: This chart is deprecated
W0418 22:55:25.850179   13784 warnings.go:70] rbac.authorization.k8s.io/v1beta1 Role is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 Role
W0418 22:55:25.857833   13784 warnings.go:70] rbac.authorization.k8s.io/v1beta1 RoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 RoleBinding
W0418 22:55:25.964283   13784 warnings.go:70] rbac.authorization.k8s.io/v1beta1 Role is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 Role
W0418 22:55:25.971541   13784 warnings.go:70] rbac.authorization.k8s.io/v1beta1 RoleBinding is deprecated in v1.17+, unavailable in v1.22+; use rbac.authorization.k8s.io/v1 RoleBinding
NAME: loki-stack-1618754123
LAST DEPLOYED: Sun Apr 18 22:55:25 2021
NAMESPACE: loki
STATUS: deployed
REVISION: 1
NOTES:
The Loki stack has been deployed to your cluster. Loki can now be added as a datasource in Grafana.

See http://docs.grafana.org/features/datasources/loki/ for more detail.
```

minikubeにlokiがデプロイされたことを確認します。
```
$ helm list -n loki
NAME                 	NAMESPACE	REVISION	UPDATED                             	STATUS  	CHART           	APP VERSION
loki-stack-1618754123	loki     	1       	2021-04-18 22:55:25.561971 +0900 JST	deployed	loki-stack-2.1.2	v2.0.0
```

あわせて、各リソースが作成されていることを確認します。
```
$ kubectl get all -n loki
NAME                                                 READY   STATUS    RESTARTS   AGE
pod/loki-stack-1618754123-0                          1/1     Running   0          44m
pod/loki-stack-1618754123-grafana-596c4fb799-jg4n9   1/1     Running   0          44m
pod/loki-stack-1618754123-promtail-ckx7l             1/1     Running   0          44m

NAME                                     TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
service/loki-stack-1618754123            ClusterIP   10.96.191.64     <none>        3100/TCP       44m
service/loki-stack-1618754123-grafana    ClusterIP   10.107.29.166    <none>        80/TCP         44m
service/loki-stack-1618754123-headless   ClusterIP   None             <none>        3100/TCP       44m

NAME                                            DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR   AGE
daemonset.apps/loki-stack-1618754123-promtail   1         1         1       1            1           <none>          44m

NAME                                            READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/loki-stack-1618754123-grafana   1/1     1            1           44m

NAME                                                       DESIRED   CURRENT   READY   AGE
replicaset.apps/loki-stack-1618754123-grafana-596c4fb799   1         1         1       44m

NAME                                     READY   AGE
statefulset.apps/loki-stack-1618754123   1/1     44m
```

### grafanaのadminユーザのパスワードを取得
grafanaのadminパスワードを取得するために生成されたsecretsリソースを確認します。
```
$ kubectl get secrets -n loki
NAME                                             TYPE                                  DATA   AGE
default-token-bps6m                              kubernetes.io/service-account-token   3      50m
loki-stack-1618754123                            Opaque                                1      49m
loki-stack-1618754123-grafana                    Opaque                                3      49m
loki-stack-1618754123-grafana-test-token-lb98v   kubernetes.io/service-account-token   3      49m
loki-stack-1618754123-grafana-token-fs9w5        kubernetes.io/service-account-token   3      49m
loki-stack-1618754123-promtail-token-frk78       kubernetes.io/service-account-token   3      49m
loki-stack-1618754123-token-wfv8l                kubernetes.io/service-account-token   3      49m
sh.helm.release.v1.loki-stack-1618754123.v1      helm.sh/release.v1                    1      49m
```

grafanaのadminパスワードを取得します。
```
$ kubectl get secrets -n loki loki-stack-1618754123-grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo 
ZHMiVmX1YmATUQADj96YZFUrrgVo29V29B2kD0WQ
```

### lokiにアクセスするためのserviceを作成
`service-grafana.yaml`というファイルを作成し、NodePortのserviceを作成します。
instanceはご自身のloki-stackインタンス名に修正してください。

```
apiVersion: v1
kind: Service
metadata:
  labels:
    app.kubernetes.io/instance: loki-stack-1618754123
    app.kubernetes.io/managed-by: Helm
    app.kubernetes.io/name: grafana
    app.kubernetes.io/version: 6.7.0
    helm.sh/chart: grafana-5.7.10
  name: loki-stack-grafana-nodeport
  namespace: loki
spec:
  ports:
  - name: service
    port: 80
    protocol: TCP
    targetPort: 3000
  selector:
    app.kubernetes.io/instance: loki-stack-1618754123
    app.kubernetes.io/name: grafana
  sessionAffinity: None
  type: NodePort
status:
  loadBalancer: {}
```

```
$ kubectl apply -f service-grafana.yaml -n loki
```

### Grafanaへアクセス
ご自身のminikubeのIPアドレスを確認します。
```
$ minikube ip
192.168.64.4
```
```
$ kubectl get svc loki-stack-grafana-nodeport -n loki
NAME                          TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
loki-stack-grafana-nodeport   NodePort   10.111.192.237   <none>        80:30999/TCP   37m
```

ブラウザを起動し、以下のアドレスを入力します。
`http://192.168.64.4:30999`

![](https://raw.githubusercontent.com/NakamuraYosuke/Day08-helm/main/images/login.png)
