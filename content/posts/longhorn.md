+++
date = '2025-05-07T20:52:24+09:00'
draft = false
title = 'longhorn 구조 정리'
+++

# Kubernetes Longhorn 스토리지 구조와 운영 가이드

## 1. Longhorn 내부 구조

**Longhorn 아키텍처 개요:** Longhorn은 Kubernetes 상에서 동작하는 **분산 블록 스토리지 시스템**으로, 모든 제어 메타데이터를 Kubernetes Custom Resource로 관리합니다. Longhorn은 각 볼륨마다 독립적인 스토리지 컨트롤러(엔진)를 생성하고, 다수 노드에 걸쳐 \*\*복제본(Replica)\*\*들을 동기 복제하여 높은 가용성을 제공합니다. 이러한 Longhorn의 컨트롤 플레인은 `longhorn-manager`라는 DaemonSet으로 구성되며, Kubernetes API (etcd)를 통해 CRD 자원으로 상태를 저장/조회합니다. Longhorn의 주요 컴포넌트와 데이터 흐름 관계를 아래 그림에 나타냈습니다.

&#x20;**그림 1: Longhorn 볼륨 엔진(Controller)과 다수 Replica의 관계** – 각 애플리케이션 Pod가 사용하는 Longhorn 볼륨마다 전용 **엔진(스토리지 컨트롤러)** 프로세스가 생성되고, 엔진은 여러 노드에 분산된 **Replica**들과 통신하며 블록 장치를 제공합니다. 모든 **쓰기** 요청은 \*\*엔진(Controller)\*\*을 통해 모든 Replica에 동기적으로 복제되고, **읽기** 요청은 정상 Replica 중 하나에서 수행됩니다. 하단의 Node1, Node2는 실제 물리 노드와 해당 디스크(SSD) 자원을 표현한 것입니다.

Longhorn은 설계상 **마이크로서비스**로 구성되어 있으며, Kubernetes에서 동작하는 여러 컨트롤러들에 의해 관리됩니다. **Longhorn Manager** (각 노드에 하나씩 실행되는 `longhorn-manager` Pod) 가 전체 제어를 담당하며, **Kubernetes CRDs**를 통해 상태를 유지합니다. 주요 CRD와 그 역할은 다음과 같습니다.

* **Volume CRD**: Longhorn **볼륨** 객체를 나타내며, 각 볼륨의 원하는 상태(spec)와 현재 상태(status)를 저장합니다. **Volume Controller**는 Volume CRD를 감시하며 **볼륨 생성/삭제**, **Attach/Detach**, **엔진 업그레이드** 등을 수행합니다. 이러한 작업은 **Engine/Replica 객체를 생성/삭제/갱신**하는 방식으로 이루어집니다.

* **Engine CRD**: 볼륨당 하나씩 존재하는 **엔진(스토리지 컨트롤러)** 인스턴스를 나타냅니다. **Engine Controller**는 엔진 프로세스의 시작/중지와 상태 모니터링을 담당합니다. 엔진은 연결된 Replica들의 상태를 파악하여 볼륨의 종합적인 건강 상태(예: `healthy`, `degraded`, `faulted`)를 보고합니다.

* **Replica CRD**: 각 **복제본(Replica)** 인스턴스를 나타냅니다. **Replica Controller**는 복제본 프로세스의 시작/중지와 라이프사이클을 관리합니다. 하나의 Longhorn 볼륨은 다수 Replica를 가질 수 있으며(기본 3개 등), 각 Replica는 다른 노드의 디스크 상에 **볼륨 데이터의 전체 사본**을 보유합니다. 따라서 **단 한 개의 Replica만 살아남아도 해당 데이터로 볼륨을 복구할 수 있습니다**.

* **Instance Manager CRD**: Longhorn에서 엔진/복제본 프로세스를 실제로 실행하는 **인스턴스 관리자**를 나타냅니다. Longhorn은 하나의 별도 Pod 안에서 다수의 엔진 또는 복제본 프로세스를 관리하는데, 이 때 Instance Manager가 사용됩니다. 노드별로 **엔진 전용(instance-manager-e-*)\*\*과 \*\*복제본 전용(instance-manager-r-*)** 두 종류의 Instance Manager Pod가 실행되며, 실제 엔진/복제본 프로세스들은 해당 Pod 내에서 독립 프로세스로 가동됩니다. Instance Manager CRD의 status에는 해당 노드에서 실행 중인 엔진/복제본 프로세스들의 목록과 상태가 실시간으로 갱신됩니다.

* **Node CRD**: Longhorn 클러스터 내 **각 노드**를 나타내며, 노드의 디스크 정보와 스케줄 가능 여부, 태그 등을 저장합니다. **Node Controller**는 노드 및 디스크 상태(예: 남은 용량, 상태)를 수집하고, 스케줄링 관련 플래그나 장애 여부를 표시합니다. (예: 디스크 가득 참, 노드 오프라인 등).

* **Engine Image CRD**: Longhorn **엔진 바이너리 이미지**(버전)를 나타냅니다. Longhorn 엔진은 버전에 따라 호환성이 다를 수 있으므로, 클러스터에 설치된 Longhorn Manager의 버전과 무관하게 특정 볼륨은 이전 버전 엔진으로 동작할 수 있습니다. Engine Image CRD는 해당 엔진 버전의 바이너리를 각 노드에 배포하고(\**engine-image-* Pod\*\*로 표시), 필요한 경우 새로운 Instance Manager를 생성하여 해당 버전의 엔진/복제본 프로세스를 실행할 수 있게 합니다. (Longhorn 업그레이드 시 구 버전 엔진과 신 버전 엔진이 동시에 존재할 수 있는 이유입니다.)

* **Share Manager CRD**: RWX(Longhorn ReadWriteMany) 볼륨을 위해 추가된 **NFS 공유 관리** 객체입니다. RWX 볼륨 사용 시 Longhorn은 내부적으로 NFS 서버 Pod인 \*\*share-manager-<볼륨명>\*\*를 생성하는데, 이 CRD가 해당 share-manager의 생명주기를 관리합니다. (Share Manager의 자세한 동작은 아래 RWX 섹션에서 설명)

이 외에도 백업(Backup), 스냅샷(Snapshot), 설정(Settings) 등 다양한 CRD가 존재하지만, 핵심은 위와 같습니다. 모든 Longhorn CRD 객체는 Kubernetes etcd에 저장되며, 각 객체의 `spec`(Desired State)과 `status`(Observed State)를 Longhorn Manager가 **컨트롤러 패턴**으로 지속적으로 조정합니다. 이러한 Operator 모델을 통해 사용자/CSI의 요청이 들어오면 CR의 spec을 변경하고, Longhorn Manager의 컨트롤러들이 실제 리소스를 생성/삭제/조작하여 상태를 맞추는 방식으로 동작합니다. 예를 들어 **볼륨 Attach** 동작은 Volume CR의 `spec.nodeID` 필드를 설정함으로써 시작되고, **Snapshot 생성**은 실행 중인 엔진 프로세스에 API 호출을 통해 수행되며 결과가 CR에 반영됩니다.

**Longhorn Manager와 CSI Driver:** Longhorn은 Kubernetes와 연동하기 위해 **CSI (Container Storage Interface) Driver**를 사용합니다. Longhorn CSI 플러그인은 배포 시 자동으로 설치되며 (`longhorn-csi-plugin` DaemonSet 등으로 각 노드에 설치) Kubernetes의 볼륨 작업 요청을 Longhorn 시스템에 전달합니다. Longhorn CSI 드라이버는 **Volume의 생성/삭제, Attach/Detach, Mount/Unmount, Snapshot 등**의 기능을 제공하며, 내부적으로 Longhorn Manager의 REST API를 호출하거나 필요한 호스트 작업을 수행합니다. 예를 들어 **CSI NodePublishVolume** 단계에서 Longhorn CSI 드라이버는 해당 볼륨의 블록 디바이스(`/dev/longhorn/<vol>`)를 인식한 뒤 \*\*파일시스템을 포맷(ext4 등)\*\*하고 노드 경로에 마운트합니다. 이후 kubelet이 그 마운트를 파드에 바인드하여 최종적으로 Pod가 볼륨에 접근할 수 있게 됩니다.

Longhorn CSI는 **driver.longhorn.io** 라는 Provisioner 이름으로 동작하며, StorageClass에 이 Provisioner를 지정하면 PVC 생성 시 Longhorn 볼륨이 동적으로 할당됩니다. 또한 **CSI Attach** 과정에서 kubelet은 Longhorn CSI를 통해 **iscsiadm 등 호스트 툴을 호출**하여 Longhorn 엔진이 제공하는 iSCSI Target에 연결하고 디바이스(`/dev/longhorn/volname`)를 생성하는 작업도 수행합니다. (Longhorn v1 데이터 엔진은 내부적으로 iSCSI를 사용하므로 노드에 `open-iscsi/iscsiadm` 패키지가 필요합니다.)

**노드별 구성요소:** 각 노드에는 Longhorn Manager 외에도 여러 구성요소 Pod가 존재합니다:

