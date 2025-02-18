# RAGEN: A General-Purpose Reasoning Agent Training Framework

<p align="center" style="font-size: 18px;">
  <strong>RAGEN</strong> is the first reproduction of the <strong>DeepSeek-R1(-Zero)</strong> methods for <em>training agentic models</em>.<br>
  <em>We strongly believe in the future of RL + LLM + Agents. The release is a minimally viable leap forward.</em>
</p>

## Performance

We run RAGEN on Qwen-2.5-{0.5B, 3B}-{Instruct, None} and DeepSeek-R1-Distill-Qwen-1.5B, on the [Gym-Sokoban](https://github.com/mpSchrader/gym-sokoban) task. 

About the sokoban task (from the official repo): Sokoban is Japanese for warehouse keeper and a traditional video game. The game is a transportation puzzle, where the player has to push all boxes in the room on the storage locations/ targets. The possibility of making irreversible mistakes makes these puzzles so challenging especially for Reinforcement Learning algorithms, which mostly lack the ability to think ahead.

NOTE: See [Visualization](https://github.com/ZihanWang314/ragen/#visualization) Section for details. The maximum reward of this environment is **10.9**. Action spaces are 0-4 (0: Stand, 1: Up, 2: Down, 3: Left, 4: Right).

<img src="./public/loss_curve.png" width="800px" alt="s" />

The loss curve have not converged (since our compute is currently limited...). But we already see some trends:
 - Instruct-finetuned models are not significantly advantaged ahead Pretrained-only models, although they are better at start.
 - 3B models are performing better than 0.5B models as well, but the advantages are also not that obvious at around 40 steps.
 - Interestingly, R1-distilled 1.5B model do less well than 0.5B models for now.

We prepare to release a complete wandb plot for these experiment runs, although you can try it your own and it may even be faster than our run (reasons above).









## Environment Setup

```bash
conda create -n ragen python=3.9 -y
conda activate ragen

git clone git@github.com:ZihanWang314/ragen.git
cd ragen

# setup install
pip install -e . # includes verl-ragen (by us) and verl-core (by the verl team)
pip install torch==2.4.0 --index-url https://download.pytorch.org/whl/cu121


# Optional: to install flash-attn, you may need to install cuda-toolkit first if you don't have
# WARNING: Building flash-attn from source requires significant GPU compute capability
# Attempted build on NVIDIA RTX 3080 (32GB RAM) did not complete after 24 hours
# Consider using a more powerful GPU setup for successful compilation
conda install -c "nvidia/label/cuda-12.1.0" cuda-toolkit -y
export CUDA_HOME=$CONDA_PREFIX # /opt/conda/envs/zero
pip3 install flash-attn --no-build-isolation


pip install -r requirements.txt # other packages
```


## Train Models

### Create data

On the [Gym-Sokoban](https://github.com/mpSchrader/gym-sokoban) task, We create 10k data for training and run for <=1 epoch. 
```bash
# sokoban env settings. will determine game difficulty
# it's normal to see some SOKOBAN errors, but the data will be created and it's fine

export DIM_X=6
export DIM_Y=6
export NUM_BOXES=1
export MAX_STEPS=5
export SEARCH_DEPTH=30


python scripts/dataset_curation.py \
    --output data/sokoban \
    --seed 10000 \
    --train_size 10000 \
    --test_size 10 \
    --prefix qwen-instruct # we find it could work for base models
```

### Export variables and train
```bash
export DATA_DIR=data/sokoban
export DIM_X=6
export DIM_Y=6
export NUM_BOXES=1
export MAX_STEPS=5
export SEARCH_DEPTH=30

# export CUDA_VISIBLE_DEVICES=0
# export BASE_MODEL=Qwen/Qwen2.5-0.5B
# export EXPERIMENT_NAME=test-qwen2.5-0.5b

export CUDA_VISIBLE_DEVICES=0
export BASE_MODEL=checkpoints/Agent-R1/test-qwen2.5-0.5b-instruct-1mbsz/actor/global_step_100
export EXPERIMENT_NAME=test-qwen2.5-0.5b-imagetest


export MICRO_BATCH_SIZE=1
export TRAIN_BATCH_SIZE=128 # 256
export PPO_BATCH_SIZE=64 # 128
export MAX_START_LENGTH=400 # the first round prompt max length
export MAX_RESPONSE_LENGTH=100
export MAX_OBS_LENGTH=120
export MAX_TURNS=5
export NUM_UPDATE_PER_ROLL=1 # roll out for a batch, then the model do N times of update. Currently not implemented.
export LOG_MODE="['wandb']" # or 'console'
export GCP=True # gradient checkpointing
export N_GPUS=1
export ROLLOUT_TP_SIZE=1

bash ./train.sh # more arguments in this file

# default config file is verl/trainer/config/ppo_trainer.yaml

```


## Visualization
1. By setting arguments in `train.sh`, you can visualize the trajectory:
```bash
logging.log_images=True # set to True to log images
logging.log_image_dir=.log.debug/trajectory # set to the directory to save images
logging.log_image_step_size=1 # save image every _ steps
logging.log_n_image_per_batch=8 # save _ images per batch   
```

2. You may also need to install fonts to make the figures displayed correctly:
```bash
sudo apt-get install fonts-noto-cjk
```

3. Example image for one trajectory: 
<p align="center" style="display: flex; justify-content: center; gap: 10px;">
    <img src="./public/step_1.png" width="200px" alt="s" />
    <img src="./public/step_2.png" width="200px" alt="s" />
    <img src="./public/step_3.png" width="200px" alt="s" />
    <img src="./public/step_4.png" width="200px" alt="s" />
    <img src="./public/step_5.png" width="200px" alt="s" />
</p>






## Cases
Please see cases/ file.
There are only limited cases for now, including [reward hacking](https://github.com/ZihanWang314/agent-r1/blob/main/cases/reward_hacking.txt) and the [suck moment](https://github.com/ZihanWang314/agent-r1/blob/main/cases/suck_moment.txt). we will add more cases recently.


## Authors

[Zihan Wang*](https://zihanwang314.github.io/)

[Kangrui Wang](https://jameskrw.github.io/)

[Qineng Wang](https://qinengwang-aiden.github.io/)

[Pingyue Zhang](https://williamzhangsjtu.github.io/)

[Manling Li†](https://limanling.github.io)

*: Project Lead; †: Advising.
Remaining authors are alphabetical order.



## Acknowledgements


We thank [DeepSeek](https://github.com/deepseek-ai/DeepSeek-R1) for providing the DeepSeek-R1 model and ideas. We thank the [veRL](https://github.com/volcengine/verl) team for their infrastructure. We thank the [TinyZero](https://github.com/Jiayi-Pan/TinyZero) team for their discoveries that inspired our early exploration. We thank Yiping Lu, Runxin Xu, Kyunghyun Cho for insightful discussions with them.

## Citation
```md
@misc{RAGEN,
  author       = {Zihan Wang and Kangrui Wang and Qineng Wang and Pingyue Zhang and Manling Li},
  title        = {RAGEN: A General-Purpose Reasoning Agent Training Framework},
  year         = {2025},
  organization = {GitHub},
  url          = {https://github.com/ZihanWang314/ragen},
}
```

## Replication

**Phase 1: Understanding the Repository**

*   Explore the file structure: (Completed) I've examined the file structure and identified key directories and files related to model loading, training, data handling, and configuration.
*   Identify the model loading mechanism: (Completed) The DeepSeek model clone is implemented using Megatron-LM, as confirmed by the files in `verl/models/llama/megatron`.
*   Understand the training process: (Completed) The repository supports both PPO and SFT, with training scripts located in `verl/trainer`. `main_ppo.py` is used for PPO training, and `fsdp_sft_trainer.py` is used for SFT training.
*   Analyze the data loading and formatting: (Completed) The `ragen/utils/dataset.py` file defines a custom `Dataset` class for trajectory-based data, and `verl/utils/dataset/sft_dataset.py` defines a class for SFT data. The `transform` method in `ragen/utils/dataset.py` and the `__getitem__` method in `verl/utils/dataset/sft_dataset.py` are used to format the data.
*   Identify the fine-tuning task: (In Progress) The fine-tuning task depends on the configuration files used with the training scripts. The `cases` directory contains example data for reward hacking and "suck moment" scenarios, which may provide clues about the fine-tuning task.

**Phase 2: Replicating Results Locally**

*   Identify the necessary dependencies: (Completed) The `requirements.txt` file lists the required Python packages.
*   Set up the environment:
    *   I will guide the user to create a new virtual environment using `conda` or `venv`.
    *   I will guide the user to install the required packages using `pip install -r requirements.txt`.
    *   I will guide the user to ensure that CUDA is properly configured and that PyTorch can access the GPU.
*   Run the training script:
    *   I will guide the user to run the PPO training script using `python verl/trainer/main_ppo.py --config-path verl/trainer/config --config-name ppo_trainer`.
    *   I will guide the user to run the SFT training script using `python verl/trainer/fsdp_sft_trainer.py --config-path verl/trainer/config --config-name sft_trainer`.
    *   I will guide the user to modify the config files as needed.
*   Verify the results:
    *   I will guide the user to monitor the training logs and metrics.
    *   I will guide the user to compare the results with the original paper or repository.

**Phase 3: Fine-tuning on Custom Tasks**

*   Understand the data format:
    *   For SFT, the data should be in parquet files with columns for prompts and responses. The `SFTDataset` class in `verl/utils/dataset/sft_dataset.py` provides more details on the expected format.
    *   For PPO, the data should be in a trajectory format, as handled by the `Dataset` class in `ragen/utils/dataset.py`.
*   Modify the training script:
    *   I will guide the user to modify the configuration files to use their own data files.
    *   I will guide the user to modify the `RewardManager` class in `verl/trainer/main_ppo.py` to use their own reward function.
*   Run the fine-tuning script:
    *   I will guide the user to run the modified training script with their own data.


## Installation Progress Documentation

### Current State (as of wheel building attempt)
1. Environment setup completed:
   - Conda environment 'ragen' created with Python 3.9
   - CUDA toolkit 12.1.0 installed via conda
   - CUDA_HOME set to conda environment path

2. Current status:
   - Attempted building flash-attn wheel for 24 hours without success
   - Build process showed normal indicators (CPU usage, nvcc processes)
   - RTX 3080 (32GB RAM) appears insufficient for compilation
   - More powerful GPU setup likely required

### Recovery Steps (if build fails)
1. Start Anaconda:
   ```bash
   # Launch Anaconda Navigator or use command line
   conda activate base
   ```

2. Navigate to project:
   ```bash
   cd path/to/ragen
   conda activate ragen
   ```

3. Check environment:
   ```bash
   # Verify CUDA toolkit
   nvcc --version
   
   # Check running processes
   tasklist | findstr "python pip conda nvcc cl"
   ```

4. Resume installation:
   ```bash
   # Set environment variable
   set CUDA_HOME=%CONDA_PREFIX%
   
   # Attempt flash-attn installation
   pip3 install flash-attn --no-build-isolation
   ```

5. Monitor progress:
   - Check Task Manager for CPU/GPU usage
   - Watch for nvcc.exe process
   - Normal build shows high CPU, low GPU usage

Note: If build fails after >1 hour, consider:
- Checking system logs for errors
- Verifying all CUDA prerequisites
- Attempting with a different flash-attn version
