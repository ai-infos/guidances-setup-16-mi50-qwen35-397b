
# Guidances for Test setup of 16 AMD MI50 32GB (Qwen3.5 397B A17B)

![illustration](illustration.jpg)

**Goal: run Qwen3.5 397B A17B GPTQ 4-bit on most cost effective hardware like 16*MI50 at decent speed (token generation & prompt processing)**

**Power draw**: 550W (idle) / 2400W (peak inference)


**Feel free to ask any questions and/or share any comments in the issues section here or in the medium article or in the reddit publication or in gfx906 discord:** 
 - https://medium.com/@ai-infos/16x-amd-mi50-32gb-at-32-t-s-tg-2k-t-s-pp-with-qwen3-5-397b-vllm-gfx906-mobydick-54584a699a81
 - https://www.reddit.com/r/LocalLLaMA/comments/1s9ivgl/16x_amd_mi50_32gb_at_32_ts_tg_2k_ts_pp_with/

## Hardware details

- 16x AMD MI50 32GB (with 2 small fans 50mm per GPU at the back)
- Motherboard with 7 PCIe 4.0 ports x16 (ROMED8-2T)
- AMD EPYC CPU with 128 lanes (like 7642 with its 48 cores 2.3 GHz or other)
- 2x 64 GB ram DDR4 3200 ECC
- 4x PSU 1800W (with 3 add2psu)
- 12x SlimSAS PCIe device adapters 2x 8i
- 4x SlimSAS PCie device 1x 4i (C-payne)
- 12x SlimSAS cables 8i
- 2x SlimSAS cable 8i to 2x4i
- 7x SlimSAS PCIe host adapter
- 1x NVME drive
- 4x PWM HUB FAN Sata
- (optional) 10x Fans 140mm 

### Some relevant advices to avoid fires (non-exhaustive):

1) Do your own researches and always understand what you do when it's related to safety.
2) Draw an electrical plan of your home (avoid trusting existing plan, especially for old building) and check the maximum amperage/voltage allowed per circuit / power strip / smart plug. 
3) Always check the specs of the PSU you want to buy to see max wattage per cable (usually but not always: SATA ~54W, Molex ~132W, 6pin ~75W, 8pin ~150W per cable). The 'per cable' is important because it means that if you have 1 cable with 2*8pin at the end, it's actually 75W per 8pin in this case. For example, if you have a PSU saying that it has 8*(2*8pin), it's in total 16 8pin but you can only power 4 MI50 (having each 2*8pin and requiring 300W per card). /!\ Even if you cap the power draw of you card to 150W, this is not advised to plug only 1 cable of 2*8pin; this is better to have 2 cables of 2*8pin per MI50, and for the remaining 8pin, it can be used for the extender (if 6+2pin). 
4) Don't forget that your PSU needs also the right number of SATA/molex/6pin for PWM HUB, add2psu and extenders (extenders with molex or 6 pin are better than sata as they usually required more power to be steady)
5) The fan port of your motherboard usually supports ~1A so don't plug too many fans (having more than 1A in total) on it, if you don't want to burn your motherboard (or more). That's why PWM HUB FAN is often required for this kind of setup as the motherboard does not have enough fan ports.

## Software details

- Ubuntu v24.04 (kernel: 6.11.0-17-generic)
- ROCM v6.3.4
- torch v2.10.0
- triton-gfx906 v3.6.0
- vllm-gfx906-mobydick v0.17.1rc0.x (https://github.com/ai-infos/vllm-gfx906-mobydick/tree/gfx906/v0.17.1rc0.x)
- MI50 bios: 32G_UEFI.rom  (available there: https://gist.github.com/evilJazz/14a4c82a67f2c52a6bb5f9cea02f5e13 /!\ don't flash your bios if you don't know what you do; the stock bios might work in your setup)
- open-webui
- Custom motherboard bios to boot with 16 MI50: ask ASRock Rack support for this ROM or in the meantime, boot with 14 GPU and use hotplug to make it run with 16 under Ubuntu (see below for more details)


## Run 16 AMD MI50 under Ubuntu (required: hotplug support for the motherboard) if boot is capped at 14 GPU

1) Modify the following motherboard bios settings from default (NB: some setting modifications are optional and some settings might be shown differently for your bios/motherboard):

- advanced > chipset config > primary graphic adapter: onboard / onboard debug port LED: off
- advanced > chipset config > amd pcie link speed > pcie1 to 7 gen3, ocu1 ocu2 m2_1 gen1, m2_2 gen4 / amd pcie link width > pcie1 x4x4x4x4 and all other pcie x8x8 / amd pcie hot plug > pcie7 hotplug: enabled
- advanced > sata > hotplug disabled
- advanced > super io config: all disabled
- advanced > amd cbs > fch common options > all sata & sata controller options disabled
- advanced > amd cbs > nbio common options > hd audio disabled 
- boot > csm > csm: custom / all pcie1 to 7 & ocu1 & ocu2 slot oprom : disabled ; m2_2 slot oprom: UEFI

2) Linux kernel update:

