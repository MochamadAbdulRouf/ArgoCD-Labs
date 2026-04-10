# Error Log on Argo CD 
Di sini saya akan membuat log error yang saya temukan ketika melakukan konfigurasi di `ArgoCD`, disini saya mencatat error dan cara troubleshootingnya


## Error Akses Dashboard ArgoCD
Error akses dashboard `ArgoCD` menggunakan `svc` `NodePort`, case dimana ketika saya melakukan konfigurasi agar `svc` `argocd-server` menjadi NodePort. Saya menemukan error yang unik, dimana `NodePort` hanya bisa diakses di node-1, saya memiliki 3 VM masing masing adalah `master-node` dan 2 `worker-node`. Namun NodePort hanya bisa di akses menggunakan `IP address` `node-1` membuat saya tidak bisa mengakses dashboard menggunakan ip address `master-node` dan `worker-node 2`. Jadi langkah" troubleshoot yang saya implementasikan berikut:


- Pertama, Intip label selector yang digunakan service untuk mencari pod
```bash
kubectl describe svc argocd-server -n argocd | grep Selector
```
- Lalu gunakan label tersebut untuk mencari pod, dan tambahkan `-o wide` untuk mencari lokasi node nya.
```bash
kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o wide
```
Implementation
```bash
ubuntu@master-node:~$ kubectl describe svc argocd-server -n argocd | grep Selector
Selector:                 app.kubernetes.io/name=argocd-server
ubuntu@master-node:~$ kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o wide
NAME                             READY   STATUS    RESTARTS   AGE   IP           NODE     NOMINATED NODE   READINESS GATES
argocd-server-79fdfd7f5b-vspg5   1/1     Running   0          52m   10.42.1.22   node-1   <none>           <none>
ubuntu@master-node:~$
```

## ArgoCD Repo Server CrashLoopBackOff 
Sebenarnya di awal pods `argocd-repo-server` berhasil running, namun setelah itu mengalami error `CrashhLoopBackOff`. Error ini di sebabkan oleh sistem kubernetes itu sendiri, di karenakan `Error serving health check request (duration: 5000983299 ns):` Pod butuh waktu lebih dari 5 detik (5000ms) untuk menjawab tes kesehatan (Liveness Probe). Waktunya keburu habis (timeout). `got signal terminated:` Kubernetes akhirnya mengirim sinyal untuk mematikan Pod tersebut karena dianggap tidak responsif. Akar masalah ini biasanya disebabkan oleh resource `instance/VM` yang terlalu sedikit, jadi ibarat RAM (Pods) mengalami lag parah sehingga tidak sanggup membuka "aplikasi" (Liveness Probe) dalam waktu 5 detik. 


Logs Error:
1. Logs pertama ketika membuat sebuah deployment akan terjadi error berikut
```bash
Unable to create application: application spec for app-api is invalid: InvalidSpecError: repository not accessible: repositories not accessible: &Repository{Repo: "https://github.com/MochamadAbdulRouf/app-api", Type: "", Name: "", Project: ""}: repo client error while testing repository: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing: dial tcp 10.43.1.130:8081: i/o timeout"
```

