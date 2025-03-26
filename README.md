# NFS-Server-Setup-K8S-Cluster

## Go through the complete documentation and understand it before doing installation because u need to setup server which is master and client which is worker.

This guide provides step-by-step instructions to configure an NFS (Network File System) server and use it as persistent storage in a Kubernetes cluster.

---

## 📌 Step 1: Install NFS Server on a Node
Choose one of your EC2 instances (e.g., master or a dedicated storage node) as the NFS Server.

### 1.1 Install NFS Server
```sh
sudo apt update -y
sudo apt install -y nfs-kernel-server
```

### 1.2 Create an NFS Share Directory
```sh
sudo mkdir -p /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share
```

### 1.3 Configure NFS Exports
Edit the exports file:
```sh
sudo nano /etc/exports
```
Add the following line (adjust the IP range to match your Kubernetes worker nodes' subnet):
```sh
/mnt/nfs_share *(rw,sync,no_subtree_check,no_root_squash)
```
Save and exit.

### 1.4 Apply Export Configuration
```sh
sudo exportfs -ra
```

### 1.5 Start and Enable the NFS Server
```sh
sudo systemctl restart nfs-server
sudo systemctl enable nfs-server
```

### 1.6 Verify NFS is Running
```sh
sudo systemctl status nfs-server
```
Ensure the service is running before proceeding.

---

## 📌 Step 2: Install NFS Client on Worker Nodes
Run the following commands on each Kubernetes worker node:
```sh
sudo apt update -y
sudo apt install -y nfs-common
```

Test the NFS connection from a worker node:
```sh
sudo mount -t nfs <NFS_SERVER_IP>:/mnt/nfs_share /mnt
```
Check if it's mounted:
```sh
df -h | grep nfs
```
If successful, unmount it:
```sh
sudo umount /mnt
```

---

## 📌 Step 3: Create an NFS Persistent Volume in Kubernetes

### 3.1 Create Persistent Volume (PV)
Create `nfs-pv.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  nfs:
    path: /mnt/nfs_share
    server: <NFS_SERVER_IP>
```
Apply it:
```sh
kubectl apply -f nfs-pv.yaml
```

### 3.2 Create Persistent Volume Claim (PVC)
Create `nfs-pvc.yaml`:
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-pvc
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 5Gi
```
Apply it:
```sh
kubectl apply -f nfs-pvc.yaml
```

---

## 📌 Step 4: Deploy a Pod Using NFS Storage
Create `nfs-test-pod.yaml`:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nfs-test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: ["/bin/sh", "-c", "sleep 3600"]
      volumeMounts:
        - mountPath: "/mnt/nfs"
          name: nfs-volume
  volumes:
    - name: nfs-volume
      persistentVolumeClaim:
        claimName: nfs-pvc
```
Apply it:
```sh
kubectl apply -f nfs-test-pod.yaml
```

Check if the pod is running:
```sh
kubectl get pods
```
Enter the pod and verify the mount:
```sh
kubectl exec -it nfs-test-pod -- df -h
```
Expected output:
```
<NFS_SERVER_IP>:/mnt/nfs_share  5.0G  0G  5.0G  /mnt/nfs
```

Test writing a file inside the pod:
```sh
kubectl exec -it nfs-test-pod -- touch /mnt/nfs/testfile
```
Check on the NFS server:
```sh
ls -l /mnt/nfs_share/
```
If `testfile` appears, the setup is successful! 🎉

---

## 🚀 Next Steps
✅ Successfully set up NFS storage for Kubernetes!
🔹 If needed, you can use NFS StorageClass to provision dynamic storage.
❌ If you face any issues, open an issue in this repository!

---

📌 **Author**: Sai Veera Teja

