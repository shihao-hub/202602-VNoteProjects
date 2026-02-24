# 进阶模块四：模型微调

> 定制化 LLM 以适应特定任务

---

## 第35章：模型微调基础

### 35.1 什么是微调

```
微调（Fine-tuning）是在预训练模型基础上，用特定数据进一步训练

┌─────────────────────────────────────────────────────────┐
│                    微调的概念                           │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  预训练模型                                             │
│  ───────────                                            │
│  • 在海量通用数据上训练                                │
│  • 学习了语言的通用知识                                │
│  • 类比：受过通识教育的人                              │
│                                                         │
│  微调                                                   │
│  ─────                                                  │
│  • 在特定任务/领域数据上继续训练                       │
│  • 让模型适应特定需求                                  │
│  • 类比：专业培训                                      │
│                                                         │
│  微调后的模型                                           │
│  ─────────────                                          │
│  • 保留通用能力                                        │
│  • 在特定任务上表现更好                                │
│  • 类比：专业人士                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 35.2 微调 vs 其他方法

```
┌─────────────────────────────────────────────────────────┐
│              不同定制化方法对比                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  Prompt Engineering                                     │
│  ─────────────────                                      │
│  • 不改变模型权重                                      │
│  • 通过提示引导模型                                    │
│  • 成本：最低                                          │
│  • 效果：有限，依赖模型原有能力                        │
│  • 适合：通用任务、快速实验                            │
│                                                         │
│  RAG（检索增强）                                        │
│  ───────────────                                        │
│  • 不改变模型权重                                      │
│  • 提供外部知识                                        │
│  • 成本：中等（需要构建知识库）                        │
│  • 效果：好，适合知识密集型任务                        │
│  • 适合：需要特定知识的问答                            │
│                                                         │
│  Fine-tuning（微调）                                    │
│  ─────────────────                                      │
│  • 改变模型权重                                        │
│  • 学习特定模式/风格/知识                              │
│  • 成本：较高（需要数据和计算资源）                    │
│  • 效果：最好，深度定制                                │
│  • 适合：需要特定行为/风格的任务                       │
│                                                         │
│  选择指南：                                             │
│  ┌────────────────────────────────────────────────┐    │
│  │ 需求                    │ 推荐方法             │    │
│  ├────────────────────────────────────────────────┤    │
│  │ 使用最新/私有知识       │ RAG                  │    │
│  │ 改变输出风格/格式       │ Fine-tuning          │    │
│  │ 学习特定任务模式        │ Fine-tuning          │    │
│  │ 快速实验                │ Prompt Engineering   │    │
│  │ 成本敏感                │ Prompt > RAG > FT    │    │
│  └────────────────────────────────────────────────┘    │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 35.3 微调的类型

```
全量微调 vs 参数高效微调

┌─────────────────────────────────────────────────────────┐
│                                                         │
│  全量微调（Full Fine-tuning）                           │
│  ────────────────────────────                           │
│  • 更新模型所有参数                                    │
│  • 效果最好                                            │
│  • 需要大量 GPU 内存                                   │
│  • 训练成本高                                          │
│                                                         │
│  7B 模型全量微调需要：~60GB+ GPU 内存                  │
│                                                         │
│  ─────────────────────────────────────────────────     │
│                                                         │
│  参数高效微调（PEFT）                                   │
│  ────────────────────                                   │
│  • 只更新少量参数                                      │
│  • 冻结大部分模型权重                                  │
│  • GPU 内存需求大幅降低                                │
│  • 效果接近全量微调                                    │
│                                                         │
│  常见 PEFT 方法：                                       │
│  • LoRA（Low-Rank Adaptation）★ 最流行                 │
│  • QLoRA（Quantized LoRA）                             │
│  • Adapter                                              │
│  • Prefix Tuning                                        │
│  • Prompt Tuning                                        │
│                                                         │
│  7B 模型 LoRA 微调：~16GB GPU 内存                     │
│  7B 模型 QLoRA 微调：~8GB GPU 内存                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## 第36章：LoRA 与 QLoRA

### 36.1 LoRA 原理

```
LoRA = Low-Rank Adaptation（低秩适配）

