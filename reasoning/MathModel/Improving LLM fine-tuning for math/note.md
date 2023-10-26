# IMPROVING LARGE LANGUAGE MODEL FINE-TUNING FOR SOLVING MATH PROBLEMS

## Movivation
- LLM在解决数学能力方面pass@1和pass@k之间的准确率差别很大(对于PALM在MATH数据集上，pass@1=33.4%，pass@64=79.4%)
- LLM有能力找到正确答案，但是区分正确答案和错误答案的能力比较弱。

## Methods
构建了task-specific的微调方法，主要探索了三种方法
#### SSFT
Supervised Step-by-step solution finetuning (SSFT)
baseline方法: 对于数学问题P，有step-by-step的解答S和最终答案A. 将S和A拼接为X，然后使用交叉熵损失(最大似然估计MLE)+下个token预测:
$$L_{mle}=-log P_M(X|P)=-log_{P_M} \Pi_i p_M(x_i|X_{0,...,i-1}, P)$$
本文选取了两类step-by-step的解答，一类来自于MATH数据集原始的解答，比较抽象，而另一类是GPT的解答，详细而具体。
#### SCR
Solution-cluster Re-ranking(SCR): 
使用SSFT时，pass@1 和 maj1@N 的准确率差别很大(maj1@1指N次解答,选频率最高的)
![Alt text](src\image.png){:width=100 height=130}
所以，将LLM微调为一个==验证者solution verfier==，从对N个答案进行重排，选个最有可能正确的答案. 本文有稍微的改进，即先对答案进行聚类/频率统计，用频率最高的K个答案进行重排，然后选择最有可能的答案。

verifier的训练方法: 
    将问题P和答案X对放进一个模板里:
    > Here is a math problem: P. Here is a candidate solution: X. The above candidate solution is
    这个答案正确的概率 $S_{cls}(X|P)=\frac{p_M("correct"|T(P,X))}{p_M("incorrect"|T(P,X)) + p_M("correct"|T(P,X))}$
    使用两种损失函数训练
    1. $L_{cls-margin}=\operatorname{max}(0, \operatorname{log}S(X_{incorrect}|P) - \operatorname{log}S(X_{correct}|P) + \lambda)$
    2. $L_{cls-xent}= -1_{\{X \space is \space correct\}} \operatorname{log}_{p_M}("correct"|X, P) + 1_{\{X \space  is \space correct\}} \operatorname{log} p_{cls}("incorrect"|X,P)$

>！！注意：这个方法是分为两个阶段的，首先生成模型提出N个答案，然后verifier重排，选出最好的。所以用于生成答案的LLM还是用SSFT方法训练，以上训练都是针对Verifier做的

#### Multi-Task Sequential Fine-tuning
最大似然估计(可能指的是下个token预测？)与最终的二元评估(答案正确与否)之间存在矛盾. 有方法使用对比学习, 将模型预测一个答案的概率作为其质量分数，然后提高正确解法的分数/生成概率，降低错误解法的分数/生成概率，即
$$L_{seq}=max(0, \operatorname{log} P_M(X_{incorrect} |P) - \operatorname{log}p_M(X_{correct} | P) + \lambda)$$

最终loss为对比学习loss+下个token预测loss. ==相当于多了对比学习loss，相比于下个token预测loss，更加关心最终答案的正确性.==
$$L_{ctr} = L_{seq} + \alpha_1 L_{mle}$$

这里的对比学习loss，提高的正确答案$X_{correct}$整个序列的生成的概率，降低错误答案$X_{incorrect}$，整个序列生成的概率. 但是数学题比较特殊，最终答案正确与否只和某几个token有关，最终错误的也不一定前面的token都是有害的，所以用于训练verifier的loss要更加合理一些。

最终的训练方法: 先将LLM作为生成模型SFT，然后作为verifier，但后再作为生成模型。
> 中间这个阶段可能就是锻炼LLM评估答案正确与否的能力，减少pass@1和pass@k之间的差距

## Experiment
- SSFT
![Alt text](src\image.png){:width=100 height=130}
    - 用GPT4的解答而微调，效果更好

- SCR
![Alt text](src\image-2.png){:width=100 height=140}
    - pass@1是baseline
    - RR.ALL是使用verifier的结果
    - RR.top-8是先统计频率，选最高的8个，然后用verifier重排的结果

- Multi-task sequential fine-tuning
![Alt text](src\image-3.png){:width=100 height=140}
    - 先训练为生成模型，然后训练为评估模型，最后再训练为生成模型
    - 200step收敛到最佳性能
    - 优于单纯的下个token预测

## Analysis
- few-shot不怎么受到样例质量的影响
- PaLM2-S作为verifier不太行-> 评估也需要一个比较大的模型