# registry-operator 설치 가이드

## 구성요소

* Git Repoistory
    * https://github.com/tmax-cloud/registry-operator
* Container Images
    * [tmaxcloudck/registry-operator](https://hub.docker.com/r/tmaxcloudck/registry-operator) v0.3.5
    * [tmaxcloudck/registry-job-operator](https://hub.docker.com/r/tmaxcloudck/registry-job-operator) v0.3.5

## Prerequisites
1. Nginx Ingress Controller 설치
    * [설치 가이드](https://github.com/tmax-cloud/install-ingress/tree/5.0)
2. Hyperauth 설치
    * [설치 가이드](https://github.com/tmax-cloud/install-hyperauth/tree/5.0)
    * 설치시 hyperauth Deployment에 args 필드 내용 추가 필요
        * spec.template.spec.containers.args: ["-Dkeycloak.profile.feature.docker=enabled", "-b", "0.0.0.0"]
            * 설치가이드에서 [`install-hyperauth/manifest/2.hyperauth_deployment.yaml`](https://github.com/tmax-cloud/install-hyperauth/blob/5.0/manifest/2.hyperauth_deployment.yaml#L32) 파일 내용 확인
3. Clair 설치
    * [설치 가이드](https://github.com/tmax-cloud/install-clair/tree/5.0)

## Related Add-ons
1. image-validating-webhook
    * [Github](https://github.com/tmax-cloud/image-validating-webhook)
    * [설치 가이드](https://github.com/tmax-cloud/install-image-validating-webhook/tree/5.0)

1. Elastic Search
    * [설치 가이드](https://github.com/tmax-cloud/install-EFK/tree/5.0#step-1-elasticsearch-%EC%84%A4%EC%B9%98)

## registry-operator 폐쇄망 구축 가이드

1. 폐쇄망에서 사용할 이미지 파일과 오퍼레이터 설치 파일 준비
    
    1.1 아래 이미지 목록을 파일로 저장 ([참조](https://github.com/tmax-cloud/install-registry/blob/5.0/podman.md#%EC%9D%B4%EB%AF%B8%EC%A7%80-%ED%91%B8%EC%8B%9C%ED%95%98%EA%B8%B0))
      * registry:2.7.1                                         
      * tmaxcloudck/notary_server:0.6.2-rc1                    
      * tmaxcloudck/notary_signer0.6.2-rc1                     
      * tmaxcloudck/notary_mysql:0.6.2-rc2                     
      * tmaxcloudck/registry-operator:v0.3.5    
      * tmaxcloudck/registry-job-operator:v0.3.5

    1.2 설치 파일 다운로드
      ```bash
      wget -O registry-operator.tar.gz https://github.com/tmax-cloud/registry-operator/archive/v0.3.5.tar.gz
      ```

2. 폐쇄망 환경으로 파일복사 및 설치 준비

    2.1 1.에서 준비한 registry-operator.tar.gz와 이미지 압축파일들을 복사

    2.2 복사한 이미지들을 로컬에 로딩 후 대상 레지스트리에 푸시 ([참조](https://github.com/tmax-cloud/install-registry/blob/5.0/podman.md#%EC%9D%B4%EB%AF%B8%EC%A7%80-%ED%91%B8%EC%8B%9C%ED%95%98%EA%B8%B0))

    2.3 registry-operator.tar.gz 압축해제 및 manager.yaml, job_manager.yaml의 이미지 경로 수정
    ```bash
        tar -xzf registry-operator.tar.gz
        cd registry-operator-0.3.4 
        
        # PodSpec의 image 경로를 올바르게(2.2에서 푸시한 이미지를 가리키도록) 수정
        vi config/manager/manager.yaml 
        # PodSpec의 image 경로를 올바르게(2.2에서 푸시한 이미지를 가리키도록) 수정
        vi config/manager/job_manager.yaml
    ```

## registry-operator 설치
1. [Step 0. 설치 파일 준비](#Step-0-설치-파일-준비)
2. [Step 1. 인증서 생성](#Step-1-인증서-생성)
3. [Step 2. config 설정](#Step-2-config-설정)
4. [Step 3. Hyperauth 계정 정보 입력](#Step-3-Hyperauth-계정-정보-입력)
5. [Step 4. install script 실행](#Step-4-install-script-실행)
6. [Step 5. 신뢰할 수 있는 인증서로 등록](#Step-5-신뢰할-수-있는-인증서로-등록)

### Step 0. 설치 파일 준비
폐쇄망 구축이 아닌 경우 아래의 명령어를 실행하여 설치 파일을 Github Repository로부터 받아 온다.

```bash
REG_OP_VERSION=v0.3.5
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