核心思想：
• 冻结原始模型权重
• 只训练小的"适配器"矩阵
• 推理时将适配器合并到原始权重

┌─────────────────────────────────────────────────────────┐
│                    LoRA 原理图解                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  原始权重矩阵 W (d × d)：冻结，不更新                  │
│                                                         │
│        输入 x                                           │
│           │                                             │
│     ┌─────┴─────┐                                      │
│     │           │                                       │
│     ▼           ▼                                       │
│  ┌─────┐    ┌─────┐                                    │
│  │  W  │    │  A  │ ← 降维矩阵 (d × r)，可训练        │
│  │冻结 │    │     │   r << d（如 r=8,16,32）          │
│  └──┬──┘    └──┬──┘                                    │
│     │          │                                        │
│     │          ▼                                        │
│     │       ┌─────┐                                    │
│     │       │  B  │ ← 升维矩阵 (r × d)，可训练        │
│     │       └──┬──┘                                    │
│     │          │                                        │
│     └────┬─────┘                                       │
│          │                                              │
│          ▼                                              │
│       输出 = Wx + BAx                                  │
│                                                         │
│  参数量对比（假设 d=4096, r=8）：                       │
│  • 原始：4096 × 4096 = 16M 参数                       │
│  • LoRA：4096 × 8 × 2 = 64K 参数                      │
│  • 压缩比：约 250 倍！                                 │
│                                                         │
└─────────────────────────────────────────────────────────┘

为什么 LoRA 有效？
• 微调时的权重变化是低秩的
• 用小矩阵就能近似表达这些变化
• 类比：用几个方向的变化来近似高维空间的变化
```

### 36.2 QLoRA 原理

```
QLoRA = Quantized LoRA（量化 LoRA）

在 LoRA 基础上增加量化，进一步降低内存

┌─────────────────────────────────────────────────────────┐
│                    QLoRA 核心技术                       │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 4-bit NormalFloat (NF4) 量化                       │
│  ─────────────────────────────                          │
│  • 将模型权重从 FP16 量化到 4-bit                      │
│  • 特殊的数据类型，适合正态分布的权重                  │
│  • 比普通 INT4 效果更好                                │
│                                                         │
│  2. 双重量化（Double Quantization）                     │
│  ─────────────────────────────────                      │
│  • 对量化常数再量化                                    │
│  • 进一步节省内存                                      │
│                                                         │
│  3. 分页优化器（Paged Optimizers）                      │
│  ─────────────────────────────────                      │
│  • 使用 NVIDIA 统一内存                                │
│  • 防止 GPU 内存溢出                                   │
│                                                         │
│  内存对比（7B 模型）：                                  │
│  • 全量微调 FP16：~60GB                                │
│  • LoRA FP16：~16GB                                    │
│  • QLoRA NF4：~6GB ★                                   │
│                                                         │
│  效果：                                                 │
│  • 接近全量微调的质量                                  │
│  • 可以在消费级 GPU（如 RTX 4090）上微调大模型        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 36.3 实战：使用 QLoRA 微调

