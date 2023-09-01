[toc]

# Task description

## 1. Task workload and input

The AI task, BLOOM Inference, is a **distributed inference of BLOOM-176B model** on **two NCI Gadi V100x4 GPU compute nodes**. 

Input is **BLOOM-176B model files released by BigScience and Microsoft**, which can be downloaded at https://huggingface.co.

The python project codes of the downstream task, token generation by HuggingFace, are available at https://github.com/huggingface/transformers-bloom-inference/blob/main/bloom-inference-scripts.

The objective is to speed up the inference without adapting the model itself.

It is recommended to use the **ds-inference shard-int8** code for less loading time and higher throughputs.

## 2. Rules

The user is not allowed to make any changes to the model itself. Any changes preserving the problem's output is allowed.

The results will be executed in **two Gadi GPU gpuvolta servers** and graded based on best "**Throughput per token including tokenize**" achieved.

## 3. How to run the task and validate the results

Refer to the BLOOM Inference Application Notes to run the task.

It is demanded that the generated tokens be identical for runs that use the same precision(int8/fp16/bf16) and framework(HuggingFace/DeepSpeed).

## 4. Submission and Presentation 

\- Submit all your build scripts, setup scripts, run scripts, inputs, and output text files.

\- Do **not** submit the large data files.

\- Prepare slides for the team’s interview based on your work for this application.
