[toc]

# An exmaple of BLOOM-175B inferencing

This note is for students who are not familiar with distributed BLOOM-175B inferencing on two V100x4 GPU nodes.

It provides a workable example of "ds-inference shard-int8", which is the AI task workload of the competition.

The steps have been tested on two GPU nodes from Gadi "gpuvolta" queue. 

## 1. Set up DeepSpeed Python environment (only on login node)

### 1.1. Create a scratch directory and offline model directory

```bash
export BLOOM_DIR=/scratch/${YourGroupID}/${YourUserID}/BLOOM # Replace the variable values with your own group and user id
export BLOOM_DIR=/scratch/il82/pz7344/BLOOM # Pengzhi's environment as an example
mkdir -p ${BLOOM_DIR} && cd ${BLOOM_DIR}
mkdir offline_models
```

### 1.2. Install and activate miniconda
```bash
time wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ${BLOOM_DIR}/miniconda.sh
time bash ${BLOOM_DIR}/miniconda.sh -b -p ${HOME}/miniconda3

source ${HOME}/miniconda3/etc/profile.d/conda.sh
conda init
conda config --set auto_activate_base false

# real	0m28.179s
```

### 1.3. Create Python 3.8 DeepSpeed Conda environment

```bash
cd ${BLOOM_DIR}
time conda create -p deepspeed073/p38 python=3.8 -y
time deepspeed073/p38/bin/pip --cache-dir /scratch/public/HPC-AI-BLOOM/cache/pip install torch==1.12.1+cu116 --extra-index-url https://download.pytorch.org/whl/cu116 transformers==4.26.1 deepspeed==0.7.6 accelerate==0.16.0 gunicorn==20.1.0 flask flask_api fastapi==0.89.1 uvicorn==0.19.0 jinja2==3.1.2 pydantic==1.10.2 huggingface_hub==0.12.1 grpcio-tools==1.50.0 bitsandbytes -v
# real	1m1.448s
# real	3m39.252s
```

### 1.4. Download BLOOM-176B models from /scratch/public

```bash
# Sync the existing model files to your BLOOM directory
time rsync -avP /scratch/public/HPC-AI-BLOOM/models--microsoft--bloom-deepspeed-inference-fp16 ${BLOOM_DIR}/offline_models

# Create the following links, so that huggingface transformer can work on the offline GPU nodes
ln -s ${BLOOM_DIR}/offline_models ${HOME}/.cache/huggingface/hub -f


# The following command is used to download the DeepSpeed sharded int8 model to public shared storage
# deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="microsoft/bloom-deepspeed-inference-int8",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",ignore_patterns=["*.safetensors"],)'

# To get DeepSpeed fp16 and HugginFace models, simple run the following two commands
#deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="microsoft/bloom-deepspeed-inference-fp16",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",ignore_patterns=["*.safetensors"],)'

#deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="bigscience/bloom",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",)'
```

### 1.5. Edit ${HOME}/.deepspeed_env

Add variables to ${HOME}/.deepspeed_env

```bash
echo "CUDA_HOME=/apps/cuda/11.6.1" > ${HOME}/.deepspeed_env
echo "NCCL_DEBUG=INFO" >> ${HOME}/.deepspeed_env
echo "TORCH_DISTRIBUTED_DETAIL=DEBUG" >> ${HOME}/.deepspeed_env
echo "TRANSFORMERS_OFFLINE=1" >> ${HOME}/.deepspeed_env
echo "LD_LIBRARY_PATH=/apps/cuda/11.6.1/extras/CUPTI/lib64:/apps/cuda/11.6.1/lib64" >> ${HOME}/.deepspeed_env
echo "PDSH_SSH_ARGS='-o StrictHostKeyChecking=no'" >> ${HOME}/.deepspeed_env
```

### 1.6. Clone the BLOOM Inference project

```bash
mkdir ${BLOOM_DIR}/github -p
cd ${BLOOM_DIR}
git clone https://github.com/huggingface/transformers-bloom-inference
```

### 1.6. Install pdsh with ssh support

```bash
cd ${BLOOM_DIR}/github
git clone https://github.com/chaos/pdsh
cd pdsh
./bootstrap
./configure --prefix=${HOME}/.local --with-ssh
make install -j $(nproc)
```

### 1.7. Configure SSH authentication

On the login node, run the following command.

```
ssh -o PasswordAuthentication=no -o StrictHostKeyChecking=no localhost
```

If it works successfully, you don't need to configure ssh authentication in the cluster.
Else, run the following two ssh-keygen, then run the above "ssh localhost" to confirm that SSH authentication has been configured successfully. 

