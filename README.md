# TensorRT-LLM Backend
The Triton backend for TensorRT-LLM.

## Usage

### Launch the backend *within Docker*

```bash
# 1. Pull the docker image
nvidia-docker run -it --rm -e LOCAL_USER_ID=`id -u ${USER}` --shm-size=2g -v <your/path>:<mount/path> <image> bash

# 2. Modify parameters:
1. all_models/<model>/tensorrt_llm/config.pbtxt
2. all_models/<model>/preprocessing/config.pbtxt
3. all_models/<model>/postprocessing/config.pbtxt

# 3. Launch triton server
python3 scripts/launch_triton_server.py --world_size=1 \
    --model_repo=all_models/<model>
```

### Launch the backend *within Slurm based clusters*
1. Prepare some scripts

`tensorrt_llm_triton.sub`
```bash
#!/bin/bash
#SBATCH -o logs/tensorrt_llm.out
#SBATCH -e logs/tensorrt_llm.error
#SBATCH -J gpu-comparch-ftp:mgmn
#SBATCH -A gpu-comparch
#SBATCH -p luna
#SBATCH --nodes=1
#SBATCH --ntasks-per-node=8
#SBATCH --time=00:30:00

sudo nvidia-smi -lgc 1410,1410

srun --mpi=pmix --container-image <image> \
    --container-mounts <your/path>:<mount/path> \
    --container-workdir <workdir> \
    --output logs/tensorrt_llm_%t.out \
    bash <workdir>/tensorrt_llm_triton.sh
```

`tensorrt_llm_triton.sh`
```
TRITONSERVER="/opt/tritonserver/bin/tritonserver"
MODEL_REPO="<workdir>/triton_backend/"

${TRITONSERVER} --model-repository=${MODEL_REPO} --disable-auto-complete-config --backend-config=python,shm-region-prefix-name=prefix${SLURM_PROCID}_
```

2. Submit a Slurm job
```
sbatch tensorrt_llm_triton.sub
```

### Kill the backend

```bash
pgrep tritonserver | xargs kill -9
```

## Examples

### GPT/OPT/LLaMA/GPT-J...
```bash
cd tools/gpt/

# Download vocab and merge table for HF models
# Take GPT as an example:
rm -rf gpt2 && git clone https://huggingface.co/gpt2
pushd gpt2 && rm pytorch_model.bin model.safetensors && \
    wget -q https://huggingface.co/gpt2/resolve/main/pytorch_model.bin && popd

python3 client.py \
    --text="Born in north-east France, Soyer trained as a" \
    --output_len=10 \
    --tokenizer_dir gpt2 \
    --tokenizer_type auto

# Exmaple output:
# [INFO] Latency: 92.278 ms
# Input: Born in north-east France, Soyer trained as a
# Output:  chef and a cook at the local restaurant, La
```
*Please note that the example outputs are only for reference, specific performance numbers depend on the GPU you're using.*

## Test

```bash
cd tools/gpt/

# Identity test
python3 identity_test.py \
    --batch_size=8 --start_len=128 --output_len=20
# Results:
# [INFO] Batch size: 8, Start len: 8, Output len: 10
# [INFO] Latency: 70.782 ms
# [INFO] Throughput: 113.023 sentences / sec

# Benchmark using Perf Analyzer
python3 gen_input_data.py
perf_analyzer -m tensorrt_llm \
    -b 8 --input-data input_data.json \
    --concurrency-range 1:10:2 \
    -u 'localhost:8000'

# Results:
# Concurrency: 1, throughput: 99.9875 infer/sec, latency 79797 usec
# Concurrency: 3, throughput: 197.308 infer/sec, latency 121342 usec
# Concurrency: 5, throughput: 259.077 infer/sec, latency 153693 usec
# Concurrency: 7, throughput: 286.18 infer/sec, latency 195011 usec
# Concurrency: 9, throughput: 307.067 infer/sec, latency 233354 usec
```
*Please note that the example outputs are only for reference, specific performance numbers depend on the GPU you're using.*