```python
# 安装依赖
# pip install transformers peft bitsandbytes accelerate datasets trl

from transformers import (
    AutoModelForCausalLM,
    AutoTokenizer,
    BitsAndBytesConfig,
    TrainingArguments,
)
from peft import LoraConfig, get_peft_model, prepare_model_for_kbit_training
from datasets import load_dataset
from trl import SFTTrainer

# === 1. 配置量化 ===
bnb_config = BitsAndBytesConfig(
    load_in_4bit=True,              # 使用 4-bit 量化
    bnb_4bit_quant_type="nf4",      # NF4 量化类型
    bnb_4bit_compute_dtype="float16", # 计算精度
    bnb_4bit_use_double_quant=True,  # 双重量化
)

# === 2. 加载模型 ===
model_name = "meta-llama/Llama-2-7b-hf"  # 或其他模型

model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto",
    trust_remote_code=True,
)

tokenizer = AutoTokenizer.from_pretrained(model_name)
tokenizer.pad_token = tokenizer.eos_token

# === 3. 配置 LoRA ===
lora_config = LoraConfig(
    r=16,                    # LoRA 秩
    lora_alpha=32,           # LoRA alpha
    target_modules=[         # 要适配的模块
        "q_proj", "k_proj", "v_proj", "o_proj",  # 注意力
        "gate_proj", "up_proj", "down_proj"      # FFN
    ],
    lora_dropout=0.05,
    bias="none",
    task_type="CAUSAL_LM",
)

# === 4. 准备模型 ===
model = prepare_model_for_kbit_training(model)
model = get_peft_model(model, lora_config)

# 打印可训练参数
model.print_trainable_parameters()
# 输出类似：trainable params: 4,194,304 || all params: 6,742,609,920 || trainable%: 0.0622

# === 5. 准备数据 ===
# 数据格式示例
"""
{
    "instruction": "将以下英文翻译成中文",
    "input": "Hello, how are you?",
    "output": "你好，你怎么样？"
}
"""

def format_instruction(sample):
    """格式化为对话格式"""
    return f"""### 指令：
{sample['instruction']}

### 输入：
{sample['input']}

### 回答：
{sample['output']}"""

dataset = load_dataset("your_dataset", split="train")

# === 6. 配置训练 ===
training_args = TrainingArguments(
    output_dir="./lora_output",
    num_train_epochs=3,
    per_device_train_batch_size=4,
    gradient_accumulation_steps=4,
    learning_rate=2e-4,
    warmup_steps=100,
    logging_steps=10,
    save_steps=500,
    fp16=True,
    optim="paged_adamw_8bit",  # 使用分页优化器
)

# === 7. 开始训练 ===
trainer = SFTTrainer(
    model=model,
    train_dataset=dataset,
    tokenizer=tokenizer,
    args=training_args,
    formatting_func=format_instruction,
    max_seq_length=512,
)

trainer.train()

# === 8. 保存 LoRA 权重 ===
trainer.save_model("./lora_weights")

# === 9. 使用微调后的模型 ===
from peft import PeftModel

# 加载基础模型
base_model = AutoModelForCausalLM.from_pretrained(
    model_name,
    quantization_config=bnb_config,
    device_map="auto",
)

# 加载 LoRA 权重
model = PeftModel.from_pretrained(base_model, "./lora_weights")

# 推理
inputs = tokenizer("你好，", return_tensors="pt").to("cuda")
outputs = model.generate(**inputs, max_new_tokens=50)
print(tokenizer.decode(outputs[0]))
```

---

## 第37章：微调数据准备

### 37.1 数据格式

```
常见的微调数据格式：

┌─────────────────────────────────────────────────────────┐
│                    数据格式类型                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 指令微调格式（Instruction Tuning）                  │
│  ─────────────────────────────────────                  │
│  {                                                      │
│    "instruction": "将英文翻译成中文",                  │
│    "input": "Hello World",                             │
│    "output": "你好世界"                                │
│  }                                                      │
│                                                         │
│  2. 对话格式（Chat Format）                             │
│  ──────────────────────────                             │
│  {                                                      │
│    "conversations": [                                   │
│      {"role": "user", "content": "你好"},              │
│      {"role": "assistant", "content": "你好！有什么..."}│
│    ]                                                    │
│  }                                                      │
│                                                         │
│  3. Alpaca 格式                                         │
│  ───────────────                                        │
│  {                                                      │
│    "instruction": "...",                               │
│    "input": "...",       # 可选                        │
│    "output": "..."                                     │
│  }                                                      │
│                                                         │
│  4. ShareGPT 格式                                       │
│  ─────────────────                                      │
│  {                                                      │
│    "conversations": [                                   │
│      {"from": "human", "value": "..."},               │
│      {"from": "gpt", "value": "..."}                  │
│    ]                                                    │
│  }                                                      │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 37.2 数据质量原则

```
高质量数据是微调成功的关键

