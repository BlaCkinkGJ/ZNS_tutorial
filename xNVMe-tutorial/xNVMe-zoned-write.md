# xNVMe를 통한 쓰기

[링크](../ZNS-emulate/ZNS-emulate-xNVMe.md)의 내용 중에 `qemu::kill` 전까지
모든 과정을 수행했다고 가정하겠습니다.

일단, 도커의 안에서 `ssh localhost:2222`를 수행하여 qemu에 들어가도록 합니다.
qemu에 들어가서 `fdisk`를 실행하면 아래와 같은 내용과 유사한 내용이 나와야 합니다.

```bash
Disk /dev/vda: 2 GiB, 2147483648 bytes, 4194304 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: ABAEAB5F-EF19-B446-B1EE-75344B88CA4C

Device      Start     End Sectors  Size Type
/dev/vda1  262144 4194270 3932127  1.9G Linux filesystem
/dev/vda14   2048    8191    6144    3M BIOS boot
/dev/vda15   8192  262143  253952  124M EFI System

Partition table entries are not in disk order.


Disk /dev/nvme0n1: 8 GiB, 8589934592 bytes, 2097152 sectors
Disk model: QEMU NVMe Ctrl
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes


Disk /dev/nvme0n2: 8 GiB, 8589934592 bytes, 2097152 sectors
Disk model: QEMU NVMe Ctrl
Units: sectors of 1 * 4096 = 4096 bytes
Sector size (logical/physical): 4096 bytes / 4096 bytes
I/O size (minimum/optimal): 4096 bytes / 4096 bytes
```

이렇게 된 상태에서 아래의 명령을 수행하여 정상적으로
ZNS가 구축이 되어있는 지를 확인해야 합니다.

일단은 `/dev/nvme0n1`, `/dev/nvme0n2`에 대해서 `zoned info`를 통해서
`type`이 `XNVME_GEO_ZONED`인 디바이스를 찾도록 합니다.

이를테면, 아래와 같은 명령을 치도록 하겠습니다.

```bash
zoned info /dev/nvme0n2
```

그러면 아래와 같은 결과가 나오게 됩니다.

```bash
xnvme_dev:
  xnvme_ident:
    trgt: '/dev/nvme0n2'
    schm: 'file'
    opts: ''
    uri: 'file:/dev/nvme0n2'
  xnvme_cmd_opts:
    mask: '00000000000000000000000000000001'
    iomd: 'SYNC'
    payload_data: 'DRV'
    payload_meta: 'DRV'
    csi: 0x2
    nsid: 0x2
    ssw: 12
  xnvme_geo:
    type: XNVME_GEO_ZONED
    npugrp: 1
    npunit: 1
    nzone: 512
    nsect: 4096
    nbytes: 4096
    nbytes_oob: 0
    tbytes: 8589934592
    mdts_nbytes: 520192
    lba_nbytes: 4096
    lba_extended: 0
```

만약, ZNS 장치가 맞다면 `zoned report /dev/nvme0n2`를 통해서
zone의 상태를 확인해주도록 합니다.
