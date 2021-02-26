# registry-operator 및 clair 설치 가이드

## 구성 요소

* clair

* registry-operator
  * [Github](https://github.com/tmax-cloud/registry-operator)
  * [Dockerhub](https://hub.docker.com/r/tmaxcloudck/registry-operator/tags?page=1&ordering=last_updated)

## Prerequisites

1. Nginx Ingress Controller 설치

    * [설치 가이드](https://github.com/tmax-cloud/install-ingress/tree/4.1#nginx-ingress-controller-%EC%84%A4%EC%B9%98-%EA%B0%80%EC%9D%B4%EB%93%9C)

1. Hyperauth 설치

    * [설치 가이드](https://github.com/tmax-cloud/install-hyperauth)

## Related Add-ons

1. image-validating-webhook

    * [Github](https://github.com/tmax-cloud/image-validating-webhook)
    * [설치 가이드](https://github.com/tmax-cloud/install-image-validating-webhook)

1. Elastic Search

    * [설치 가이드](https://github.com/tmax-cloud/hypercloud-install-guide/tree/4.1/EFK#step-1-elasticsearch-%EC%84%A4%EC%B9%98)

## clair 폐쇄망 구축 가이드

1. 도커 이미지 파일시스템에 저장

    ```bash
    SCANNER=quay.io/coreos/clair
    sudo docker pull ${SCANNER}
    sudo docker save ${SCANNER} > ${SCANNER}.tar
    ```

2. 저장한 이미지 파일을 설치할 폐쇄망 환경으로 복사

3. 폐쇄망 Registry 서버에 이미지를 푸시

    ```bash
    REGISTRY={REGISTRY}   # ex: REGISTRY=192.168.6.100:5000
    SCANNER=quay.io/coreos/clair
    
    sudo docker load < ${SCANNER}.tar
    sudo docker tag ${SCANNER} ${REGISTRY}/${SCANNER}
    sudo docker push ${REGISTRY}/${SCANNER}
    ```

4. 아래 clair 설치 가이드에 따라 Clair 서버 배치

## clair 설치 가이드

1. registry-operator 설치 디렉터리 이동 후 deploy 스크립트 실행

    ```bash
    cd ${REG_OP_HOME}/config/manager/clair 
    make dev
    ```

## clair 삭제 가이드

* 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall -s
    ```

## registry-operator 폐쇄망 구축 가이드

1. 폐쇄망 Image Registry에서 사용할 Image 및 설치 파일 준비

    * 작업 디렉토리 생성 및 환경 설정

      ```bash
        mkdir -p ~/registry-operator-install
        REG_OP_HOME=~/registry-operator-install
        REG_OP_VERSION=v0.2.2
        cd ${INSTALL_HOME}
      ```

    * Image 저장

      ```bash
        IMG=registry
        VER=2.7.1
        sudo docker pull ${IMG}:${VER}
        sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

        IMG=tmaxcloudck/notary_server
        VER=0.6.2-rc1
        sudo docker pull ${IMG}:${VER}
        sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

        IMG=tmaxcloudck/notary_signer
        VER=0.6.2-rc1
        sudo docker pull ${IMG}:${VER}
        sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

        IMG=tmaxcloudck/notary_mysql
        VER=0.6.2-rc1
        sudo docker pull ${IMG}:${VER}
        sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar
        
        IMG=tmaxcloudck/registry-operator
        VER=${REG_OP_VERSION}
        sudo docker pull ${IMG}:${VER}
        sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar
      ```

    * 설치 파일 저장

      ```bash
        wget -O registry-operator.tar.gz https://github.com/tmax-cloud/registry-operator/archive/${REG_OP_VERSION}.tar.gz
      ```

1. 폐쇄망 Registry에 필요한 Image Push 및 설치 파일 압축 해제 및 Image 주소 수정

    * 위에서 저장한 tar 압축 파일들을 모두 폐쇄망 환경의 ${INSTALL_HOME} 디렉토리로 옮긴다.

    * 아래의 명령어를 실행하여 폐쇄망 Image Registry 주소등 환경설정을 한다.

      ```bash
        mkdir -p ~/registry-operator-install
        REG_OP_HOME=~/registry-operator-install
        REG_OP_VERSION=v0.2.2
        REGISTRY={REGISTRY}   # ex: REGISTRY=192.168.6.100:5000
        cd ${INSTALL_HOME}
      ```

    * 아래의 명령어를 실행하여 Image를 Load하고 Push한다.

      ```bash
        cd ${INSTALL_HOME}
        IMG=registry
        VER=2.7.1
        sudo docker load < ${IMG}_${VER}.tar
        sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
        sudo docker push ${REGISTRY}/${IMG}:${VER}
        
        IMG=tmaxcloudck/notary_server
        VER=0.6.2-rc1
        sudo docker load < ${IMG}_${VER}.tar
        sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
        sudo docker push ${REGISTRY}/${IMG}:${VER}
        
        IMG=tmaxcloudck/notary_signer
        VER=0.6.2-rc1
        sudo docker load < ${IMG}_${VER}.tar
        sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
        sudo docker push ${REGISTRY}/${IMG}:${VER}
        
        IMG=tmaxcloudck/notary_mysql
        VER=0.6.2-rc1
        sudo docker load < ${IMG}_${VER}.tar
        sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
        sudo docker push ${REGISTRY}/${IMG}:${VER}

        IMG=tmaxcloudck/registry-operator
        VER=${REG_OP_VERSION}
        sudo docker load < ${IMG}_${VER}.tar
        sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
        sudo docker push ${REGISTRY}/${IMG}:${VER}
      ```

      * 아래의 명령어를 실행하여 설치 파일 압축 해제 및 Image 주소 수정

        ```bash
          cd ${INSTALL_HOME}
          mkdir registry-operator-${REG_OP_VERSION}
          tar -xzf registry-operator.tar.gz -C registry-operator-${REG_OP_VERSION} --strip-components=1
          REG_OP_HOME=${INSTALL_HOME}/registry-operator-${REG_OP_VERSION}

          IMG=tmaxcloudck\\/registry-operator:${REG_OP_VERSION}
          sed -i 's/'${IMG}'/'${REGISTRY}'\/'${IMG}'/g' ${REG_OP_HOME}/config/manager/manager.yaml
        ```

## registry-operator 설치 가이드

1. [Step 0. 설치 파일 준비](#Step-0-설치-파일-준비)

1. [Step 1. 인증서 생성](#Step-1-인증서-생성)

1. [Step 2. config 설정](#Step-2-config-설정)

1. [Step 3. install script 실행](#Step-3-install-script-실행)

1. [Step 4. 신뢰할 수 있는 인증서로 등록](#Step-4-신뢰할-수-있는-인증서로-등록)

### Step 0. 설치 파일 준비

폐쇄망 구축이 아닌 경우 아래의 명령어를 실행하여 설치 파일을 Github Repository로부터 받아 온다.

```bash
  REG_OP_VERSION=v0.2.2
  REG_OP_DIR=registry-operator-${REG_OP_VERSION}
  mkdir ${REG_OP_DIR}
  wget -c https://github.com/tmax-cloud/registry-operator/archive/${REG_OP_VERSION}.tar.gz -O - |tar -xz -C ${REG_OP_DIR} --strip-components=1
  REG_OP_HOME=$(pwd)/${REG_OP_DIR}
  cd ${REG_OP_HOME}
```

### Step 1. 인증서 생성

* Root CA가 있는 경우

    Root CA를 인증서 디렉토리(./config/pki/)로 옮긴다. (단, 인증서의 이름을 `ca.crt`와 `ca.key`로 해야한다.)

* Hyperauth 인증서를 추가로 신뢰해야 하는 경우(Root CA와 다른 인증서로 Hyperauth를 설치한 경우)

    Hyperauth 인증서를 인증서 디렉토리(config/pki/)로 옮긴다. (단, 인증서의 이름을 `keycloak.crt`로 해야한다.)

* Root CA가 없는 경우 아래의 명령어를 실행하여 인증서를 새로 생성한다.

    ```bash
      cd ${REG_OP_HOME}
      sudo chmod 755 ./config/scripts/newCertFile.sh
      ./config/scripts/newCertFile.sh
      cp ca.crt ca.key ./config/pki/
    ```

### Step 2. config 설정

${REG_OP_HOME}/config/manager/manager_config.yaml 파일에 환경변수를 설정할 수 있다.

환경변수에 대한 설명은 ${REG_OP_HOME}/docs/envs.md 를 보거나 [Github](https://github.com/tmax-cloud/registry-operator/blob/master/docs/envs.md)를 참고하여 설정한다.(Github의 경우 tag를 해당 버전으로 변경해야한다.)

* Check!!

  * hyperauth url 주소 설정: manager_config.yaml 파일에서 keycloak.service 설정
  * clair url 주소 설정: manager_config.yaml 파일에서 clair.url 설정
  * (`폐쇄망의 경우 필수`)폐쇄망 레지스트리 주소 설정: manager_config.yaml 파일에서 image.registry 설정
  * multi cluster의 경우: manager_config.yaml 파일에서 cluster.name 설정으로 클러스터를 구분

### Step 3. install script 실행

* 아래의 명령어를 실행하여 registry-operator를 설치합니다.

    ```bash
    cd ${REG_OP_HOME}
    sudo chmod 755 ./config/scripts/newCertSecret.sh install.sh
    ./install.sh
    ```

### Step 4. 신뢰할 수 있는 인증서로 등록

**Note**: 클러스터 내 모든 노드에 적용

1. [Step 1.](#Step-1.-인증서-생성)에서 만든 Root CA파일(${REG_OP_HOME}/config/pki/ca.crt)을 노드의 인증서 경로로 복사하고 신뢰할 수 있는 인증서 목록을 갱신한다.

    * 노드의 운영체제가 CentOS 7인 경우:

        ```bash
          REMOTE={REMOTE}   # ssh 주소 설정: USER@IP (ex: root@192.168.6.100)
          cd ${REG_OP_HOME}
          scp ./config/pki/ca.crt ${REMOTE}:/etc/pki/ca-trust/source/anchors/registry_ca.crt
          ssh ${REMOTE}
          update-ca-trust
        ```

    * 노드의 운영체제가 Ubuntu 18.04인 경우:

        ```bash
          REMOTE={REMOTE}   # ssh 주소 설정: USER@IP (ex: root@192.168.6.100)
          cd ${REG_OP_HOME}
          scp ./config/pki/ca.crt ${REMOTE}:/usr/local/share/ca-certificates/registry_ca.crt
          ssh ${REMOTE}
          update-ca-certificates
        ```

1. 컨테이너 런타임을 재기동하여 갱신된 CA 목록을 적용한다.

    * Container Runtime이 docker 인 경우:

        ```bash
          systemctl restart docker
        ```

    * Container Runtime이 cri-o 인 경우:

        ```bash
          systemctl restart crio
        ```

## registry-operator 삭제 가이드

* registry-operator와 모든 crd 리소스 삭제를 원하는 경우, 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall -a
    ```

* registry-operator만 삭제를 원하는 경우, 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall -m
    ```

* crd 리소스만 삭제를 원하는 경우, 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall -c
    ```