* `longhorn-manager` Pod: 앞서 설명한 Longhorn Manager로, DaemonSet으로 모든 노드에서 실행됩니다. 각 매니저는 자기 노드의 Longhorn 구성요소를 관리하고, CRD 변경을 감지해 작업을 수행하며, 서로 간에 gRPC 통신으로 조율합니다.

* `instance-manager-r` Pod: Replica용 인스턴스 관리자 Pod로, 해당 노드에서 실행되는 모든 Replica 프로세스를 통합하여 실행합니다. 하나의 instance-manager Pod 내에서 여러 Replica 프로세스가 컨테이너가 아닌 **프로세스** 형태로 동작하며, gRPC로 Manager와 통신합니다.

* `instance-manager-e` Pod: Engine(컨트롤러)용 인스턴스 관리자 Pod로, 이 노드에서 실행되는 모든 Engine 프로세스를 실행/관리합니다. (Longhorn 1.5부터는 instance-manager-r/e 통합 논의도 있으나 기본적으로 분리되어 있음)

* `engine-image` Pod: Longhorn Engine 바이너리를 담은 Pod입니다. 각 Longhorn 버전은 대응되는 엔진 이미지를 가지며, 엔진 이미지 CRD에 따라 DaemonSet으로 노드마다 배포됩니다. 엔진 이미지 Pod는 해당 바이너리를 컨테이너 레이어에 가지고 있을 뿐 특별한 데몬 역할은 하지 않으며, 주로 **엔진/인스턴스 매니저 프로세스 실행에 필요한 바이너리 제공 및 버전 호환성 표시** 역할을 합니다.

* `longhorn-csi-plugin` Pod: CSI Driver의 Node 플러그인으로, DaemonSet으로 각 노드에 배포됩니다. kubelet이 CSI 호출을 할 때 이 플러그인이 응답하며 볼륨 attach/mount 작업을 처리합니다.

* (필요시) `share-manager-<vol>` Pod: RWX 볼륨이 사용 중인 경우 해당 볼륨을 export하는 NFS 서버 Pod가 생성됩니다. RWX 관련 구성은 뒤에서 다룹니다.

이러한 구성요소들은 **Kubernetes API Server와 etcd**를 통해 느슨하게 연계됩니다. Longhorn Manager는 필요한 시점에 Kubernetes에 새로운 Pod (예: share-manager, instance-manager 등)을 생성하며, 상태 정보는 CRD로 etcd에 저장됩니다. 결과적으로 Longhorn은 etcd를 단일 소스로 사용하여 메타데이터 일관성을 유지하고, **데이터 평면**은 각 노드의 프로세스들이 네트워크로 협동하여 구현하는 구조입니다.

## 2. 볼륨 생성부터 연결, 삭제까지의 흐름

이 섹션에서는 **PersistentVolumeClaim(PVC)** 생성부터 실제 Pod에 마운트되고, 삭제되는 전체 과정을 단계별로 설명합니다. 또한 각 단계에서 Kubernetes와 Longhorn 내부에서 어떤 일이 일어나는지, 그리고 리눅스 노드 상에서의 디바이스/마운트 변화도 함께 살펴봅니다.

### 2-1. PVC 생성 및 Volume 프로비저닝

1. **PVC 및 StorageClass 생성:** 사용자는 Longhorn 스토리지 클래스가 지정된 PVC를 생성합니다. 예를 들어 Longhorn 기본 StorageClass(`provisioner: driver.longhorn.io`)를 사용하여 5Gi 용량의 PVC를 생성하면, Kubernetes **동적 프로비저닝**에 의해 Longhorn CSI 드라이버가 호출됩니다. StorageClass에 미리 정의된 매개변수(복제본 개수 등)에 따라 Longhorn **볼륨 생성 요청**이 이루어집니다.

2. **Volume CR 생성:** Longhorn CSI **CreateVolume** 호출을 받은 Longhorn Manager는 새로운 **Volume CR 객체**(`volumes.longhorn.io`)를 생성합니다. Volume CR에는 볼륨 이름(보통 PVC의 UID), 크기, 복제본 개수 등의 spec이 기록됩니다. Kubernetes는 PVC와 자동으로 바인딩된 **PV 객체**도 생성하는데, 이 PV는 VolumeHandle로 Longhorn 볼륨의 이름을 참조하고 `driver: driver.longhorn.io` 로 설정됩니다. 결과적으로 사용자의 PVC는 **Longhorn Volume**에 매핑되어 바인딩됩니다.

3. **Replica 스케줄링 및 생성:** Volume CR의 컨트롤러가 동작하면서, Longhorn는 해당 볼륨의 **복제본(Replica)들을 생성**합니다. 볼륨 spec에 지정된 복제본 수(기본 3개 등)에 따라, 사용 가능한 노드들 중 서로 다른 노드에 복제본을 배치합니다. 이 때 Node CR의 디스크 용량, 태그, 스케줄 가능 여부를 고려하여 자동 스케줄링되며, 각 선택된 노드에 **Replica CR 객체**가 생성됩니다. Replica CR에는 어떤 볼륨의 복제본인지와 데이터 경로 등이 명시되고, 해당 노드의 Replica Controller에 의해 실제 **복제본 프로세스**(또는 데이터 파일)가 준비됩니다. (볼륨이 아직 Attach되지 않은 상태에서는 실제 프로세스 대신 **대기 상태**로 Replica 정보만 생성해둡니다. 실제 데이터 파일은 일반적으로 해당 노드의 `/var/lib/longhorn/replicas/<볼륨명>-<replicaID>` 경로 아래에 생성됩니다. 이 파일이 **가상 디스크 이미지**로서 할당된 용량 내에서 **Thin Provisioning** 됩니다.)

4. **엔진(Controller) 준비:** Longhorn Manager는 Volume CR 생성 시 **Engine CR 객체**도 생성합니다. 다만 엔진은 \*\*어느 노드에 붙을지 (spec.nodeID)\*\*가 정해지지 않았으므로, 이 시점에서는 엔진 프로세스가 시작되지 않고 **대기 상태**입니다. Volume은 현재 “Detached” (어태치 안 됨) 상태로 존재하며, Longhorn UI나 `kubectl get volumes.longhorn.io`로 조회하면 `state: detached`, `robustness: healthy` (데이터는 문제없으나 attach되지 않음) 등으로 나타납니다.

   * 참고: 볼륨 생성 직후 Longhorn UI나 CR 정보를 보면 **Scheduled** 필드가 `True`로 표시되는데, 이는 요청한 복제본 수만큼 스케줄링이 완료되었음을 의미합니다. 아직 Attach 전이므로 실제 엔진/복제본 프로세스는 비활성 상태이며, Volume CR의 `spec.nodeID`도 비어 있습니다.

### 2-2. 볼륨 Attach 및 Pod 마운트

5. **Pod 스케줄링 및 Attach 요청:** PVC를 사용하는 Pod이 스케줄되면, 해당 Pod가 예약된 노드에서 구동되기 전에 **AttachVolume** 단계가 진행됩니다. Kubernetes는 스케줄된 노드 정보를 토대로 Longhorn CSI 드라이버의 **ControllerPublishVolume** (Attach) 함수를 호출하여 볼륨을 해당 노드에 연결하도록 요청합니다. Longhorn Manager는 이를 받아들여 **Volume CR의 `spec.nodeID`를 대상 노드로 설정**합니다. 이 순간 Volume CR의 desired state가 “해당 노드에 연결”로 바뀌며, Volume Controller는 Attach 절차를 시작합니다.

6. **엔진 프로세스 시작:** Volume Controller는 Engine CR을 업데이트하여 엔진을 실행시킵니다. 즉, Engine CR의 spec에 nodeID가 설정되고, Longhorn은 해당 노드의 **instance-manager-e Pod**에 엔진 프로세스 생성을 지시합니다. Instance Manager는 **엔진 바이너리**(해당 볼륨에 대응되는 버전)를 사용해 새로운 엔진 프로세스를 컨테이너 내에서 실행합니다. 이 엔진 프로세스는 곧바로 self-check를 거쳐 자신의 상태를 Engine CR의 status로 보고하기 시작합니다.

7. **복제본 프로세스 시작 및 연결:** Volume Controller는 볼륨에 속한 모든 Replica CR들을 확인하여, 이들이 **엔진에 연결되도록** 준비합니다. 일반적으로 Attach 이전에는 Replica 프로세스가 비활성이므로, 각 해당 노드의 **instance-manager-r Pod**에 복제본 프로세스를 시작하라는 명령이 내려집니다. Replica 프로세스는 자신이 참조할 데이터 파일(이미 존재함)을 오픈하고 대기합니다. 이제 **엔진 프로세스**는 Volume CR의 정보 및 Longhorn Manager와의 통신을 통해 어떤 Replica들이 있어야 하는지 인지하고, 각 Replica에 순차적으로 접속을 시도합니다. 엔진은 TCP (v1 엔진의 경우 gRPC + iscsi 연결)로 각 Replica 프로세스와 연결을 맺고, **모든 Replica가 연결되면 볼륨이 RW 가능**한 상태가 됩니다. 이 순간 Longhorn UI에서는 해당 Volume이 `state: attached` (어떤 노드에 attach됨) 및 `robustness: healthy`로 표시됩니다. 만약 일부 Replica가 오프라인이면 `degraded` (강등) 상태로 표시되며, Longhorn은 자동으로 부족한 Replica를 재빌드하여 healthy 상태로 복구를 시도합니다.