```bash
ssh-keygen -t ecdsa -f ${HOME}/.ssh/id_ecdsa -N "" -vvv
ssh-keygen -y -f ${HOME}/.ssh/id_ecdsa >> ${HOME}/.ssh/authorized_keys
```

## 2. Run DeepSpeed Infernece Benchmark (Need 2 GPU compute nodes)

### 2.1. Allocate two GPU nodes and start a interactive session

Interaction sessions are strongly discouraged. Please use batch scripts to run your inference tasks. 

```bash
qsub -q gpuvolta -I -l walltime=00:20:00,ncpus=96,mem=300gb,ngpus=8 -M 393958790@qq.com -m abe -P il82
```

### 2.3. Edit hostfile

You need to run the following sed command every time when you allocated nodes.

```bash
cat ${PBS_NODEFILE} | cut -f 1 -d . | sed -e 's/$/ slots=48/' | sort -u > ${HOME}/hostfile
```

### 2.5. Run the inference benchmark

```bash
export BLOOM_DIR=/scratch/${YourGroupID}/${YourUserID}/BLOOM # Replace the variable values with your own group and user id
export BLOOM_DIR=/scratch/il82/pz7344/BLOOM # Pengzhi's environment as an example

cd ${HOME}
time ${BLOOM_DIR}/deepspeed073/p38/bin/deepspeed --hostfile ${HOME}/hostfile --num_gpus 4 ${BLOOM_DIR}/github/transformers-bloom-inference/bloom-inference-scripts/bloom-ds-inference.py --name microsoft/bloom-deepspeed-inference-int8 --dtype int8 --benchmark --batch_size 8
```



## 3. How to read the results

**As the result is the time it takes to generate a token, the smaller the time value is, the better.**

```bash
Throughput per token including tokenize: 26.93 msecs
```

A PBS output example

