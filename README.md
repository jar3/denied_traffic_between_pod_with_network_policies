# Kubernetes Network Policy Example: Deny Traffic Between Pods

This guide demonstrates how to create and apply a Kubernetes Network Policy to deny traffic between pods across namespaces while allowing communication within the same namespace. It also includes steps to set up the environment and test the behavior.

## 1. Create the Cluster

### Add Nodes to the Cluster
1. **Node 1**:
   ```bash
   kubeadm init --apiserver-advertise-address $(hostname -i) --pod-network-cidr 10.5.0.0/16
   kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml
   alias k='kubectl'
   ```

2. **Node 2**:
   ```bash
   kubeadm join 192.168.0.18:6443 --token <TOKEN> --discovery-token-ca-cert-hash <HASH>
   ```

## 2. Deploy an Nginx Pod

1. **Run Nginx Pod**:
   ```bash
   k run www --image=nginx --labels="app=www" --expose --port=80
   ```

2. **Verify Pod and Service**:
   ```bash
   k get pods
   k get svc
   ```

   Example output:
   ```
   NAME  READY  STATUS       RESTARTS  AGE
   www   1/1    Running      0         7s

   NAME         TYPE        CLUSTER-IP      EXTERNAL-IP  PORT(S)  AGE
   kubernetes   ClusterIP   10.96.0.1       <none>       443/TCP  10m
   www          ClusterIP   10.108.115.37   <none>       80/TCP   17s
   ```

## 3. Create Network Policy

1. **Define Network Policy**:
   Create `deny-namespace.yaml`:
   ```yaml
   kind: NetworkPolicy
   apiVersion: networking.k8s.io/v1
   metadata:
     namespace: default
     name: deny-namespaces
   spec:
     podSelector:
       matchLabels: {}
     ingress:
       - from:
           - podSelector: {}
   ```

2. **Apply the Network Policy**:
   ```bash
   k apply -f deny-namespace.yaml
   ```

3. **Verify the Network Policy**:
   ```bash
   k get networkpolicy
   ```
   Example output:
   ```
   NAME              POD-SELECTOR  AGE
   deny-namespaces   <none>        34s
   ```

## 4. Create a New Namespace

```bash
k create namespace foo
```

## 5. Test Connectivity Between Namespaces

1. **Access from `foo` Namespace**:
   ```bash
   k run test-$RANDOM --namespace=foo --rm -i -t --image=alpine -- sh
   wget -qO- --timeout=2 http://www.default
   ```

   Expected output:
   ```
   wget: can't connect to remote host (10.108.115.37): Connection refused
   ```

2. **Access from `default` Namespace**:
   ```bash
   k run test-$RANDOM --namespace=default --rm -i -t --image=alpine -- sh
   wget -qO- --timeout=2 http://www.default
   ```

   Expected output:
   ```html
   <!DOCTYPE html>
   <html>
   <head>
   <title>Welcome to nginx!</title>
   ...
   </html>
   ```
