# Kubernetes Storage

## emptyDir

### 왜 필요할까?

초기 컨테이너 환경에서 임시 데이터를 저장할 공간이 필요할 때, 사용하던 임시 볼륨 개념에서 가져온 개념이다.

Pod 단위에서 이러한 임시 저장소를 제공하기 위해 emptyDir 볼륨 타입을 지원하게 되었다.

Pod 단위에서의 임시저장소라 함은 Pod 내에 있는 컨테이너 간의 공유되는 저장소임을 의미한다.

### 어떻게 동작하지?

1. Pod가 스케줄링되어 특정 노드에서 생성되면, 동시에 노드의 파일 시스템상에 디렉터리가 하나 만들어진다.
    - Pod는 모두 실제 노드의 도움을 받아서 돌아가는 존재이기 때문에 실제 노드의 파일 시스템을 기반으로 동작하는 것!
2. Pod 내 컨테이너들은 이를 Mount 받아서 공용 임시 저장공간처럼 활용할 수 있다.
3. Pod가 종료되거나 제거되면, 해당 emptyDir도 함께 영구 삭제된다.
    - Pod도 저장된 내용이 저장되지 않는 휘발성이므로, 결국 내부에 존재하는 emptyDir도 함께 삭제되는 존재라는 것이다.

### 특징

- 휘발성 → 운영 단계에서는 활용할 수 없는 존재..
- 빠른 I/O → 노드의 로컬 디스크를 사용하기 때문에
- 용도 → 빌드 과정에서의 캐시 디렉터리, 임시 파일 저장 등 일시적 데이터가 필요할 때 적합
- 주요 설정 → Pod 선언 내 별도 옵션없이 `emptyDir: {}`로 선언하면 바로 사용 가능

### 어떤 상황에서 쓰면 좋을까?

빌드 캐싱이나 임시 로그, 애플리케이션 내부 처리용 임시 파일, 성능 테스트…

### 사용법

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: emptydir-multi-container
spec:
  replicas: 4
  selector:
    matchLabels:
      app: emptydir-multi-container
  template:
    metadata:
      labels:
        app: emptydir-multi-container
    spec:
      containers:
        - name: container-a
          image: 
          imagePullPolicy: IfNotPresent 
          volumeMounts:
            - name: shared-volume
              mountPath: /usr/src/app/shared
          ports:
            - containerPort: 3000     

        - name: container-b
          image: 
          imagePullPolicy: IfNotPresent
          volumeMounts:
            - name: shared-volume
              mountPath: /usr/src/app/shared
          ports:
            - containerPort: 2000

      volumes:
        - name: shared-volume
          emptyDir: {}
