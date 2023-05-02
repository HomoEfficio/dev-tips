# Mac M1 Python Setup

## miniconda

- https://docs.conda.io/en/latest/miniconda.html#latest-miniconda-installer-links 에서 M1 64-bit pkg 다운로드 후 실행

### base 환경 비활성화

- pkg 설치 후 shell을 실행하면 다음과 같이 맨 앞에 `(base)`라고 표시된다

    ```
    (base) ~ 🦑🍺 ❯
    ```

- 기본으로 conda base 환경이 활성화 되기 때문인데 파이썬이 아닌 다른 작업을 할 때도 표시되어 불편하므로 기본 비활성화하고 필요할 때만 `conda activate <<env-name>>`으로 활성화한다

    ```
    (base) ~ 🦑🍺 ❯ conda config --set auto_activate_base False
    ```


### channel 추가

- 채널은 패키지 저장소(repository) 같은 개념
- 처음에는 기본 채널만 존재

    ```
    ~ 🦑🍺 ❯ conda config --show channels                  
    channels:
      - defaults
    ```

- 기본(defaults) 채널보다 더 다양하고 편리한 conda-forge 채널 추가

    ```
    ~ 🦑🍺 ❯ conda config --add channels conda-forge && conda config --set channel_priority strict
    ~ 🦑🍺 ❯ conda config --show channels
    channels:
      - conda-forge
      - defaults
    ```

