# Hosting LLama 3.1 405B on Azure A100 VM's

## Introduction

- Gettin nvidia NIM llama 3.1 405B on Azure A100 VM's: Standard_ND96amsr_A100_v4
- Using Kubernetes to deploy the model
- Based on NVIDIA's recommendation on compute matrix.

## Prerequisites

- Azure subscription
- Get A100 VM's from Azure: Standard_ND96amsr_A100_v4
- Get NVIDIA NIM Llama 3.1 405B from NVIDIA NGC
- Need to install kubernets on 3 nodes minimum, one master and 2 worker nodes.
- Check this documentation for nodes needs: https://docs.nvidia.com/nim/large-language-models/latest/support-matrix.html
- Scroll down to Llama 3.1 405B Instruct for hardware requirements.
  
## Steps

- First install kubernetes on the nodes - multi node setup
- Need one master and one worker as minimum using the above GPU SKU

```
SYSTEM INFO
- Free GPUs:
  -  [20b2:10de] (0) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (1) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (2) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (3) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (4) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (5) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (6) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
  -  [20b2:10de] (7) NVIDIA A100-SXM4-80GB (A100 80GB) [current utilization: 0%]
MODEL PROFILES
- Compatible with system and runnable: <None>
- Incompatible with system:
  - b80e254301eff63d87b9aa13953485090e3154ca03d75ec8eff19b224918c2b5 (tensorrt_llm-h100-fp8-tp8-latency)
  - 8860fe7519bece6fdcb642b907e07954a0b896dbb1b77e1248a873d8a1287971 (tensorrt_llm-h100-fp8-tp8-throughput)
  - f8bf5df73b131c5a64c65a0671dab6cf987836eb58eb69f2a877c4a459fd2e34 (tensorrt_llm-a100-fp16-tp8-latency)
  - b02b0fe7ec18cb1af9a80b46650cf6e3195b2efa4c07a521e9a90053c4292407 (tensorrt_llm-h100-fp16-tp8-latency)

```

- Install kubelet, kubeadm, and kubectl on all nodes, with containerd as the container runtime.
- Install CUDA drivers on the nodes.
- Setup NFS storage on the nodes. Use Blob storage for all nodes to share the model.
- If Shared storage NFS is not available deployment will fail.
- create a Persistent Volume (PV) which allowed Kubernetes to automatically create the required Persistent Volume Claim (PVC) and bind it to the PV.
- Install GPU operator: https://github.com/NVIDIA/gpu-operator
- Install network operators on the nodes. https://docs.nvidia.com/networking/display/kubernetes2410/nvidia+network+operator
- Network operator is needed for CUDA to work across multiple nodes for large language models.
- Here is the documenenation for LWS in Kubernetes: https://github.com/kubernetes-sigs/lws/blob/main/docs/setup/install.md
- Now download the NIM Llama 3.1 405B from NVIDIA NGC and install it on the nodes.
- Here is the docker image: http://nvcr.io/nim/meta/llama-3.1-405b-instruct:1.3.0 
- Defineity need NVIDIA KEY to access the image.
- Here is the command to check the containers running

```
kubectl get pods -n min -o wide
```

- Here is the help script to install the model: https://catalog.ngc.nvidia.com/orgs/nim/helm-charts/nim-llm/files
- Get the Helm chart version 1.3.0 and install it on the nodes. Tested and worked.
- After image is download and installed, test the model and make sure it is working.

```
curl-X'POST'\
'http://10.244.1.126:8000/v1/chat/completions'\
-H'accept:application/json'\
-H'Content-Type:application/json'\
-d '{
 "model":"meta/llama-3.1-405b-instruct"
 "messages":[{"role":"user","content":"Explain Quantum computing in details"}],
 "max_tokens":64
 }'
```

- once you validate the output, you can now use the model for your application.
- Now we can move on to next running in AKS and Azure Machine learning

## Conclusion

- This is to show how to get started with NVIDIA NIM Llama 3.1 405B on Azure A100 VM's.
- Subjective to change based on the NVIDIA and Azure updates.
- Based on hardware requirements and software requirements above process might change.