```

- volumes 섹션에서 `emptyDir:{}` 설정시 별도 과정 없이 임시 볼륨 생성 가능
- volumeMounts 섹션에서 `mountPath` 를 `/usr/src/app/cache` 로 지정시 이 디렉토리를 임시 캐시되는 저장소 path로 활용 가능
- container-a에 접속하여 임시 파일을 생성했다면, container-b에서도 동일한 데이터를 확인 가능하다.
    
    ```bash
    kubectl exec -it emptydir-multi-conatiner -c container-a -- /bin/sh
    ```
    
    해당 명령어를 통해서 Pod 내 컨테이너에 /bin/sh를 사용하여 진입 가능함
    

## hostPath

### 왜 필요할까?

Pod 내에서의 통신이 아니라 Pod끼리의 통신이 필요하다면?

호스트 노드 내의 파일 시스템 일부를 직접 마운트해서 사용할 수 있다면 좋지 않을까?

동일한 노드 내의 파드끼리 파일을 공유할 수 있도록 하는 것이 hostPath이다.

### 어떻게 동작하지?

1. 개발자가 Pod 정의 YAML에 hostPath 볼륨을 명시하여 API Server에게 제출
2. API 서버가 워커 노드를 결정하고, 워커의 kubelet이 Pod와 볼륨 생성 과정 수행
3. 워커노드에서 hostPath를 확인하여 없으면 생성
4. Pod의 컨테이너가 실제 노드 파일 시스템(=hostPath)을 공유 마운트하여 사용

### 특징

- 로컬 노드 종속적 → 해당 노드의 로컬 파일 시스템을 직접 사용한다
    - 만약 Pod가 다른 노드로 옮겨간다면? 내가 원하는 노드에 배정받지 못한다면?
    - 이런 상황들을 고려하여 사용을 고민해야 한다.
- 경량 → 사용이 어렵지 않음
- 개발 / 디버깅 용도로 활용 → 로컬 환경에서 개발시 호스트 파일시스템과 컨테이너를 공유하고자 할 때 유용하지만, 프로덕션 환경에서는 PV를 활용하는 것이 더 권장됨

### 실습

- 실습에서 중요한 부분은 “동일한 노드” 내에서 공유가 되는 점을 테스트하기 위함이므로, `nodeSelector` 를 활용하는 부분이다.
- `nodeSelector` 의 역할은 간단하게 말하자면, 특정 Pod가 특정 노드에게만 스케줄링될 수 있도록 미리 지정해놓는 것!

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostpath-deployment-a
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hostpath-app-a
  template:
    metadata:
      labels:
        app: hostpath-app-a
    spec:
      nodeSelector:
        kubernetes.io/hostname: worker1
      containers:
      - name: jeff-server
        image: ghcr.io/ej31/jeff-ex:1.3-amd64
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: shared-volume
          mountPath: /app/data
      volumes:
      - name: shared-volume
        hostPath:
          path: /tmp/hostpath-shared
          type: DirectoryOrCreate
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hostpath-deployment-b
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hostpath-app-b
  template:
    metadata:
      labels:
        app: hostpath-app-b
    spec:
      nodeSelector:
        kubernetes.io/hostname: worker2
      containers:
      - name: jeff-server
        image: ghcr.io/ej31/jeff-ex:1.3-amd64
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: shared-volume
          mountPath: /app/data
      volumes:
      - name: shared-volume
        hostPath:
          path: /tmp/hostpath-shared
          type: DirectoryOrCreate
```

- `hostPath` 를 사용하기 위해서는 배포하고자 하는 Pod에 Volumes 부분을 설정해주고, 해당 Pod내에서 어떤 Volume을 mount 할 것인지 설정해주어야한다.
- `path` 설정은 실제 노드의 어떤 디렉토리 부분에 저장할 것인지를 지정해주는 곳이다.
- `type` 설정은 다양한 설정이 가능하나, `DirectoryOrCreate` 의 경우에는 해당 디렉토리가 없을 때 자동 생성되는 부분을 이야기한다.
- 이렇게 volumes 부분을 shared-volume이라는 이름으로 지어줬다면, `volumeMounts` 부분에서 해당 volumes을 사용할 것이라고 명시해주어야 한다.
    - `mountPath` 부분은 Pod 내의 어떤 디렉토리에 마운트할 것인지를 말하는 것!

⇒ 이후 테스트를 진행해보면 동일한 노드 내에 있는 다른 Pod에서 각자 `/app/data` 부분에 새로운 파일을 작성했다면, 모두 동기화되는 것을 확인할 수 있다.

## PersistentVolume(PV) & PersistentVolumeClaim(PVC)

### 왜 필요할까?

방금 Pod 단위, Node 단위 내에서 공유되는 저장공간들의 문제점을 봤다.

모두 각자 단위에 종속적일 수 밖에 없다는 점이다.

특정 노드에 한정짓는 것이 아니라 클러스터 단위에서 모든 Pod간의 동일한 스토리지를 공유해야할 경우에 사용하는 스토리지가 필요하다.

### PV와 PVC는 어떻게 동작하지?

1. PV 생성을 통해 클러스터 자체에 스토리지 자원을 먼저! 등록해둔다.
    - 이는 운영자가 미리 자원을 만들어두는 것이다.
2. PVC 생성을 통해 이런 용량과 접근 모드로 스토리지를 사용한다고 명시하는 것
    - 개발자가 실제로 쿠버네티스 스토리지 자원을 필요로 할 때, 이만큼 사용하겠다고 요청을 보내는 것이다.
    - 미리 만들어져있다면, 요청대로 빠르게 제공할 수 있기에 이러한 분리 제도를 도입했다.
