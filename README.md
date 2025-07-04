# LLM finetuning
# Language Model Finetuning with Unsloth
## Overview

This guide provides a step-by-step walkthrough for finetuning large language models (LLMs) using the [Unsloth](https://github.com/unslothai/unsloth) library. It covers the entire workflow, including setup, data preparation, training, inference, model saving, and deployment optimizations for both Qwen 2 7B and CodeGemma 7B models.

---

## Table of Contents

- [Unsloth Library](#unsloth-library)
- [Setup](#setup)
- [Model Loading and PEFT Setup](#model-loading-and-peft-setup)
- [Data Preparation](#data-preparation)
- [Training](#training)
- [Inference](#inference)
- [Saving the Model](#saving-the-model)
- [WandB Logging](#wandb-logging)
- [CodeGemma 7B Finetuning](#codegemma-7b-finetuning)
- [Merging and Quantization](#merging-and-quantization)
- [Colab Notebook](#colab-notebook-)

---

> **Note:**  
> - This README assumes familiarity with Python and basic machine learning concepts.  
> - GPU acceleration is highly recommended for efficient training and inference.  
> - For more details on Unsloth features and advanced usage, refer to the [official documentation](https://github.com/unslothai/unsloth).

---
This notebook demonstrates finetuning two different language models, Qwen 2 7B and CodeGemma 7B, using the Unsloth library for faster and more memory-efficient finetuning.

## Unsloth Library

[Unsloth](https://github.com/unslothai/unsloth) is a library that provides optimizations for finetuning large language models, resulting in faster training and reduced memory usage. It achieves this through various techniques like kernel optimizations and efficient memory management. This notebook leverages Unsloth to finetune both Qwen 2 and CodeGemma models effectively on limited resources.

## Setup

The notebook starts by installing the necessary libraries, including `unsloth`, `bitsandbytes`, `accelerate`, `peft`, `trl`, and `datasets`. This is handled in the initial setup cells.



### Model Loading and PEFT Setup

The Qwen 2 7B model is loaded using `FastLanguageModel.from_pretrained` with 4-bit quantization, which is optimized by Unsloth for faster loading and lower memory footprint. PEFT adapters are then applied using `FastLanguageModel.get_peft_model`, leveraging Unsloth's optimizations for PEFT training.
### Qwen 2 7B Model Loading and PEFT Setup

The Qwen 2 7B model is loaded using Unsloth's `FastLanguageModel.from_pretrained` method, which supports efficient 4-bit quantization for reduced memory usage and faster training. After loading, PEFT (Parameter-Efficient Fine-Tuning) adapters are applied using `FastLanguageModel.get_peft_model`, allowing for efficient adaptation of the model to new tasks with minimal additional parameters.

```python
from unsloth import FastLanguageModel

max_seq_length = 4096  # Adjust as needed for your use case
dtype = None  # Auto-detects the best dtype (float16/bfloat16)
load_in_4bit = True  # Enables 4-bit quantization

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/qwen2-7b-bnb-4bit",  # Replace with your preferred Qwen 2 7B model
    max_seq_length = max_seq_length,
    dtype = dtype,
    load_in_4bit = load_in_4bit,
    # token = "hf_..."  # Use if accessing gated models
)

model = FastLanguageModel.get_peft_model(
    model,
    r = 16,  # LoRA rank, adjust based on resources and task
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj"],
    lora_alpha = 16,
    lora_dropout = 0,  # 0 is optimized for Unsloth
    bias = "none",
    use_gradient_checkpointing = "unsloth",  # Enables efficient memory usage for long contexts
    random_state = 3407,
    use_rslora = False,
    loftq_config = None,
)
```

This setup ensures that the Qwen 2 7B model is loaded efficiently and is ready for parameter-efficient finetuning using Unsloth's optimizations.
## Setup

The notebook starts by installing the necessary libraries for finetuning large language models, including `unsloth`, `bitsandbytes`, `accelerate`, `peft`, `trl`, and `datasets`. This is handled in the initial setup cells of the notebook.

The `%%capture` magic command is used in the installation cell to suppress the extensive output generated by the installation process, keeping the notebook clean.

The following code snippet shows the installation process, including a conditional check for Google Colab environments to ensure the correct dependencies are installed:

```python
%%capture
import os
if "COLAB_" not in "".join(os.environ.keys()):
    !pip install unsloth
else:
    # Do this only in Colab notebooks! Otherwise use pip install unsloth
    !pip install --no-deps bitsandbytes accelerate xformers==0.0.29.post3 peft trl triton cut_cross_entropy unsloth_zoo
    !pip install sentencepiece protobuf "datasets>=3.4.1" huggingface_hub hf_transfer
    !pip install --no-deps unsloth
```
### Data Preparation

The dataset for finetuning is loaded using the `load_dataset` function from the `datasets` library. In this case, the `yahma/alpaca-cleaned` dataset is used.

A formatting function, `formatting_prompts_func`, is defined to structure the instruction, input, and output from the dataset into a prompt format suitable for training. The `EOS_TOKEN` is added to the end of each formatted text to indicate the end of a sequence. The dataset is then mapped using this function.
alpaca_prompt = """Below is an instruction that describes a task, paired with an input that provides further context. Write a response that appropriately completes the request.

### Instruction:
{}

### Input:
{}

### Response:
{}
"""

from datasets import load_dataset

# Ensure tokenizer is defined before using EOS_TOKEN
EOS_TOKEN = tokenizer.eos_token  # Must add EOS_TOKEN

def formatting_prompts_func(examples):
    instructions = examples["instruction"]
    inputs = examples["input"]
    outputs = examples["output"]
    texts = []
    for instruction, input, output in zip(instructions, inputs, outputs):
        # Must add EOS_TOKEN, otherwise your generation will go on forever!
        text = alpaca_prompt.format(instruction, input, output) + EOS_TOKEN
        texts.append(text)
    return {"text": texts}

dataset = load_dataset("yahma/alpaca-cleaned", split="train")
dataset = dataset.map(formatting_prompts_func, batched=True)

from trl import SFTTrainer
from transformers import TrainingArguments
from unsloth import is_bfloat16_supported

trainer = SFTTrainer(
    model=model,
    tokenizer=tokenizer,
    train_dataset=dataset,
    dataset_text_field="text",
    max_seq_length=max_seq_length,
    dataset_num_proc=2,
    args=TrainingArguments(
        per_device_train_batch_size=2,
        gradient_accumulation_steps=4,
        # Use num_train_epochs = 1, warmup_ratio for full training runs!
        warmup_steps=5,
        max_steps=60,
        learning_rate=2e-4,
        fp16=not is_bfloat16_supported(),
        bf16=is_bfloat16_supported(),
        logging_steps=1,
        optim="adamw_8bit",
        weight_decay=0.01,
        lr_scheduler_type="linear",
        seed=3407,
        output_dir="outputs",
        report_to="none",  # Use this for WandB etc
    ),
)

trainer_stats = trainer.train()
)

FastLanguageModel.for_inference(model)  # Unsloth has 2x faster inference!
inputs = tokenizer(
    [
        alpaca_prompt.format(
            "Continue the fibonnaci sequence.",  # instruction
            "1, 1, 2, 3, 5, 8",  # input
            "",  # output - leave this blank for generation!
        )
    ],
    return_tensors="pt"
).to("cuda")

outputs = model.generate(**inputs, max_new_tokens=64, use_cache=True)
print(tokenizer.batch_decode(outputs))

from transformers import TextStreamer
text_streamer = TextStreamer(tokenizer)
_ = model.generate(**inputs, streamer=text_streamer, max_new_tokens=128)

model.save_pretrained("lora_model")  # Local saving
tokenizer.save_pretrained("lora_model")

model.save_pretrained("lora_model")  # Local saving
tokenizer.save_pretrained("lora_model")

import wandb
model.save_pretrained("lora_model")  # Local saving
tokenizer.save_pretrained("lora_model")

wandb.login()

run = wandb.init(project="qwen2-finetuning", job_type="upload-model")

# Create a W&B artifact for the model
artifact = wandb.Artifact(
    name="qwen2-lora-model",
    type="model",
    description="LoRA adapters for Qwen2-7B finetuned on Alpaca dataset",
)

# Add the model directory to the artifact
artifact.add_dir("lora_model")

# Log the artifact
run.log_artifact(artifact)

# Finish the WandB run
run.finish()
from unsloth import FastLanguageModel
from unsloth.chat_templates import get_chat_template
from datasets import load_dataset

model, tokenizer = FastLanguageModel.from_pretrained(
    model_name = "unsloth/codegemma-7b-bnb-4bit",  # Choose ANY! eg teknium/OpenHermes-2.5-Mistral-7B
    max_seq_length = max_seq_length,
    dtype = dtype,
    load_in_4bit = load_in_4bit,
    # token = "hf_...", # use one if using gated models like meta-llama/Llama-2-7b-hf
)

model = FastLanguageModel.get_peft_model(
    model,
    r = 16, # Choose any number > 0 ! Suggested 8, 16, 32, 64, 128
    target_modules = ["q_proj", "k_proj", "v_proj", "o_proj",
                      "gate_proj", "up_proj", "down_proj",],
    lora_alpha = 16,
    lora_dropout = 0, # Supports any, but = 0 is optimized
    bias = "none",    # Supports any, but = "none" is optimized
    use_gradient_checkpointing = "unsloth", # True or "unsloth" for very long context
    random_state = 3407,
    use_rslora = False,  # We support rank stabilized LoRA
    loftq_config = None, # And LoftQ
)

tokenizer = get_chat_template(
    tokenizer,
    chat_template = "chatml", # Supports zephyr, chatml, mistral, llama, alpaca, vicuna, vicuna_old, unsloth
    mapping = {"role" : "from", "content" : "value", "user" : "human", "assistant" : "gpt"}, # ShareGPT style
    map_eos_token = True, # Maps <|im_end|> to </s> instead
)
    mapping = {"role" : "from", "content" : "value", "user" : "human", "assistant" : "gpt"}, # ShareGPT style
    map_eos_token = True, # Maps <|im_end|> to </s> instead
)

def formatting_prompts_func(examples):
    convos = examples["conversations"]
    texts = [tokenizer.apply_chat_template(convo, tokenize = False, add_generation_prompt = False) for convo in convos]
    return { "text" : texts, }

dataset = load_dataset("philschmid/guanaco-sharegpt-style", split = "train")
dataset = dataset.map(formatting_prompts_func, batched = True,)
def formatting_prompts_func(examples):
    convos = examples["conversations"]
    texts = [tokenizer.apply_chat_template(convo, tokenize = False, add_generation_prompt = False) for convo in convos]
    return { "text" : texts, }

dataset = load_dataset("philschmid/guanaco-sharegpt-style", split = "train")
dataset = dataset.map(formatting_prompts_func, batched = True)

from trl import SFTTrainer, SFTConfig

trainer = SFTTrainer(
    model = model,
    tokenizer = tokenizer,
    train_dataset = dataset,
    packing = False,  # Can make training 5x faster for short sequences.
    args = SFTConfig(
        per_device_train_batch_size = 1,
        gradient_accumulation_steps = 4,
        warmup_steps = 5,
        max_steps = 20,
        learning_rate = 2e-4,
        optim = "adamw_8bit",
        weight_decay = 0.01,
        lr_scheduler_type = "linear",
        seed = 3407,
        dataset_text_field = "text",
        report_to = "none",  # Use this for WandB etc
        max_grad_norm = 0.3,
    ),
)

trainer_stats = trainer.train()
    messages,
    tokenize = True,
    add_generation_prompt = True, # Must add for generation
    return_tensors = "pt",
).to("cuda")

outputs = model.generate(input_ids = inputs, max_new_tokens = 64, use_cache = True)
tokenizer.batch_decode(outputs)

text_streamer = TextStreamer(tokenizer)
_ = model.generate(input_ids = inputs, streamer = text_streamer, max_new_tokens = 128, use_cache = True)

You can merge the finetuned LoRA adapters with the base model into different formats using `model.save_pretrained_merged` or push them to the Hugging Face Hub using `model.push_to_hub_merged`.

To merge to a 16-bit format:python
# Merge to 16bit
if False: model.save_pretrained_merged("model", tokenizer, save_method = "merged_16bit",)
if False: model.push_to_hub_merged("hf/model", tokenizer, save_method = "merged_16bit", token = "")

To merge to a 4-bit format:python
from transformers import TextStreamer

FastLanguageModel.for_inference(model) # Enable native 2x faster inference

messages = [
    {"from": "human", "value": "Continue the fibonnaci sequence: 1, 1, 2, 3, 5, 8,"},
]
inputs = tokenizer.apply_chat_template(
    messages,
    tokenize = True,
    add_generation_prompt = True, # Must add for generation
    return_tensors = "pt",
).to("cuda")

outputs = model.generate(input_ids = inputs, max_new_tokens = 64, use_cache = True)
print(tokenizer.batch_decode(outputs))

text_streamer = TextStreamer(tokenizer)
_ = model.generate(input_ids = inputs, streamer = text_streamer, max_new_tokens = 128, use_cache = True)
# Save to 8bit Q8_0
if False: model.save_pretrained_gguf("model", tokenizer,)
if False: model.push_to_hub_gguf("hf/model", tokenizer, token = "")

To save to 16-bit GGUF format:python
# Save to 16bit GGUF
if False: model.save_pretrained_gguf("model", tokenizer, quantization_method = "f16")
if False: model.push_to_hub_gguf("hf/model", tokenizer, quantization_method = "f16", token = "")

To save to q4_k_m GGUF format:python
# Save to q4_k_m GGUF
if False: model.save_pretrained_gguf("model", tokenizer, quantization_method = "q4_k_m")
if False: model.push_to_hub_gguf("hf/model", tokenizer, quantization_method = "q4_k_m", token = "")

# Colab Notebook :
https://colab.research.google.com/drive/1XTETt9ZmcZc_Kuo-logdgK8G3ivO2Fm1