8. **호스트에 디바이스 생성 (iSCSI 연결):** 엔진 프로세스는 **frontend**로 기본 블록디바이스 인터페이스를 사용합니다. Longhorn v1 엔진에서는 이를 위해 **iSCSI 타겟**을 생성하여 호스트에 노출하는 방식을 취합니다. 엔진 프로세스 내부에는 변경된 tgtd(iSCSI target daemon)가 포함되어 있고, 엔진은 localhost의 특정 포트에 iSCSI Target을 개방합니다. Longhorn CSI의 Attach 로직은 kubelet을 통해 **호스트에서 해당 iSCSI Target에 연결**하도록 `iscsiadm` 명령을 실행합니다. 그 결과, 해당 노드의 커널에 새로운 iSCSI 디스크가 생성되고 `/dev/sd*` 와 같은 경로로 인식됩니다. Longhorn은 udev 규칙 등을 통해 이 디스크를 **`/dev/longhorn/<볼륨명>` 경로로 연결**해줍니다 (여러 볼륨을 쉽게 구분하기 위함). 예를 들어 볼륨 `pvc-5242...`가 attach되면 노드의 `/dev/longhorn/pvc-5242...` 가 블록 장치로 나타납니다. 아래는 실제 Longhorn 볼륨이 마운트되기 전, 호스트에서 확인한 장치 파일 예시입니다.

   ```bash
   # 노드 (r8s-worker1)에서 Longhorn 볼륨 attach 후 장치 확인
   $ ls -l /dev/longhorn/
   total 0
   brw-rw---- 1 root root 8, 0 Oct 15 07:53 pvc-5242ccd4-5a41-4bb4-b5f8-6f64118f4d33:contentReference[oaicite:48]{index=48}

   $ sudo lsblk -o NAME,MAJ:MIN,RO,SIZE,TYPE,MOUNTPOINT | grep '8, *0'
   sda    8:0    0   5G  disk               # Longhorn 볼륨 (5Gi) 장치
   └─sda1 8:1    0   5G  part /var/lib/longhorn/… (마운트 전일 경우 비어 있음)
   ```

   위 예시에서 볼 수 있듯이 `/dev/longhorn/pvc-...` 장치는 메이저:마이너 번호 `8,0`으로 표시되며(일반 SCSI 디스크로 인식), 실제 디바이스 이름 `/dev/sda`로 매핑되어 있습니다. Longhorn이 해당 경로를 `/dev/longhorn/` 밑으로 제공하여 사용자는 직관적으로 장치를 식별할 수 있습니다.

9. **파일시스템 포맷 및 Pod에 마운트:** PVC 생성 시 별도로 PV의 `volumeMode`를 지정하지 않았다면 기본적으로 **Filesystem 모드**로 동작합니다. 따라서 첫 사용시 Longhorn CSI **NodePublishVolume** 단계에서 `mkfs.ext4` 등의 포맷이 실행됩니다. Longhorn StorageClass의 매개변수로 `fsType: ext4`가 지정되어 있으면 ext4로 포맷하며 (기본값), 이후 **mount** 명령으로 해당 블록디바이스를 kubelet의 Pod 마운트 경로 (예: `/var/lib/kubelet/pods/<podUID>/volumes/kubernetes.io~csi/<pvcUUID>/mount`)에 마운트합니다. 이 작업이 완료되면 kubelet은 해당 마운트된 디렉토리를 컨테이너 내부에 `/mnt` (사용자가 PVC를 마운트한 경로)로 bind mount하여 최종적으로 Pod 내에서 볼륨을 사용할 수 있게 됩니다.

   * **Mount 확인:** 볼륨이 마운트되면 호스트 노드에서 `mount | grep longhorn` 등을 통해 마운트 정보를 확인할 수 있습니다. 또한 `df -h` 명령을 통해 용량과 사용량을 볼 수 있습니다:

     ```bash
     $ mount | grep pvc-5242ccd4
     /dev/longhorn/pvc-5242... on /var/lib/kubelet/pods/.../pvc-5242.../mount type ext4 (rw,relatime)

     $ df -h | grep pvc-5242ccd4
     /dev/longhorn/pvc-5242...   5.0G   33M  5.0G   1% /var/lib/kubelet/pods/.../mount
     ```

   * **데이터 확인:** 예를 들어 Pod에서 `/mnt` 경로로 마운트했다면, 해당 노드에서 `/var/lib/kubelet/pods/.../mount` 경로를 통해 데이터가 보입니다. 앞서 longhorn 볼륨에 데이터를 채운 뒤 노드에서 이를 확인하면 다음과 같습니다:

     ```bash
     $ sudo ls -l /var/lib/kubelet/pods/.../mount
     total 40
     drwxr-xr-x 2  911  911  4096 Oct 14 14:09 keys
     drwxr-xr-x 4  911  911  4096 Oct 14 14:09 log
     drwx------ 2 root root 16384 Oct 14 14:09 lost+found
     ... (생략) ...
     drwxr-xr-x 2  911  911  4096 Oct 14 14:09 www
     ```

   위 예시는 블로그의 Longhorn 볼륨 내용을 노드에서 직접 마운트하여 확인한 것으로, `lost+found`가 생성되어 있고 나머지는 애플리케이션이 기록한 폴더들입니다.

> **Note:** Longhorn 볼륨은 **Block Device 모드**로도 사용할 수 있습니다. 이 경우 PVC의 `volumeMode: Block`으로 설정하면, Longhorn CSI는 포맷/마운트를 하지 않고 `/dev/longhorn/vol` 장치를 그대로 컨테이너에 전달합니다. 컨테이너는 raw 블록 장치(`/dev/xvd...` 형태)로 PVC가 제공되며, 애플리케이션에서 직접 파일시스템을 만들거나 raw 디바이스로 사용합니다.

### 2-3. 볼륨 사용 중 동작 및 삭제

10. **데이터 쓰기/복제 동작:** Pod 내 애플리케이션이 볼륨에 데이터를 쓰면, 해당 I/O는 호스트를 거쳐 Longhorn 엔진에 전달됩니다. 엔진(컨트롤러)은 **쓰기 작업을 모두 연결된 Replica에 동기적으로 전달**하여 **모든 Replica에 데이터가 기록되면 응답**합니다. 하나의 Replica 쓰기가 실패하면 엔진은 그 Replica를 **faulted** 처리하고 제외한 뒤 나머지 Replica로 계속 운영합니다 (이 때 Volume 상태는 `degraded`로 바뀜). **읽기 작업**의 경우 엔진은 연결된 건강한 Replica 중 하나에서 데이터를 읽어옵니다. 일반적으로 가장 최신으로 sync된 Replica 또는 첫 번째 Replica가 선택되지만, 네트워크 지연 등을 고려하여 최적의 Replica를 고를 수도 있습니다.

    Longhorn 엔진은 **스냅샷** 기반으로 복제본을 관리하여, 실시간으로 모든 Replica를 동일한 스냅샷 시점으로 유지합니다. 새로운 데이터는 마지막 스냅샷 이후 영역에 기록되며, 추후 Snapshot 명령 시 메타데이터를 생성합니다. 이러한 내부 메커니즘 덕분에 Replica 프로세스가 재시작되거나 새로운 Replica가 추가될 때 **증분 동기화**가 가능합니다.

11. **볼륨 삭제 (PVC 삭제):** 사용자가 PVC를 삭제하면, Kubernetes는 Longhorn CSI 드라이버의 **DeleteVolume** 함수를 호출합니다. Longhorn은 우선 해당 볼륨이 **Attached 상태이면 자동으로 Detach**(분리)를 시도합니다. Pod이 삭제되었거나 더 이상 사용 중이 아니므로, Longhorn Manager는 Volume CR의 `spec.nodeID`를 비우고 해당 엔진 프로세스를 종료시킵니다. 연결되었던 Replica 프로세스들도 순차적으로 종료됩니다. Volume CR의 상태가 `detached`로 바뀌면 Longhorn CSI는 PV와 Volume 데이터를 삭제합니다. Longhorn Manager는 Volume CR, Engine CR, Replica CR 등 관련 CR들을 etcd에서 삭제하고, 백엔드 디스크에 남은 데이터 파일(.img 및 메타데이터)을 제거합니다. 최종적으로 PVC/PV 리소스도 Kubernetes에서 삭제되어, 볼륨 삭제가 완료됩니다.

    * 만약 **삭제 시점에 볼륨이 `attached` 상태로 응답이 지연된다면**, Longhorn은 강제로 detach 후 삭제하거나, 오류를 리턴할 수 있습니다. 예를 들어 볼륨이 사용 중이라 `NodePublishVolume`가 해제되지 않은 경우 삭제가 실패할 수 있습니다. 이 때는 해당 Pod를 완전히 제거하고 다시 PVC를 삭제하면 됩니다.
    * **삭제되지 않은 리소스 정리:** 드물게 Longhorn CRD들이 삭제되었는데도 관련 Pod나 디렉토리가 남아있는 경우, `longhorn-manager`가 백그라운드 정리를 수행합니다. 필요 시 수동으로 `kubectl delete -n longhorn-system volumes.longhorn.io <vol>` 등으로 CR를 삭제할 수도 있으나, 일반적인 상황에서는 PVC 삭제만으로 모든 것이 정리됩니다.