2. Lalu saya mencoba melihat pods dari `namespace` argocd nya dan BOOM ketemu masalahnya.
```bash
ubuntu@master-node:~$ k get pods -n argocd

NAME                                               READY   STATUS             RESTARTS         AGE

argocd-application-controller-0                    1/1     Running            0                146m

argocd-applicationset-controller-cf6df4d44-z84ns   1/1     Running            24 (6m8s ago)    146m

argocd-dex-server-694c74cbb8-fkp5t                 1/1     Running            2 (146m ago)     146m

argocd-notifications-controller-67dd6d74b5-8d27d   1/1     Running            0                146m

argocd-redis-85b9d55dc4-rfgtp                      1/1     Running            0                146m

argocd-repo-server-6fd5c47689-5kwf7                0/1     CrashLoopBackOff   35 (2m16s ago)   146m

argocd-server-79fdfd7f5b-vspg5                     1/1     Running            0                146m


ubuntu@master-node:~$ k logs argocd-repo-server-6fd5c47689-5kwf7 -n argocd

Defaulted container "argocd-repo-server" out of: argocd-repo-server, copyutil (init)

{"built":"2026-03-27T13:41:15Z","commit":"998fb59dc355653c0657908a6ea2f87136e022d1","level":"info","msg":"ArgoCD Repository Server is starting","port":8081,"time":"2026-04-10T13:41:14Z","version":"v3.3.6"}

{"level":"info","msg":"Loading Redis credentials from environment variables","time":"2026-04-10T13:41:14Z"}

{"level":"info","msg":"Generating self-signed TLS certificate for this session","time":"2026-04-10T13:41:14Z"}

{"level":"info","msg":"Initializing GnuPG keyring at /app/config/gpg/keys","time":"2026-04-10T13:41:14Z"}

{"dir":"","execID":"6b6f9","level":"info","msg":"gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe4060927311","time":"2026-04-10T13:41:14Z"}

{"args":"[gpg --no-permission-warning --logger-fd 1 --batch --gen-key /tmp/gpg-key-recipe4060927311]","dir":"","level":"info","msg":"Trace","operation_name":"exec gpg","time":"2026-04-10T13:41:14Z","time_ms":209.85725599999998}

{"level":"info","msg":"Populating GnuPG keyring with keys from /app/config/gpg/source","time":"2026-04-10T13:41:14Z"}

{"dir":"","execID":"26623","level":"info","msg":"gpg --no-permission-warning --list-public-keys","time":"2026-04-10T13:41:14Z"}

{"args":"[gpg --no-permission-warning --list-public-keys]","dir":"","level":"info","msg":"Trace","operation_name":"exec gpg","time":"2026-04-10T13:41:14Z","time_ms":3.953906}

{"dir":"","execID":"a5777","level":"info","msg":"gpg --no-permission-warning -a --export CAB819794C8F9193","time":"2026-04-10T13:41:14Z"}

{"args":"[gpg --no-permission-warning -a --export CAB819794C8F9193]","dir":"","level":"info","msg":"Trace","operation_name":"exec gpg","time":"2026-04-10T13:41:14Z","time_ms":3.122808}

{"dir":"","execID":"bafb7","level":"info","msg":"gpg-wrapper.sh --no-permission-warning --list-secret-keys CAB819794C8F9193","time":"2026-04-10T13:41:14Z"}

{"args":"[gpg-wrapper.sh --no-permission-warning --list-secret-keys CAB819794C8F9193]","dir":"","level":"info","msg":"Trace","operation_name":"exec gpg-wrapper.sh","time":"2026-04-10T13:41:14Z","time_ms":6.257695}

{"level":"info","msg":"Loaded 0 (and removed 0) keys from keyring","time":"2026-04-10T13:41:14Z"}

{"level":"info","msg":"argocd-repo-server is listening on [::]:8081","time":"2026-04-10T13:41:14Z"}

{"level":"info","msg":"Starting GPG sync watcher on directory '/app/config/gpg/source'","time":"2026-04-10T13:41:14Z"}

{"level":"info","msg":"starting grpc server","time":"2026-04-10T13:41:14Z"}

{"component":"healthcheck","duration":5000983299,"error":"rpc error: code = Canceled desc = context canceled","level":"error","msg":"Error serving health check request","time":"2026-04-10T13:42:13Z"}

{"component":"healthcheck","duration":5000448495,"error":"rpc error: code = Canceled desc = context canceled","level":"error","msg":"Error serving health check request","time":"2026-04-10T13:42:43Z"}

{"level":"info","msg":"got signal terminated, attempting graceful shutdown","time":"2026-04-10T13:43:13Z"}

{"level":"info","msg":"clean shutdown","time":"2026-04-10T13:43:13Z"}
```

Solusi fix:

1. Karena keterbatasan resource `VM/Instance`, namun sebenarnya sistem masih kuat untuk menjalankan `argocd`, saya mengubah `timeoutsecond liveness Probe` menjadi 30 detik pada deployment `argocd-repo-server`, agar memberikan waktu pods `argocd-repo-server` aktif terlebih dahulu.
```bash
kubectl patch deployment argocd-repo-server -n argocd -p '{"spec": {"template": {"spec": {"containers": [{"name": "argocd-repo-server", "livenessProbe": {"timeoutSeconds": 30}, "readinessProbe": {"timeoutSeconds": 30}}]}}}}'
deployment.apps/argocd-repo-server patched
```

## argocd-applicationset-controller 0/1 CrashLoopBackOff // ArgoCD CRD Component CrashLoopBackOff

Analogi singkat dari error ini Kubernetes secara bawaan hanya mengenali istilah-istilah dasar seperti Pod, Service, atau Deployment. Ia tidak tahu apa itu ApplicationSet. jadi ini disebut `CRD (Custom Resource Definition)`. Akar error ini kemungkinan Saat kamu menginstal ArgoCD sebelumnya proses instalasinya terputus di tengah jalan. Kubernetes belum sempat membaca dan menyimpan CRD milik ArgoCD, tapi komponen controller-nya sudah keburu menyala dan mencari CRD tersebut.