3. 바인딩 과정을 통해 PVC 요구사항을 충족하는 PV가 있으면 자동으로 연결된다.
4. Pod에서 사용하고자 하는 PVC만 써주면, 실제 물리 스토리지를 자동으로 마운트한다. 
5. Pod가 죽어도 PV는 남아있으니 PVC가 계속 PV를 유지
이후 새로운 Pod가 동일 PVC를 재마운트 가능! ⇒ 우리에게 필요한 영속성을 보장해주는 존재

### 특징

- 영구성 → Pod가 재시작되거나 심지어 삭제되더라도 데이터를 보존 가능
- 분리된 라이프사이클 → PV는 클러스터 자원으로 독립적으로 존재, PVC는 필요할 때마다 청구하는 개념
    - Pod를 삭제해도 PVC는 남아있을 수 있음 → 재사용 가능
    - PVC를 삭제하면 PV가 Released 될 수 있음
- Access Mode 제공
    - `ReadWriteOnce (RWO)`  : 단일 노드에서만 읽기/쓰기가 가능
    - `ReadOnlyMany (ROX)`  : 여러 노드에서 읽기 가능, 쓰기는 단일 노드만 가능
    - `ReadWriteMany (RWX)`  : 여러 노드에서 동시에 읽기/쓰기 가능
- 스토리지 클래스를 통한 동적 프로비저닝 ⇒ PV 대체 가능
    - 사용자가 PVC만 작성하면, 적절한 StorageClass를 통해 PV가 자동으로 생성, 할당되어 편의성 제공
    - `storageClassName: aws-ebs` 와 같이 수동으로 생성하지 않아도, 해당 옵션에 정의된 프로비저너가 스토리지 백엔드에 요청을 보내 PV를 자동으로 생성하고 PVC에 바인딩
    
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: aws-ebs
    provisioner: kubernetes.io/aws-ebs # AWS EBS 프로비저너
    parameters:
      type: gp3                         # EBS 볼륨 타입 (gp2, gp3, io1 등)
      fsType: ext4                      # 파일 시스템 타입 (ext4, xfs 등)
      encrypted: "true"                 # 데이터 암호화 여부
      iopsPerGb: "10"                   # (선택) io1/ gp3 타입일 경우 IOPS 설정
      throughput: "125"                 # (선택) gp3 타입일 경우 스루풋 설정
    reclaimPolicy: Delete               # PVC 삭제 시 EBS 볼륨 삭제
    volumeBindingMode: WaitForFirstConsumer # PVC가 파드에 연결된 후 볼륨 바인딩
    ```
    

### 실습

PV 생성

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-pv
spec:
  capacity:
    storage: 1Gi        # PV 용량
  accessModes:
    - ReadWriteOnce     # 단일 노드에 읽기/쓰기 가능
  hostPath:
    path: /data/hostpath # 실제 워커 노드(또는 마스터 노드)의 /data/hostpath 경로
  persistentVolumeReclaimPolicy: Retain
```

- `capacity.storage`: PV가 제공하는 스토리지 크기
- `accessModes`: RWX, RWO, ROX 등 지원 모드 중 하나 사용
- `hostPath`: 실습 편의를 위해 호스트 경로 기반의 PV 사용, storageClassName도 활용 가능
- `persistentVolumeReclaimPolicy`: PV가 릴리즈 후 삭제되거나 유지될지 결정

PVC 생성

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi  # 원하는 스토리지 용량
```

PV에 대한 아무런 언급 없이도 직접 PV를 찾아서 바인딩 하게 된다.

Deployment에서 PVC 활용

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jeff-ex-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jeff-ex
  template:
    metadata:
      labels:
        app: jeff-ex
    spec:
      containers:
      - name: jeff-ex-container
        image: ghcr.io/ej31/jeff-ex:1.3-amd64
        ports:
        - containerPort: 3000
        volumeMounts:
        - name: jeff-data-volume
          mountPath: /app/data    # 컨테이너 내부에 마운트할 경로
      volumes:
      - name: jeff-data-volume
        persistentVolumeClaim:
          claimName: my-pvc
```