> **요약 흐름 도표:** (PVC 생성 → Longhorn Volume/PV 생성 → Pod에 Attach/Mount → Pod 사용 → PVC 삭제) 과정을 정리하면 다음과 같습니다:
>
> * **PVC 생성:** StorageClass `longhorn` → **CSI Provisioner**가 Longhorn API 호출 → **Volume CR 생성** (spec에 크기/복제본 etc) → **Replica CR들** 생성 (각 노드에 할당, 데이터파일 준비) → (엔진 CR 생성 대기)
> * **Pod 스케줄:** AttachVolume 요청 → **Volume CR.spec.nodeID 지정** → Longhorn **엔진 프로세스** 시작 (해당 노드) → **복제본 프로세스들** 시작 (각 노드) → 엔진-복제본 연결 완료 → **iSCSI 연결**로 `/dev/longhorn/*` 장치 생성 → **CSI NodePublish**: ext4 포맷 후 마운트 → Pod에 볼륨 연결 완료.
> * **데이터 I/O:** 엔진이 Replica들과 통신하여 R/W 수행 (동기 복제).
> * **볼륨 삭제:** PVC/PV 삭제 → **Detach** (엔진/복제본 프로세스 종료) → Longhorn **Volume/Engine/Replica CR 삭제** 및 데이터파일 삭제 → 리소스 정리 완료.

## 3. RWO vs RWX 볼륨 지원 구조

일반적인 Longhorn 볼륨은 **RWO(ReadWriteOnce)** 접근 모드를 가지며, 동시에 한 노드에서만 마운트가 가능합니다. 이는 블록 스토리지의 특성상 하나의 노드에서만 iSCSI 디바이스를 어태치하여 사용할 수 있기 때문입니다. 그러나, 다수의 Pod가 **다른 노드에서 동시에 같은 데이터에 접근**해야 하는 **RWX(ReadWriteMany)** 요구사항도 있습니다. Longhorn은 RWX 볼륨을 지원하기 위해 **내부적으로 NFS 서버를 사용하는 방식**을 채택하고 있습니다. 이번 섹션에서는 Longhorn의 RWX 구현 방법과 사용 시 고려사항을 설명합니다.

### 3-1. Longhorn RWX 구현 방식 (Share Manager를 통한 NFS)

Longhorn은 **RWX 볼륨을 직접 다중 노드에 Attach**하지 않습니다. 대신, **하나의 노드에 볼륨을 Attach한 후 그 노드에서 NFS 서버를 구동하여** 다른 노드들과 공유합니다. 이 역할을 수행하는 컴포넌트가 **Longhorn Share Manager**입니다. RWX로 PVC가 생성되면 Longhorn은 자동으로 **share-manager-<볼륨명> Pod**를 띄우고, 그 안에서 **NFSv4 서버**가 동작하도록 합니다. 동시에 Kubernetes `longhorn-system` 네임스페이스에 \*\*Service (ClusterIP)\*\*가 생성되어, NFS 서버에 접근할 수 있는 가상 IP를 제공합니다. 결과적으로 RWX PVC를 사용하는 모든 Pod는 내부적으로 이 NFS 서비스를 통해 공유된 스토리지를 읽고 쓰게 됩니다.

RWX 볼륨의 동작 흐름은 다음과 같습니다:

* **RWX PVC 생성:** 사용자가 `accessModes: ReadWriteMany`로 PVC를 생성하면, Longhorn CSI는 우선 RWO 볼륨과 동일하게 Longhorn Volume을 프로비저닝합니다 (Volume CR, Replica CR 등 생성). 이 단계까지는 일반 볼륨과 동일하며, 실제 데이터는 복제본들로 구성됩니다.

* **Share Manager 생성:** Volume이 RWX로 요청된 것을 인지한 Longhorn은 **Share Manager Pod** (이름: `share-manager-<volume>`)을 `longhorn-system`에 생성합니다. 이 Pod는 경량 컨테이너로 구성되어 있으며, 내부에서 Longhorn CSI와 연계된 **NFS Ganesha** 서버가 구동됩니다. Share Manager는 Longhorn Manager를 통해 제어되며, 특정 노드에 스케줄됩니다. (기본적으로 해당 볼륨의 첫 Attach를 요청한 그 노드에 생성됨)

* **볼륨 Attach to Share Manager:** Share Manager Pod 내부에서는 Longhorn **엔진**을 활용해 볼륨을 **자기 자신에게 Attach**합니다. 즉, share-manager Pod가 실행된 노드에 Longhorn Volume을 마운트하고, 그 경로를 NFSv4로 export합니다. 이 때 share-manager는 Longhorn API를 사용하여 **Volume CR의 nodeID를 자신의 노드로 설정**하고 엔진/복제본을 연결하는 절차를 자동화합니다. 결과적으로 share-manager Pod 안에서 `/export` 등의 디렉토리에 Longhorn 볼륨이 마운트됩니다.

* **NFS Service 생성:** Longhorn은 해당 share-manager Pod 앞단에 `share-manager-<volume>`라는 Service를 생성합니다. 이 Service는 Pod 내 NFS 서버 포트를 가리키며, ClusterIP (또는 필요시 NodePort 등)로 클러스터 내에서 접근 가능하게 됩니다. 보통 Service 명은 Volume 이름과 연계되어 CSI에서 사용됩니다.

* **Pod들에 RWX 마운트:** 이제 RWX PVC를 사용하는 애플리케이션 Pod들이 스케줄될 때, Longhorn CSI는 각 Pod의 NodePublish 단계에서 **NFS 클라이언트 마운트**를 수행합니다. 즉, kubelet은 해당 Node에 NFS client(`mount -t nfs4`)를 사용하여 위에서 만든 Service (ClusterIP)로 연결하여 마운트하는 것입니다. 이로써 여러 노드에 걸친 여러 Pod들이 동일한 NFS 서버(share-manager Pod)를 통해 하나의 Longhorn 볼륨을 공유하게 됩니다. 각 Pod의 관점에서는 RWX PVC가 마치 다중 접속 가능한 볼륨처럼 보이지만, 실제로는 백엔드에서 **하나의 노드**에 attach된 Longhorn 볼륨에 접근하고 있는 것입니다.

&#x20;**그림 2: Longhorn RWX 볼륨 아키텍처** – RWX 볼륨의 내부 구성 다이어그램【49†look】. `volume-0` Longhorn 볼륨은 한 노드에만 attach되어 있고, 그 노드에서 동작 중인 `share-manager-0` Pod 내의 **NFSv4 서버**를 통해 export됩니다. 여러 클라이언트 노드 (`node-0` \~ `node-N`)의 애플리케이션 Pod들은 `csi-plugin`과 Longhorn Manager의 orchestration으로 자동 마운트된 NFS 경로를 통해 `volume-0`에 접근합니다. (Longhorn 1.1+ 버전부터 RWX 기능이 제공되며, 내부에서 share-manager를 활용한 NFSv4 방식으로 구현됨.)

**Share Manager 동작:** share-manager는 Longhorn의 **Share Manager Controller**에 의해 관리되며, Pod가 비정상 종료되거나 노드가 장애가 나면 **자동으로 다른 노드에 재생성**됩니다. 이를 위해 Longhorn은 RWX 볼륨별로 **Recovery Backend**라는 보조 컴포넌트를 두어 NFS 클라이언트들의 상태를 추적하고(어떤 노드 호스트네임들이 접속 중인지 등을 기록) 장애 시 재마운트를 지원합니다. Longhorn 1.3 이후로는 **빠른 Failover** 옵션이 추가되어, share-manager Pod이 죽었을 때 즉시 다른 노드로 Attach 및 NFS 재구동을 수행함으로써 RWX 다운타임을 최소화할 수 있습니다. (실험적인 기능이었으나 최신 버전에서는 안정화됨)

### 3-2. RWX 사용 시 고려사항 (성능 및 제한)

**성능 이슈:** RWX 볼륨은 NFS 레이어를 추가로 거치므로 RWO 볼륨보다 성능이 저하될 수 있습니다. 모든 I/O가 한 노드를 경유하기 때문에 **네트워크 대역폭**이 병목이 될 가능성이 있고, NFS 특성상 **latency**가 증가할 수 있습니다. 특히 **동시 쓰기 작업**이 많은 워크로드의 경우 NFS 서버에서의 **락(Lock) 경합**이나 캐시 일관성 오버헤드가 성능에 영향을 줄 수 있습니다. 따라서 RWX 볼륨은 주로 다중 노드에서 읽기 위주로 공유하거나, 빈번한 동시 쓰기가 없는 워크로드에 적합합니다. 데이터베이스와 같이 강한 일관성이 필요한 애플리케이션은 RWX(NFS) 보다는 RWO를 권장합니다.