┌─────────────────────────────────────────────────────────┐
│                    数据质量原则                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  1. 多样性（Diversity）                                 │
│  ───────────────────────                                │
│  • 覆盖不同类型的任务                                  │
│  • 不同的表达方式                                      │
│  • 避免数据过于集中                                    │
│                                                         │
│  2. 准确性（Accuracy）                                  │
│  ─────────────────────                                  │
│  • 输出内容正确                                        │
│  • 没有事实错误                                        │
│  • 人工验证关键数据                                    │
│                                                         │
│  3. 一致性（Consistency）                               │
│  ─────────────────────                                  │
│  • 格式统一                                            │
│  • 风格一致                                            │
│  • 遵循相同的标注规范                                  │
│                                                         │
│  4. 相关性（Relevance）                                 │
│  ─────────────────────                                  │
│  • 与目标任务相关                                      │
│  • 去除无关数据                                        │
│                                                         │
│  5. 数量平衡                                            │
│  ───────────                                            │
│  • 不同类别数量平衡                                    │
│  • 避免过度偏向某类数据                                │
│                                                         │
│  数据量建议：                                           │
│  • 简单任务：几百到几千条                              │
│  • 复杂任务：几千到几万条                              │
│  • 通用对话：几万到几十万条                            │
│                                                         │
│  质量 > 数量！                                         │
│  1000 条高质量数据 > 10000 条低质量数据                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 37.3 数据准备示例

```python
import json
from typing import List, Dict
import random

class DatasetBuilder:
    def __init__(self):
        self.data = []
    
    def add_instruction(self, instruction: str, input_text: str, output: str):
        """添加指令数据"""
        self.data.append({
            "instruction": instruction,
            "input": input_text,
            "output": output
        })
    
    def add_conversation(self, messages: List[Dict[str, str]]):
        """添加对话数据"""
        self.data.append({
            "conversations": messages
        })
    
    def from_qa_pairs(self, qa_pairs: List[tuple]):
        """从问答对构建"""
        for question, answer in qa_pairs:
            self.add_instruction(
                instruction="回答以下问题",
                input_text=question,
                output=answer
            )
    
    def validate(self) -> List[str]:
        """验证数据质量"""
        issues = []
        for i, item in enumerate(self.data):
            # 检查空值
            if "instruction" in item:
                if not item.get("output", "").strip():
                    issues.append(f"第 {i} 条数据：output 为空")
            elif "conversations" in item:
                if len(item["conversations"]) < 2:
                    issues.append(f"第 {i} 条数据：对话轮数不足")
        return issues
    
    def split(self, train_ratio: float = 0.9):
        """划分训练集和验证集"""
        random.shuffle(self.data)
        split_idx = int(len(self.data) * train_ratio)
        return self.data[:split_idx], self.data[split_idx:]
    
    def save(self, path: str):
        """保存数据集"""
        with open(path, "w", encoding="utf-8") as f:
            json.dump(self.data, f, ensure_ascii=False, indent=2)


# 使用示例
builder = DatasetBuilder()

# 添加指令数据
builder.add_instruction(
    instruction="将以下文本总结为一句话",
    input_text="人工智能是计算机科学的一个分支，致力于创建能够执行通常需要人类智能的任务的系统...",
    output="人工智能是计算机科学的一个分支，旨在创建模拟人类智能的系统。"
)

# 添加对话数据
builder.add_conversation([
    {"role": "user", "content": "Python 中如何反转列表？"},
    {"role": "assistant", "content": "在 Python 中有几种方法反转列表：\n1. 使用切片：`reversed_list = original[::-1]`\n2. 使用 reverse() 方法：`original.reverse()`\n3. 使用 reversed() 函数：`reversed_list = list(reversed(original))`"}
])

# 验证
issues = builder.validate()
if issues:
    print("发现问题：", issues)

# 划分并保存
train_data, val_data = builder.split()
builder.data = train_data
builder.save("train.json")
builder.data = val_data
builder.save("val.json")
```

