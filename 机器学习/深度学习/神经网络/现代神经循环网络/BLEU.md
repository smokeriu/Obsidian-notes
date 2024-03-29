BLEU用于预估序列预测的准确性，其值越高说明越准确，当两个句子完全一致是，其值为1。
公式如下：
$$
\exp\left(\min\left(0, 1 - \frac{\mathrm{len}_{\text{label}}}{\mathrm{len}_{\text{pred}}}\right)\right) \prod_{n=1}^k p_n^{1/2^n}
$$
其中：
- `label`表示真实数据。也称为标签序列。
- `pred`表示模型的预测数据。

BLEU评估的是，对于预测序列中的任意n元语法，这个n元语法是否出现在标签序列中。另外，用$p_n$表示$n$元语法的精确度，它是两个数量的比值：
- 第一个是预测序列与标签序列中匹配的$n$元语法的数量。
- 第二个是预测序列中$n$元语法的数量。
具体地说，给定标签序列$A$、$B$、$C$、$D$、$E$、$F$和预测序列$A$、$B$、$B$、$C$、$D$，我们有：
- $p_1 = 4/5$——4表示两个序列的1元语法交集（ABCD），5表示预测序列的1元语法数量（ABBCD）。
- $p_2 = 3/4$——3表示2元语法交集（AB、BC、CD），4表示预测序列中的2元语法数量（AB、BB、BC、CD）。
- $p_3 = 1/3$——1表示3元语法交集（BCD），3表示预测序列中的3元语法数量（ABB，BBC，BCD）。
- $p_4 = 0$则同理。

BLEU适合用于衡量结果的准确性的原因在于，其将句子的长度纳入了评价标准中，正常而言，句子越长，预测难度越大。当预测的句子过短时，其的`p_n`很容易拿到非常高的分数，但$\exp\left(\min\left(0, 1 - \frac{\mathrm{len}_{\text{label}}}{\mathrm{len}_{\text{pred}}}\right)\right)$作为惩罚会降低最终的结果得分。

BLEU的代码如下：
```python
def bleu(pred_seq, label_seq, k):  #@save
    """计算BLEU"""
    pred_tokens, label_tokens = pred_seq.split(' '), label_seq.split(' ')
    len_pred, len_label = len(pred_tokens), len(label_tokens)
    score = math.exp(min(0, 1 - len_label / len_pred))
    for n in range(1, k + 1):
        num_matches, label_subs = 0, collections.defaultdict(int)
        for i in range(len_label - n + 1):
            label_subs[' '.join(label_tokens[i: i + n])] += 1
        for i in range(len_pred - n + 1):
            if label_subs[' '.join(pred_tokens[i: i + n])] > 0:
                num_matches += 1
                label_subs[' '.join(pred_tokens[i: i + n])] -= 1
        score *= math.pow(num_matches / (len_pred - n + 1), math.pow(0.5, n))
    return score
```