**클라이언트 환경:** RWX 볼륨을 사용하려면 **모든 노드에 NFSv4 클라이언트**가 설치되어 있어야 합니다. (예: 우분투의 `nfs-common` 패키지 등) 이는 Longhorn이 NFS 마운트를 시도할 때 필요한 `/sbin/mount.nfs4` 헬퍼가 존재해야 함을 의미합니다. 설치되지 않은 경우 Pod 이벤트 등에 “`mount.nfs4: program not found`” 식의 오류가 발생하며 마운트에 실패합니다.

**Attachment 및 Failover:** RWX 볼륨은 내부적으로 한 번에 하나의 Longhorn 엔진만 실행되므로, **동시에 두 곳에 attach되지 않습니다**. 만약 share-manager Pod가 실행 중인 노드 전체가 다운되면, Longhorn은 일정 시간 이후 해당 RWX 볼륨을 강제로 분리(detach)하고 다른 건강한 노드에 재attach하여 새로운 share-manager를 띄웁니다. 이 과정에서 **일시적인 서비스 중단**이 발생할 수 있습니다 (전환 시간 동안 NFS 경로 I/O 블록). Longhorn 1.1에는 이 자동 재배치가 다소 지연되었으나, 이후 버전에서 **fast failover** 기능으로 개선되었습니다. 운영 시 중요한 RWX 볼륨이라면 **백업 NFS 서버**(Recovery Backend) 구성을 활성화하여, 주 NFS 서버 장애 시 클라이언트들이 빠르게 재연결하도록 튜닝할 수 있습니다.

**용량 및 접근 제한:** 한 RWX PVC를 여러 Pod이 공유하면 **각 Pod이 쓰는 데이터 총합이 PVC 용량을 초과하지 않도록** 유의해야 합니다. PVC 용량 쿼타는 Volume 단위로 적용되므로, Pod 간 조율이 필요합니다. 또한 다수 Pod이 동시에 쓰는 경우 예상보다 빨리 용량이 찰 수 있으므로 **모니터링**이 중요합니다.

**구성 예시:** RWX PVC를 사용하기 위해서는 StorageClass는 RWO와 동일한 Longhorn 클래스를 쓰고, PVC `accessModes`만 `ReadWriteMany`로 지정하면 됩니다. 예:

```yaml
kind: PersistentVolumeClaim
metadata:
  name: my-vol-rwx
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: longhorn
  resources:
    requests:
      storage: 5Gi
```

위와 같이 생성한 PVC를 Deployment 등 여러 Pod에서 마운트하면, Longhorn이 자동으로 NFS 공유를 설정합니다. 별도의 NFS Server 설정이나 PV 사전 준비가 필요 없다는 점에서 **Longhorn RWX는 매우 간편**합니다. 다만, **성능과 안정성 측면에서 RWO보다 신중한 운영**이 필요하며, 중요한 워크로드의 경우 충분한 테스트 후 사용하는 것이 좋습니다.

## 4. 장애 시 동작 및 디버깅 사례

Longhorn 스토리지는 기본적으로 여러 노드에 데이터를 복제하여 **고가용성**을 추구하지만, 운영 환경에서 다양한 오류 상황이 발생할 수 있습니다. 여기서는 Longhorn 볼륨 사용 중 나타날 수 있는 일반적인 문제 유형과 원인, 그리고 이를 진단하고 해결하는 방법을 다룹니다. 또한 노드 장애나 Pod 재스케줄 등 이벤트 발생 시 Longhorn 볼륨의 동작 방식과 운영 시 유의사항도 함께 설명합니다.

### 4-1. 볼륨 Attach/Mount 실패 관련

* **문제 증상:** Pod가 스케줄되었지만 `ContainerCreating` 상태에서 멈춰 있고, `kubectl describe pod` 출력에 `"AttachVolume.Attach failed for volume ..."` 또는 `"MountVolume.SetUp failed for volume ..."` 등의 에러가 보입니다. 예를 들어 이벤트 로그에 *“MountVolume.MountDevice failed ... /dev/longhorn/pvc-... already mounted or mount point busy”* 같은 메시지가 출력될 수 있습니다.

* **원인 진단:** 이러한 문제는 대개 **볼륨 Attach/Mount 과정에서의 실패** 때문에 발생합니다. 세부 원인으로는:

  * **호스트 설정 문제:** Longhorn v1 엔진의 경우 노드에 iSCSI 이니시에이터가 설치되어 있어야 하는데(`iscsiadm`), 누락된 경우 attach 실패가 발생합니다. (이 경우 Longhorn UI에서 해당 노드가 `Unschedulable` 상태로 표시될 수도 있습니다.)
  * **잔류 세션/리소스 문제:** 이전에 사용하던 iSCSI 세션이나 device mapper가 남아있어 “already mounted” 오류가 나는 경우입니다. 예컨대 multipathd가 활성화된 시스템에서 잘못된 경로로 처리하거나, CrashLoop후 devicemapper가 dev를 점유하는 케이스 등이 있습니다.
  * **퍼미션 문제:** Mount 지점 디렉토리 권한이나 SELinux 설정 등이 원인이 되기도 합니다.

* **해결책:**

  * iSCSI 패키지 미설치의 경우 해당 노드에 `open-iscsi` (Ubuntu) 또는 `iscsi-initiator-utils`(CentOS) 등을 설치하고 노드를 재등록하면 됩니다.
  * `"already mounted"` 오류의 경우 Longhorn 공식 KB에서는 **multipathd 비활성화**를 해결책으로 제시합니다. 노드에서 `ls -l /dev/longhorn/`으로 장치 major\:minor 번호를 확인한 뒤, 해당 장치를 multipathd에서 제외하거나 `multipath -f <dev>`로 해제해야 할 수 있습니다. 또한 필요 시 `iscsiadm -m session -u`로 기존 세션을 로그아웃하고 한 번 Pod를 재시작시키면 문제가 해결되는 경우가 많습니다.
  * SELinux가 enforcing인 환경에서는 Longhorn 볼륨 마운트가 차단될 수 있으므로, 컨텍스트 설정이나 SELinux Permissive 모드로 전환을 고려해야 합니다.

* **팁:** Pod가 Attach/Mount 단계에서 멈춘 경우, **이벤트 메세지**(`kubectl describe pod`)와 Longhorn Manager 및 CSI Driver의 **로그**를 함께 확인하면 원인 파악에 도움이 됩니다. `kubectl -n longhorn-system logs -l app=longhorn-csi-plugin`으로 CSI 노드 플러그인 로그를 보면 구체적인 mount 명령어와 에러가 나옵니다. 또한 Longhorn UI의 `Events` 탭에서도 해당 볼륨/PVC 관련 이벤트를 볼 수 있습니다.

### 4-2. 엔진(Controller) 오류와 볼륨 Faulted

* **문제 증상:** Longhorn UI에서 특정 볼륨이 `Faulted` 또는 `Unavailable` 상태로 표시되고, 해당 PVC를 사용하는 Pod가 I/O 에러로 CrashLoopBackOff 되는 경우가 있습니다. Volume 상태를 보면 `state: detached`지만 `robustness: faulted` 로 나타나거나, 엔진 프로세스가 비정상 종료되었다는 이벤트가 기록됩니다.

* **원인 진단:** \*\*엔진 프로세스(스토리지 컨트롤러)\*\*가 비정상 종료되면 볼륨이 Faulted 상태가 됩니다. 원인으로는:

  * 심각한 버그나 assertion 실패로 엔진이 크래시 난 경우 (엔진 로그 확인 필요).
  * 엔진이 돌아가던 노드가 갑작스럽게 셧다운/리부트되어 (노드 다운) 볼륨이 예상치 못하게 분리된 경우.
  * Longhorn Manager와의 통신 이슈로 엔진 상태가 sync되지 않아 Fault 처리된 경우.

* **해결책:**

  * **일시적 노드/엔진 이슈**라면, Longhorn UI에서 해당 볼륨을 **Detach**한 후 **Attach**를 다시 시도해볼 수 있습니다. Faulted 상태에서는 자동 Attach가 안 되므로 수동 detach->attach 시도를 통해 엔진을 재시작합니다.
  * 그래도 Attach가 되지 않거나 엔진이 시작하자마자 죽는다면, Longhorn UI의 **Salvage** 기능을 고려해야 합니다. Salvage는 Faulted 볼륨에서 **정상 Replica를 골라 새로운 엔진을 기동**하는 절차입니다. 예를 들어 3개 Replica 중 2개는 데이터 손상, 1개만 정상인 상황에서 Salvage를 실행하면 그 1개의 복제본으로 엔진을 띄워 볼륨을 구동합니다.
  * 엔진 오류 시 **엔진 로그**를 확인해야 원인을 파악할 수 있습니다. `kubectl -n longhorn-system logs <instance-manager-e-...> --since=1h` 등으로 해당 엔진 프로세스의 stdout/stderr을 볼 수 있습니다. 심각한 에러 (예: segmentation fault 등)가 있었다면 버그로 간주하고 Longhorn 이슈 트래커를 참조해야 할 수도 있습니다.