---

## 第38章：微调最佳实践

### 38.1 超参数选择

```
┌─────────────────────────────────────────────────────────┐
│                   关键超参数指南                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  LoRA 参数                                              │
│  ────────────                                           │
│  r（秩）：                                              │
│  • 常用值：8, 16, 32, 64                               │
│  • 越大表达能力越强，但参数越多                        │
│  • 建议从 16 开始，根据效果调整                        │
│                                                         │
│  lora_alpha：                                           │
│  • 通常设为 2 × r                                      │
│  • 控制 LoRA 权重的缩放                                │
│                                                         │
│  target_modules：                                       │
│  • 一般选择注意力层的 Q、K、V、O                       │
│  • 可以增加 FFN 层获得更好效果                         │
│                                                         │
│  训练参数                                               │
│  ──────────                                             │
│  学习率：                                               │
│  • LoRA 建议：1e-4 到 3e-4                            │
│  • 全量微调：1e-5 到 5e-5                             │
│                                                         │
│  batch_size：                                           │
│  • 受 GPU 内存限制                                     │
│  • 用 gradient_accumulation 模拟大 batch              │
│  • 有效 batch = batch_size × accumulation_steps       │
│                                                         │
│  epochs：                                               │
│  • 通常 2-5 epochs                                    │
│  • 数据少时可以多训几轮                                │
│  • 注意过拟合                                          │
│                                                         │
│  warmup：                                               │
│  • 通常 3-10% 的总步数                                │
│  • 帮助训练稳定                                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 38.2 训练监控

```python
from transformers import TrainerCallback
import wandb

class MonitorCallback(TrainerCallback):
    """训练监控回调"""
    
    def on_log(self, args, state, control, logs=None, **kwargs):
        if logs:
            # 记录到 wandb
            wandb.log({
                "train/loss": logs.get("loss"),
                "train/learning_rate": logs.get("learning_rate"),
                "train/epoch": logs.get("epoch"),
            })
    
    def on_evaluate(self, args, state, control, metrics=None, **kwargs):
        if metrics:
            wandb.log({
                "eval/loss": metrics.get("eval_loss"),
                "eval/perplexity": 2 ** metrics.get("eval_loss", 0),
            })

# 检查过拟合
def check_overfitting(train_losses: list, eval_losses: list, threshold: float = 0.2):
    """
    检查是否过拟合
    如果训练损失持续下降而验证损失上升，可能过拟合
    """
    if len(train_losses) < 5 or len(eval_losses) < 5:
        return False
    
    # 最近几步的趋势
    train_trend = train_losses[-1] - train_losses[-5]
    eval_trend = eval_losses[-1] - eval_losses[-5]
    
    # 训练损失下降，验证损失上升
    if train_trend < -threshold and eval_trend > threshold:
        return True
    
    return False
