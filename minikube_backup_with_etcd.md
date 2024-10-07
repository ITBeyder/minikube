
https://medium.com/@mehmetodabashi/backup-and-restore-etcd-cluster-on-kubernetes-93c19b1c070

# Backup Minikube with etcd

## 1. Install etcd CLI

First, SSH into your Minikube cluster:

```bash
minikube ssh
```

## 2. Retrieve etcd Endpoints and Certificates

Retrieve information about the etcd endpoints:

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml | grep listen
```

Example output:

```bash
- --listen-client-urls=https://127.0.0.1:2379,https://192.168.49.2:2379
- --listen-metrics-urls=http://127.0.0.1:2381
- --listen-peer-urls=https://192.168.49.2:2380
```

Retrieve certificate information:

```bash
sudo cat /etc/kubernetes/manifests/etcd.yaml | grep file
```

Example output:

```bash
- --cert-file=/var/lib/minikube/certs/etcd/server.crt
- --key-file=/var/lib/minikube/certs/etcd/server.key
- --peer-cert-file=/var/lib/minikube/certs/etcd/peer.crt
- --peer-key-file=/var/lib/minikube/certs/etcd/peer.key
- --peer-trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt
- --trusted-ca-file=/var/lib/minikube/certs/etcd/ca.crt
```

## 3. Install `etcdctl`

Run the following commands inside Minikube to install `etcdctl`:

```bash
sudo apt update
sudo apt install nano
sudo nano etcdcli.sh
```

Edit the `etcdcli.sh` file with the following content:

```bash
#!/bin/bash
ETCD_VERSION=${ETCD_VERSION:-v3.3.1}
curl -L https://github.com/coreos/etcd/releases/download/$ETCD_VERSION/etcd-$ETCD_VERSION-linux-amd64.tar.gz -o etcd-$ETCD_VERSION-linux-amd64.tar.gz
tar xzvf etcd-$ETCD_VERSION-linux-amd64.tar.gz
rm etcd-$ETCD_VERSION-linux-amd64.tar.gz
cd etcd-$ETCD_VERSION-linux-amd64
sudo cp etcd /usr/local/bin/
sudo cp etcdctl /usr/local/bin/
rm -rf etcd-$ETCD_VERSION-linux-amd64
etcdctl --version
```

Make the script executable and run it:

```bash
sudo chmod +x etcdcli.sh
./etcdcli.sh
```

## 4. Create an etcd Backup

Run the following command to create a backup of your etcd data:

```bash
sudo ETCDCTL_API=3 etcdctl --endpoints=https://127.0.0.1:2379 \
--cacert=/var/lib/minikube/certs/etcd/ca.crt \
--cert=/var/lib/minikube/certs/etcd/server.crt \
--key=/var/lib/minikube/certs/etcd/server.key \
snapshot save ./etcdbackup.db
```

## 5. Restore from Snapshot

To restore the etcd snapshot, run:

```bash
ETCDCTL_API=3 etcdctl --data-dir="/home/docker/etcdbackup" \
--endpoints=https://127.0.0.1:2379 \
--cacert=/var/lib/minikube/certs/etcd/ca.crt \
--cert=/var/lib/minikube/certs/etcd/server.crt \
--key=/var/lib/minikube/certs/etcd/server.key \
snapshot restore etcdbackup.db
```

## 6. Update `etcd.yaml` for Restored Data

After restoring the snapshot, edit the etcd manifest to use the new data directory. Open the `etcd.yaml` file:

```bash
sudo nano /etc/kubernetes/manifests/etcd.yaml
```

Change the `--data-dir` option to point to the restored directory:

```bash
- --data-dir=/home/docker/etcdbackup
```

## 7. Final `etcd.yaml` Example

Here is what the updated `etcd.yaml` file will look like after making the changes:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: etcd-minikube
  namespace: kube-system
spec:
  containers:
  - command:
    - etcd
    - --data-dir=/home/docker/etcdbackup
    ...
```