* **데이터 복구:** 엔진 Faulted 상태에서 Salvage로 띄운 볼륨은 일단 데이터를 읽을 수 있으므로, 즉시 백업을 뜨는 것이 좋습니다. Longhorn이 관리하는 **스냅샷**을 생성하고, 필요한 경우 **Backup Target**(NFS/S3)에 백업본을 만들어 두세요. 그런 다음 문제가 된 Replica들을 제거하고 (Longhorn UI에서 해당 볼륨 > Replicas > 삭제) 새로운 Replica를 rebuild하면 정상 상태로 돌아올 수 있습니다. 최악의 경우 salvage도 실패하면 **개별 Replica로부터 데이터 복구**하는 방법이 있습니다 – 한 Replica의 디스크 이미지를 가져와서 마운트하는 고급 방법으로, Longhorn docs의 “Exporting a Volume from a Single Replica” 가이드를 참고하십시오.

### 4-3. Replica 불일치 및 Degraded 상태

* **문제 증상:** Longhorn UI에서 Volume의 `Robustness`가 `Degraded`로 표시됩니다. 볼륨은 어태치되어 사용 가능하나, 내부적으로 복제본 중 하나 이상에 문제가 있다는 의미입니다. `kubectl get volumes.longhorn.io -n longhorn-system` 명령으로 상태를 보면 `ROBUSTNESS` 필드가 `degraded`이며, attached 노드와 크기는 정상으로 나옵니다. 또한 UI에서 해당 Volume을 클릭하면 한 이상의 Replica가 `ERR` 혹은 `WO`(out of sync) 상태로 표시될 수 있습니다.

* **원인 진단:** Degraded는 **한 개 이상의 Replica가 손실 또는 오프라인**되었음을 뜻합니다. 일반 원인:

  * 어떤 노드가 다운되어 그 노드에 있던 Replica가 사라진 경우 (노드 장애).
  * 디스크 용량 부족이나 I/O 오류로 특정 Replica가 오류 상태로 전환된 경우.
  * 수동으로 관리자가 Longhorn UI에서 Replica 하나를 삭제한 경우 (재빌드 중에는 degraded로 표시).

* **Longhorn의 동작:** Longhorn은 Volume이 Degraded 상태가 되면 **자동으로 신규 Replica를 스케줄링**하여 데이터 복구를 시도합니다. 예를 들어 3/3 복제본 중 1개 상실 → 즉시 다른 노드에 새로운 Replica 생성 → 남은 정상 Replica로부터 **백그라운드 복제**가 진행됩니다. 이 동안 Volume은 정상 I/O가 가능하나 성능이 조금 저하될 수 있습니다 (복구 트래픽). 복제가 완료되면 Robustness가 다시 `healthy`로 바뀝니다.

* **운영자 개입:** 대부분의 Replica 손실 상황은 Longhorn이 자동 복구하지만, 다음에 유의해야 합니다:

  * **디스크 용량 부족:** 만약 어떤 Replica가 있는 디스크가 가득 차서(Eviction 상태) write 실패가 난 경우, 해당 Replica가 Error가 됩니다. 이 경우 Longhorn이 새 Replica를 만들더라도, 클러스터에 남은 여유 용량이 없다면 복구 실패합니다. *→ 해결:* 불필요한 데이터/스냅샷 제거로 공간 확보 후 복구 재시도.
  * **동시 다중 Replica 손실:** 3중 복제에서 두 Replica가 동시에 손실되면 Volume이 즉시 Faulted 되며 자동복구 불가입니다. 한 Replica만 살아있으므로 (quorum 없음) I/O 중단 -> Fault. 이런 경우 **Salvage** 절차로 단일 Replica에서 복구를 시도해야 합니다. *→ 예방:* 다중 노드 다운을 피하고, 백업 활성화.
  * **Replica rebuild 지연:** 네트워크가 느리거나 Volume 크기가 매우 크다면, 신규 Replica 동기화에 오랜 시간이 걸립니다. 이 동안 다시 다른 Replica가 죽으면 위험해지므로, 가능하면 **가용한 모든 노드에 복제본을 분산**하여 한 노드에 동시에 문제가 생겨도 다수 Replica가 안죽도록 합니다. Longhorn 설정에서 `Replica Soft Anti-Affinity`를 \*\*false(기본)\*\*로 두면, 동일 노드에 두 Replica가 배치되지 않도록 강제하여 노드당 하나씩만 유지하게 합니다. (만약 cluster 크기가 작아 어쩔 수 없이 한 노드에 2개 복제본을 둔 경우, 그 노드 장애 시 volume salvage 외엔 답이 없습니다.)

* **디버깅:** Replica 상태는 `kubectl -n longhorn-system get replicas.longhorn.io`로 개별 CR을 확인할 수 있습니다. 각 Replica CR에는 어떤 노드, 어떤 디스크 경로, 현재 replica mode (RW, ERR, WO 등) 정보가 있습니다. `kubectl describe`로 이벤트를 보면 왜 ERR로 갔는지 힌트가 있을 수 있습니다. 또한 `longhorn-manager` Pod 로그에 해당 볼륨 이름을 grep하면 (예: `kubectl -n longhorn-system logs -l app=longhorn-manager | grep pvc-xxxx`) 복제본 관련 메시지를 찾을 수 있습니다.

* **복구:** Longhorn UI에서 Volume을 선택하고 **“Replicas”** 탭에서 문제있는 Replica를 **삭제(Delete)** 하면 즉시 새로운 Replica가 생성되어 rebuild가 시작됩니다. 자동으로 안되거나 hanging될 때 수동으로 트리거하는 방법입니다. 또한 Snapshot이 너무 많아 성능이 저하된 경우 (스냅샷이 쌓이면 복제 성능에 영향) **Snapshot 꾸준히 정리**하는 것이 중요합니다.

### 4-4. 기타 흔한 이슈와 대처

* **볼륨이 Attach 되지 않고 `attached node` 필드가 빈 경우:** Volume CR의 `spec.nodeID`가 비어있는데 Pod는 지속 대기 → 이는 Longhorn **스케줄링 실패**일 수 있습니다. 예: 3개 Replica를 각기 다른 노드에 놓아야 하는데, 노드가 2개 뿐이라 불가능한 상황 등. 이때 Longhorn UI Volume 상세에 Scheduling Failure 메시지가 표시됩니다. *대처:* Longhorn 설정에서 `Replica Soft Anti-Affinity`를 켜서 (또는 복제본 수 감소) 같은 노드에도 복제본을 허용하거나, 노드를 추가로 확보합니다. 또한 Disk나 Node에 태그가 설정되어 PVC의 StorageClass에 요구됐는데 매치가 안되는 경우도 확인해야 합니다.

* **노드 장애 및 Pod 재스케줄:** 만약 **엔진이 동작 중인 노드가 다운**되면, Longhorn은 약 60초간 해당 노드의 복귀를 기다립니다 (기본 node down eviction 시간). 그 후 Volume을 강제로 Detach 처리합니다. Kubernetes는 장애 노드의 Pod를 새 노드로 옮길 것이고, 그 시점에 Volume Attach가 새 노드로 시도됩니다. Longhorn은 남아있는 건강한 Replica들로 즉시 Attach를 수행하고 (엔진 프로세스를 새 노드에 생성) Volume을 재개시킵니다. 이 때 이전 장애 노드에 있었던 Replica는 연결되지 않으므로 Volume은 degraded로 시작하며, 새로운 Replica를 만들어 복구를 진행합니다. 운영자는 노드 장애 복구 후 Longhorn UI에서 해당 다운 노드를 \*\*`Disabled` 처리(스케줄링 불가)\*\*하여, 문제가 완전히 해결될 때까지 새로운 볼륨 스케줄이 안 되게 할 수 있습니다. 노드가 복귀하면 Longhorn이 automount했던 iSCSI 세션이 잔존할 수 있으므로, `systemctl restart iscsi` 등으로 초기화하면 좋습니다.

  *주의:* **노드 Drain/재부팅**을 할 경우, Pod 이동 전에 **볼륨을 미리 Detach**하지 않도록 합니다. Kubernetes에서 Pod를 내리면 Longhorn CSI가 자동으로 Detach를 수행하므로, 관리자가 손쓸 필요는 없습니다. 다만 한꺼번에 여러 노드를 drain하면 일시적으로 여러 볼륨이 동시에 faulted 될 수 있으므로 **한 번에 한 노드씩** drain하십시오.

* **Longhorn UI/CLI로 상태 모니터링:** 문제 발생 시 **Longhorn UI**의 **Volume 상세 페이지**를 확인하면 Replica별 상태와 최근 이벤트를 볼 수 있습니다. UI가 없을 경우 **Longhorn CLI**(`longhornctl`) 또는 kubectl로 CR을 확인합니다. 예: `kubectl -n longhorn-system get volumes.longhorn.io <volName> -o yaml`로 세부사항을 볼 수 있고, `kubectl describe`로 이벤트를 확인합니다. Longhorn CLI(`longhornctl volume ls`, `longhornctl volume inspect <vol>`)도 유사한 정보를 제공합니다. Longhorn Manager logs (`kubectl logs`)와 `instance-manager` logs를 조합하여 보면 **거의 모든 문제의 원인**을 파악할 수 있습니다.

