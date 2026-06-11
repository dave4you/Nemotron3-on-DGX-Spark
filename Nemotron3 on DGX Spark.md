# Nemotron3 on DGX Spark

## Introduction

Before we start, a quick disclaimer: I am not a data scientist, AI engineer, or infrastructure specialist. I am a programme and project manager.

What drove me down this path was a combination of two observations. First, I saw AI SaaS costs increasing across projects and organizations. Second, I became increasingly aware of the privacy and governance implications of sending large amounts of company information to commercial AI providers.

At the same time, the availability of powerful open-source AI models has grown rapidly. Companies such as NVIDIA actively support this ecosystem. From a business perspective, NVIDIA benefits from selling hardware, while organizations benefit from having more freedom, flexibility, and control over how AI is deployed and used.

As someone responsible for delivering projects and managing budgets, I wanted to better understand what it actually means to run AI within an organization. Rather than starting with a large enterprise deployment, I decided to start small and learn by doing.

The journey was not always straightforward. I ran into configuration issues, deployment challenges, and plenty of technical concepts that were new to me. However, by combining the experience I had built throughout my career with the ability to research, experiment, and leverage tools such as ChatGPT, I gradually managed to get a working system up and running.

This manual is the result of that experience. It is written for people who may not consider themselves highly technical but who are curious about running AI locally and want more control over costs, privacy, and infrastructure. If you have basic computer knowledge, access to a DGX Spark environment, and a willingness to learn, this guide should help you make **NVIDIA Nemotron 3 Super** available on your cluster.

My goal is not to explain every technical detail. Instead, I aim to provide a practical, straightforward guide that helps you get from zero to a working deployment as quickly as possible.