- Add in linux kernel grub option via vim /etc/default/grub: `pcie_ports=native pci=realloc=on pciehp.pciehp_force=1 pci=assign-busses pci=hpmmioprefsize=128G pci=hpmemsize=128G pci=hpmmiosize=16G pci=hpiosize=4M pci=hpbussize=16` 
- `sudo update-grub`
- `sudo reboot`
- Some command examples to verify everything's ok: 
```code
sudo dmesg | grep pciehp
sudo dmesg | grep -i -E "pci|bar|resource|alloc|fail|mi50|amdgpu"
sudo dmesg -T | tail -n 100
lspci -tv && rocm-smi
```

## Relevant commands to run

### ROCm & amdgpu drivers

```code
# Get the script that adds the AMD repo for 24.04 (noble)
wget https://repo.radeon.com/amdgpu-install/6.3.4/ubuntu/noble/amdgpu-install_6.3.60304-1_all.deb
sudo apt install ./amdgpu-install_6.3.60304-1_all.deb

# Install ROCm  6.3.4 including hip, rocblas, amdgpu-dkms etc (assuming the machine has already the advised compatible kernel 6.11)
sudo amdgpu-install --usecase=rocm --rocmrelease=6.3.4    

sudo usermod -aG render,video $USER

# Verify ROCm installation
rocm-smi --showproductname --showdriverversion
rocminfo

# Add iommu=pt if you later grow beyond two GPUs
# ROCm’s NCCL-/RCCL-based frameworks can hang on multi-GPU rigs unless the IOMMU is put in pass-through mode
# see https://rocm.docs.amd.com/projects/install-on-linux/en/docs-6.3.3/reference/install-faq.html#multi-gpu

sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="/GRUB_CMDLINE_LINUX_DEFAULT="iommu=pt /' /etc/default/grub
sudo update-grub
sudo reboot
cat /proc/cmdline  # >>> to check: must return: "BOOT_IMAGE=... iommu=pt"

```

### vllm-gfx906-mobydick fork with its dependencies (torch, triton, python, etc)

```code

pyenv install 3.12.11
pyenv virtualenv 3.12.11 venv312
pyenv activate venv312

# PYTORCH 2.10.0

git clone --branch v2.10.0 --recursive https://github.com/pytorch/pytorch.git
cd pytorch

# Install Python Dependencies
pip install -r requirements.txt
pip install mkl-static mkl-include

# Hipify the Source (Convert CUDA to ROCm code)
python tools/amd_build/build_amd.py

# Build the wheel and install
export MAX_JOBS=96 # to be adjusted according to your setup to avoid OOM / freeze / crash
export USE_ROCM=1
export PYTORCH_ROCM_ARCH=gfx906
export CMAKE_PREFIX_PATH="${VIRTUAL_ENV}:${CMAKE_PREFIX_PATH}"

pip wheel --no-build-isolation -v -w dist -e . 2>&1 | tee build.log
pip install ./dist/torch*.whl


# TORCHVISION 0.25.0

# Install dependencies
sudo apt-get update && sudo apt-get install -y libpng-dev libjpeg-dev ffmpeg

# Build and Install
git clone --branch v0.25.0 https://github.com/pytorch/vision.git
cd vision
export FORCE_CUDA=1
export USE_ROCM=1
export PYTORCH_ROCM_ARCH=gfx906

python setup.py install


# TORCHAUDIO 2.10.0

# Build and Install
git clone --branch v2.10.0 https://github.com/pytorch/audio.git
cd audio
export PYTORCH_ROCM_ARCH=gfx906
export USE_ROCM=1

python setup.py install


# TRITON

git clone --branch v3.6.0+gfx906 https://github.com/ai-infos/triton-gfx906.git
cd triton-gfx906 
pip install -r python/requirements.txt
TRITON_CODEGEN_BACKENDS="amd" pip wheel --no-build-isolation -w dist . 2>&1 | tee build.log
pip install ./dist/triton-*.whl  


# FLASH-ATTENTION-GFX906 (triton backend)

git clone https://github.com/ai-infos/flash-attention-gfx906.git
cd flash-attention-gfx906
FLASH_ATTENTION_TRITON_AMD_ENABLE="TRUE" python setup.py install


# VLLM

git clone --branch gfx906/v0.17.1rc0.x --single-branch https://github.com/ai-infos/vllm-gfx906-mobydick
cd vllm-gfx906-mobydick
pip install 'cmake>=3.26.1,<4' 'packaging>=24.2' 'setuptools>=77.0.3,<80.0.0' 'setuptools-scm>=8' 'jinja2>=3.1.6' 'amdsmi>=6.3,<6.4' 'timm>=1.0.17'
pip install -r requirements/rocm.txt
pip wheel --no-build-isolation -v -w dist . 2>&1 | tee build.log
pip install ./dist/vllm-*.whl

```

