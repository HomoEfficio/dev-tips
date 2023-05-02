# helm

[helm](https://helm.sh/)은 k8s 패키지 매니저다.

공식 문서에 흔한 그림 하나 없고 이래저래 영 마음에 들지 않아서 개념적으로 이해하는 데 필요 이상의 노력이 드는 것 같아서 어쩔 수 없이 따로 핵심만 추려서 정리해본다.


# 동작 구조

일단 k8s 패키지 매니저가 뭔지 그림으로 맛을 보자.

이해하기 쉽게 한 가지 방식만을 골라서 단순화 했으며, 실제로는 물론 여러가지 시나리오, 방식으로 구성할 수 있다.

![Imgur](https://i.imgur.com/SeEuKqD.png)

- 개발자가 작성한 Helm Chart 를 `helm push` 명령으로 `Helm Chart Repository`에 업로드 한다.
- 개발자가 만든 Container Image 를 `docker push` 명령으로 `Container Registry`에 업로드 한다.
- 개발자가 k8s Control Plane 에 values.yaml 파일을 지정하면서 `helm install` 명령을 전달하면,
  - values.yaml 에 있는 값이 Helm Chart 에 주입되고 Helm Release 가 생성되고,
  - Helm Chart 에 들어있는 Image 정보를 통해 Container Image 를 가져와서 Helm Chart 정보를 토대로 Pod 등 k8s 자원이 생성된다.


# 주요 용어

## helm chart

- k8s 자원 yaml 파일을 만들 수 있는 여러 yaml 템플릿 파일과 설정값이 들어 있는 values.yaml 파일로 이루어진 파일 세트
- 대략 아래와 같은 파일이 모여있는 디렉터리라고 봐도 크게 틀리지 않는다. 아래 내용은 helm 한글 문서 https://helm.sh/ko/docs/topics/charts/#차트-파일-구조 에서 가져왔다.

    ```
    CHART이름/  
      Chart.yaml          # 차트에 대한 정보를 가진 YAML 파일
      LICENSE             # 옵션: 차트의 라이센스 정보를 가진 텍스트 파일
      README.md           # 옵션: README 파일
      values.yaml         # 차트에 대한 기본 환경설정 값들
      values.schema.json  # 옵션: values.yaml 파일의 구조를 제약하는 JSON 파일
      charts/             # 이 차트에 종속된 차트들을 포함하는 디렉터리
      crds/               # 커스텀 자원에 대한 정의
      templates/          # values와 결합될 때, 유효한 쿠버네티스 manifest 파일들이 생성될 템플릿들의 디렉터리
      templates/NOTES.txt # 옵션: 간단한 사용법을 포함하는 텍스트 파일
    ```

- templates 폴더 안에는 다음과 같은 k8s 자원 yaml 파일이 들어있다. 아래 내용은 helm 한글 문서 https://helm.sh/ko/docs/chart_template_guide/getting_started/#mycharttemplates-훑어보기 에서 가져왔다.

    ```
    NOTES.txt : 차트의 "도움말". 이것은 helm install 을 실행할 때 사용자에게 표시될 것이다.
    deployment.yaml : 쿠버네티스 디플로이먼트를 생성하기 위한 기본 매니페스트
    service.yaml : 디플로이먼트의 서비스 엔드포인트를 생성하기 위한 기본 매니페스트
    _helpers.tpl : 차트 전체에서 다시 사용할 수 있는 템플릿 헬퍼를 지정하는 공간
    ```

  - 예를 들어 deployment.yaml 파일에 아래와 같은 내용이 있다면, `helm install` 실행 시 사용되는 values.yaml 파일에 `spring.configLocation`이라는 항목(key)이 있다면 그 값을 `SPRING_CONFIG_LOCATION` 환경변수에 저장한다.

      ```yaml
          env:
            {{- if hasKey .Values.spring "configLocation" }}
            - name: SPRING_CONFIG_LOCATION
              value: '{{ toString .Values.spring.configLocation }}'
            {{- end}}
      ```

- values.yaml 파일도 포함돼 있는데 이 파일을 사용할 수도 있고, helm chart 밖에 존재하는 별도의 values.yaml 파일을 지정해서 사용할 수도 있다. 위 그림에 나온 방식은 별도의 values.yaml 파일을 사용하고 있다.

## helm release

- `helm install` 명령 실행결과로 생성되며, helm chart의 인스턴스라고 볼 수 있다.
- 하나의 helm chart 에
  - 서로 다른 values.yaml 을 적용해서 내용적으로 다른 여러 helm release 를 만들 수도 있고,
  - 동일한 values.yaml 을 적용하되 release 이름을 다르게 지정해서 내용적으로는 동일한 여러 helm release 를 만들 수도 있다.
- helm chart 및 values.yaml 파일의 내용 변경을 k8s 자원에 반영해야 할 때 `helm upgrade` 명령을 통해 release 의 내용을 변경할 수 있다.
  - values.yaml 파일 변경 시 `helm upgrade`만 하면 되지만, deployment.yaml 파일이 변경됐을 때는 `helm push`로 차트 먼저 리포지토리에 푸시해둬야 한다.


# 주요 명령

사실 이런 건 그냥 공식 문서를 보고 알 수 있어야 하는데, helm 문서에 나온 설명에 혼동을 불러일으키는 부분이 많아서 어쩔 수 없이 일부만 정리한다.

이 명령 사용법만 알면 나머지 다른 명령은 큰 혼동 없이 문서를 보고도 이해할 수 있을 것이다.


## helm push

helm chart를 helm repository에 저장

```
형식: helm push 올릴chart경로 helmRepo이름
예제: helm push deploy/my-chart my-repo
```

helm v3.8.0 부터는 위와 같이 `helm push deploy/my-chart my-repo` 명령을 싫행하면 아래와 같이 에러가 발생한다.  

```bash
❯ helm push deploy/my-chart my-repo --debug

Error: scheme prefix missing from remote (e.g. "oci://")
helm.go:84: [debug] scheme prefix missing from remote (e.g. "oci://")
```

이유는 https://helm.sh/docs/topics/registries/ 에 나와 있는데, 결론적으로 v3.8.0 부터는 `helm push` 명령은 아래와 같은 형식으로만 사용 가능하다.

```bash
helm push mychart-0.1.0.tgz oci://localhost:5000/helm-charts
```
tgz 파일은 `helm package deploy/my-chart` 명령으로 만들 수 있는데, oci 지원 리포지토리는 담당하는 곳에서 따로 구성해주지 않으면 어쩔 도리가 없다.

그래서 결국에는 아래와 같이 plugin을 설치하고

```bash
❯ helm plugin install https://github.com/chartmuseum/helm-push

Downloading and installing helm-push v0.10.3 ...
https://github.com/chartmuseum/helm-push/releases/download/v0.10.3/helm-push_0.10.3_darwin_arm64.tar.gz
Installed plugin: cm-push
```

아래와 같이 push 대신 cm-push 를 사용하면 된다.
```bash
❯ helm cm-push deploy/my-chart my-repo --debug

Pushing my-chart-0.1.0.tgz to my-repo...
Done.
```
push 후 `helm repo update` 명령 실행 후 `helm search repo my-repo`를 실행하면 push 한 차트를 확인할 수 있다.

## helm install

helm repository에 저장된 helm chart를 가져오고 여기에 특정 values-xxx.yaml 을 명시적으로 지정해서 helm chart에 포함된 k8s yaml template files 에 values-xxx.yaml 파일에 있는 값을 주입하고 k8s 자원 yaml을 생성할 수 있는 helm release 를 생성하고 실제로 k8s에 자원 배포

```
형식: helm install -n k8sNamespace 생성될release이름 사용할chart이름 -f 사용할valuesyaml파일경로
예제: helm install -n my-namespace my-release my-repo/my-chart -f deploy/values-my-values.yaml
```


## helm uninstall

install 에 의해 생성된 helm release를 삭제하고 k8s에 생성됐던 deployment 등 관련 자원도 모두 삭제된다

```
형식: helm uninstall -n k8sNamespace 삭제할release이름
예제: helm uninstall -n my-namespace my-release
```


## helm upgrade

install 에 의해 생성된 helm release의 내용 변경

```
형식: helm upgrade -n k8sNamespace release이름 사용할chart이름 -f 사용할valuesyaml파일경로
예제: helm upgrade -n my-namespace my-release my-repo/my-chart -f deploy/values-my-other-values.yaml
```

## helm list

install 에 의해 생성된 helm release 의 목록 조회

```
helm list
```


# 실무 사용 참고

- 소스 코드에서 컨테이너 이미지를 만들어 컨테이너 레지스트리에 업로드 하는 일은 보통 jenkins 등 CI 도구를 사용해서 처리한다.
- 소스 코드가 공개 repo에 있다면 ArgoCD 를 k8s 클러스터 내부에 구성해서 편리하게 배포할 수 있다.
- 소스 코드가 비공개 repo에 있다면 이 비공개 repo에 접근할 수 있는 곳에 CI용 jenkins를 두고, k8s 클러스터 내부에 CD용 jenkins를 둬서, CI용 jenkins가 이미지를 컨테이너 레지스트리에 올린 후에 CD용 jenkins를 호출(HTTP API)해서 배포할 수 있다.
- 소스 코드가 변경됐다면 그에 따른 image 만 변경하면 되므로 CI-CD만 하면 되고, helm 작업(install 또는 upgrade)은 다시 할 필요가 없다.
- 소스 코드는 변경이 없는데 helm chart나 values.yaml 파일만 변경됐다면 helm upgrade만 다시 하면 배포까지 되고 CI-CD 작업은 다시 할 필요가 없다.
  - values.yaml 파일 변경 시 `helm upgrade`만 하면 되지만, deployment.yaml 파일이 변경됐을 때는 `helm push`로 차트 먼저 리포지토리에 푸시해둬야 한다.


# helm chart 작성

작성 방법은 공식 문서 https://helm.sh/ko/docs/chart_template_guide/ 를 참고한다.



