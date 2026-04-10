# ArgoCD
Argo CD adalah sebuah GitOps continuous delivery tool untuk Kubernetes. Sebelum ke detailnya, kita pahami dulu masalah yang dia selesaikan.

### Masalah sebelum Argo CD
Cara lama deploy ke K8s: developer push code → CI pipeline build → lalu seseorang (atau script) jalankan kubectl apply dari luar cluster. Ini model "push" — artinya ada pihak eksternal yang aktif mendorong perubahan ke cluster. Masalahnya: tidak ada yang menjaga apakah kondisi cluster tetap sesuai dengan yang seharusnya. Kalau ada yang iseng kubectl edit langsung di production, tidak ada yang tahu.
Argo CD hadir dengan pendekatan berbeda: GitOps. Git adalah satu-satunya sumber kebenaran. Argo CD yang aktif "menarik" dan memastikan cluster selalu sinkron dengan Git — bukan sebaliknya.
![tp-after-before-argocd](./image/1.png)