### Download Qwen official quant: Qwen3.5-397B-A17B-GPTQ-Int4

```code
mkdir -p ~/llm/models/Qwen3.5-397B-A17B-GPTQ-Int4 && cd ~/llm/models
hf download Qwen/Qwen3.5-397B-A17B-GPTQ-Int4 --local-dir Qwen3.5-397B-A17B-GPTQ-Int4
```

### Run Qwen3.5 397B GPTQ Int4 in vllm-gfx906-mobydick


```code
pip install transformers==5.3.0

VLLM_EXECUTE_MODEL_TIMEOUT_SECONDS=1200 FLASH_ATTENTION_TRITON_AMD_ENABLE="TRUE" OMP_NUM_THREADS=4 VLLM_LOGGING_LEVEL=DEBUG vllm serve \
    ~/llm/models/Qwen3.5-397B-A17B-GPTQ-Int4 \
    --served-model-name Qwen3.5-397B-A17B-GPTQ-Int4 \
    --dtype float32 \
    --kv-cache-dtype half \
    --max-model-len 262144 \
    --enable-log-requests \
    --enable-log-outputs \
    --log-error-stack \
    --max-num-seqs 4 \
    --gpu-memory-utilization 0.98 \
    --enable-auto-tool-choice \
    --tool-call-parser qwen3_coder \
    --reasoning-parser qwen3 \
    --mm-processor-cache-gb 1 \
    --speculative-config '{"method":"qwen3_next_mtp","num_speculative_tokens":5}' \
    --limit-mm-per-prompt.image 1 --limit-mm-per-prompt.video 1 --skip-mm-profiling \
    --tensor-parallel-size 16 --pipeline-parallel-size 1 \
    --host 0.0.0.0 \
    --quantization moe_wna16 \
    --port 8000 2>&1 | tee log.txt 
```

**Performance peak**: TG (token generation): 32.8 tok/s / PP (prompt processing): variable according to request length (911 tok -> 91,1 tok/s ; 16k tok -> 1600 tok/s etc... but a long request implies also longer pre processing, it lasts in reality ~1min06 to handle 16k tok request before decoding phase)


### Run Open-WebUI

```code
sudo docker run -d --network=host \
  --name open-webui-mi50 \
  -v open-webui:/app/backend/data \
  -e OPENAI_API_BASE_URL=http://localhost:8000/v1 \
  --restart always \
  ghcr.io/open-webui/open-webui:main
```

Go to http://localhost:8080 and enjoy!

## TODO LIST

- improve this guidance draft (content/form)
- add docker files for easy setup
- open source a future test setup of 32 AMD MI50 32GB for Kimi K2.5 Thinking and/or GLM5

## FINAL NOTES

- You can find more technical details about the vllm-gfx906-mobydick fork (ex vllm-gfx906-deepseek) in this PR: https://github.com/nlzy/vllm-gfx906/pull/62 and you can check similar guidances to run deepseek v3.2 awq here: https://github.com/ai-infos/guidances-setup-16-mi50-deepseek-v32 
- Qwen3.5 397B GPTQ does not use the same attention backend as Deepseek v3.2 AWQ but we could successfully run both models on old consumer GPU like GFX906 thanks to the support of FP32 activation in vllm-gfx906-deepseek fork instead of BF16 (not supported by GFX906) and instead of FP16 (supported but producing garbage output). Then this fork also focus on performance, allowing dot product in FP16 with precision accumulation in FP32 (in triton kernels) and it also supports dtype float32 associated with kv-cache-dtype half (float16) for performance purpose and to save some VRAM. 
- That's a first attempt and this is not the most optimized setup to run Qwen3.5 397B on gfx906 hardware, so there's still some room for speed/memory improvements. It would be great to have someone with AMD/ROCM/HIP/Triton/etc kernel skills to work on it and improve this proposal! 


**Credits: Global Open source Community**
