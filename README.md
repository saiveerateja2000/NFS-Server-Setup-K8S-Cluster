# Kubernetes NFS Setup and Usage with Persistent Volumes

## Step 1: Check Kubernetes Nodes
Run the following command to check the available nodes in the cluster:

```bash
kubectl get nodes
```

Example output:

```bash
NAME               STATUS   ROLES           AGE    VERSION
ip-172-31-12-235   Ready    <none>          6h2m   v1.29.15
ip-172-31-2-209    Ready    control-plane   6h3m   v1.29.15
```

## Step 2: Setup NFS Server on Control Plane (172.31.2.209)
### 1. Install NFS Server
```bash
sudo apt update
sudo apt install -y nfs-kernel-server
```

### 2. Create NFS Share Directory
```bash
sudo mkdir -p /mnt/nfs_share
sudo chmod 777 /mnt/nfs_share  # Give full access for testing
sudo chown nobody:nogroup /mnt/nfs_share  # Set ownership
```

### 3. Configure NFS Export
Edit the exports file:
```bash
sudo nano /etc/exports
```
Add this line:
```bash
/mnt/nfs_share 172.31.12.235(rw,sync,no_subtree_check,no_root_squash)
```
Save and exit.

### 4. Apply and Restart NFS
```bash
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

## Step 3: Setup NFS Client on Worker Node (172.31.12.235)
### 1. Install NFS Client
```bash
sudo apt update
sudo apt install -y nfs-common
```

### 2. Create Mount Directory
```bash
sudo mkdir -p /mnt/nfs_client
```

### 3. Mount NFS Share
```bash
sudo mount 172.31.2.209:/mnt/nfs_share /mnt/nfs_client
```

To check if it is mounted:
```bash
df -h | grep nfs
```

## Step 4: Verify NFS Functionality
### 1. Create a File on Server
```bash
echo "Hello from NFS Server" | sudo tee /mnt/nfs_share/serverfile.txt
```

### 2. Check File on Client
```bash
ls /mnt/nfs_client
cat /mnt/nfs_client/serverfile.txt
```

### 3. Create a File on Client
```bash
echo "Hello from NFS Client" | sudo tee /mnt/nfs_client/clientfile.txt
```

### 4. Check File on Server
```bash
ls /mnt/nfs_share
cat /mnt/nfs_share/clientfile.txt
```

## Step 5: Make Mount Persistent (Optional)
To ensure the mount persists after a reboot, add this line to `/etc/fstab` on the client (172.31.12.235):
```bash
172.31.2.209:/mnt/nfs_share /mnt/nfs_client nfs defaults 0 0
```
Then reload:
```bash
sudo mount -a
```

## Step 6: Use NFS in Kubernetes with PV and PVC
### 1. Create PersistentVolume (PV)
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
  nfs:
    path: /mnt/nfs_share
    server: 172.31.2.209
  persistentVolumeReclaimPolicy: Retain
```
Apply it:
```bash
kubectl apply -f pv.yaml
```

### 2. Create PersistentVolumeClaim (PVC)
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
      storage: 2Gi
```
Apply it:
```bash
kubectl apply -f pvc.yaml
```

### 3. Deploy an Application Using NFS Storage
Example Nginx deployment using the PVC:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx
          volumeMounts:
            - mountPath: /usr/share/nginx/html
              name: nfs-storage
      volumes:
        - name: nfs-storage
          persistentVolumeClaim:
            claimName: nfs-pvc
```
Apply it:
```bash
kubectl apply -f deployment.yaml
```

### 4. Verify Everything
Check PV, PVC, and Pod:
```bash
kubectl get pv
kubectl get pvc
kubectl get pods
```

Inside the running Nginx pod:
```bash
kubectl exec -it <nginx-pod-name> -- sh
ls /usr/share/nginx/html
```

Any files written in `/usr/share/nginx/html` will be stored persistently in NFS!

### Summary:
✅ Data is shared across pods
✅ Data persists even if the pod restarts
✅ Multiple pods can access the same data (ReadWriteMany)

This setup ensures a clear, structured, and reliable NFS integration with Kubernetes. 🚀

