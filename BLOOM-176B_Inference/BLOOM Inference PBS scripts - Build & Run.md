[toc]



# Build DeepSpeed

Simply run the following commands.

Internet connection is needed, so you need to run it on Login nodes.

```bash
# Create Work Directoy
export BLOOM_DIR=/scratch/il82/pz7344/BLOOM # Pengzhi's environment as an example
mkdir -p ${BLOOM_DIR} && cd ${BLOOM_DIR}
mkdir offline_models

# Install and activate miniconda
time wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh -O ${BLOOM_DIR}/miniconda.sh
time bash ${BLOOM_DIR}/miniconda.sh -b -p ${HOME}/miniconda3
# real	0m28.179s

# Initialize Conda
source ${HOME}/miniconda3/etc/profile.d/conda.sh
conda init
conda config --set auto_activate_base false

# Create Python 3.8 DeepSpeed Conda environment
cd ${BLOOM_DIR}
time conda create -p deepspeed073/p38 python=3.8 -y
# real	1m1.448s
time deepspeed073/p38/bin/pip --cache-dir /scratch/public/HPC-AI-BLOOM/cache/pip install torch==1.12.1+cu116 --extra-index-url https://download.pytorch.org/whl/cu116 transformers==4.26.1 deepspeed==0.7.6 accelerate==0.16.0 gunicorn==20.1.0 flask flask_api fastapi==0.89.1 uvicorn==0.19.0 jinja2==3.1.2 pydantic==1.10.2 huggingface_hub==0.12.1 grpcio-tools==1.50.0 bitsandbytes -v
# real	3m39.252s

# Sync the existing model files to your BLOOM directory
time rsync -avP /scratch/public/HPC-AI-BLOOM/models--microsoft--bloom-deepspeed-inference-fp16 ${BLOOM_DIR}/offline_models

# Create the following links, so that huggingface transformer can work on the offline GPU nodes
rm -rf ${HOME}/.cache/huggingface/hub
ln -sf ${BLOOM_DIR}/offline_models ${HOME}/.cache/huggingface/hub

# The following command is used to download the DeepSpeed sharded int8 model to public shared storage
# deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="microsoft/bloom-deepspeed-inference-int8",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",ignore_patterns=["*.safetensors"],)'

# To get DeepSpeed fp16 and HugginFace models, simple run the following two commands
#deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="microsoft/bloom-deepspeed-inference-fp16",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",ignore_patterns=["*.safetensors"],)'

#deepspeed073/p38/bin/python -c 'from huggingface_hub import snapshot_download; snapshot_download(repo_id="bigscience/bloom",local_files_only=False,cache_dir="/scratch/public/HPC-AI-BLOOM",)'

# Add variables to ${HOME}/.deepspeed_env, we will start all training's from $HOME directory
echo "CUDA_HOME=/apps/cuda/11.6.1" > ${HOME}/.deepspeed_env
echo "NCCL_DEBUG=INFO" >> ${HOME}/.deepspeed_env
echo "TORCH_DISTRIBUTED_DETAIL=DEBUG" >> ${HOME}/.deepspeed_env
echo "TRANSFORMERS_OFFLINE=1" >> ${HOME}/.deepspeed_env
echo "LD_LIBRARY_PATH=/apps/cuda/11.6.1/extras/CUPTI/lib64:/apps/cuda/11.6.1/lib64" >> ${HOME}/.deepspeed_env
echo "PDSH_SSH_ARGS='-o StrictHostKeyChecking=no'" >> ${HOME}/.deepspeed_env

# Clone the BLOOM Inference project
mkdir ${BLOOM_DIR}/github -p
cd ${BLOOM_DIR}/github
git clone https://github.com/huggingface/transformers-bloom-inference

# Install pdsh with ssh support
cd ${BLOOM_DIR}/github
git clone https://github.com/chaos/pdsh
cd pdsh
./bootstrap
./configure --prefix=${HOME}/.local --with-ssh
make install -j $(nproc)

# Configure SSH authentication
yes | ssh-keygen -t ecdsa -f ${HOME}/.ssh/id_ecdsa -N "" -vvv
ssh-keygen -y -f ${HOME}/.ssh/id_ecdsa >> ${HOME}/.ssh/authorized_keys
```

# Run BLOOM Inference

## Submit inference script

It takes about 320 seconds to run the python script, so we request for 10 minutes.

```bash
qsub -q gpuvolta -l walltime=00:10:00,ncpus=$((48*2)),mem=$((376*2))gb,ngpus=$((4*2)) -M 393958790@qq.com -m abe -P il82 -N bloom2nodes bloom_inference.sh
```

## The script "bloom_inference.sh"

```bash
export BLOOM_DIR=/scratch/il82/pz7344/BLOOM # Pengzhi's environment as an example

export PATH=${PATH}:${HOME}/.local/bin
ss -lnpte
env

cat ${PBS_NODEFILE} | cut -f 1 -d . | sed -e 's/$/ slots=48/' | sort -u > ${HOME}/hostfile

cd ${HOME}
time ${BLOOM_DIR}/deepspeed073/p38/bin/deepspeed --hostfile ${HOME}/hostfile --num_gpus 4 ${BLOOM_DIR}/github/transformers-bloom-inference/bloom-inference-scripts/bloom-ds-inference.py --name microsoft/bloom-deepspeed-inference-int8 --dtype int8 --benchmark --batch_size 8
```

