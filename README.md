# raspbernetes

# Instalaciones:

# Deshabilitar wifi y Bluetooth
```
  systemctl disable wpa_supplicant
  systemctl disable bluetooth
  systemctl disable hciuart
```
  Añadir a /boot/config.txt:
``` 
  dtoverlay=pi3-disable-wifi
  dtoverlay=pi3-disable-bt
```
# Instalar kubernetes:


  IP estática:
  
  Añadir a /etc/dhcpcd.conf:
  ```
  interface eth0
  static ip_address=192.168.0.104/24
  static routers=192.168.0.1
  static domain_name_servers=192.168.0.1
  ```
  Install Docker:
  ```
  curl -sSL get.docker.com | sh && \
  sudo usermod pi -aG docker
  newgrp docker
```
  Disable swap:
  ```
  sudo dphys-swapfile swapoff && \
  sudo dphys-swapfile uninstall && \
  sudo update-rc.d dphys-swapfile remove
  ```
  Añadir al final de la linea /boot/cmdline.txt
  ```
  cgroup_enable=cpuset cgroup_memory=1 cgroup_enable=memory
  ```
  
  REBOOT
  
  Añadir repositorio e instalar kubernetes
  ```
  curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add - && \
  echo "deb http://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list && \
  sudo apt-get update -q && \
  sudo apt-get install -qy kubeadm
```
  Instalar Helm:
  ```
  wget https://kubernetes-helm.storage.googleapis.com/helm-v2.12.2-linux-arm.tar.gz
  tar xvzf helm-$HELM_VERSION-linux-arm.tar.gz
  ```  

Inicio:

Para precargar los contenedores iniciales de despliegue
```
sudo kubeadm config images pull

sudo kubeadm init

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
Configurar res:
```
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"

kubectl get nodes
kubectl get pods --all-namespaces
```

Dashboard:
```
kubectl create -f https://raw.githubusercontent.com/kubernetes/dashboard/master/aio/deploy/recommended/kubernetes-dashboard-arm.yaml

grep 'client-certificate-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.crt
grep 'client-key-data' ~/.kube/config | head -n 1 | awk '{print $2}' | base64 -d >> kubecfg.key
openssl pkcs12 -export -clcerts -inkey kubecfg.key -in kubecfg.crt -out kubecfg.p12 -name "kubernetes-client"
```

Crear Usuario:
```
cat <<EOF | kubectl create -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin-user
  namespace: kube-system
EOF

cat <<EOF | kubectl create -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin-user
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin-user
  namespace: kube-system
EOF
```
```
kubectl -n kube-system describe secret $(kubectl -n kube-system get secret | grep admin-user | awk '{print $1}')
```

-Acceso:
```
https://{dns}:6443/api/v1/namespaces/kube-system/services/https:kubernetes-dashboard:/proxy/
```

Join a posteriori
```
kubeadm token create --print-join-command
```

# Montar disco para gluster:

-Estado actual del montaje:
lsblk

-particionar:
```
sudo fdisk -w auto /dev/sda
```
-formatear:
```
sudo mkfs.xfs -f -L myvol-brick1 /dev/sda1
```
-montar:
```
sudo mkdir -p /data/glusterfs/k8s-volume/brick1
printf $(sudo blkid -o export /dev/sda1|grep PARTUUID)" /data/glusterfs/k8s-volume/brick1 xfs defaults,noatime 1 2\n" | sudo tee -a /etc/fstab
sudo mount /data/glusterfs/k8s-volume/brick1
```