- `mountPath` 경로에 컨테이너 내부 마운트 경로를 적어준다
- PVC를 마운트할 수 있도록 `volumes` 섹션에 `persistentVolumeClaim` 를 지정해서 PVC를 연결

## NFS 볼륨

### 왜 필요할까?

클러스터 내부에 데이터를 저장하게 되면, 클러스터 자체에 문제가 생겼을 경우 데이터 자체도 영향을 받을 수 있다. 만약 이러한 데이터들을 외부에 저장할 수 있다면? 에서 시작하는 기술이다.

### 어떻게 동작할까?

1. 외부에 NFS 서버에 데이터를 저장할 공유 디렉터리가 존재한다.
2. Kubernetes의 Pod가 PVC를 통해 NFS 서버와 연결한다. 
    - PVC는 Pod와 NFS 서버를 연결하는 추상화된 요청이다.
    - 내부적으로 PVC는 PV와 매핑되어 NFS 서버의 디렉터리와 연결된다.
3. Pod가 실행되면 NFS 서버의 디렉터리가 로컬 디렉터리처럼 마운트되어 사용 가능
    - 이후 NFS 서버의 데이터를 읽거나 쓸 수 있다.

### NFS 이외에는… 없나?

AWS EBS, GCE Persistent Disk, Ceph, GlusterFS 등 다양한 타입의 외부 볼륨을 제공한다.

이러한 외부 볼륨을 자동 프로비저닝하는 기능이 StorageClass이다.

다양한 선택지가 존재하기 때문에 가장 적합한 스토리지를 선택하는 것이 중요하다.

### **다양한 외부 볼륨 옵션 선택시 고려하면 좋을 점**

- **워크로드 특성**:
    - 데이터베이스와 같은 고성능 워크로드에는 AWS EBS, GCP PD, Azure Disk.
    - 로그 저장소나 공유 데이터에는 NFS, Ceph, GlusterFS.
- **접근 모드**:
    - 단일 파드에서만 접근 가능한 스토리지는 `ReadWriteOnce`.
    - 여러 파드에서 동시에 접근하려면 `ReadWriteMany`를 지원하는 스토리지 필요.
- **가용성 및 확장성**:
    - 클라우드 네이티브 애플리케이션에는 Ceph 또는 클라우드 제공 스토리지.
- **운영 비용**:
    - 클라우드 스토리지는 비용이 발생하므로, 워크로드와 비용 간의 균형을 고려.

### NFS 서버를 직접 구축해서 실습해보기

> EC2를 하나 생성해서 해당 NFS 패키지를 설치하여 서버를 구축한다.
> 

```bash
# 1) NFS 패키지 설치
sudo apt-get update
sudo apt-get install -y nfs-kernel-server

# 2) 공유 디렉터리 생성
sudo mkdir -p /var/nfs_share
sudo chown nobody:nogroup /var/nfs_share
sudo chmod 777 /var/nfs_share

# 3) /etc/exports 파일에 공유 설정 추가
echo "/var/nfs_share  *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports

# 4) NFS 서비스 재시작
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

> 마스터노드에 NFS 서버 데이터 업로드를 위한 설정 추가하기
> 

```bash
# 1) NFS 클라이언트 패키지 설치 (Ubuntu/Debian 기준)
sudo apt-get update
sudo apt-get install -y nfs-common

# 2) NFS 마운트할 디렉터리 생성
sudo mkdir -p /mnt/nfs_test

# 3) NFS 서버 마운트 -> 여기서도 IP 주소는 본인이 생성한 NFS 서버 IP로 설정해야 함!!!
sudo mount -t nfs 172.31.45.83:/var/nfs_share /mnt/nfs_test

# 4) 테스트 파일 생성
echo "Hello NFS!" > /mnt/nfs_test/test.txt
date > /mnt/nfs_test/current_time.txt

# 5) 파일 확인
ls -l /mnt/nfs_test/

