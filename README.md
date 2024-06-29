# Self-play with Execution Feedback: Improving Instruction-following Capabilities of Large Language Models

[![PWC](https://img.shields.io/endpoint.svg?url=https://paperswithcode.com/badge/self-play-with-execution-feedback-improving/instruction-following-on-ifeval)](https://paperswithcode.com/sota/instruction-following-on-ifeval?p=self-play-with-execution-feedback-improving)

*Guanting Dong, Keming Lu, Chengpeng Li, Tingyu Xia, Bowen Yu, Chang Zhou, Jingren Zhou*

Qwen, Alibaba Inc.

---

## :sparkles: Overview


This is the repository contains core implementations of the **AutoIF**, proposed by [Self-play with Execution Feedback: Improving Instruction-following Capabilities of Large Language Models](https://arxiv.org/abs/2406.13542).

**AutoIF** is the first scalable and reliable method for automatically generating instruction-following data and verifying its quality using code execution feedback.

![image](https://github.com/dongguanting/AutoIF/assets/60767110/6c222465-25a4-4dec-ade6-d3a5af80ba39)



## :rocket: Data Synthesis of AutoIF
We divided the AutoIF's data synthesis process into steps and provided 10-20 samples per step to facilitate your reproduction. Please remember to replace them with your own input.

### :wrench: Dependencies
General Setup Environment:
- Python 3.9
- [PyTorch](http://pytorch.org/) (currently tested on version 2.1.2+cu121)
- [Transformers](http://huggingface.co/transformers/) (version 4.41.2, unlikely to work lower than this version)

```bash
cd ./AutoIF/
pip install -r requirements.txt
```
---

## Instruction Augmentation and Verification

Firstly, we hand-write 36 seed instructions：
![image](https://github.com/dongguanting/AutoIF/assets/60767110/62518bd7-f5d9-4a33-a85f-1327b77e4dcb)



**Step1: Self-instruct Seed Instructions** 

Concatenate the instruction with the RFT prompt.

```bash
python 1_RFT.py
```

Please perform k times RFT with a supervised model (e.g., GPT-4, Qwen2-72B), save as format in seed_instruction.txt.


**Step2: Verification Funcs and Cases Generation**

Using seed and augmented instructions for generating verification funcs and cases.

```bash
python 2_verification_funcs_cases_generation.py
```

Please generate K verification functions and cases for each sample, save it in eval_func_rft.jsonl


**Step3: Quality Cross-validation**

Cross-validate the pass rates of verification functions and cases to ensure high-quality instructions.

```bash
python 3_cross_validation.py
```

**Step4 & 5: Back Translation**

Please back transalte verification funcs to instructions, and then use [mDeBERTa](https://huggingface.co/MoritzLaurer/mDeBERTa-v3-base-xnli-multilingual-nli-2mil7) for consistency filtering.

```bash
python 4_eval_func_backtranslator.py
python 5_eval_func_backtranslator_filter.py
```

---

## Query Augmentation and Verification

**Step1: Query Reforming and Augmentation**

We randomly concat each query with K queries of [
ShareGPT](https://huggingface.co/datasets/anon8231489123/ShareGPT_Vicuna_unfiltered) and reformat them using our response RFT template:

```bash
python 6_concat_sharegpt_query.py
```

Please use supervision model to generate k responses for each query.


**Step2: Instruction-following Verification**

Cross-validate the pass rate of verification functions and augmented responses to obtain high-quality queries.

```bash
python 7_query_vertification.py
```

In this step, we also concatenate each sample with a consistency scoring prompt. Please score them using the supervision model.


**Step3: Query Quality Verification**

Finally, we fliter out the sample with score > 8 and save it into [LlaMA-Factory](https://github.com/hiyouga/LLaMA-Factory)'s SFT data format.

```bash
python 8_query_score_filiter.py
python 9_sft_data_construction.py
```

---

## ⚡ DPO Data Construction


![image](https://github.com/dongguanting/AutoIF/assets/60767110/f339b287-d11f-4c01-9e9e-f77971f93de3)



:sparkles:Tips:
In our paper, DPO includes two settings, the following are their differences:
- **Offline DPO:** the reponses are obtained from your SFT data generated by supervision model.
- **Online DPO:** the reponses are obtained from your response generated by your base model during each training iteration.

Please process your SFT data using the eval functions generated in the previous step, and format the results as dpo_query_eval_score_results.jsonl.


**Step1: Verification Funcs Scoring**

We use verfy the pass rate of each response by using corresponding verfication funcs.

```bash
python 1_dpo_rft_wash.py
```

**Step1: Data selection**

We construct DPO pairs with postive samples (Acc>=0.5) and nagative samples (Acc=0).

```bash
python 2_dpo_data_query_construct.py
```

After construction you need to process as the DPO data format in [LlaMA-Factory](https://github.com/hiyouga/LLaMA-Factory).

---

## 🎯 Training

We use the version of [LlaMA-Factory v0.6.3](https://github.com/hiyouga/LLaMA-Factory/releases/tag/v0.6.3). Thanks for their excellent work.

:sparkles:Tips:
the difference between our two setups:
- **Strong-to-Weak Distillation:** we use powerful model as supervision model (e.g., GPT-4, Qwen2-72B, Llama3-70B), and weak model (e.g., Qwen2-7B, Llama3-8B) as base model.
- **Self-Alignment:** we use the same model (e.g., Qwen2-72B, Llama3-70B) as supervision and base model.


(1) SFT Training:

```bash
deepspeed --num_gpus=8 train_bash.py \
        --deepspeed $deepspeed_zero3_config_path \
        --stage sft \
        --do_train \
        --use_fast_tokenizer \
        --flash_attn \
        --adam_beta1 0.9 \
        --adam_beta2 0.95 \
        --model_name_or_path $MODEL_PATH \
        --dataset $dataset \
        --template $Template \
        --finetuning_type full \
        --output_dir $OUTPUT_PATH \
        --overwrite_cache \
        --overwrite_output_dir \
        --warmup_steps 20 \
        --weight_decay 0.1 \
        --per_device_train_batch_size 4 \
        --gradient_accumulation_steps 4 \
        --ddp_timeout 9000 \
        --learning_rate 7e-6 \
        --lr_scheduler_type "linear" \
        --logging_steps 1 \
        --cutoff_len 8192 \
        --save_steps 200 \
        --num_train_epochs 3.0 \
        --plot_loss \
        --bf16 
```

(2) DPO Training:

```bash
deepspeed --num_gpus 8 train_bash.py \
        --deepspeed $deepspeed_zero3_config_path \
        --stage dpo \
        --do_train \
        --model_name_or_path $MODEL_PATH \
        --dataset $dataset \
        --dataset_dir $DATA_PATH \
        --template $Template \
        --finetuning_type full \
        --output_dir $OUTPUT_PATH \
        --overwrite_cache \
        --overwrite_output_dir \
        --cutoff_len 4096 \
        --preprocessing_num_workers 1 \
        --per_device_train_batch_size 1 \
        --gradient_accumulation_steps 2 \
        --lr_scheduler_type cosine \
        --logging_steps 10 \
        --warmup_ratio 0.1 \
        --save_steps 1000 \
        --learning_rate 5e-6 \
        --num_train_epochs 2.0 \
        --max_samples 200000 \
        --ddp_timeout 180000000 \
        --plot_loss \
        --fp16
```
For the implementations details between training 7B and 70B models, please refer to our paper.

---

## :mag_right: Overall Results


![image](https://github.com/dongguanting/AutoIF/assets/60767110/6cffd39b-3d34-42ce-9739-79bd0c23208f)


---


## Citation

If you find this work helpful for your research, please kindly cite it.


```bibtex
@misc{dong2024selfplay,
      title={Self-play with Execution Feedback: Improving Instruction-following Capabilities of Large Language Models}, 
      author={Guanting Dong and Keming Lu and Chengpeng Li and Tingyu Xia and Bowen Yu and Chang Zhou and Jingren Zhou},
      year={2024},
      eprint={2406.13542},
      archivePrefix={arXiv},
      primaryClass={id='cs.CL' full_name='Computation and Language' is_active=True alt_name='cmp-lg' in_archive='cs' is_general=False description='Covers natural language processing. Roughly includes material in ACM Subject Class I.2.7. Note that work on artificial languages (programming languages, logics, formal systems) that does not explicitly address natural-language issues broadly construed (natural-language processing, computational linguistics, speech, text retrieval, etc.) is not appropriate for this area.'}
}
```


