## Node 1 (RAY HEAD)

`export HEAD\_IP=10.100.72.1

export NODE\_IP=$HEAD\_IP

export IFACE=enp1s0f0np0



docker run -d \\

&#x20; --name nemotron \\

&#x20; --gpus all \\

&#x20; --network host \\

&#x20; --shm-size 32g \\

&#x20; -e VLLM\_HOST\_IP=$NODE\_IP \\

&#x20; -e GLOO\_SOCKET\_IFNAME=$IFACE \\

&#x20; -e NCCL\_SOCKET\_IFNAME=$IFACE \\

&#x20; -e TP\_SOCKET\_IFNAME=$IFACE \\

&#x20; -v \~/models/Nemotron120B:/models/Nemotron120B \\

&#x20; nvcr.io/nvidia/vllm:26.05.post1-py3 \\

&#x20; vllm serve /models/Nemotron120B \\

&#x20;   --host 0.0.0.0 \\

&#x20;   --port 8000 \\

&#x20;   --tensor-parallel-size 1 \\

&#x20;   --pipeline-parallel-size 2 \\

&#x20;   --distributed-executor-backend ray \\

&#x20;   --gpu-memory-utilization 0.80 \\

&#x20;   --max-model-len 4096 \\

&#x20;   --trust-remote-code`



## Node 2 (WORKER)

`export HEAD\_IP=10.100.72.1

export NODE\_IP=10.100.72.2

export IFACE=enp1s0f0np0



docker run -d \\

&#x20; --name nemotron-worker \\

&#x20; --gpus all \\

&#x20; --network host \\

&#x20; --shm-size 32g \\

&#x20; -e VLLM\_HOST\_IP=$NODE\_IP \\

&#x20; -e GLOO\_SOCKET\_IFNAME=$IFACE \\

&#x20; -e NCCL\_SOCKET\_IFNAME=$IFACE \\

&#x20; -e TP\_SOCKET\_IFNAME=$IFACE \\

&#x20; -v \~/models/Nemotron120B:/models/Nemotron120B \\

&#x20; nvcr.io/nvidia/vllm:26.05.post1-py3 \\

&#x20; bash -lc "ray start --address=$HEAD\_IP:6379 --node-ip-address=$NODE\_IP --block"`