```bash
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15834:16156 [2] NCCL INFO Channel 01 : 2[b1000] -> 6[b1000] [send] via NET/IB/0
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688396:1688736 [2] NCCL INFO Channel 01 : 6[b1000] -> 2[b1000] [send] via NET/IB/0
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15834:16156 [2] NCCL INFO Connected all trees
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15834:16156 [2] NCCL INFO threadThresholds 8/8/64 | 64/8/64 | 8/8/512
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15834:16156 [2] NCCL INFO 2 coll channels, 2 p2p channels, 1 p2p channels per peer
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15835:16059 [3] NCCL INFO Connected all trees
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15835:16059 [3] NCCL INFO threadThresholds 8/8/64 | 64/8/64 | 8/8/512
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15835:16059 [3] NCCL INFO 2 coll channels, 2 p2p channels, 1 p2p channels per peer
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15835:16059 [3] NCCL INFO comm 0x7fb4700030d0 rank 3 nranks 8 cudaDev 3 busId b2000 - Init COMPLETE
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15834:16156 [2] NCCL INFO comm 0x7fe9980030d0 rank 2 nranks 8 cudaDev 2 busId b1000 - Init COMPLETE
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688396:1688736 [2] NCCL INFO Connected all trees
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688396:1688736 [2] NCCL INFO threadThresholds 8/8/64 | 64/8/64 | 8/8/512
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688396:1688736 [2] NCCL INFO 2 coll channels, 2 p2p channels, 1 p2p channels per peer
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15833:16157 [1] NCCL INFO comm 0x7fa2000030d0 rank 1 nranks 8 cudaDev 1 busId 3e000 - Init COMPLETE
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15832:16058 [0] NCCL INFO comm 0x7fbf880030d0 rank 0 nranks 8 cudaDev 0 busId 3d000 - Init COMPLETE
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688397:1688738 [3] NCCL INFO Connected all trees
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688397:1688738 [3] NCCL INFO threadThresholds 8/8/64 | 64/8/64 | 8/8/512
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688397:1688738 [3] NCCL INFO 2 coll channels, 2 p2p channels, 1 p2p channels per peer
gadi-gpu-v100-0010: gadi-gpu-v100-0010:15832:15832 [0] NCCL INFO Launch mode Parallel
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688396:1688736 [2] NCCL INFO comm 0x7fb9bc0030d0 rank 6 nranks 8 cudaDev 2 busId b1000 - Init COMPLETE
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688397:1688738 [3] NCCL INFO comm 0x7f4ee40030d0 rank 7 nranks 8 cudaDev 3 busId b2000 - Init COMPLETE
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688395:1688735 [1] NCCL INFO comm 0x7f73040030d0 rank 5 nranks 8 cudaDev 1 busId 3e000 - Init COMPLETE
gadi-gpu-v100-0062: gadi-gpu-v100-0062:1688394:1688737 [0] NCCL INFO comm 0x7fcddc0030d0 rank 4 nranks 8 cudaDev 0 busId 3d000 - Init COMPLETE
gadi-gpu-v100-0010: *** Running generate
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=DeepSpeed is a machine learning framework
gadi-gpu-v100-0010: out=DeepSpeed is a machine learning framework for fast and accurate prediction of protein structure and function. It is based on a novel deep learning architecture, which is able to learn from a large amount of data, and to make accurate predictions. The framework is able to learn from a large amount of data, and to make accurate predictions. The framework is able to learn from a large amount of data, and to make accurate predictions. The framework is able to learn from a large amount of data, and to make accurate predictions. The framework is able
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=He is working on
gadi-gpu-v100-0010: out=He is working on a new book, and he is also working on a new book on the history of the American Revolution. He is also working on a new book on the history of the American Revolution. He is also working on a new book on the history of the American Revolution. He is also working on a new book on the history of the American Revolution. He is also working on a new book on the history of the American Revolution. He is also working on a new book on the history of the American Revolution
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=He has a
gadi-gpu-v100-0010: out=He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of money.
gadi-gpu-v100-0010: He has a lot of
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=He got all
gadi-gpu-v100-0010: out=He got all the way to the top of the mountain, and he was just about to jump off the edge, and he looked down and he saw a little boy, and he said, “Hey, little boy, what are you doing here?” And the little boy said, “I’m going to jump off this mountain.” And the man said, “Well, you can’t jump off this mountain.” And the little boy said, “Well, I can.” And the man said, “Well
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=Everyone is happy and I can
gadi-gpu-v100-0010: out=Everyone is happy and I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future. I can see the future.
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=The new movie that got Oscar this year
gadi-gpu-v100-0010: out=The new movie that got Oscar this year is a movie about a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is a man who is
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=In the far far distance from our galaxy,
gadi-gpu-v100-0010: out=In the far far distance from our galaxy, there is a planet called the planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead is a planet of the dead.
gadi-gpu-v100-0010: The planet of the dead
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: ------------------------------------------------------------
gadi-gpu-v100-0010: in=Peace is the only way
gadi-gpu-v100-0010: out=Peace is the only way to solve the problem of the world. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world is in a state of war. The world
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: [2023-09-01 14:25:32,566] [INFO] [utils.py:827:see_memory_usage] end-of-run
gadi-gpu-v100-0010: [2023-09-01 14:25:32,566] [INFO] [utils.py:828:see_memory_usage] MA 26.84 GB         Max_MA 27.66 GB         CA 26.95 GB         Max_CA 28 GB 
gadi-gpu-v100-0010: [2023-09-01 14:25:32,566] [INFO] [utils.py:836:see_memory_usage] CPU Virtual Memory:  used = 31.56 GB, percent = 8.4%
gadi-gpu-v100-0010: *** Running benchmark
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: *** Performance stats:
gadi-gpu-v100-0010: Throughput per token including tokenize: 26.93 msecs
gadi-gpu-v100-0010: Start to ready to generate: 51.278 secs
gadi-gpu-v100-0010: Tokenize and generate 4000 (bs=8) tokens: 21.544 secs
gadi-gpu-v100-0010: Start to finish: 72.822 secs
gadi-gpu-v100-0010: 
gadi-gpu-v100-0010: [2023-09-01 14:27:42,681] [INFO] [launch.py:350:main] Process 15832 exits successfully.
gadi-gpu-v100-0010: [2023-09-01 14:27:42,682] [INFO] [launch.py:350:main] Process 15833 exits successfully.
gadi-gpu-v100-0062: [2023-09-01 14:27:42,942] [INFO] [launch.py:350:main] Process 1688397 exits successfully.
gadi-gpu-v100-0062: [2023-09-01 14:27:42,942] [INFO] [launch.py:350:main] Process 1688394 exits successfully.
gadi-gpu-v100-0062: [2023-09-01 14:27:42,942] [INFO] [launch.py:350:main] Process 1688396 exits successfully.
gadi-gpu-v100-0062: [2023-09-01 14:27:42,942] [INFO] [launch.py:350:main] Process 1688395 exits successfully.
gadi-gpu-v100-0010: [2023-09-01 14:27:43,682] [INFO] [launch.py:350:main] Process 15835 exits successfully.
gadi-gpu-v100-0010: [2023-09-01 14:27:43,682] [INFO] [launch.py:350:main] Process 15834 exits successfully.

real	4m11.809s
user	0m1.699s
sys	0m0.595s
```