* **데이터 손상 의심 상황:** 만약 애플리케이션 레벨에서 데이터 오류가 발견되어 Longhorn을 의심한다면, 우선 **다른 Replica들의 스냅샷으로 복구**를 고려하세요. Longhorn 자체는 쓰기 시 모든 Replica에 동일 데이터 기록을 보장하지만, **기본적으로 데이터 무결성 검증(Checksum)을 하지 않습니다**. 디스크에 silent corruption이 발생할 경우 감지 못할 수 있으므로, 중요 데이터는 **정기적으로 백업 및 검증**해야 합니다. Longhorn 1.2+ 버전에서는 **CRC 검사** 옵션을 도입하여 백업 시 데이터 검증을 지원하므로, 백업 기능을 적극 활용하는 것이 좋습니다.

## 5. 운영 팁 (스냅샷, 백업, 성능 최적화 등)

마지막으로 Longhorn 스토리지를 운영하면서 활용할 수 있는 여러 가지 팁들을 정리합니다. 스냅샷/백업 관리, 성능 개선, 데이터 무결성 확보 등을 위한 설정과 권장 사항입니다.

* **스냅샷 전략:** Longhorn 볼륨은 **스냅샷** 기능을 통해 시점 복사본을 유지할 수 있습니다. 스냅샷은 **매우 경량**으로 관리되지만 누적될 경우 성능에 영향을 줄 수 있으므로 **정기적인 관리**가 필요합니다. Longhorn UI의 **Recurring Snapshot/Backup** 설정을 활용하여 예를 들어 “매 시간 스냅샷, 하루 한 번 Backup” 등의 정책을 설정하세요. Recurring Job을 설정해두면 Longhorn이 자동으로 스냅샷을 만들고, retention 정책에 따라 오래된 스냅샷을 삭제합니다. 스냅샷은 복제본 디스크 상에서 **증분** 형태로 저장되므로 공간은 비교적 효율적으로 쓰이지만, **너무 많은 스냅샷이 쌓이면** 복제본 체인이 길어져 복구 시간이 길어질 수 있습니다. 따라서 중요 볼륨은 **일정 갯수 이상 스냅샷이 쌓이지 않도록**(예: 50개 제한) 설정하는 것이 좋습니다.

* **백업 및 오프사이트 스토리지:** Longhorn은 **Backup** 기능으로 스냅샷을 외부 저장소(NFS 서버나 S3 호환 오브젝트 스토리지)에 저장할 수 있습니다. 이는 클러스터 자체가 손실되더라도 데이터를 복원할 수 있게 해주므로, 프로덕션 데이터는 반드시 정기 백업을 설정해야 합니다. BackupTarget (예: S3 버킷 주소)을 설정하고, Volume마다 Recurring Backup을 스케줄하면 Longhorn이 지정된 간격으로 새로운 스냅샷을 백업 대상으로 업로드합니다. 백업 후에는 Longhorn UI의 **Backups** 메뉴에서 개별 파일을 확인하고 필요 시 다른 클러스터로 Restore할 수 있습니다. *Tip:* 백업 대상이 S3인 경우, 네트워크 대역폭과 비용을 고려하여 야간 시간대에 수행하거나 증분 백업 전략을 사용하세요.

* **데이터 무결성 검증:** 앞서 언급했듯 Longhorn은 실시간 I/O 경로에 데이터 검증을 수행하지 않습니다 (성능 오버헤드 때문에). 따라서 애플리케이션 레벨에서 주기적으로 체크섬을 검증하거나, 일정 주기마다 데이터를 읽어서 오류를 확인하는 전략이 필요합니다. 예를 들어 DB의 경우 주기적인 **CHECKSUM** 혹은 **백업 리스토어 검증**으로 데이터 손상을 조기에 발견해야 합니다. Longhorn 자체적으로는 백업 시 **CRC**를 계산하여 데이터 무결성을 확인하는 기능이 있습니다. Backup 생성 로그를 보면 소스와 대상 CRC 값을 비교하므로, 백업 성공 여부와 함께 무결성도 확보할 수 있습니다. 또한 Longhorn 1.3부터 **Snapshot Integrity** 옵션이 추가되어, 스냅샷 생성 시 각 Replica에 요청하여 디스크상 데이터를 MD5 체크하는 기능이 있습니다. 다만 이는 시간/성능 비용이 크므로 필요한 경우에만 사용합니다.

* **성능 최적화 – 복제본 개수:** Longhorn의 **복제본 개수**는 성능과 가용성 사이의 트레이드오프 요소입니다. **복제본이 많을수록** 한 번의 write 시 병렬로 여러 노드에 기록하므로 **레이턴시가 증가**합니다. 반대로 **복제본 1개**로 하면 네트워크 왕복이 없으므로 로컬 디스크 성능에 가깝게 향상되지만, 노드 장애 시 데이터 손실 위험이 있습니다. 성능이 중요한 비프로덕션 워크로드는 복제본 1\~2개로, 가용성이 중요한 워크로드는 3개로 운용하는 것이 일반적입니다. Longhorn은 **실시간으로 복제본 개수를 조정**할 수 있으므로 (Volume Attached 중에도 증설 가능), 부하나 상황에 따라 변경할 수 있습니다. 예를 들어 대용량 일괄 적재 작업 중에는 1개 replica로 성능을 높이고, 완료 후 3개로 다시 늘려 복제본을 동기화하는 방법도 고려할 수 있습니다.

* **성능 최적화 – 데이터 지역성(Data Locality):** Longhorn Setting에서 \*\*“Data Locality”\*\*를 활성화하면, Pod가 실행 중인 노드에 항상 해당 볼륨의 Replica 하나를 유지하도록 합니다. 이를 통해 **읽기 작업은 로컬 디스크에서 수행**되어 지연을 줄일 수 있습니다. (쓰기 시에는 여전히 다른 노드로도 복제됨) 단, Pod가 다른 노드로 옮겨가면 새로운 노드에 복제본을 즉시 생성하고 이전 노드의 복제본은 삭제함으로써, 데이터가 Pod와 같은 노드에 존재하도록 합니다. 이 기능은 네트워크가 불안정하거나 지연이 큰 환경에서 읽기 성능을 높여주지만, 잦은 스케줄 변경이 있을 경우 과도한 복제본 재구축을 일으킬 수 있으니 사용에 주의합니다.

* **성능 최적화 – 네트워크 전용 NIC:** Longhorn은 **복제 트래픽 전용 네트워크**를 사용할 수 있습니다. 설정에서 `Network` 관련 옵션을 지정하면, 복제본 간 동기화에 특정 NIC (예: separate VLAN)이 사용되어 데이터 I/O와 애플리케이션 트래픽이 분리됩니다. 만약 클러스터에 스토리지 전용 네트워크가 있다면 Longhorn 설정의 `Default Data Path`를 해당 네트워크 인터페이스로 지정하세요. 이는 잦은 replication sync나 백업 작업 시 클러스터의 서비스 트래픽에 영향을 주지 않도록 해줍니다.

* **자원 요청 & CPU Pinning:** Longhorn 엔진/복제본 프로세스는 기본적으로 사용자 Pod와 동일한 노드의 리소스를 사용합니다. 성능 민감한 워크로드의 경우 Longhorn `Guaranteed Engine CPU` 설정을 통해 엔진 프로세스에 전용 CPU를 예약할 수 있습니다. 복제본 프로세스도 `guaranteed-engine-cpu`에 비례하여 요청값이 증가합니다. 이를 통해 엔진이 Kube 스케줄러 상 **Guaranteed QoS**로 동작하면 예상치 못한 CPU 스틸을 방지하여 I/O 지연을 줄일 수 있습니다. 또한 underlying 디스크의 성능 (SSD vs HDD)에 따라 Longhorn Volume의 성능이 결정되므로, 중요 볼륨은 **SSD 디스크 풀**을 태그로 구분해 운영하시고, `storageClass.parameters.diskSelector` 기능으로 특정 태그 디스크에만 볼륨을 배치할 수 있습니다.

* **Replica Soft Anti-Affinity:** 기본적으로 Longhorn은 동일 볼륨의 두 Replica를 **같은 노드에 배치하지 않음**(hard anti-affinity)으로 안전성을 높입니다. 그러나 노드 수보다 복제본 수가 많은 경우 볼륨이 스케줄되지 않을 수 있습니다. 이때 **Soft Anti-Affinity** 설정을 **true**로 바꾸면, 불가피한 경우 같은 노드에 둘 이상의 Replica를 허용합니다. 이 옵션은 개발/테스트 환경처럼 노드가 적은 상황에서 볼륨 프로비저닝을 가능하게 하지만, 한 노드 장애 시 다수 Replica 동시 손실로 이어질 수 있으므로 프로덕션에서는 가급적 유지하지 않는 것이 좋습니다.

* **모니터링 및 경고:** Longhorn은 Prometheus 메트릭을 제공합니다. `longhorn_frontend_latency_seconds`나 `longhorn_volume_readwrite_seconds`와 같은 메트릭을 수집하여 볼륨 I/O 지연이나 에러율을 모니터링하세요. 또한 Longhorn 자체 UI에서도 Volume 조건(Degraded, Faulted)을 UI 알림으로 볼 수 있지만, **Prometheus Alertmanager** 등을 이용해 Longhorn 상태에 대한 경고를 설정하면 문제를 조기에 발견해 대응할 수 있습니다. (예: `LonghornVolumeRobustnessDegraded` 경고 등)

