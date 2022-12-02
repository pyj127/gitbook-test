# \[Doollee] DevOps Pipeline을 이용하여 클릭 몇 f7728095b4014f00a4a5270ddb5596a9

## \[Doollee] DevOps Pipeline을 이용하여 클릭 몇 번으로 CI/CD 환경 구축하기

💡 이번 가이드에서는 카카오 i 클라우드에서 DevOps Pipeline을 사용하여 CD/CD 환경을 구축해 IKE를 배포하는 방식을 설명합니다.

## 실습 목표

***

카카오 i 클라우드에서는 웹 콘솔을 통해 GUI 기반으로 쉽게 DevOps Pipeline을 생성하고, CI/CD 환경을 구축 및 관리할 수 있습니다.

💡 \*\*안내\*\*

* 소요 시간:
* 사용자 로컬 권장 환경
  * OS: macOS, Linux(ubuntu)
* 마지막 업데이트 날짜: 2022년 11월 일

_파이프라인 - IKE 구조 그림_

## 사전 작업

***

CI/CD 환경 구축을 위해 사전 작업이 필요합니다. 사전 작업들이 완료된 분들은 [Step 1](https://www.notion.so/Doollee-DevOps-Pipeline-CI-CD-f7728095b4014f00a4a5270ddb5596a9)부터 진행해주시기 바랍니다.

#### Container Registry 만들기

카카오 i 클라우드의 Container Registry(CR)를 이용해 컨테이너 이미지 저장소를 생성합니다.

1. [카카오 i 클라우드 웹 콘솔](https://console.kakaoi.io/)에 접속합니다.
2.  [**Container Registry**](https://console.kakaoi.io/container-registry) 탭에서 \[리포지토리 만들기] 버튼을 클릭해 리포지토리를 생성합니다.

    | 공개 여부 | 리포지토리 이름 | 태그 덮어쓰기 | 이미지 스캔 |
    | ----- | -------- | ------- | ------ |
    | 공개    | test-cr  | 가능      | 수동     |
3. CR 생성 후 \[커맨드 보기] 버튼 클릭 시, 도커 커맨드와 CR 접근 URI를 확인할 수 있습니다.

#### IKE를 이용하여 쿠버네티스 클러스터 구축하기

[Kubernetes Engine(IKE)를 이용하여 K8S 환경 구축하기](https://kicdocs.oopy.io/ike\_webservice)

위 가이드의 Step 2까지 진행해주시기 바랍니다.

#### 예제 리포지토리 설정

> **예제 리포지토리 fork**

[\*\*Spring-petclinic](https://github.com/spring-projects/spring-petclinic.git)\*\* git 리포지토리에 접속하여 git 프로젝트를 내 리포지토리로 fork 합니다.

> **리포지토리 clone**

핸즈온 진행을 위한 별도의 작업 디렉터리를 생성하고 fork한 리포지토리를 Clone 합니다.

```bash
sudo mkdir ~/handson
cd ~/handson
sudo git clone https://github.com/${USER_NAME}/spring-petclinic.git
```

> **리포지토리 설정파일 추가**

Clone 받은 리포지토리에 도커 빌드를 위한 Dockerfile을 생성합니다.

*   `Code`

    ```bash
    cd ~/handson/spring-petclinic
    vi Dockerfile
    ```

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

Helm 배포를 위한 chart 디렉토리를 생성합니다.

*   `Code`

    ```bash
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
    ```

chart 디렉토리 내에 templates 디렉토리를 생성하고 해당 디렉토리 내에서 deployment, service에 대한 yaml 파일을 작성합니다.

*   `Code`

    ```bash
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

> **리포지토리에 변경사항 push**

파일 및 디렉토리를 모두 생성한 뒤, git 명령어를 통해 remote repository에 변경사항을 push 합니다.

```bash
git add .
git commit -m "add Dockerfile & Chart directory"
git push origin main
```

## Step 1. 환경 설정

***

파이프라인 구성을 위해 환경 설정이 필요합니다. DevOps Pipeline의 [환경 설정](https://console.kakaoi.io/devops-pipeline/configuration/) 탭으로 이동합니다.

#### 외부 서비스(소스) 만들기

배포할 서비스의 소스 저장소(Github)를 등록합니다.

| 외부 서비스 이름            | 서비스 타입 | 엔드 포인트 URL                                        |
| -------------------- | ------ | ------------------------------------------------- |
| spring-petclinic-git | Github | https://github.com/${USER\_NAME}/spring-petclinic |

사전 작업에서 설정한 Github URL을 입력합니다.

#### Container Registry 접근용 크리덴셜 만들기

Container Registry에 접근하기 위해 크리덴셜(인증 정보)을 등록합니다.

| 크리덴셜 이름   | 인증 유형         | 사용자 이름       | 비밀번호         |
| --------- | ------------- | ------------ | ------------ |
| cr-secret | 사용자 이름 / 비밀번호 | 사용자 액세스 키 ID | 사용자 액세스 보안 키 |

사용자 액세스 키 발급 및 관리를 위한 자세한 설명은 **Kakao i Cloud 콘솔 > 설정 가이드 >** [**사용자 엑세스 키**](https://console.kakaoi.io/docs/posts/kic\_console/2022-06-13-kic\_console\_setting/kic\_console\_setting#%EC%82%AC%EC%9A%A9%EC%9E%90-%EC%95%A1%EC%84%B8%EC%8A%A4-%ED%82%A4) 를 참고하시기 바랍니다.

#### 외부 서비스(Container Registry) 만들기

| 외부 서비스 이름    | 서비스 타입 | 엔드 포인트 URL                   |
| ------------ | ------ | ---------------------------- |
| docker-image | Docker | https://${CR 접근 URI}/test-cr |

`CR 접근 URI`에는 [사전 작업](https://www.notion.so/Doollee-DevOps-Pipeline-CI-CD-f7728095b4014f00a4a5270ddb5596a9)에서 생성한 Container Registry의 URI를 기입합니다.

`test-cr`은 Container Registry 생성 시 입력한 리포지토리 이름입니다.

#### ike-cluster 접근용 크리덴셜 만들기

| 크리덴셜 이름            | 인증 유형      | Kubeconfig 파일      |
| ------------------ | ---------- | ------------------ |
| ike-cluster-secret | Kubeconfig | kubeconfig 파일 불러오기 |

kubeconfig 파일은 [사전 작업](https://www.notion.so/Doollee-DevOps-Pipeline-CI-CD-f7728095b4014f00a4a5270ddb5596a9)에서 생성한 쿠버네티스 클러스터의 kubeconfig 파일을 불러옵니다.

💡 \*\*외부 서비스\*\*는 외부의 서비스에 대해 접근할 수 있는 정보를 저장합니다. 엔드포인트 URL을 입력하며, 주로 소스 저장소에 대한 접근 설정을 할 수 있습니다.💡 \*\*크리덴셜(Credential)\*\*은 외부 서비스 또는 서비스 타입을 사용하기 위한 인증 목록으로, \*사용자 이름/비밀번호, 인증키, 토큰\* 등의 방식으로 인증 정보를 정의할 수 있습니다.

## Step 2. 파이프라인 만들기

***

환경 설정을 마쳤으니 본격적으로 DevOps Pipeline을 생성해 CI/CD 환경을 구축해봅시다.

카카오 i 클라우드의 DevOps Pipeline을 이용해 사용자는 버튼 클릭 몇 번 만으로 CI/CD 파이프라인을 구축할 수 있습니다.

1. 카카오 i 클라우드 웹 콘솔에서 \*\*[DevOps Pipeline](https://console.kakaoi.io/devops-pipeline/pipeline/pipelines)\*\*을 클릭해 이동합니다.
2. 파이프라인 만들기를 클릭한 후, 처음부터 만들기를 선택합니다.
3. 좌측 태스크 추가 탭에서 아래 리스트의 태스크를 추가합니다.
   1. `소스 > Github`
   2. `빌드 > Docker`
   3. `배포 > Helm`
4. 우측 태스크 설정 탭에서 각 태스크 별로 설정 값을 입력합니다.
   *   **Github**

       | 외부 서비스               | 리비전  |
       | -------------------- | ---- |
       | spring-petclinic-git | main |
   *   **Docker**

       | 외부 서비스       | 도커 파일 경로 | 도커 이미지 이름 | 태그     |
       | ------------ | -------- | --------- | ------ |
       | docker-image | .        | test-img  | latest |
   *   **Helm**

       | 배포 차트 경로 | 크리덴셜               | 헬름 차트 이름 | 네임스페이스  | 벨류 파일       |
       | -------- | ------------------ | -------- | ------- | ----------- |
       | chart    | ike-cluster-secret | app      | default | values.yaml |
5. Github → Docker → Helm 순으로 연결합니다.
6. 파이프라인을 저장합니다.

카카오 i 클라우드의 Container Registry, IKE, DevOps Pipeline을 가지고 CI/CD 파이프라인 구축이 완성되었습니다.

KiC CR, IKE, Pipeline 이용해서 \~배포. 빠르게 된다. 이것만으로도 서비스 구축이 가능하다. 개발 환경 구성이 우리 플랫폼만에서 다 된다.

## Step 3. DevOps Pipeline 실행하기

***

[**DevOps Pipeline**](https://console.kakaoi.io/devops-pipeline/pipeline/pipelines)으로 이동하여 파이프라인 목록을 확인하고, 실행 버튼을 눌러 파이프라인을 실행합니다.

파이프라인 실행 이력 탭에서 상세 정보(실행 결과, 소요 시간, 로그, …)를 확인할 수 있습니다.

그림. 파이프라인 실행 및 실행 이력 확인

💡 카카오 i 클라우드 DevOps Pipeline은 수동 실행과 스케쥴 설정을 모두 지원합니다. 자세한 설명은 \*\*사용자 가이드 > DevOps Pipeline > \[파이프라인 관리하기]\(https://console.kakaoi.io/docs/posts/dev\_pipeline/dev\_pipeline\_htg/2022-09-08-dev\_pipeline\_htg\_managing/dev\_pipeline\_htg\_managing#%ED%8C%8C%EC%9D%B4%ED%94%84%EB%9D%BC%EC%9D%B8-%EC%8B%A4%ED%96%89%ED%95%98%EA%B8%B0)\*\* 탭을 참고하세요.

## Step 4. 서비스 접속 확인하기

***

파이프라인과 쿠버네티스 클러스터를 통해 spring-petclinic 서비스 배포가 완료되었습니다.

[**카카오 i 클라우드의 LoadBalancer**](https://console.kakaoi.io/load-balancer) 서비스 목록에 petclinic 로드 밸런서가 생성된 것을 확인하실 수 있습니다.

생성된 로드밸런서에 공인 IP를 연결하고, 해당 공인 IP의 8080 포트로 접속하여 잘 구현되었는지 확인하시기 바랍니다.

그림. 서비스 접근

😃 수고 많으셨습니다! \~\~카카오 i 클라우드에서 NVIDIA GPU 환경을 사용하기 위한 준비가 되었습니다. 이제 \[GPU Virtual Machine으로 Jupyter Notebook 환경 구성]\(https://www.notion.so/GPU-Virtual-Machine-Jupyter-Notebook-568012406e0349778acd0c9afb4b9535)을 시작해보세요!\~\~

[Kakao i Cloud 이용약관](https://policy.kakaoicloud.com/terms/kakaoicloud) | [**개인정보처리방침**](https://policy.kakaoi.ai/service\_privacy) Copyright © Kakao Enterprise Corp. All Rights Reserved.