```

### 38.3 常见问题排查

```
┌─────────────────────────────────────────────────────────┐
│                   微调常见问题                          │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  问题 1：训练不收敛                                     │
│  ────────────────────                                   │
│  症状：loss 不下降或剧烈波动                           │
│  可能原因：                                             │
│  • 学习率太大                                          │
│  • 数据格式错误                                        │
│  • 梯度爆炸                                            │
│  解决：                                                 │
│  • 降低学习率                                          │
│  • 检查数据格式                                        │
│  • 添加梯度裁剪                                        │
│                                                         │
│  问题 2：过拟合                                         │
│  ────────────                                           │
│  症状：训练 loss 很低，验证 loss 上升                 │
│  可能原因：                                             │
│  • 数据太少                                            │
│  • 训练轮数太多                                        │
│  • 模型太大                                            │
│  解决：                                                 │
│  • 增加数据或数据增强                                  │
│  • 减少训练轮数                                        │
│  • 增加 dropout                                        │
│  • 使用更小的 LoRA r                                  │
│                                                         │
│  问题 3：灾难性遗忘                                     │
│  ────────────────                                       │
│  症状：微调后在其他任务上表现变差                      │
│  可能原因：                                             │
│  • 微调数据太单一                                      │
│  • 训练太过头                                          │
│  解决：                                                 │
│  • 混入通用对话数据                                    │
│  • 减少训练轮数                                        │
│  • 使用更小的学习率                                    │
│                                                         │
│  问题 4：输出重复或退化                                 │
│  ────────────────────                                   │
│  症状：模型输出重复内容或质量下降                      │
│  可能原因：                                             │
│  • 训练数据有重复模式                                  │
│  • 过拟合                                              │
│  解决：                                                 │
│  • 检查数据多样性                                      │
│  • 使用早停（Early Stopping）                         │
│  • 调整生成参数（temperature, top_p）                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 38.4 模型合并与部署

```python
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

# === 合并 LoRA 权重到基础模型 ===
def merge_lora_weights(base_model_path: str, lora_path: str, output_path: str):
    """
    将 LoRA 权重合并到基础模型，生成完整模型
    合并后推理更快，但文件更大
    """
    # 加载基础模型（FP16）
    base_model = AutoModelForCausalLM.from_pretrained(
        base_model_path,
        torch_dtype="float16",
        device_map="cpu",  # 在 CPU 上合并
    )
    
    # 加载 LoRA
    model = PeftModel.from_pretrained(base_model, lora_path)
    
    # 合并权重
    merged_model = model.merge_and_unload()
    
    # 保存
    merged_model.save_pretrained(output_path)
    
    # 复制 tokenizer
    tokenizer = AutoTokenizer.from_pretrained(base_model_path)
    tokenizer.save_pretrained(output_path)
    
    print(f"合并后的模型已保存到：{output_path}")


# === 使用 vLLM 部署 ===
# pip install vllm
"""
# 启动 vLLM 服务
python -m vllm.entrypoints.openai.api_server \
    --model ./merged_model \
    --dtype float16 \
    --api-key your-api-key

# 使用 OpenAI SDK 调用
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="your-api-key"
)

response = client.chat.completions.create(
    model="./merged_model",
    messages=[{"role": "user", "content": "你好"}]
)
"""


# === 使用 Ollama 部署 ===
"""
# 1. 创建 Modelfile
FROM ./merged_model
PARAMETER temperature 0.7
SYSTEM "你是一个有用的助手。"

# 2. 创建模型
ollama create my-model -f Modelfile

# 3. 运行
ollama run my-model
"""
```

---

## 本章小结

```
核心要点回顾：

1. 微调基础
   ├── 微调 vs Prompt vs RAG：根据需求选择
   ├── 全量微调 vs PEFT：资源和效果的权衡
   └── 适用场景：改变风格、学习特定模式

2. LoRA 与 QLoRA
   ├── LoRA：低秩适配，只训练少量参数
   ├── QLoRA：量化 + LoRA，内存需求更低
   └── 效果接近全量微调，成本大幅降低

3. 数据准备
   ├── 常见格式：指令、对话、Alpaca、ShareGPT
   ├── 质量原则：多样性、准确性、一致性
   └── 质量 > 数量

4. 最佳实践
   ├── 超参数：r、学习率、epochs 等
   ├── 监控：loss、过拟合检测
   ├── 问题排查：不收敛、过拟合、遗忘
   └── 部署：合并权重、vLLM、Ollama
```

---

**下一章 → [第39章：多 Agent 框架概览](./12-multi-agent.md)**
