## xNVMe를 통해 ZNS 생성
![xNVMe](https://github.com/OpenMPDK/xNVMe)는 NVMe device를 위한 크로스 플랫폼 라이브러리입니다.
리눅스 환경에서 아래 순서로 커맨드를 입력하시면 ZNS SSD를 생성하고 테스트해볼 수 있습니다.
테스트 결과는 html 파일로 생성됩니다.

이 내용은 [링크](https://github.com/OpenMPDK/xNVMe/runs/1065843776?check_suite_focus=true)
내용을 기반으로 합니다.

``` bash
docker network prune --force --filter "label=xNVMe-net"
docker network create --label xNVMe-net xNVMe-net
docker pull refenv/qemu-nvme:latest
docker create --name xnvme-test --label xnvme --workdir /__w/xNVMe/xNVMe --network xNVMe-net --privileged -e "HOME=/github/home" -e GITHUB_ACTIONS=true -e CI=true -v \
"/var/run/docker.sock":"/var/run/docker.sock" -v "/opt/ghar/_work":"/__w" -v "/opt/ghar/externals":"/__e":ro -v "/opt/ghar/_work/_temp":"/__w/_temp" -v \
"/opt/ghar/_work/_actions":"/__w/_actions" -v "/opt/ghar/_work/_tool":"/__w/_tool" -v "/opt/ghar/_work/_temp/_github_home":"/github/home" -v \
"/opt/ghar/_work/_temp/_github_workflow":"/github/workflow" --entrypoint "tail" refenv/qemu-nvme:latest "-f" "/dev/null"
docker start xnvme-test
docker exec -it xnvme-test bash
```

아래 내용은 `docker exec -it xnvme-test bash`를 하여 도커 프로세스의 bash에서
작성하는 내용에 해당합니다.

```bash
rm -rf
source /opt/scripts/suitup.sh
export RESULTS=/tmp/results
export TARGET_ENV=/usr/local/share/cijoe/envs/refenv-xnvme-qemu.sh
export CLOUD_IMG_URL=https://cloud.debian.org/images/cloud/bullseye/daily/20200807-351/debian-11-generic-amd64-daily-20200807-351.qcow2
service ssh restart
pip3 uninstall -y cijoe
pip3 install --pre cijoe
pip3 install --pre cijoe-pkg-xnvme
mkdir -p ${RESULTS}/base
mkdir -p ${RESULTS}/lblk
mkdir -p ${RESULTS}/zoned
source /opt/scripts/suitup.sh
qemu::img_from_url "${CLOUD_IMG_URL}"
source /opt/scripts/suitup.sh
source "${CIJ_TESTFILES}/qemu_setup_nvme_1c2ns_nvm_zns.sh"
qemu_setup_pcie
QEMU_ARGS_EXTRA="$QEMU_SETUP_PCIE"
qemu::run
source /opt/scripts/suitup.sh
wget https://github.com/OpenMPDK/xNVMe/releases/download/v0.0.19/xnvme-0.0.19.bin-dev.deb
wget https://github.com/OpenMPDK/xNVMe/releases/download/v0.0.19/xnvme-0.0.19.bin-examples.deb
wget https://github.com/OpenMPDK/xNVMe/releases/download/v0.0.19/xnvme-0.0.19.bin-tests.deb
wget https://github.com/OpenMPDK/xNVMe/releases/download/v0.0.19/xnvme-0.0.19.bin-tools.deb
for deb in *.deb; do
  ssh::push "$deb" "/tmp/"
  done
ssh::cmd "dpkg -i /tmp/*.deb"
source /opt/scripts/suitup.sh
git clone https://github.com/OpenMPDK/xNVMe.git --recursive
cd /__w/xNVMe/xNVMe/xNVMe
apt install -y cmake meson libcunit1-dev libnuma-dev libssl-dev libncurses5-dev 
make
ssh::push "/__w/xNVMe/xNVMe/xNVMe/third-party/fio/repos/fio" "/opt/aux/"
cij_runner $CIJ_TESTPLANS/xnvme-base-linux.plan ${TARGET_ENV} --output ${RESULTS}/base
cij_runner $CIJ_TESTPLANS/xnvme-lblk-linux.plan ${TARGET_ENV} --output ${RESULTS}/lblk
cij_runner $CIJ_TESTPLANS/xnvme-zoned-linux.plan ${TARGET_ENV} --output ${RESULTS}/zoned
source /opt/scripts/suitup.sh
qemu::kill
source /opt/scripts/suitup.sh
cij_reporter ${RESULTS}/base
cij_reporter ${RESULTS}/lblk
cij_reporter ${RESULTS}/zoned
```

아래 내용은 도커를 빠져 나와서 수행하는 내용입니다.

```bash
docker cp xnvme-test:/tmp/results/base/report.html ./base_report.html
docker cp xnvme-test:/tmp/results/lblk/report.html ./lblk_report.html
docker cp xnvme-test:/tmp/results/zoned/report.html ./zoned_report.html

docker rm --force xnvme-test
docker network rm xnvme_test_net
```

### 추가 내용

#### Qemu의 접속

ZNS 디스크를 사용하기 위해서는 도커 프로세스에서 구동 중인
Qemu에 들어가야 합니다.

따라서, 다음의 명령을 통해 접속을 진행하도록 합니다.

```bash
# 안되는 경우에는 반드시 export를 다시 해주십시오.
# export TARGET_ENV=/usr/local/share/cijoe/envs/refenv-xnvme-qemu.sh
ssh $QEMU_HOST -p $QEMU_GUEST_SSH_FWD_PORT
```

#### 자원 할당 내용의 변경

Qemu의 하드웨어 리소스 할당 내역을 변경하고 싶은 경우에는
`/usr/local/share/cijoe/envs/refenv-xnvme-qemu.sh`의 내용을 확인하고
변경하고자 하는 값에 해당하는 환경 변수 값을 수정해줍니다.

그리고 Qemu의 크기가 너무 작아서 작업을 할 수 없는 경우에는
`/opt/guests/emujoe/`로 이동하여 아래의 명령을 수행해주도록 합니다.

```bash
qemu-img resize boot.img +<변경하고자 하는 크기>G
```

#### 재실행을 하고 싶은 경우

Qemu를 재실행을 하고 싶은 경우에는 아래 절차를 따라야 합니다.

```bash
# 삭제 과정
source /opt/scripts/suitup.sh
qemu::kill

# 환경 변수의 설정
export TARGET_ENV=/usr/local/share/cijoe/envs/refenv-xnvme-qemu.sh

# 실행 과정
source /opt/scripts/suitup.sh
source "${CIJ_TESTFILES}/qemu_setup_nvme_1c2ns_nvm_zns.sh"
qemu_setup_pcie
QEMU_ARGS_EXTRA="$QEMU_SETUP_PCIE"
qemu::run
```

