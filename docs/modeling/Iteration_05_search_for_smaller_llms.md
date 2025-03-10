# Iteration 5. Search for smaller LLMs

_09-08-2024_

<!---
The work is done using short iterations. Each iteration needs to have a very
clear goal. This allows to gain greater knowledge of the problem on each iteration.
--->

## Goal

Search an LLM that is smaller than Phi-3 and is fast enough to be fine-tuned on Kaggle within the submission time (12 hours)

Can I get a better result than Phi-3 by using a smaller model with test time fine-tuning?

## Motivation

In the previous iteration I have seen that test time fine-tuning works on the eval set. However when doing a proof of concept
on Kaggle's hardware I have found that to be able to fine-tune Phi-3 with a sequence length of 4096 first I have
to use int4 quantization and second the fine-tuning would take 56 hours for 1k steps.

## Development

### Candidate search

- I could try with previous versions of Phi-3 that I believe were smaller.
- I have also read recently about [smollm](https://huggingface.co/blog/smollm)
- [MobileLLM](https://github.com/facebookresearch/MobileLLM) is cited by smollm, seems to be developed by Facebook but the models do not seem to be available.

However Smollm models have only 2k context length.

## Results

- [Fine-tuning on ARC tasks](https://wandb.ai/guillermobarbadillo/20240809_smaller_models?nw=nwuserguillermobarbadillo)
- [Test-time fine-tuning](https://wandb.ai/guillermobarbadillo/20240812_smaller_models_test-time_fine-tuning?nw=nwuserguillermobarbadillo)

### Results summary

| model               | parameters (B) | eval accuracy | ttft eval accuracy | test accuracy | ttft test accuracy |
|---------------------|----------------|---------------|--------------------|---------------|--------------------|
| Phi-3               | 3.8            | 6.50%         | **16.80%**         | 3.00%         | -                  |
| Qwen2-1.5B-Instruct | 1.5            | **7.10%**     | 15.90%             | **4.00%**     | *4.00%             |
| Qwen2-0.5B-Instruct | 0.5            | 4.60%         | 12.40%             | 0.00%         | ***5.00%**         |

We have been able to improve accuracy on the test set by using Qwen2 models instead of Phi-3. I'm not sure if test-time
fine-tuning is correctly implemented on Kaggle, there were some weird results. I have to unify the code to verify that everything is correct.

### Test time fine-tuning speed on Kaggle

| Model                    | parameters (M) | max context length (k) | seq_len | 10 steps time (min) | 1000 steps time (hours) |
|--------------------------|----------------|------------------------|---------|---------------------|-------------------------|
| Phi-3 (3.8B)             | 3800           | 128                    | 1024    | 3                   | 5.0                     |
| Phi-3 int4 (3.8B)        | 3800           | 128                    | 4096    | 30                  | 50.0                    |
| Qwen/Qwen2-1.5B-Instruct | 1500           | 32                     | 3584    | 2                   | 3.3                     |
| Qwen/Qwen2-0.5B-Instruct | 500            | 32                     | 4096    | 1                   | 1.7                     |

So far Phi-3 and Qwen2 are the only small LLMs that I have found that have enough context lenght to deal with ARC tasks. However
Phi-3 requires too much memory for fine-tuning and if I use int4 quantization it is very slow.

The option of using Qwen2 seems to be the best one given the current hardware limitations.

This inference times are measured when using `float16` type, I have found that it is 4 times faster than `bfloat16` on the T4 GPU. The next section shows
the same measurements with more models and `bfloat16` for reference.

#### Slower previous results with `bfloat16`

| Model                        | parameters (M) | max context length (k) | seq_len | 10 steps time (min) | 1000 steps time (hours) |
|------------------------------|----------------|------------------------|---------|---------------------|-------------------------|
| Phi-3 (3.8B)                 | 3800           | 128                    | 1024    | 16                  | 26.7                    |
| Phi-3 int4 (3.8B)            | 3800           | 128                    | 4096    | 34                  | 56.7                    |
| Phi-2 (2.7B) int4            | 2700           | 2                      | 4096    | 15                  | 25.0                    |
| **Phi-1.5 (1.3B)**           | 1300           | 2                      | 3072    | 7                   | 11.7                    |
| **Phi-1.5 (1.3B) int4**      | 1300           | 2                      | 4096    | 7                   | 11.7                    |
| SmolLM-1.7B-Instruct         | 1700           | 2                      | 2048    | 11                  | 18.3                    |
| SmolLM-1.7B-Instruct int4    | 1700           | 2                      | 4096    | 15.5                | 25.8                    |
| **SmolLM-360M-Instruct**     | 360            | 2                      | 4096    | 3.75                | 6.3                     |
| **SmolLM-135M-Instruct**     | 135            | 2                      | 4096    | 1.5                 | 2.5                     |
| **Qwen/Qwen2-0.5B-Instruct** | 500            | 32                     | 3072    | 4                   | 6.7                     |

- To be able to run the code on Kaggle I have a limit of 12 hours. The table above shows that many of the tried models cannot be fine-tuned as fast as necessary to be able to make a submission.
- `SmolLM` models only have a context length of 2k tokens, that is not enough for the ARC challenge
- `Qwen2-0.5B-Instruct` is a good candidate because it has a context length of 32 and I should be able to train in 7h which is enough to make a submission.

### Role of the rank of LoRA

![role of r](res/role-of-r.png)

I have made experiments with different values for the rank of LoRA. It has a clear effect on the train loss: when we use a higher rank we get a lower train loss. However the effect on the validation loss is not clear, it seems that using a very small rank such as 4 is harmful but other than that the differences do not seem to be significative.

### Kaggle runtimes for reference

- `qwen2-0.5b-instruct` test set evaluation: 1h15
- `qwen2-0.5b-instruct` 1k steps fine-tuning: 2h19
- `qwen2-1.5b-instruct` test set evaluation: 1h45
- `qwen2-1.5b-instruct` 1k steps fine-tuning: ~5h

### 2 stage test-time fine-tuning

I had the intuition that maybe doing the test-time fine-tuning in two stages could improve the validation loss: a first one with data augmentation and a second one without data augmentation.

However I have tried with a wide range of learning rates for fine-tuning and no consistent improvement was seen.

![2-stage-fine-tuning](res/2-stage-fine-tuning.png)

### float16 vs bfloat16

I have already seen that in Kaggle fine-tuning with float16 is 4 times faster than with bfloat16.
Let's see if that is the case for my computer (2x3090):

- `bfloat16` 50 steps fine-tuning: 2m45, 2m48
- `float16` 50 steps fine-tuning: 2m43, 2m46

Memory use is the same and speed is the same. Maybe it is using `bfloat16` internally all the time, I remember listening something like that long ago.

I have also run a quick experiment with 2 samples per GPU but there was no speedup.

### Qwen2-0.5B-Instruct vs Qwen2-1.5B-Instruct

![qwen-comparison](res/qwen-comparison.png)

The bigger 1.5B parameters model learned faster on the train set and generalized better to the eval set.

Would this continue to scale with bigger models? That would be interesting to see, f.e. fine-tuning GPT4-o.
Given the hardware constraints of the challenge I cannot afford using much bigger models than this.

For reference Qwen2-1.5B with LoRA r32 uses 8.5M of parameters, while Qwen2-0.5B with r128 uses ~17M of parameters.
So the number of trainable parameters is not the difference betweeen trainings. Simply the bigger model seems
to have more prior knowledge about the task.

### Optimal number of steps for test-time fine-tuning

There are no clear signs of overfitting, but it seems to be around 3k steps when using the whole eval set.

## Conclusion

`Qwen2-1.5B-Instruct` and `Qwen2-0.5B-Instruct` are two replacement for `Phi-3` that are possible to be test-time fine-tuned on Kaggle.

We have improved the test leaderboard score from 3 to 5 on this iteration.

- I have found that `float16` is 4 times faster than `bfloat16` when fine-tuning on Kaggle
- The bigger qwen2-1.5b model learns faster than the 0.5b version

## Next steps

- Decomposing the task into step by step could be helpful. That is, creating multiple grids until creating the final output.
- [ ] What is the effect of changing the train data? Keep the same train configuration and just change the data.
    - [ ] How the test accuracy changes if I train in the eval set?
    - [ ] How the test accuracy changes if I train in both the train and eval set?
    - [ ] What if I use the train dataset and n-1 eval dataset?
    - [ ] What if I only do test-time fine-tuning?
    - [ ] Is it helpful to first train with re-arc?
    - [ ] Use other arc-like datasets for training
- [ ] Could I speedup Qwen2 training by using a single gpu?
- [ ] Maybe I have to start using just 100 eval tasks for validation, that way my test-time fine-tuning experiments will match the Kaggle setup. Also I would have an additional 300 tasks for training.

## TODO

- [x] Can I run Smollm models with VLLM? The architecture is supported... https://docs.vllm.ai/en/latest/models/supported_models.html
- [ ] Long context fine-tuning https://huggingface.co/blog/wenbopan/long-context-fine-tuning
- [x] What is the role of LoRA `r`?
- [ ] Read about previous editions of the ARC challenges to see if any other data augmentation is used.
- [x] Make a submission with test fine-tuning
- [ ] Remember that the test is 4 times smaller than the eval set
- [x] 2 stage test-time fine-tuning
- [x] Bfloat vs float
- [x] Qwen/Qwen2-1.5B-Instruct
- [x] What is the number of steps in the test-time fine-tuning where overfitting starts to happen? On previous experiments it seemed to be above 5k steps
- [x] Document Kaggle training and inference speeds.
- [ ] Why experiments on Kaggle are getting worse training losses?
- [x] Document results and close iteration
