---
description: >-
  이번 가이드에서는 카카오 i 클라우드에서 DevOps Pipeline을 사용하여 CD/CD 환경을 구축해 IKE를 배포하는 방식을
  설명합니다.
---

# 📚 DevOps Pipeline을 이용하여 클릭 몇 번으로 CI/CD 환경 구축하기

## 실습 목표

카카오 i 클라우드에서는 웹 콘솔을 통해 GUI 기반으로 쉽게 DevOps Pipeline을 생성하고, CI/CD 환경을 구축 및 관리할 수 있습니다.

{% hint style="info" %}
안내

* 소요 시간:&#x20;
* 사용자 로컬 권장 환경
  * OS: macOS
* 마지막 업데이트 날짜: 2022년 12월 일
{% endhint %}



## 사전 작업

CI/CD 환경 구축을 위해 사전 작업이 필요합니다. 사전 작업이 완료된 분들은 [**Step 1**](./#step-1.)부터 진행해주시기 바랍니다.

### Container Registry 만들기

카카오 i 클라우드의 Container Registry(CR)를 이용해 컨테이너 이미지 저장소를 생성합니다.

1. ****[**카카오 i 클라우드 웹 콘솔**](https://console.kakaoi.io/)에 접속합니다.
2.  ****[**Container Registry**](https://console.kakaoi.io/container-registry) **** 탭에서 \[리포지토리 만들기] 버튼을 클릭해 리포지토리를 생성합니다.



| 공개 여부 | 리포지토리 이름 | 태그 덮어쓰기 | 이미지 스캔 |
| ----- | -------- | ------- | ------ |
| 공개    | test-cr  | 가능      | 수ㅇ     |

1. CR 생성 후 \[커맨드 보기] 버튼 클릭 시, 도커 커맨드와 CR 접근 URI를 확인할 수 있습니다.



### IKE를 이용하여 쿠버네티스 클러스터 구축하기

{% embed url="https://kicdocs.oopy.io/ike_webservice" %}
Hands On 2. IKE를 이용하여 K8S 환경 구축하기
{% endembed %}

위 가이드의 Step 2 과정까지 진행해주시기 바랍니다.



### 예제 리포지토리 설정

<details>

<summary>예제 리포지토리 fork</summary>

****[**spring-petclinic git 리포지토리**](https://github.com/spring-projects/spring-petclinic.git)에 접속하여 git 프로젝트를 내 리포지토리로 fork 합니다.

</details>

<details>

<summary>리포지토리 clone</summary>

핸즈온 진행을 위한 별도의 작업 디렉터리를 생성하고 fork한 리포지토리를 Clone 합니다.

```shell
sudo mkdir ~/handson
cd ~/handson
sudo git clone https://github.com/${USER_NAME}/spring-petclinic.git
```

</details>

<details>

<summary>리포지토리 설정파일 추가</summary>

Clone 받은 리포지토리에 `Dockerfile`과 `Helm chart` 디렉토리를 생성합니다.&#x20;

```shell
cd ~/handson/spring-petclinic
vi Dockerfile

# Dockerfile
FROM docker.io/library/gradle:7.5.1-jdk11 AS build

WORKDIR /app

# add project
ADD . /app/

# server mvn install
RUN gradle clean bootJar

FROM docker.io/library/eclipse-temurin:11-jre-focal AS app

WORKDIR /app
COPY --from=build /app/build/libs/spring-petclinic-2.7.3.jar .

EXPOSE 8080

# server run
ENTRYPOINT ["java"]
CMD ["-jar", "/app/spring-petclinic-2.7.3.jar"]
```

```shell
cd ~/handson/spring-petclinic
mkdir chart
cd chart

cat -v << \EOF | sudo tee values.yaml
namespace: default

app:
  name: petclinic-k8s
  profile:
  replicaCount: 1
  hostAliases: []
  image: ${PROJECT_NAME}.kr-central-1.kcr.dev/${CR_NAME}/test-img:latest
  resources: {}
  service:
    type: LoadBalancer

EOF

cat -v << \EOF | sudo tee Chart.yaml
apiVersion: v2
name: helm
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: 1.16.0

EOF

mkdir templates
cd templates

cat -v << \EOF | sudo tee deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.namespace }}
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.app.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      {{- with .Values.app.hostAliases }}
      hostAliases:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: app
          imagePullPolicy: Always
          image: {{ .Values.app.image }}
          {{- with .Values.app.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          command: [ "java" ]
          args:
            - "-jar"
            - "spring-petclinic-2.7.3.jar"
          {{- with .Values.app.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          ports:
            - containerPort: 8080
EOF

cat -v << \EOF | sudo tee service.yaml
apiVersion: v1
kind: Service
metadata:
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.app.name }}
  name: {{ .Values.app.name }}-svc
spec:
  ports:
    - name: petclinic
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: {{ .Values.app.name }}
  type: {{ .Values.app.service.type }}
EOF
```

</details>

```shell
cd ~/handson/petclinic
```

{% tabs %}
{% tab title="${pr}/Dockerfile" %}
```docker
# Dockerfile
FROM docker.io/library/gradle:7.5.1-jdk11 AS build

WORKDIR /app

# add project
ADD . /app/

# server mvn install
RUN gradle clean bootJar

FROM docker.io/library/eclipse-temurin:11-jre-focal AS app

WORKDIR /app
COPY --from=build /app/build/libs/spring-petclinic-2.7.3.jar .

EXPOSE 8080

# server run
ENTRYPOINT ["java"]
CMD ["-jar", "/app/spring-petclinic-2.7.3.jar"]
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="${PROJECT}/chart/values.yaml" %}
```yaml
namespace: default

app:
  name: petclinic-k8s
  profile:
  replicaCount: 1
  hostAliases: []
  image: ${PROJECT_NAME}.kr-central-1.kcr.dev/${CR_NAME}/test-img:latest
  resources: {}
  service:
    type: LoadBalancerdoc
```
{% endtab %}

{% tab title="${PROJECT}/chart/Chart.yaml" %}
```yaml
apiVersion: v2
name: helm
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: 1.16.0
```
{% endtab %}
{% endtabs %}

{% tabs %}
{% tab title="${PROJECT}/chart/templates/deployment.yaml" %}
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  namespace: {{ .Values.namespace }}
  name: {{ .Values.app.name }}
  labels:
    app: {{ .Values.app.name }}
spec:
  replicas: {{ .Values.app.replicaCount }}
  selector:
    matchLabels:
      app: {{ .Values.app.name }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
    spec:
      {{- with .Values.app.hostAliases }}
      hostAliases:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      containers:
        - name: app
          imagePullPolicy: Always
          image: {{ .Values.app.image }}
          {{- with .Values.app.resources }}
          resources:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          command: [ "java" ]
          args:
            - "-jar"
            - "spring-petclinic-2.7.3.jar"
          {{- with .Values.app.env }}
          env:
            {{- toYaml . | nindent 12 }}
          {{- end }}
          ports:
            - containerPort: 8080
```
{% endtab %}

{% tab title="${PROJECT}/chart/templates/service.yaml" %}
```yaml
apiVersion: v1
kind: Service
metadata:
  namespace: {{ .Values.namespace }}
  labels:
    app: {{ .Values.app.name }}
  name: {{ .Values.app.name }}-svc
spec:
  ports:
    - name: petclinic
      port: 8080
      protocol: TCP
      targetPort: 8080
  selector:
    app: {{ .Values.app.name }}
  type: {{ .Values.app.service.type }}
```
{% endtab %}
{% endtabs %}

<details>

<summary>리포지토리에 변경사항 Push</summary>

파일 및 디렉토리를 모두 생성한 뒤, git 명령어를 통해 remote repository에 변경사항을 push 합니다.

```shell
git add .
git commit -m "add Dockerfile & Chart directory"
git push origin main
```

</details>



## Step 1. 환경 설정

파이프라인 구성을 위해 환경 설정이 필요합니다. DevOps Pipeline의 [환경 설정](https://console.kakaoi.io/devops-pipeline/configuration/) 탭으로 이동합니다.

### 외부 서비스(소스) 만들기



### Container Registry 접근용 크리덴셜 만들기



### 외부 서비스(Container Registry) 만들기



### ike-cluster 접근용 크리덴셜 만들기



