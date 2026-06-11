

`docker run -d \
  --name nemotron \
  --gpus all \
  --network host \
  --shm-size 32g \
  -v ~/models/Nemotron120B:/models/Nemotron120B \
  nvcr.io/nvidia/vllm:26.05.post1-py3 \
  vllm serve /models/Nemotron120B \
    --host 0.0.0.0 \
    --port 8000 \
    --tensor-parallel-size 1 \
    --pipeline-parallel-size 2 \
    --distributed-executor-backend ray \
    --gpu-memory-utilization 0.80 \
    --max-model-len 4096 \
    --trust-remote-code`