# 6) 만약 마운트 된 걸 없애고 싶다면 주석 지우고 언마운트하면 된다.
sudo umount /mnt/nfs_test
```

미리 설치해둔 NFS 서버의 Private IP를 3번에 넣어주면 된다.

NFS가 `/mnt/nfs_test` 폴더 아래에 마운트되고, 테스트 파일을 생성하면 서버에 자동으로 저장된다.

이후 쿠버네티스에서 이를 활용하려면 지금까지 배운 PV, PVC를 활용하면 된다.

과정은 동일하다. PV에서 hostPath가 NFS 대상으로만 바뀌는 것 뿐이다!

> PV 리소스 생성
> 

미리 만들어둔 NFS 서버를 PV에 지정해주자

```yaml
# nfs-pv.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteMany
  nfs:
    path: "/var/nfs_share"
    server: "172.31.45.83"  # 중요 : nfs-server 가 설정 된 EC2의 privateIP 로 변경!
  persistentVolumeReclaimPolicy: Retain
```

- `nfs` 관련 정의를 넣어주면 된다
    - `path` 부분은 NFS의 어떤 디렉토리를 활용할 것인지에 대한 명시이다.
    - `server` 부분은 NFS 서버의 IP 명을 적어주면 된다.
    - `persistentVolumeReclaimPolicy` : `Retain` , `Delete` 옵션으로 존재한다.
    이는 PV가 더이상 사용되지 않을 때(PVC와 바인딩이 해제된 경우) 해당 볼륨을 어떻게 처리할지 정의하는 것으로 그대로 유지하거나 삭제하는 두가지 옵션을 제공한다.

Pod에서 실제로 사용하기 위한 볼륨을 요청하는 PVC 리소스를 만들자

```yaml
# nfs-pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Gi
```

- `accessModes` 를 활용하여 읽기, 쓰기의 모드를 지정해준다.
    - PV에서 사용하는 서버와 호환되는 모드를 사용하여야만 한다.
    - ex) AWS EBS 처럼 다중 읽기,쓰기가 지원되지 않는 PV를 사용하면서 `ReadWriteMany` 를 지정해주면 안된다.
    - 이는 PV와 PVC의 용량, accessModes 등이 일치하면 자동 바인딩된다.

이후 직접 컨테이너에 볼륨을 마운트해보자

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nfs-test-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nfs-test
  template:
    metadata:
      labels:
        app: nfs-test
    spec:
      containers:
        - name: nfs-test-container
          image: ghcr.io/ej31/jeff-ex:1.3-amd64
          ports:
            - containerPort: 3000
          volumeMounts:
            - name: nfs-volume
              mountPath: "/usr/src/app/data"  # Node.js 앱 내부 디렉터리로 가정
      volumes:
        - name: nfs-volume
          persistentVolumeClaim:
            claimName: nfs-pvc
```

- `volumeMounts`: 컨테이너 내 `/usr/src/app/data` 디렉터리에 `nfs-volume`(PVC) 마운트
- NFS 서버에 있는 실제 경로(`/var/nfs_share`)가 Pod 내부 `/usr/src/app/data`로 매핑됨

만약 NFS 서버를 StorageClass로 정의하고, PV를 동적으로 프로비저닝하고싶다면 아래와 같은 방법으로 하면 된다.

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: nfs-storageclass
provisioner: nfs-subdir-external-provisioner  # 설치된 NFS 프로비저너
parameters:
  server: "172.31.45.83"       # NFS 서버 IP
  path: "/var/nfs_share"       # 공유 디렉터리 경로
  readOnly: "false"            # 읽기/쓰기 가능 여부
mountOptions:
  - hard                       # NFS 연결 시 재시도 옵션
  - nfsvers=4.1                # NFS 버전 설정
reclaimPolicy: Delete           # PVC 삭제 시 PV 및 데이터 삭제
volumeBindingMode: Immediate    # PVC 요청 시 PV 바로 바인딩
```

### 동적 프로비저닝 VS 정적 프로비저닝

가장 큰 차이점은 관리자가 직접 PV를 생성하는지 안하는지의 차이이다.

동적으로 생성하는 경우 PVC 요청 시 StorageClass를 통해 PV를 자동 생성한다.

정적 생성의 경우, 직접 PV를 생성하고 PVC를 요청해야한다.