## 6. 구성 예제 및 명령어 모음

마지막으로 Longhorn 사용에 유용한 **YAML 구성 예제**와 **명령어 사용법**을 정리합니다. 실제 운영 환경에서 참조하여 활용할 수 있도록 하였습니다.

### 6-1. StorageClass 및 PVC YAML 예제

* **StorageClass 예제:** Longhorn 기본 StorageClass (`longhorn`) 정의 YAML입니다. 복제본 수, 시간 초과 등의 파라미터를 보여줍니다.

  ```yaml
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: longhorn
  provisioner: driver.longhorn.io
  allowVolumeExpansion: true
  parameters:
    numberOfReplicas: "3"
    staleReplicaTimeout: "2880"        # 분 단위, 2880분=2일 (복제본 응답 타임아웃)
    fromBackup: ""                    # 백업에서 복원시 백업 URL 지정
    fsType: "ext4"                    # 파일시스템 타입 (기본 ext4)
    # backupTargetName: "default"     # 기본 백업대상 (설정되어 있을 경우)
    # mkfsParams: "-O ^metadata_csum" # ext4 포맷시 커스텀 매개변수 (옵션)
  ```

  위 예시는 Longhorn 공식 예제와 동일합니다. 필요에 따라 `diskSelector`나 `nodeSelector` 파라미터를 추가로 지정하여 특정 태그를 가진 디스크/노드에만 스케줄링하도록 할 수도 있습니다.

* **RWO PVC 예제:** StorageClass `longhorn`를 사용하는 일반 ReadWriteOnce PVC 예제입니다.

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: example-vol-rwo
    namespace: my-app
  spec:
    accessModes:
      - ReadWriteOnce
    storageClassName: longhorn
    resources:
      requests:
        storage: 2Gi
  ```

  PVC를 생성하면 Longhorn이 자동으로 PV 및 Volume을 프로비저닝합니다. 이 PVC를 Deployment/StatefulSet 등에 마운트하여 사용하면 됩니다.

* **RWX PVC 예제:** 동일 StorageClass를 사용하되 accessModes만 변경한 예제입니다.

  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: shared-vol-rwx
    namespace: my-app
  spec:
    accessModes:
      - ReadWriteMany
    storageClassName: longhorn
    resources:
      requests:
        storage: 5Gi
  ```

  RWX PVC를 생성/마운트하면 Longhorn이 내부적으로 share-manager Pod와 NFS 서비스를 설정하므로, 추가 작업 없이 다중 Pod에서 이 PVC를 사용할 수 있습니다 (단, RWX 기능은 Longhorn **v1.1 이상**에서 지원되므로 버전 확인 필요).

### 6-2. Longhorn 상태 조회 및 디버깅 명령어

* **볼륨 리스트 조회:** Longhorn의 Volume CR들을 조회하려면 아래와 같이 실행합니다.

  ```bash
  $ kubectl -n longhorn-system get volumes.longhorn.io
  NAME                                       STATE     ROBUSTNESS   SIZE        NODE       AGE
  pvc-5242ccd4-5a41-4bb4-b5f8-6f64118f4d33   attached  healthy      2Gi         worker1    5h
  pvc-bcd123...                              detached  healthy      1Gi         <none>     2d
  pvc-ef789...                               attached  degraded     10Gi        worker2    3h
  ```

  `STATE`가 `attached/detached`, `ROBUSTNESS`가 `healthy/degraded` 등을 표시합니다. `NODE`는 Attach된 노드 (없으면 `<none>`).

* **볼륨 상세 확인:** 특정 볼륨의 상세 CR 내용을 보려면:

  ```bash
  $ kubectl -n longhorn-system describe volumes.longhorn.io pvc-5242ccd4-5a41-...
  ```

  이를 통해 현재 spec과 status 필드 (attached 노드, frontend, 복제본 목록, 마지막 이벤트 등)를 확인할 수 있습니다. 예를 들어 `Ready` 상태의 복제본 ID와 위치, `Last Salvage` 여부 등의 정보를 볼 수 있습니다. `kubectl edit volumes.longhorn.io <name>`으로 수동 편집도 가능하지만, 잘못 수정하면 위험하므로 권장되진 않습니다 (디버깅 시 읽기전용으로 참고만).

* **복제본 상태 확인:**

  ```bash
  $ kubectl -n longhorn-system get replicas.longhorn.io -o wide
  NAME                                           INSTANCE_MANAGER   MODE   NODE       DISK               AGE
  pvc-5242ccd4-...-replica-1ae4f092              instance-manager-r-76sd   RW   worker1   default-disk    5h
  pvc-5242ccd4-...-replica-bc5e1234              instance-manager-r-abcd   RW   worker2   default-disk    5h
  pvc-5242ccd4-...-replica-fa001122              instance-manager-r-334e   RW   worker3   default-disk    5h
  ```

  위 예시는 3개 복제본이 **모두 RW(읽기/쓰기 가능)** 모드로 정상 동작 중임을 보여줍니다. 만약 하나가 `ERR`로 표시된다면 해당 Replica가 오류 상태임을 뜻합니다. `INSTANCE_MANAGER` 필드는 어느 인스턴스 매니저 Pod에서 실행 중인지 가리킵니다 (문제 발생 시 이 이름으로 logs를 볼 수 있음).

* **Longhorn CLI 사용:** Longhorn은 별도로 `longhorn-cli` (`lhctl`이라고도 함) 바이너리를 제공하며, `longhorn kubectl plugin` 형식으로 동작하기도 합니다. 예를 들어:

  ```bash
  $ longhorn list                           # 모든 볼륨 요약 리스트
  $ longhorn inspect <볼륨이름>             # 특정 볼륨 상세 (JSON 출력)
  $ longhorn snapshot ls <볼륨이름>         # 볼륨의 스냅샷 목록 나열
  $ longhorn snapshot create <볼륨이름>     # 스냅샷 생성 (출력된 ID 확인)
  $ longhorn backup create <볼륨이름> --snapshot <ID>   # 해당 스냅샷을 백업
  ```

  등의 명령이 있습니다. (`lhctl`을 사용할 경우 `lhctl volume list`와 같이 서브커맨드를 명시해야 할 수 있습니다.) Longhorn CLI는 UI 없이도 거의 모든 조작을 할 수 있지만, 내부적으로 REST API 호출을 하기 때문에 **Longhorn UI API Endpoint**(보통 `http://<managerIP>:9500/v1`)에 접근 가능해야 합니다. 클러스터 외부에서 실행할 경우 port-forward 등을 해야 합니다. 보통은 `kubectl`로 CRD 조작이 가능하기에, CLI는 스냅샷/백업 관리 등 제한적으로 사용됩니다.

* **호스트 장치 확인 및 수동 마운트:** 앞서 소개한 대로, Longhorn 볼륨은 노드의 `/dev/longhorn/<vol>` 경로로 블록 디바이스를 노출합니다. 운영 중 문제가 발생했을 때 **응급 조치**로 해당 볼륨을 수동 Mount해서 데이터 확인 또는 백업을 할 수 있습니다 (단, **Pod가 사용 중인 볼륨을 임의로 mount하면 안됩니다**. 반드시 해당 볼륨이 Longhorn상 **detached** 상태여야 합니다).

  1. Longhorn UI에서 문제가 된 볼륨을 **Detach** (아무 노드에도 Attach되지 않은 상태로 만듦).
  2. 임의의 작업용 노드에서 Longhorn UI를 통해 그 볼륨을 **Attach** (마운트는 안 함) – UI에서 Attach만 하면 kubelet이 아닌 Longhorn만 attach해서 `/dev/longhorn/vol`이 나타납니다.
  3. 작업 노드에 SSH 접속하여 `ls -l /dev/longhorn/`으로 장치 확인.
  4. 마운트 포인트 디렉토리 생성 (예: `/mnt/test`).
  5. `sudo mount -o nouuid /dev/longhorn/<vol> /mnt/test` 로 수동 마운트. (`nouuid` 옵션은 XFS 등일 때 유용; ext4는 필요없음).
  6. `/mnt/test` 경로에서 데이터 확인 및 복사/백업 등 작업 수행.
  7. 완료 후 `sudo umount /mnt/test` 하고 Longhorn UI에서 **Detach**.

  이러한 방법으로 Longhorn이 관리하는 볼륨을 임시로 접근할 수 있습니다. 다만 이 절차는 **운영 중인 볼륨에 대해서는 신중히 적용**해야 합니다. 잘못 다룰 경우 Longhorn 매니저와 상태 불일치가 생길 수 있으므로, 가급적 UI/CLI를 통해서만 조작하고 직접 마운트는 읽기전용 진단용으로만 사용하세요.

* **로그 및 상태 수집:** Longhorn에는 **Support Bundle** 생성 기능이 있습니다 (`longhorn-support-bundle` ConfigMap). UI 상단 **Generate Support Bundle**을 클릭하면, 최근 이벤트, 설정, 로그를 번들로 압축해 다운로드할 수 있습니다. 장애 원인 분석에 필요한 자료를 한 번에 수집해주므로, 심각한 문제가 발생하면 이 번들을 보관하고 커뮤니티나 지원 채널에 공유해 도움을 받을 수 있습니다.