## Usefull sources
[link text](https://build.nvidia.com/station)
[link text](https://deepwiki.com/NVIDIA/dgx-spark-playbooks)


## Prerequisites
To follow this guide, you will need:

1. Basic understanding of Git and command-line interfaces.
2. Basic understanding of AI models and their applications.
3. Some experience with Networking and Linux command line.
4. Some experience with Docker and containerization concepts.

_If you do not have these, I recommend taking some time to familiarize yourself with these topics before proceeding. There are many online resources available that can help you get up to speed.
or do get some help from a colleague or friend who has this experience._

5. Make sure your DGX Spark environment is up to data (use NVIDIA SYNC to check)
6. Have a 2 node spark cluster running on DGX Spark and a **proper description of the interfaces and IP addresses**. Follow the instructions provided by NVIDIA to get your cluster up and running. Tip: instead of scripting, use the Cluster Config UI available on **NVIDIA Sync**, which is more user-friendly and less error-prone. 

## My Environment
My DGX Spark environments consists of 2 Acer Veriton GN100 AI Mini Workstations directly connected through a QSFP/CX7 cable for high-performance interconnect.
I chose the Veriton GN100 AI Mini Workstation for its strong thermal design. An independent [review](https://www.storagereview.com/review/nvidia-dgx-spark-thermal-test-how-oem-cooling-designs-stack-up) found its cooling design to be particularly effective, delivering excellent cooling performance under load, which is critical for maintaining sustained AI and compute-intensive workloads.

## Nemotron 3
### Download Nemotron 3
For me step 1 was to download the Nemotron 3 model. There are mutiple ways to do this, but I found it easiest to use the command line interface (CLI) provided by NVIDIA. This allows you to download the model directly to your local machine or to a location on your cluster.

The models are listed here: [NVIDIA Models](https://build.nvidia.com/nvidia)


I created a /models/Nemotron120B/ directory on my local machine and used the following command to download the model:
```bash
uvx hf download \
  nvidia/NVIDIA-Nemotron-3-Super-120B-A12B-NVFP4 \
  --local-dir /home/<username>/models/Nemotron120B
 ```

### Copy model to both nodes
I also copied the downloaded model to the other node in my cluster. This is important because the model needs to be accessible from both nodes for it to work properly. 
I used the following command to copy the model to node2:

```bash
rsync -av --progress --mkpath \
  /home/<username>/models/Nemotron120B/ \
  <username>@<ip address node 2>:/home/<username>/models/Nemotron120B/
```

I used the ip address of the high-speed 200GbE direct QSFP connection. Why wait for the model to copy over the slower Wifi/1Gbe/10GbE network when you can use the faster connection between the nodes? This significantly reduced the time it took to transfer the model, allowing me to get up and running much faster.

### Use the following steps derived from https://github.com/eugr
I stumbled upon this repository, which provided a clear and concise set of instructions for running models on DGX Spark. 
It circumvented compatibility issues between NVDIA's DGX OS, NVIDIA SMI, CUDUA and associated drivers as well as model quantization compatibility issues (MIXED_PRECISION) and allowed me to get the model up and running without having to troubleshoot these issues myself (which I tried but could not solve due to the technical complexity of the issues).


**1. Set up passwordless ssh. On spark**
```
wget https://raw.githubusercontent.com/NVIDIA/dgx-spark-playbooks/refs/heads/main/nvidia/connect-two-sparks/assets/discover-sparks
chmod +x discover-sparks
./discover-sparks
```

**2. Make the scripts from the eugr repository locally available**
```
git clone https://github.com/eugr/spark-vllm-docker.git
cd spark-vllm-docker
```

**3. Build and verify**
3.1 Build the docker image and copy it to the cluster nodes. This will take a while, so be patient. The -c flag will also copy the image to the cluster nodes after building it.
```
./build-and-copy.sh -c
``` 

3.2	Verify
Verify that the image is available on both nodes by running the following command on each node:
```
docker images | grep vllm-node
```


**4. Launch and verify**
4.1. Launch the cluster and run the model. This command will start the cluster and run the model in the background. The -d flag will run the cluster in detached mode, allowing you to continue using the terminal while the cluster is running. The -n flag specifies the IP addresses of the nodes in the cluster, and the --eth-if flag specifies the network interface to use for communication between the nodes. The --master-port flag specifies the port to use for communication between the master and worker nodes. The rest of the command specifies the parameters for running vllm serve, including the model path, port, host, GPU memory utilization, tensor parallelism, pipeline parallelism, distributed executor backend, maximum model length, and load format.
```
export VLLM_SPARK_EXTRA_DOCKER_ARGS="-v /home/<username>/models/Nemotron120B:/models/Nemotron120B:ro"

./launch-cluster.sh \
  -d \
  -n <ip address node 1>,<ip address node 2> \
  --eth-if enp1s0f0np0 \
  --master-port 29501 \
  exec vllm serve \
  /models/Nemotron120B \
  --port 8000 \
  --host 0.0.0.0 \
  --gpu-memory-utilization 0.7 \
  -tp 1 \
  -pp 2 \
  --distributed-executor-backend ray \
  --max-model-len 4096 \
  --load-format fastsafetensors
```

The script will:

1. Start the cluster containers if needed.
2. Start Ray head and worker.
3. Execute vllm serve to run in the background (docker)
4. Return immediately to the shell

**4.2 Verify**

Verify that the model is running by checking the logs of the cluster containers. You can do this by running the following command:

```
docker ps --filter "name=vllm_node" --format "table {{.Names}}\t{{.Status}}"
```

**5. Test**

Testing is someting that can be done in many ways. I leave the model running and use **vllm** to test the model by sending a request to the API endpoint. 

The idea is that we use vllm to send a request to the model and measure its performance. The NO_PROXY environment variable is set to ensure that the request is sent directly to the model without going through a proxy server.


The nice thing is: vllm is inside the Docker container.

The complete command looks like this:

```
docker exec -it vllm_node bash -lc '
NO_PROXY=<ip address node 1> no_proxy=<ip address node 2> \
vllm bench serve \
  --backend openai-chat \
  --base-url http://<ip address node 1>:8000 \
  --endpoint /v1/chat/completions \
  --model /models/Nemotron120B \
  --tokenizer /models/Nemotron120B \
  --dataset-name random \
  --random-input-len 1024 \
  --random-output-len 512 \
  --num-prompts 128 \
  --max-concurrency 8 \
  --temperature 0
'
```

The command specifies the backend to use (openai-chat compatible), the base URL for the API, the endpoint for chat completions, the model and tokenizer paths, the dataset to use for testing (random), the input and output lengths, the number of prompts to test, the maximum concurrency, and the temperature for generating responses.
If all goes while, after a while you will have your test results and can start using the model for your own applications!

Example test results:





## Glossary

**DGX Spark**

A platform that allows users to run large-scale AI workloads on NVIDIA's DGX systems, which are optimized for deep learning and AI applications.

**Docker**

Docker is an OS‑level virtualization (or containerization) platform, which allows applications to share the host OS kernel instead of running a separate guest OS like in traditional virtualization. 

**GitHub**

The term GitHub is derived from two parts — Git and Hub.

_Git_: A distributed version control system created by Linus Torvalds, used to track changes in source code during software development.

_Hub_: Represents a centralized platform where developers can host, share, and collaborate on Git repositories.

So, GitHub essentially means a hub for Git repositories, enabling developers worldwide to collaborate on projects efficiently.

**Nemotron3**

An open AI model family developed by NVIDIA. It is designed to be a powerful and efficient language model that can be used for a variety of natural language processing tasks. The Nemotron 3 family consists of three models: Nano, Super, and Ultra. 
Nano has 30B parameters, Super has 120B parameters and Ultra has 530B parameters. The larger the model, the more powerful it is, but also the more computational resources it requires to run.
Super is the model I used for this tutorial, as it provides a good balance between performance and resource requirements (almost uses all of the 128GB memory of the DGX Spark).

**SSH**

SSH (Secure Shell) is a network protocol that establishes encrypted connections between computers for secure remote access. It operates on TCP port 22 and provides authentication, encryption, and integrity to protect data transmitted over unsecured networks.


