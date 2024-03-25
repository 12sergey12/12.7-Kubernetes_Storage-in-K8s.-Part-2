# Домашнее задание к занятию «Хранение в K8s. Часть 2» Баранов Сергей

### Цель задания

В тестовой среде Kubernetes нужно создать PV и продемострировать запись и хранение файлов.

------

### Чеклист готовности к домашнему заданию

1. Установленное K8s-решение (например, MicroK8S).
2. Установленный локальный kubectl.
3. Редактор YAML-файлов с подключенным GitHub-репозиторием.

------

### Дополнительные материалы для выполнения задания

1. [Инструкция по установке NFS в MicroK8S](https://microk8s.io/docs/nfs). 
2. [Описание Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/). 
3. [Описание динамического провижининга](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). 
4. [Описание Multitool](https://github.com/wbitt/Network-MultiTool).

------

### Задание 1

**Что нужно сделать**

Создать Deployment приложения, использующего локальный PV, созданный вручную.

1. Создать Deployment приложения, состоящего из контейнеров busybox и multitool.

[Deployment.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/Deployment.yaml)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pvc-deployment
  labels:
    app: main2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: main2
  template:
    metadata:
      labels:
        app: main2
    spec:
      containers:
      - name: busybox
        image: busybox
        command: ['sh', '-c', 'while true; do echo Success! >> /tmp/cache/success.txt; sleep 5; done']
        volumeMounts:
        - name: my-vol-pvc
          mountPath: /tmp/cache

      - name: network-multitool
        image: wbitt/network-multitool
        volumeMounts:
        - name: my-vol-pvc
          mountPath: /static
        env:
        - name: HTTP_PORT
          value: "80"
        - name: HTTPS_PORT
          value: "443"
        ports:
        - containerPort: 80
          name: http-port
        - containerPort: 443
          name: https-port
        resources:
          requests:
            cpu: "1m"
            memory: "20Mi"
          limits:
            cpu: "10m"
            memory: "20Mi"
      volumes:
      - name: my-vol-pvc
        persistentVolumeClaim:
          claimName: pvc-vol
```

2. Создать PV и PVC для подключения папки на локальной ноде, которая будет использована в поде.

[PVC-vol.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/PVC-vol.yaml)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-vol
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
[PV-vol.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/PV-vol.yaml)

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv
spec:
  storageClassName: ""
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /data/pv
  persistentVolumeReclaimPolicy: Retain
```

```
root@baranov:/home/baranovsa/kube-2.2# kubectl get pods
NAME                           	READY    STATUS    RESTARTS       AGE
pvc-deployment-85cff6c56b-rj7vs   2/2 	 Running   2(8m28s ago)   20h
root@baranov:/home/baranovsa/kube-2.2# kubectl get deployment
NAME         	READY   UP-TO-DATE   AVAILABLE   AGE
pvc-deployment   1/1 	1        	1        20h
root@baranov:/home/baranovsa/kube-2.2# kubectl get pvc
NAME  	  STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-vol   Bound	   pv   	2Gi    	RWO                          20h
root@baranov:/home/baranovsa/kube-2.2# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM         	STORAGECLASS   REASON   AGE
pv 	2Gi       RWO        	 Retain       	  Bound	   default/pvc-vol                       	20h
root@baranov:/home/baranovsa/kube-2.2# kubectl get svc
NAME     	TYPE    	CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
kubernetes   ClusterIP          10.96.0.1	<none>     443/TCP   23d
root@baranov:/home/baranovsa/kube-2.2#

```

3. Продемонстрировать, что multitool может читать файл, в который busybox пишет каждые пять секунд в общей директории. 

```
root@baranov:/home/baranovsa/kube-2.2# kubectl exec pvc-deployment-85cff6c56b-rj7vs -c network-multitool -it -- sh
/ # ls
bin 	dev 	etc 	lib 	mnt 	proc	run 	srv 	sys 	usr
certs   docker  home	media   opt 	root	sbin	static  tmp 	var
/ # cd static
/static # ls
success.txt
/static # cat success.txt
Success!
Success!
Success!
Success!
Success!
Success!
Success!
Success!
```
```
root@baranov:/# ls
bin   dev   initrd.img    	lib32   lost+found    opt   run   srv  usr      vmlinuz.old
boot  etc   initrd.img.old  lib64   media    proc  sbin  sys  var
data  home  lib   	 	libx32  mnt   	 root  snap  tmp  vmlinuz
root@baranov:/# cd data
root@baranov:/data# ls
pv
root@baranov:/data# cd pv
root@baranov:/data/pv# ls
success.txt
root@baranov:/data/pv#cat success.txt
Success!
Success!
Success!
Success!
Success!
Success!
Success!
Success!

```

4. Удалить Deployment и PVC. Продемонстрировать, что после этого произошло с PV. Пояснить, почему.

```
root@baranov:/home/baranovsa/kube-2.2# kubectl delete deployment pvc-deployment
deployment.apps "pvc-deployment" deleted
root@baranov:/home/baranovsa/kube-2.2# kubectl get deployment
No resources found in default namespace.
root@baranov:/home/baranovsa/kube-2.2# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM         	STORAGECLASS   REASON   AGE
pv 	2Gi     	RWO        	Retain    Bound    default/pvc-vol                       	21h
root@baranov:/home/baranovsa/kube-2.2# kubectl get pvc
NAME  	  STATUS        VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
pvc-vol   Bound	pv   	2Gi      RWO                       	          21h
root@baranov:/home/baranovsa/kube-2.2#
root@baranov:/home/baranovsa/kube-2.2#
root@baranov:/home/baranovsa/kube-2.2#
root@baranov:/home/baranovsa/kube-2.2# kubectl delete pvc pvc-vol
persistentvolumeclaim "pvc-vol" deleted
root@baranov:/home/baranovsa/kube-2.2# kubectl get pvc
No resources found in default namespace.
root@baranov:/home/baranovsa/kube-2.2# kubectl get pv
NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS 	CLAIM         	STORAGECLASS   REASON   AGE
pv 	2Gi    	  RWO        	 Retain       	  Released   default/pvc-vol                       	21h
root@baranov:/home/baranovsa/kube-2.2#

```

5. Продемонстрировать, что файл сохранился на локальном диске ноды. Удалить PV.  Продемонстрировать что произошло с файлом после удаления PV. Пояснить, почему.

```root@baranov:/# ls
bin   dev   initrd.img    	lib32   lost+found    opt   run   srv  usr      vmlinuz.old
boot  etc   initrd.img.old  lib64   media    proc  sbin  sys  var
data  home  lib   	 	libx32  mnt   	 root  snap  tmp  vmlinuz
root@baranov:/# cd data
root@baranov:/data# ls
pv
root@baranov:/data# cd pv
root@baranov:/data/pv# ls
success.txt
root@baranov:/data/pv#cat success.txt
Success!
Success!
Success!
Success!
Success!
Success!
Success!
Success!

```


Ответ: Файл остался. Также при конфигурировании pv  использовался режим ReclaimPolicy: Retain при котором "Retain - после удаления PV ресурсы из внешних провайдеров автоматически не удаляются". 
Даже после удаления pv файлы также останутся.

```
root@baranov:/home/baranovsa/kube-2.2# kubectl delete pv pv
persistentvolume "pv" deleted
root@baranov:/home/baranovsa/kube-2.2#
root@baranov:/home/baranovsa/kube-2.2#
root@baranov:/home/baranovsa/kube-2.2# kubectl get pv
No resources found
root@baranov:/home/baranovsa/kube-2.2#

root@baranov:/# ls
bin   dev   initrd.img    	lib32   lost+found    opt   run   srv  usr      vmlinuz.old
boot  etc   initrd.img.old  lib64   media    proc  sbin  sys  var
data  home  lib   	 	libx32  mnt   	 root  snap  tmp  vmlinuz
root@baranov:/# cd data
root@baranov:/data# ls
pv
root@baranov:/data# cd pv
root@baranov:/data/pv# ls
success.txt
root@baranov:/data/pv#cat success.txt
Success!
Success!
Success!
Success!
Success!
Success!
Success!
Success!
Success!

```

6. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

[Deployment.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/Deployment.yaml)

[PVC-vol.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/PVC-vol.yaml)

[PV-vol.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/PV-vol.yaml)


------

### Задание 2

**Что нужно сделать**

Создать Deployment приложения, которое может хранить файлы на NFS с динамическим созданием PV.

1. Включить и настроить NFS-сервер на MicroK8S.

```
root@baranov:/home/baranovsa/kube-2.2# helm install nfs-server stable/nfs-server-provisioner
WARNING: This chart is deprecated
NAME: nfs-server
LAST DEPLOYED: Mon Mar 25 22:40:20 2024
NAMESPACE: default
STATUS: deployed
REVISION: 1
TEST SUITE: None
NOTES:
The NFS Provisioner service has now been installed.

A storage class named 'nfs' has now been created
and is available to provision dynamic volumes.

You can use this storageclass by creating a `PersistentVolumeClaim` with the
correct storageClassName attribute. For example:

    ---
    kind: PersistentVolumeClaim
    apiVersion: v1
    metadata:
      name: test-dynamic-volume-claim
    spec:
      storageClassName: "nfs"
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 100Mi
root@baranov:/home/baranovsa/kube-2.2#
```

2. Создать Deployment приложения состоящего из multitool, и подключить к нему PV, созданный автоматически на сервере NFS.

[deployment_with_nfs.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/deployment_with_nfs.yaml)

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-deployment-nfs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-deployment-nfs
  template:
    metadata:
      labels:
        app: my-deployment-nfs
    spec:
      containers:
      - name: multitool
        image: wbitt/network-multitool
        volumeMounts:
          - name: my-volume-nfs
            mountPath: /input
      volumes:
      - name: my-volume-nfs
        persistentVolumeClaim: 
          claimName: nfs-pvc
```

```
root@baranov:/home/baranovsa/kube-2.2# kubectl get pods
NAME                                  READY   STATUS    RESTARTS   AGE
nfs-server-nfs-server-provisioner-0   1/1     Running   0          2m33s
my-deployment-nfs-9d5fdf76b-kq58w     1/1     Running   0          103s
root@baranov:/home/baranovsa/kube-2.2# 

```
[pvc_nfs.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/pvc_nfs.yaml)

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  storageClassName: nfs
  volumeMode: Filesystem
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Mi
```
```
root@baranov:/home/baranovsa/kube-2.2# kubectl get pvc
NAME      STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
nfs-pvc   Bound    pvc-03fb0fef-2c54-4a92-b70f-3a10e2a35f95   100Mi      RWO            nfs            48s
root@baranov:/home/baranovsa/kube-2.2# kubectl get pv
NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM             STORAGECLASS   REASON   AGE
pvc-03fb0fef-2c54-4a92-b70f-3a10e2a35f95   100Mi      RWO            Delete           Bound    default/nfs-pvc   nfs                     51s
root@baranov:/home/baranovsa/kube-2.2#
```

3. Продемонстрировать возможность чтения и записи файла изнутри пода. 

```
root@baranov:/home/baranovsa/kube-2.2# kubectl exec -it my-deployment-nfs-9d5fdf76b-kq58w -- sh
/ # ls -l /input/
total 0
/ # touch /input/123.txt
/ # ls -l /input/
total 0
-rw-r--r--    1 root     root             0 Mar 25 15:45 123.txt

```

4. Предоставить манифесты, а также скриншоты или вывод необходимых команд.

[deployment_with_nfs.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/deployment_with_nfs.yaml)

[pvc_nfs.yaml](https://github.com/12sergey12/12.7-Kubernetes_Storage-in-K8s.-Part-2/blob/main/pvc_nfs.yaml)
------

### Правила приёма работы

1. Домашняя работа оформляется в своём Git-репозитории в файле README.md. Выполненное задание пришлите ссылкой на .md-файл в вашем репозитории.
2. Файл README.md должен содержать скриншоты вывода необходимых команд `kubectl`, а также скриншоты результатов.
3. Репозиторий должен содержать тексты манифестов или ссылки на них в файле README.md.
