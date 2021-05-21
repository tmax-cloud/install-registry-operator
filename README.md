# registry-operator 설치 가이드

## 구성 요소 및 버전

* registry-operator
  * Github
    * latest version source: [v0.3.2](https://github.com/tmax-cloud/registry-operator/tree/v0.3.2)
    * latest version release: [v0.3.2](https://github.com/tmax-cloud/registry-operator/releases/tag/v0.3.2)
  * Dockerhub(image)
    * [tamxcloudck/registry-operator:v0.3.2](https://hub.docker.com/layers/tmaxcloudck/registry-operator/v0.3.2/images/sha256-99c30ac81f274ada1b5743726f5b4b2c91122dd4c168ed2fe54dd2792ae378c5?context=explore)
    * [tamxcloudck/registry-job-operator:v0.3.2](https://hub.docker.com/layers/tmaxcloudck/registry-job-operator/v0.3.2/images/sha256-e353f082e0b950417c82059cf04c22d6aba497304cb186e91b13b539dc18d89a?context=explore)

## Prerequisites

1. Nginx Ingress Controller 설치

    * [설치 가이드](https://github.com/tmax-cloud/install-ingress/tree/5.0)

1. Hyperauth 설치

    * [설치 가이드](https://github.com/tmax-cloud/install-hyperauth/tree/5.0)
    * 설치시 hyperauth Deployment에 args 필드 내용 추가 필요
        * spec.template.spec.containers.args: ["-Dkeycloak.profile.feature.docker=enabled", "-b", "0.0.0.0"]
            * 설치가이드에서 [`install-hyperauth/manifest/2.hyperauth_deployment.yaml`](https://github.com/tmax-cloud/install-hyperauth/blob/5.0/manifest/2.hyperauth_deployment.yaml#L32) 파일 내용 확인

1. Clair 설치

    * [설치 가이드](https://github.com/tmax-cloud/install-clair/tree/5.0)

## Related Add-ons

1. image-validating-webhook

    * [Github](https://github.com/tmax-cloud/image-validating-webhook)
    * [설치 가이드](https://github.com/tmax-cloud/install-image-validating-webhook/tree/5.0)

1. Elastic Search

    * [설치 가이드](https://github.com/tmax-cloud/install-EFK/tree/5.0#step-1-elasticsearch-%EC%84%A4%EC%B9%98)

## registry-operator 폐쇄망 구축 가이드

1. 폐쇄망 Image Registry에서 사용할 Image 및 설치 파일 준비

    * 작업 디렉토리 생성 및 환경 설정

      ```bash
      mkdir -p ~/registry-operator-install
      INSTALL_HOME=~/registry-operator-install
      REG_OP_VERSION=v0.3.4
      cd ${INSTALL_HOME}
      ```

    * Image 저장

      ```bash
      IMG=registry
      VER=2.7.1
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

      mkdir tmaxcloudck
      IMG=tmaxcloudck/notary_server
      VER=0.6.2-rc1
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

      IMG=tmaxcloudck/notary_signer
      VER=0.6.2-rc1
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar

      IMG=tmaxcloudck/notary_mysql
      VER=0.6.2-rc2
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar
      
      IMG=tmaxcloudck/registry-operator
      VER=${REG_OP_VERSION}
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar
      
      IMG=tmaxcloudck/registry-job-operator
      VER=${REG_OP_VERSION}
      sudo docker pull ${IMG}:${VER}
      sudo docker save ${IMG}:${VER} > ${IMG}_${VER}.tar
      ```

    * 설치 파일 저장

      ```bash
      wget -O registry-operator.tar.gz https://github.com/tmax-cloud/registry-operator/archive/${REG_OP_VERSION}.tar.gz
      ```

1. 폐쇄망 Registry에 필요한 Image Push 및 설치 파일 압축 해제 및 Image 주소 수정

    * `위에서 저장한 tar 압축 파일들을 ${INSTALL_HOME} 디렉토리 내용 그대로` 폐쇄망 환경의 ${INSTALL_HOME} 디렉토리로 옮긴다.

    * 아래의 명령어를 실행하여 폐쇄망 Image Registry 주소등 환경설정을 한다.

      ```bash
      mkdir -p ~/registry-operator-install
      INSTALL_HOME=~/registry-operator-install
      REG_OP_VERSION=v0.3.4
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
      VER=0.6.2-rc2
      sudo docker load < ${IMG}_${VER}.tar
      sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
      sudo docker push ${REGISTRY}/${IMG}:${VER}

      IMG=tmaxcloudck/registry-operator
      VER=${REG_OP_VERSION}
      sudo docker load < ${IMG}_${VER}.tar
      sudo docker tag ${IMG}:${VER} ${REGISTRY}/${IMG}:${VER}
      sudo docker push ${REGISTRY}/${IMG}:${VER}

      IMG=tmaxcloudck/registry-job-operator
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
        
        IMG=tmaxcloudck\\/registry-job-operator:${REG_OP_VERSION}
        sed -i 's/'${IMG}'/'${REGISTRY}'\/'${IMG}'/g' ${REG_OP_HOME}/config/manager/job_manager.yaml
        ```

## registry-operator 설치

1. [Step 0. 설치 파일 준비](#Step-0-설치-파일-준비)

1. [Step 1. 인증서 생성](#Step-1-인증서-생성)

1. [Step 2. config 설정](#Step-2-config-설정)

1. [Step 3. Hyperauth 계정 정보 입력](#Step-3-Hyperauth-계정-정보-입력)

1. [Step 4. install script 실행](#Step-4-install-script-실행)

1. [Step 5. 신뢰할 수 있는 인증서로 등록](#Step-5-신뢰할-수-있는-인증서로-등록)

### Step 0. 설치 파일 준비

폐쇄망 구축이 아닌 경우 아래의 명령어를 실행하여 설치 파일을 Github Repository로부터 받아 온다.

```bash
REG_OP_VERSION=v0.3.4
REG_OP_DIR=registry-operator-${REG_OP_VERSION}
mkdir ${REG_OP_DIR}
wget -c https://github.com/tmax-cloud/registry-operator/archive/${REG_OP_VERSION}.tar.gz -O - |tar -xz -C ${REG_OP_DIR} --strip-components=1
REG_OP_HOME=$(pwd)/${REG_OP_DIR}
cd ${REG_OP_HOME}
```

### Step 1. 인증서 생성

* Root CA가 있는 경우

    `/etc/kubernetes/pki/` 디렉토리에 저장되어 있는 `hypercloud-root-ca.crt`과 `hypercloud-root-ca.key` 파일이 있으면 이 인증서를 Root CA 인증서로 사용하면 된다.

    Root CA를 인증서 디렉토리(./config/pki/)로 옮긴다. (단, 인증서의 이름을 `ca.crt`와 `ca.key`로 해야한다.)

    ```bash
    sudo cp /etc/kubernetes/pki/hypercloud-root-ca.crt ./config/pki/ca.crt
    sudo cp /etc/kubernetes/pki/hypercloud-root-ca.key ./config/pki/ca.key
    sudo chmod 644 ./config/pki/ca.crt ./config/pki/ca.key
    ```

* Root CA가 없는 경우 아래의 명령어를 실행하여 인증서를 새로 생성한다.

    ```bash
    cd ${REG_OP_HOME}
    sudo chmod 755 ./config/scripts/newCertFile.sh
    ./config/scripts/newCertFile.sh
    cp ca.crt ca.key ./config/pki/
    ```

* Root CA가 Hyperauth에서 쓰이는 Root CA와 다른 경우 Hyperauth 인증서를 추가로 신뢰 필요

    Hyperauth 인증서를 인증서 디렉토리(config/pki/)로 옮긴다. (단, 인증서의 이름을 `keycloak.crt`로 해야한다.)

### Step 2. config 설정

`${REG_OP_HOME}/config/manager/manager_config.yaml` 파일에 환경변수를 설정할 수 있다.

환경변수에 대한 설명은 ${REG_OP_HOME}/docs/envs.md 를 보거나 [Github](https://github.com/tmax-cloud/registry-operator/blob/master/docs/envs.md)를 참고하여 설정한다.(Github의 경우 tag를 해당 버전으로 변경해야한다.)

* Check!!

  * hyperauth url 주소 설정: manager_config.yaml 파일에서 keycloak.service 설정
  * clair url 주소 설정: manager_config.yaml 파일에서 scanning.scanner.url 설정
  * elasticsearch url 주소 설정: manager_config.yaml 파일에서 scanning.report.url 설정
  * (`폐쇄망의 경우 필수`)폐쇄망 레지스트리 주소 설정: manager_config.yaml 파일에서 image.registry 설정
  * multi cluster의 경우: manager_config.yaml 파일에서 cluster.name 설정으로 클러스터를 구분

### Step 3. Hyperauth 계정 정보 입력

`${REG_OP_HOME}/config/manager/keycloak_secret.yaml` 파일에서 username과 password를 Hyperauth(=Keycloak)의 admin 계정을 설정한다.

기본값으로 username/password의 값이 admin/admin으로 되어 있으므로 수정이 필요한 경우 아래의 명령어로 쉽게 수정 가능

```bash
USERNAME={USERNAME} # {USERNAME}에 수정할 username 입력
PASSWORD={PASSWORD} # {PASSWORD}에 수정할 password 입력

sed -i 's/username: admin/username: '${USERNAME}'/g' ${REG_OP_HOME}/config/manager/keycloak_secret.yaml
sed -i 's/password: admin/password: '${PASSWORD}'/g' ${REG_OP_HOME}/config/manager/keycloak_secret.yaml
```

### Step 4. install script 실행

* 아래의 명령어를 실행하여 registry-operator를 설치합니다.

    ```bash
    cd ${REG_OP_HOME}
    sudo chmod 755 ./config/scripts/newCertSecret.sh install.sh
    ./install.sh
    ```

### Step 5. 신뢰할 수 있는 인증서로 등록

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

    * (**NOTE**: ssh 접속한 상태에서 아래의 컨테이너 런타임 재기동까지 수행해준다!)

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
    ./uninstall.sh -a
    ```

* registry-operator만 삭제를 원하는 경우, 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall.sh -m
    ```

* crd 리소스만 삭제를 원하는 경우, 아래의 명령어를 실행

    ```bash
    cd ${REG_OP_HOME}
    chmod 755 ./uninstall.sh
    ./uninstall.sh -c
    ```