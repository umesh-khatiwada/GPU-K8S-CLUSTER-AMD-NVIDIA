
Don't you just love it when you submit a PR and it turns out that no code is needed? That's exactly what happened when I tried add GPU support to Kind.
In this blog post you will learn how to configure Kind such that it can use the GPUs on your device. Credit to @klueska for the solution.

Install the NVIDIA container toolkit by following the official install docs.
Configure NVIDIA to be the default runtime for docker:

```
sudo nvidia-ctk runtime configure --runtime=docker --set-as-default
sudo systemctl restart docker
```

Set accept-nvidia-visible-devices-as-volume-mounts = true in /etc/nvidia-container-runtime/config.toml:
```
sudo sed -i '/accept-nvidia-visible-devices-as-volume-mounts/c\accept-nvidia-visible-devices-as-volume-mounts = true' /etc/nvidia-container-runtime/config.toml
```


Create a Kind Cluster:
```
kind create cluster --name substratus --config - <<EOF
apiVersion: kind.x-k8s.io/v1alpha4
kind: Cluster
nodes:
- role: control-plane
  image: kindest/node:v1.27.3@sha256:3966ac761ae0136263ffdb6cfd4db23ef8a83cba8a463690e98317add2c9ba72
  # required for GPU workaround
  extraMounts:
    - hostPath: /dev/null
      containerPath: /var/run/nvidia-container-devices/all
EOF
```




# https://github.com/NVIDIA/nvidia-docker/issues/614#issuecomment-423991632
docker exec -ti substratus-control-plane ln -s /sbin/ldconfig /sbin/ldconfig.real


Install the K8s NVIDIA GPU operator so K8s is aware of your NVIDIA device:

helm repo add nvidia https://helm.ngc.nvidia.com/nvidia || true
helm repo update
helm install --wait --generate-name \
     -n gpu-operator --create-namespace \
     nvidia/gpu-operator --set driver.enabled=false



You should now have a working Kind cluster that can access your GPU. Verify it by running a simple pod:

kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: cuda-vectoradd
spec:
  restartPolicy: OnFailure
  containers:
  - name: cuda-vectoradd
    image: "nvcr.io/nvidia/k8s/cuda-sample:vectoradd-cuda11.7.1-ubuntu20.04"
    resources:
      limits:
        nvidia.com/gpu: 1
EOF