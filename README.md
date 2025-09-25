# uhn-takehome

Demonstrates zero-shot, few-shot, CoT, and LoRA tests on MedCalc-Bench with Qwen 0.6B / 1.7B.

Setup in Colab:
1. Runtime → Change runtime type → GPU (T4/A100) 
2. Run the setup cell, then proceed with whichever experiment desired

Reproduction Details:

Models: Qwen/Qwen3-0.6B, Qwen/Qwen3-1.7B
Precision: FP16; 4-bit NF4 for QLoRA
Seed: 42

By default, the notebook writes results and adapters into Google Drive:

- **Results CSVs** (accuracy by category) → `/content/drive/MyDrive/medcalc/results/*.csv`
- **LoRA adapters** (trained weights) → `/content/drive/MyDrive/medcalc/qwen*_adapter/`

If you want to reproduce from scratch:
1. Run the notebook end to end (training will regenerate adapters).
2. Or skip LoRA sections and use the zero/few shot baselines only.



## Benchmark Results

| Setting                  | Model | Overall | Lab   | Risk  | Physical | Severity | Diagnosis | Date  | Dosage |
|--------------------------|-------|---------|-------|-------|----------|----------|-----------|-------|--------|
| **Zero-shot**            | 0.6B  | 0.0334  | 0.049 | 0.038 | 0.021    | 0.025    | 0.000     | 0.000 | 0.075  |
|                          | 1.7B  | 0.127   | 0.092 | 0.133 | 0.133    | 0.088    | 0.433     | 0.050 | 0.075  |
| **Few-shot (k=3)**       | 0.6B  | 0.0926  | 0.076 | 0.121 | 0.117    | 0.050    | 0.133     | 0.000 | 0.075  |
|                          | 1.7B  | 0.1423  | 0.098 | 0.146 | 0.175    | 0.138    | 0.400     | 0.017 | 0.100  |
| **Chain-of-Thought**     | 0.6B  | 0.041   | 0.040 | 0.038 | 0.038    | 0.050    | 0.083     | 0.000 | 0.075  |
|                          | 1.7B  | 0.0439  | 0.021 | 0.067 | 0.025    | 0.125    | 0.083     | 0.000 | 0.050  |
| **LoRA (finetuned few-shot)** | 0.6B  | 0.2483  | 0.086 | 0.175 | 0.558    | 0.100    | 0.533     | 0.200 | 0.100  |
|                          | 1.7B  | 0.2073  | 0.101 | 0.138 | 0.408    | 0.150    | 0.467     | 0.167 | 0.075  |


Results Analysis: 

Consistent patterns: Small open-source LLMs struggle with structured medical calculation tasks, and the method of prompting or adapting them makes a big difference.

Starting with zero_shot, the 1.7B model achieves an overall accuracy of 12.7%, while the 0.6B model reaches only 3.3%. This gap makes sense, as with more parameters, the 1.7 has some effect of memorization to fall back on. With no guided examples and clearly not even close to enough training knowledge, the 0.6 is completely useless with 3.3% accuracy.

Few-shot prompting brings some improvements. For the 1.7B model, accuracy increases slightly from 12.7% to 14.2%. By contrast, the 0.6B model benefits more from few-shot prompting, improving from 3.3% to 9.3% overall. This makes sense as the smaller model is so weak in zero shot that simply seeing a few examples gives it valuable information on how to structure answers.

Chain of Thought reasoning had some unexpected results. For the 1.7B model, it actively hurts performance: accuracy collapses to 4.4%, worse than both zero-shot and few-shot. For the 0.6B model, CoT produces a tiny increase (3.3% → 4.1%), but the effect is marginal and essentially noise. The most plausible explanation is that reasoning chains introduce more opportunities for small models to make arithmetic mistakes and deal with other complexities that they are just not suited to handle. These results seem to show that CoT benefits appear only in models with billions more parameters while small LLMs actually regress when asked to reason step by step.

LoRA fine tuning gave by far the best improvements. For the 0.6B model, accuracy jumps from 9.3% in few-shot prompting to 24.8% after LoRA, which is massive. This is driven by large jumps in specific categories: physical tasks climb from 11.7% to 55.8%, diagnosis tasks from 13.3% to 53.3%, and dates from 0% to 20%. These are precisely the categories that require either formulaic application or disciplined formatting, which finetuning can teach directly. Strangely, the 1.7B model does improve with LoRA, however it performed worse than the 0.6B with accuracy rising from 14.2% to 20.7%. The smaller improvement at 1.7B likely reflects the short training of only 1 epoch. The model is large enough that a single epoch LoRA may not suffice to meaningfully adjust its parameters.

Overall, it is clear that smaller LLMs need more than clever prompting to perform well on the medcalc dataset. Without fine tuning, they are almost completely unreliable, with methods like chain of thought being clearly counterproductive at this scale, confirming that smaller LLMs do not have the capacity to maintain coherent reasoning chains.