```bash
ubuntu@master-node:~$ k get pod -n argocd

NAME READY STATUS RESTARTS AGE

argocd-application-controller-0 1/1 Running 0 152m

argocd-applicationset-controller-cf6df4d44-z84ns 0/1 CrashLoopBackOff 24 (5m2s ago) 153m

argocd-dex-server-694c74cbb8-fkp5t 1/1 Running 2 (152m ago) 153m

argocd-notifications-controller-67dd6d74b5-8d27d 1/1 Running 0 153m

argocd-redis-85b9d55dc4-rfgtp 1/1 Running 0 153m

argocd-repo-server-c5b4cc57-vjntc 1/1 Running 0 76s

argocd-server-79fdfd7f5b-vspg5 1/1 Running 0 153m

ubuntu@master-node:~$ k logs argocd-applicationset-controller-cf6df4d44-z84ns -n argocd

{"built":"2026-03-27T13:41:15Z","commit":"998fb59dc355653c0657908a6ea2f87136e022d1","level":"info","msg":"ArgoCD ApplicationSet Controller is starting","namespace":"argocd","time":"2026-04-10T13:51:47Z","version":"v3.3.6"}

{"level":"info","msg":"Starting configmap/secret informers","time":"2026-04-10T13:51:47Z"}

{"level":"info","msg":"Configmap/secret informer synced","time":"2026-04-10T13:51:47Z"}

{"level":"info","msg":"Cluster cache informer synced","time":"2026-04-10T13:51:47Z"}

{"level":"info","msg":"Loading TLS configuration from secret argocd/argocd-secret","time":"2026-04-10T13:51:47Z"}

{"level":"info","msg":"Starting webhook server :7000","time":"2026-04-10T13:51:47Z"}

{"level":"info","msg":"Starting manager","time":"2026-04-10T13:51:47Z"}

{"level":"info","logger":"controller-runtime.metrics","msg":"Starting metrics server","time":"2026-04-10T13:51:47Z"}

{"addr":"[::]:8081","level":"info","msg":"starting server","name":"health probe","time":"2026-04-10T13:51:47Z"}

{"bindAddress":":8080","level":"info","logger":"controller-runtime.metrics","msg":"Serving metrics server","secure":false,"time":"2026-04-10T13:51:47Z"}

{"controller":"applicationset","controllerGroup":"argoproj.io","controllerKind":"ApplicationSet","level":"info","msg":"Starting EventSource","source":"kind source: *v1.Secret","time":"2026-04-10T13:51:47Z"}

{"controller":"applicationset","controllerGroup":"argoproj.io","controllerKind":"ApplicationSet","level":"info","msg":"Starting EventSource","source":"kind source: *v1alpha1.ApplicationSet","time":"2026-04-10T13:51:47Z"}

{"controller":"applicationset","controllerGroup":"argoproj.io","controllerKind":"ApplicationSet","level":"info","msg":"Starting EventSource","source":"kind source: *v1alpha1.Application","time":"2026-04-10T13:51:47Z"}

{"error":"failed to get restmapping: no matches for kind \"ApplicationSet\" in version \"argoproj.io/v1alpha1\"","kind":"ApplicationSet.argoproj.io","level":"error","logger":"controller-runtime.source.Kind","msg":"if kind is a CRD, it should be installed before calling Start","time":"2026-04-10T13:51:47Z"}

{"error":"failed to get restmapping: no matches for kind \"ApplicationSet\" in version \"argoproj.io/v1alpha1\"","kind":"ApplicationSet.argoproj.io","level":"error","logger":"controller-runtime.source.Kind","msg":"if kind is a CRD, it should be installed before calling Start","time":"2026-04-10T13:51:57Z"}
```

Solusi Fix:
1. Kita hanya perlu menyuntikan ulang CRD itu agar cluster kubernetes paham
```bash
kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.6/manifests/crds/applicationset-crd.yaml
```

Namun akan terjadi error lagi seperti berikut 
```bash
ubuntu@master-node:~$ kubectl apply -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.6/manifests/crds/applicationset-crd.yaml

The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations: Too long: may not be more than 262144 bytes
```

Analogi error tersebut karena Setiap kali kamu menggunakan perintah kubectl apply, Kubernetes memiliki kebiasaan unik: ia akan memfotokopi seluruh isi file YAML yang kamu kirim, lalu menempelkannya sebagai "Catatan Post-it" (bernama Annotation: last-applied-configuration) di objek tersebut. Tujuannya agar Kubernetes ingat riwayat perubahannya.

Masalahnya, file CRD ApplicationSet milik ArgoCD ini ukurannya raksasa (lebih dari 256 KB). Saat Kubernetes mencoba melipat file itu ke dalam sebuah "Post-it" berukuran kecil, Terlalu besar dan memunculkan error `Too long` tersebut.


Untuk mengatasi masalah itu kita akan menggunakan fitur `Server-Side Apply.`, Dengan menambahkan flag khusus ini, kita menyuruh Kubernetes: "Hei, jangan buat catatan Post-it di komputermu (Client). Langsung saja proses filenya di dalam otak utamamu (Server) agar tidak ada batasan ukuran karakter!". Jalankan kembali perintahmu, namun kali ini sisipkan flag `--server-side`:
```bash
kubectl apply --server-side -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.6/manifests/crds/applicationset-crd.yaml
```
Karena akan membutuhkan waktu untuk recovery kita hanya perlu menghapus pod yang sudah rusak, nanti kubernetes akan memperbaruinya secara otomatis
```bash
kubectl delete pod argocd-applicationset-controller -n argocd
```
Setelah itu coba cek kembali
```bash
kubectl get pod -n argocd
```
