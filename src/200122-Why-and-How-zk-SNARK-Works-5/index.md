---
title: "从零开始学习 zk-SNARK（五）——Pinocchio 协议"
date: 2020-01-22T08:00:00+08:00
summary: "作为本系列的最后一篇文章，本文继续对 zk-SNARK 协议进行完善，最终形成一个完整的 zk-SNARK 协议。"
featuredImage: "img/5/vlad-hilitanu-m8-b0Pbkru4-unsplash.jpg"
slug: "learn zk-SNARK from zero part five"
markup: pandoc
author: Maksym Petkus，翻译 & 注解：even@安比实验室（even@secbit.io）
tags: ["Zero Knowledge Proof", "tutorials", "zkSNARKs"]
categories: ["Zero Knowledge Proof", "zkSNARKs"]
---


> even@安比实验室: 作为本系列的最后一篇文章，本文继续对 zk-SNARK 协议进行完善，最终形成一个完整的 zk-SNARK 协议。



作者：Maksym Petkus

翻译 & 注解：even@安比实验室（even@secbit.io）

校对：valuka@安比实验室

本系列文章已获作者中文翻译授权。

## 约束和公共输入

### 约束

我们的分析主要集中在运算的概念上。但是，协议实际上不是去做”计算“，而是检验输出值是否是操作数正确运算得到的结果。所以我们称之为约束，即一个 verifier 约束 prover 去为预定义的“程序”提供有效值，而无论这个“程序”是什么。多个约束组成的系统被称为“约束系统”（在我们的例子中这是一个一阶约束系统，或被称为**R1CS**）。

> @Maksym（作者）：这里其实隐含了寻找所有正确答案的一个方法就是对所有可能的组合值进行一次暴力破解，然后只选择一个满足的约束，或者使用可满足约束的更精密的技术[[con18](#b979)]。



> even@安比实验室：请注意这个约束是定义在算术电路，或者布尔电路上。因为这两类电路的可满足性问题是 NP-Complete 问题。

因而我们也可以使用约束来确保其它的关系。例如，如果我们想要确认变量 *a* 的值只能为 0 或 1（即二进制数），我们可以用一个简单的约束去做这件事：

$\textcolor{green}{a} \times \textcolor{blue}{a} = \textcolor{red}{a}$

我们也可以约束 *a* 的值只能为 2：

$\textcolor{green}{(a-2)} \times \textcolor{blue}{1} = \textcolor{red}{0}$

一个更复杂的例子是确保数字 *a* 是一个 4-bit 的数字（也称为半字节），换句话说可以用 4个bit 来表示出 *a*。我们也可以称这个为“确保取值范围” 因为一个 4-bit 的数字可以代表  2<sup>4</sup> 的组合，因而也就是从 0 到15 范围内的 16 个数字。在十进制数字系统中任何数字都可以表示为以 10为底 （也就是我们手指头的数量）并根据对应的指数求幂后的值的和，例如： 123 = 1 ⋅ 10<sup>2</sup> + 2 ⋅ 10<sup>1</sup> + 3 ⋅ 10<sup>0</sup>。同样的一个二进制数可以表示为以 2 为底在对应系数上的求幂值的和，例如，1011 (二进制) = 1 ⋅ 2<sup>3</sup> + 0 ⋅ 2<sup>2</sup> + 1 ⋅ 2<sup>1</sup> + 1 ⋅ 2<sup>0</sup> = 11 (十进制)。

因而如果 *a* 是一个 4-bit 的数，那么 *a= b<sub>3</sub>⋅2<sup>3</sup> + b<sub>2</sub> ⋅ 2<sup>2</sup> + b<sub>1</sub>⋅ 2<sup>1</sup> + b<sub>0</sub> ⋅ 2<sup>0</sup> ，其中 b<sub>3</sub>, b<sub>2</sub>, b<sub>1</sub>, b<sub>0</sub> 为布尔值。约束就可以表示为下面的形式：

$1: \qquad \textcolor{green}{a} \times \textcolor{blue}{1} = \textcolor{red}{8 \cdot b_3 +4 \cdot b_2 + 2 \cdot b_1 + 1 \cdot b_0} $

并且为了确保 b<sub>3</sub>, b<sub>2</sub>, b<sub>1</sub>, b<sub>0</sub>  都是二进制数我们需要增加约束：

$2: \qquad \textcolor{green}{b_0} \times \textcolor{blue}{b_0} = \textcolor{red}{b_0}$

$3: \qquad \textcolor{green}{b_1} \times \textcolor{blue}{b_1} = \textcolor{red}{b_1}$

$4: \qquad \textcolor{green}{b_2} \times \textcolor{blue}{b_2} = \textcolor{red}{b_2}$

$5: \qquad \textcolor{green}{b_3} \times \textcolor{blue}{b_3} = \textcolor{red}{b_3}$

更复杂的约束也可以用这种方式表示，以此来确保使用的值满足规则。注意很重要的一点是在当前的计算结构中是不能对 1 进行约束的：

$\textcolor{green}{\Sigma_{i=1}^n{c_{l,i} \cdot v_i }} \times \textcolor{blue}{\Sigma_{i=1}^n{c_{r,i} \cdot v_i}} = \textcolor{red}{\Sigma_{i=1}^n{c_{o,i} \cdot v_i }}$

因为值 1 （以及前面约束中的 2）必须通过 *c·v<sub>one </sub>* 表达出来，其中 *c* 可以被固定到 *proving key* 中，但是因为 *v* 是由 prover 提供的，所以可以是任何别的值。尽管我们可以通过设置 *c=0* 来强制 *c·v<sub>one </sub>* 变成 0，但是在我们前面受限的构造方法中很难找到一个约束来强制 v<sub>one </sub> 为 1。于是，Verifier 需要有一种办法来设置 v<sub>one </sub> 的值。

> even@安比实验室: 我们前文中提到的表达式的约束关系就称为 R1CS。

### 公共输入和 1

如果不能根据 verifier 的输入对其进行检查，那么证明的可用性将受到限制。也就是说，虽然知道 prover 将两个值相乘但是并不知道这些值和/或计算结果。尽管有可能在 *proving key* 中通过“硬编码”来进行验证一些特定的值（如，乘法运算的结果必须为12），但这就需要针对每一个想要的“verifier 的输入”生成单独的密钥对。

> even@安比实验室: 这样会严重限制实用性，电路需要支持参数。

**因而如果可以由 verifier 而不是 prover 为计算指定一些值（输入或者输出），包括 $v_{one}$，那证明就可以变得更通用了。**

首先，我们看一下要证明的值 g<sup>L(s)</sup>，g<sup>R(s)</sup>，g<sup>O(s)</sup>。因为这里使用了同态加密所以可以增大这些值，例如，我们可以与另一个加密的多项式相加使得值为*g<sup>L(s)</sup> · g<sup>l<sub>v</sub>(s)</sup> =g<sup>L(s)+l<sub>v</sub>(s)</sup>* 。

这样如果我们能够在提供给 prover 的变量多项式中排除必要的一项，verifier 就可以在这一项变量多项式上设置他自己的值，并且使得检查依然能够通过。

因为 verifier 已经可以通过加入α-转换来限制 prover 选择多项式了，所以这个检查结果就很容易得到。因而当消除了它的 α<sub>s </sub>和 *β* 校验和对应的项，这些可变多项式就可以从 *proving key* 转移到 *verification key* 当中去了。

必要的协议更新为：

* Setup

  * …将 n 个可变多项式全部分为两组：

    * verifier 的 *m+1* 项：$L_v(x) = l_0(x) + l_1(x) +…+l_m(x)$,对 $R_v(x)$ 和 $O_v(x)$ 也做同样的计算。这里对于索引0 保留值 $v_{one} =1$

    * prover 的 *n-m* 项：

      $L_p(x) = l_{m+1}(x) +…+l_n(x)$，对 $R_p(x)$ 和 $O_p(x)$ 也做同样的计算。

  * 设置 *proving key*：$(\{g^{s^k}\}_{k \in [d]},\{g_l^{l_i(s)},g_r^{r_i(s)},g_o^{o_i(s)},g_l^{α_ll_i(s)},g_r^{α_rr_i(s)},g_o^{α_oo_i(s)},g_l^{βl_i(s)},g_r^{βr_i(s)},g_o^{βo_i(s)}\}_{i \in \{m+1,…,n\}})$

  * 添加到 *verification key*：$(……，\{g_l^{l_i(s)}, g_r^{r_i(s)}, g_o^{o_i(s)}\}_{i \in \{0,…,m\}})$

* Proving

  * …为 verifier 的多项式计算 *h(x)*：$h(x) = \frac{L(x) \cdot R(x) -O(x)}{t(x)}$，其中 $L(x) = L_v(x) + L_p(x)$，$R(x)$ ，$O(x)$ 也通过同样的计算获得。

  * 提供证明：

    $(g_l^{L_p(s)},g_r^{R_p(s)},g_o^{O_p(s)},g_l^{α_lL_p(s)},g_r^{α_rR_p(s)},g_o^{α_oO_p(s)},g^{Z(s)},g^{h(s)})$

* Verification

  * 为 verifier 的变量多项式赋值，并加 1：

    $g_l^{L_v(s)} = g_l^{l_0(s)} \cdot \prod_{i=1}^m{(g_l^{l_i(s)})^{v_i}}$，对 $g_r^{R_v(s)}$ 和 $g_o^{O_v(s)}$ 做同样的计算

  * 变量多项式约束检查：

    $e(g_l^{L_p},g^{α_l}) = e(g_l^{L'_p},g)$，对 $g_r^{R_p}$ 和 $g_o^{O_p}$ 做同样的计算

  * 变量值一致性检查：

    $e(g_l^{L_p}g_r^{R_p}g_o^{O_p},h^{βγ}) = e(g^Z,g^γ)$

  * 有效计算检查：

    $e(g_l^{L_v(s)}g_l^{L_p},g_r^{R_v(s)}g_r^{R_p}) =e(g_o^t,g^h)e(g_o^{O_v(s)}g_o^{O_p},g)$

> 注意：根据协议（**单个变量操作数多项式** 的章节）的性质，由多项式 *l<sub>0</sub>(x),r<sub>0</sub>(x),o<sub>0</sub>(x)* 表示的值 1 已在相应的运算中具备了合适的值，因此不需要再赋值了。
>
> 注意：verifier 将不得不在验证步骤中做额外的工作，使得赋值的变量成比例。

**这实际上是把一些变量从 prover 手中拿到 verifier 的手中，并同时保持等式相等。** 因而只有当 prover 和 verifier 的输入中使用相同值的时候， *有效计算*检查才依然成立。

1 这个值相当重要，它能够通过与任意一个常数项相乘来生成这个值（从选择的有限域上），例如，用 123 去乘以 *a*：

$\textcolor{green}{1 \cdot a} \times \textcolor{blue}{123 \cdot v_{one}} = \textcolor{red}{1 \cdot r}$

> even@安比实验室: 这里将原本由 prover 赋值的一些变量改为由拿到 verifier 赋值，使得prover 不得不与 verifier 保持相同的输入。这不仅解决了 Verifier 参数输入的问题，也间接解决了常数赋值的问题。  



## 零知识计算

### 计算的零知识证明

自从引入通用计算协议（**计算的证明**这一章节），我们一直放弃了*零知识* 的性质，这是为了让协议的改进变得更简单。至此，我们已经构建了可验证的计算协议。

以前我们使用随机数*δ*-转换来构造多项式的“零知识” 证明，这种方法能够使得证明与随机数无法区分（**零知识**这一章节）：

$$\delta p(s ) = t(s) \cdot \delta h(s) $$

通过计算我们证明了：

$$L(s) \cdot R(s) - O(s)= p(s)h(s)$$

尽管我们可以通过用相同 *δ* 乘以多项式的方法来调整解决方案，即提供随机值 *δL*(*s*)*， δR*(*s*)*， δ*²*O*(*s*)*， δ*²*h*(*s*)，这依然能够通过*有效计算检查* 来满足配对验证：

$e(g,g)^{\delta^2L(s)R(s)}=e(g,g)^{\delta^2(t(s)h(s)+O(s))}$

但是问题是使用相同的 *δ* 会妨碍安全性，因为我们在证明中分别用了以下这些值：

* 其他人可以很容易得辨认出两个不同的多项式值是否相同，以此来获取一些知识，即：g<sup>δL(s)</sup>=g<sup>δR(s)</sup>

* *L*(*s*) 和 *R*(*s*) 的不同值之间潜在的微小关系可能会通过暴力破解来区分开来，例如如果 *L*(*s*) = 5*R*(*s*)，就可以对 *i ∈ {1…N}* 取值反复校验 g<sup>L(s)</sup>=(g<sup>R(s)</sup>)<sup>i</sup> ，只需要执行 5 步就可以揭示出两者 5 倍区别的关系。同样的暴力破解也可以用在破解加密值的加法运算上，如：*g<sup>L(s)</sup> = g<sup>R(s)+5</sup>*
* 证明元素之间的其它关系也可能会被发现，例如，如果 e(g<sup>δL(s)</sup>,g<sup>δR(s)</sup>) = e(g<sup>δ<sup>2</sup>O(s)</sup>,g)，那么也就表示 *L*(*x*) ⋅ *R*(*x*) = *O*(*x*)。

> 注意：**一致性检查优化** 使得挖掘数据关系变得更加困难了，但是依然能够发现一些关系       ，且不说 verifier 可以选择特定  ρ<sub>l</sub>， ρ<sub>r</sub>  来为揭示知识提供便利（只要这不是一个多样化的 setup）。

最终，我们需要对每一个多项式的值使用不同的随机数 (*δ<sub>s</sub>*) ，例如：

*δ<sub>l</sub>L(s) ·δ<sub>r</sub>R(s) - δ<sub>o</sub>O(s) =t(s)·(△?⃝ h(s))*

为了解决等式右边不相等的问题，我们不必改变协议，只要修改证明的值 *h(s)*即可。 这里 Delta (Δ) 代表为了平衡方程另一侧的随机性而对 *h(s)* 做的处理，?⃝代表乘法运算或者加法运算（这个反过来也适应了除法和减法）。如果我们选择用乘法 (?⃝ = ×) 来计算 Δ ，也就意味着不太可能有较大的概率可以找到一个 Δ ，因为存在随机性：

$\Delta = \frac{\delta_lL(s) \cdot \delta_rR(s) - \delta_oO(s)}{t(s)h(s)}$

设置 $\delta_o = \delta_l \cdot \delta_r$，于是就变成了：

$\Delta = \frac{\delta_l \delta_r(L(s) \cdot R(s) - O(s))}{t(s)h(s)} = \delta_l \delta_r$

但是如前文所述，这个妨碍了零知识的性质，更重要的是这个结构也不再适合 verifier 的输入多项式，因为它们必须是 *δ<sub>s</sub>*相应的倍数，这就需要额外的交互了。

我们可以尝试把随机数加到变量上：*(L(s)+δ<sub>l</sub>) ·(R(s)+δ<sub>r</sub>) - (O(s)+δ<sub>o</sub>) = t(s) ·(△ × h(s))*

$$\Delta = \frac{\overbrace{L(s)R(s)-O(s)}^{t(s)h(x)} + \delta_rL(s) + \delta_lR(s) + \delta_l \delta_r - \delta_o}{t(s)h(s)} = 1+ \frac{\delta_rL(s) + \delta_lR(s) + \delta_l \delta_r -\delta_o}{t(s)h(s)}$$ 

但是随机数是不可除尽的。尽管我们可以用 *t*(*s*)*h*(*s*) 去乘以每一个 *δ* ，但由于我们已经用了 $\Delta$  乘以 *h*(*s*) ， $\Delta$ 是组成加密结果的一部分（即*E*(*L*(*s*))等），因此在没有使用配对（它的结果在另一个数值空间内）的情况下是不能计算出g<sup>△h(s)</sup> 的。同样也不能使用 *s* 的幂（从 1 到 *d*）的加密值对 Δ*h*(*x*) 的进行加密计算，Δ*h*(*x*) 的阶将达到 2*d*。并且，基于上述同样的原因也无法计算这个随机操作数多项式的值：*g<sup>L(s)+δ<sub>l</sub>t(s)h(s)</sup>*。

于是我们应该用加法(?⃝ = +)来使用 $\Delta$，因为它可以同态地计算。

$(L(s)+ \delta_l) \cdot (R(s) +\delta_r) - (O(s) + \delta_o) = t(s) \cdot (\Delta + h(s))$

$\Delta = \frac{L(s)R(s)-O(s)+\delta_rL(s) + \delta_lR(s) + \delta_l \delta_r - \delta_o - t(s)h(s)}{t(s)} =>$

$\Delta = \frac{\delta_rL(s)+ \delta_lR(s) + \delta_l \delta_r -\delta_o}{t(s)}$

分子中的每一项都是 *δ* 的倍数，因而我们可以将其与 *t*(*s*) 相乘使它可以被分母整除：

$(L(s)+ \delta_l t(s)) \cdot (R(s) + \delta_r t(s)) -(O(s) + \delta_o t(s)) = t(s) \cdot (\Delta + h(s))$

$\cancel{L(s)R(s)-O(s)} +t(s)(\delta_rL(s) + \delta_lR(s) + \delta_l \delta_rt(s) - \delta_o) = t(s)\Delta + \cancel{t(s)h(s)}$

$\Delta = \delta_rL(s) + \delta_lR(s) + \delta_l \delta_r t(s) -\delta_o$

这样就可以在加密的空间中进行“有效计算检查”了：

$g^{L(s)+ \delta_lt(s)} = g^{L(s)} \cdot (g^{t(s)})^{\delta_l}$，等。

$g^{\Delta} = (g^{L(s)})^{\delta_r} \cdot (g^{R(s)})^{\delta_l} \cdot (g^{t(s)})^{\delta_l \delta_r} g^{-\delta_o}$

于是既隐藏了加密值，又使得等式可以通过*有效计算* 的检查。

$L \cdot R -O + \textcolor{magenta}{t(\delta_rL + \delta_lR + \delta_l \delta_r t - \delta_o)} = t(s)h+ \textcolor{magenta}{t(s)(\delta_rL+\delta_lR+\delta_l \delta_rt-\delta_o)}$

这个结构就是统计学上的零知识 因为增加了 *δ<sub>l</sub>, δ*<sub>r</sub>*, δ<sub>o</sub>* 的均匀随机倍数（参见[Gen+12] 中的定理 13）。

> 注意：这种方法和 verfier 的操作数也是一致的，即：g<sub>l</sub><sup>L<sub>p</sub>+δ<sub>l</sub>t </sup>·g<sub>l</sub><sup>L<sub>v</sub></sup> = g<sup>L<sub>p</sub>+L<sub>v</sub>+δ<sub>l</sub>t</sup>。
>
> 因而当且仅当 prover 使用了 verifier 的值来构造证明(即， Δ = δ<sub>r</sub>(L<sub>p</sub> + L<sub>v</sub>) + δ<sub>l</sub>(R<sub>p</sub>+ R<sub>v</sub> ) + δ<sub>l</sub>δ<sub>r</sub>t – δ<sub>o</sub>)，这个有效计算的检查依然是成立的，更多的细节看下一部分。

为了使得 “变量多项式限制” 和 “变量值一致性”检查与 *零知识* 的修改一致，就有必要去增加以下的参数到 *proving key* 中：

$g_l^{t(s)}, g_r^{t(s)},g_o^{t(s)}, g_l^{α_lt(s)}, g_r^{α_rt(s)},g_o^{α_ot(s)}, g_l^{βt(s)}, g_r^{βt(s)},g_o^{βt(s)}$

非常奇怪的是最初的 Pinocchio 协议[Par+13]主要关注可验证的计算，而较少涉及 *零知识* 性质，这其实只需要一点点小修改，这个几乎是没有什么成本的。

> even@安比实验室: 与前文中的零知方案不同，这里通过相加而不是相乘的方式来确保 prover 知识的零知性。
>
> Pinocchio 协议是针对 GGPR 论文的改进，在3.1节中也提到了实现零知识只需要沿用 GGPR 论文的方法即可，并不是这篇论文的贡献。另外，Pinocchio 协议论文侧重工程实践，在2013年时，零知识证明还并没有得到应用。真正的应用还是自从匿名币ZCash起。



## zk-SNARK 协议

在这一步步的改进之后，我们得到了最终版本的zkSNARK，又名 Pinocchio [Par + 13]，协议  （*零知识* 部分是可选的并用*不同的颜色* 标注出来了），就是：

* Setup

  * 选择生成元 *g* 和加密配对 *e*

  * 将变量总数为 *n* 其中输入/输出变量数位 *m* 的函数 *f(u) = y*，转换为阶数为 *d* 大小为 *n+1* 的多项式形式（QAP）（$\{l_i(x),r_i(x),o_i(x)\}_{i \in \{0,……,n\}},t(x)$）

  * 选择随机数 $s, \rho_l, \rho_r, \alpha_l, \alpha_r, \alpha_o, \beta, \gamma$

  * 设置 $\rho_o = \rho_l \cdot \rho_r$ 和操作数生成元 $g_l=g^{\rho_l}, g_r = g^{\rho_r}, g_o = g^{\rho_o}$

  * 设置 *proving key*：

    $(\{g^{s^k}\}_{k \in [d]}, \{g_l^{l_i(s)},g_r^{r_i(s)},g_o^{o_i(s)}\}_{i \in {0,…,n}}), \{g_l^{\alpha_ll_i(s)},g_r^{\alpha_rr_i(s)},$

    $g_o^{\alpha_oo_i(s)},g_l^{\beta l_i(s)},g_r^{\beta r_i(s)},g_o^{\beta o_i(s)}\}_{i \in \{m+1,…,n\}},$

    $\textcolor{magenta}{g_l^{t(s)}, g_r^{t(s)}, g_o^{t(s)}, g_l^{\alpha_l t(s)}, g_r^{\alpha_r t(s)}, g_o^{\alpha_o t(s)}, g_l^{\beta t(s)},g_r^{\beta t(s)}, g_o^{\beta t(s)}} )$

  * 设置 *verfication key*：

    $(g^1,g_o^{t(s)},\{g_l^{l_i(s)},g_r^{r_i(s)},g_o^{o_i(s)}\}_{i \in \{0,…,m\}},g^{α_l},g^{\alpha_r},g^{\alpha_o},g^{\gamma},g^{\beta \gamma})$

* Proving

  * 代入输入值 *u*，执行 *f(u)* 计算获取所有的中间变量值 $\{v_i\}_{i \in \{m+1,……，n\}}$

  * 把所有未加密的变量多项式赋值给$ L(x) = l_0(x) + \sum_{i=1}^n {v_i \cdot l_i(x)}$，并对 $R(x)$ 和 $O(x)$ 做同样的计算

  * $\textcolor{magenta}{选择随机数 \delta_l ,\delta_r 和 \delta_o}$

  * 计算 $h(x) = \frac{L(x)R(x)-O(x)}{t(x)} \textcolor{magenta}{+ \delta_r L(x) + \delta_lR(x) + \delta_l \delta_r t(x) - \delta_o}$

  * 将 prover 的变量值赋值给加密的可变多项式 $\textcolor{magenta}{并进行零知识的 \delta-转换}\quad  g_l^{L_p(s)}= \textcolor{magenta}{(g_l^{t(s)})^{\delta_l}} \cdot \prod_{i=m+1}^n{(g_l^{l_i(s)})^{v_i}}$，再用同样的方式计算 $g_r^{R_p(s)}$ 和 $g_o^{O_p(s)}$。

  * 为其 *α-*变换赋值 $g_l^{L_p'(s)}= \textcolor{magenta}{(g_l^{\alpha _l t(s)})^{\delta_l}} \cdot \prod_{i=m+1}^n{(g_l^{\alpha _l l_i(s)})^{v_i}}$，再用同样的方式计算 $g_r^{R_p'(s)}$ 和 $g_o^{O_p'(s)}$。

  * 为变量值一致性多项式赋值

    $g^{Z(s) = \textcolor{magenta}{(g_l^{\beta t(s)})^{\delta_l}(g_r^{\beta t(s)})^{\delta_r}(g_o^{\beta t(s)})^{\delta_o}}} \cdot \prod_{i=m+1}^n{(g_l^{\beta l_i(s)}g_r^{\beta r_i(s)}g_o^{\beta o_i(s)})^{v_i}}$

  * 计算证明 $(g_l^{L_p(s)}, g_r^{R_p(s)}, g_o^{O_p(s)}, g^{h(s)}, g_l^{L_p'(s)}, g_r^{R_p'(s)}, g_o^{O_p'(s)}, g^Z)$

* Verification

  * 解析提供的证明为 $(g_l^{L_p},g_r^{R_p},g_o^{O_p},g^h,g_l^{L_p'},g_r^{R_p'},g_o^{O_p'},g^Z)$

  * 将输入/输出值赋值给 verifier 的加密多项式并加 1：

    $g_l^{L_v(s)} = g_l^{l_o(s)} \cdot \prod_{i=1}^m{(g_l^{l_i(s)})^{v_i}}$，并对 $g_r^{R_v(s)}$ 和 $g_o^{O_v(s)}$ 做同样的计算

  * 可变多项式约束检查：

    $e(g_l^{L_p},g^{\alpha_l}) = e(g_l^{L_p'},g)$，并对 $g_r^{R_p}$ 和 $g_o^{O_p}$ 做同样的检查

  * 变量值一致性检查：

    $e(g_l^{L_p},g_r^{R_p},g_o^{O_p},g^{\beta \gamma}) = e(g^Z, g^{\gamma})$

  * 有效的计算检查：

    $e(g_l^{L_p}g_l^{L_v(s)},g_r^{R_p}g_r^{R_v(s)}) = e(g_o^{t(s)},g^h) \cdot e(g_o^{O_p}g_o^{O_v(s)},g)$

    

## 结论

我们最终完成了一个允许证明计算的有效协议：

* 简明 (Succinctly) —— 独立于计算量，证明是恒定的，小尺寸的

* 非交互性 (Non-interactive) —— 证明只要一经计算就可以在不直接与 prover 交互的前提下使任意数量的 verifier 确信

* 可论证的知识 (with Argument of Knowledge) —— 对于陈述是正确的这点有不可忽略的概率，即无法构造假证据；并且 prover 知道正确陈述的对应值（即：证据），例如，如果陈述是 “B 是 sha256(a) 的结果” 那么就说明 prover 知道一些值 a 能够使得 B = sha256(a) 成立，因为 B 只能够通过 a 的知识计算出来，换句话说就是无法通过 B 来反算出 a（假定 a 的熵足够）。

* 陈述有不可忽略的概率是正确的 (even@安比实验室: 这里指Soundness可靠性)，即，构造假证据是不可行的

* *零知识*( zero-knowledge)  —— 很“难”从证明中提取任何知识，即，它与随机数无法区分。

> even@安比实验室: 所谓Argument——论证，区别于 Proof —— 证明。Pinocchio 协议是 Argument 而非 Proof。这是因为 Pinocchio的可靠性是 Computational Soundness，Statistical ZK，这一类的证明系统被称为 Argument。所谓的 Computational Soundness 暗含了这样的事实：如果 Prover 计算能力足够强大的话，可以破坏可靠性。



基于多项式的特殊性质，模运算，同态加密，椭圆曲线密码学，加密配对和发明者的聪明才智才使得这个协议得以实现。

这个协议证明了一个特殊有限执行机制的计算，即在一次运算中可以将几乎任意数量的变量加在一起但是只能执行一次乘法，因而就有机会优化程序以有效地利用这种特性的同时也使用这个结构最大限度地减少计算次数。

为了验证一个证明， **verifier 并不需要知道所有的秘密数据，这一点很关键**，这就使得任何人都可以以非交互式方式发布和使用正确构造的 *verification key*。这一点与只能让一个参与者确信证明的"指定 verifier"方案相反，因而它的信任是不可转移的。在 *zkSNARK* 中，如果不可信或由单方生成密钥对，则可以实现这个属性。

零知识证明构造领域正在不断发展，包括引入了优化（[BCTV13, Gro16, GM17]），改进例如可更新的 *proving key* 和 *verification key*（[Gro+18]），以及新的构造方法（Bulletproofs [Bün+17], ZK-STARK [Ben+18], Sonic [Mal+19]）。



> even@安比实验室：至此,本系列文章的介绍已经全部结束，大家是不是已经对zkSNARK协议（Pinocchio 协议）了如指掌了？其实任何复杂的协议掌握了核心思想，都可以抽丝剥茧将其变得通俗易懂。
>
> 零知识证明的学习还有很长的路要走，本文只是一个入门的资料，正如文章中所述，零知识证明构造领域正在不断发展，不断的有新的研究突破呈现在我们面前。这是一个非常有趣的领域，后续也非常欢迎小伙伴跟我们一起继续学习和研究零知识证明技术。
>
> 再次感谢 Maksym Petkus 大神的分享和授权。文章翻译存在不足之处，欢迎纠正，补充，指导。
>



**原文链接**

https://arxiv.org/pdf/1906.07221.pdf

https://medium.com/@imolfar/why-and-how-zk-snark-works-7-constraints-and-public-inputs-e95f6596dd1c

https://medium.com/@imolfar/why-and-how-zk-snark-works-8-zero-knowledge-computation-f120339c2c55

**参考文献**

[con18] — Wikipedia contributors. *Constraint satisfaction*. Wikipedia, The Free Encyclopedia. 2018.

[Gen+12] — Rosario Gennaro, Craig Gentry, Bryan Parno, and Mariana Raykova. *Quadratic* *Span Programs and Succinct NIZKs without PCPs*. Cryptology ePrint Archive, Report 2012/215. https://eprint.iacr.org/2012/215. 2012.

[Par+13] — Bryan Parno, Craig Gentry, Jon Howell, and Mariana Raykova. *Pinocchio: Nearly* *Practical Verifiable Computation*. Cryptology ePrint Archive, Report 2013/279. https://eprint.iacr.org/2013/279. 2013.

[BCTV13] — Eli Ben-Sasson, Alessandro Chiesa, Eran Tromer, Madars Virza. *Succinct Non-Interactive Zero Knowledge for a von Neumann Architecture*. Cryptology ePrint Archive, Report 2013/879. https://eprint.iacr.org/2013/879. 2013.

[Gro10] — Jens Groth. “Short pairing-based non-interactive zero-knowledge arguments”. In: *International Conference on the Theory and Application of Cryptology and* *Information Security*. Springer. 2010, pp. 321–340.

[GM17] — Jens Groth, Mary Maller. *Snarky Signatures: Minimal Signatures of Knowledge from Simulation-Extractable SNARKs*. Cryptology ePrint Archive, Report 2017/540. https://eprint.iacr.org/2017/540. 2017.

[Gro+18] — Jens Groth, Markulf Kohlweiss, Mary Maller, Sarah Meiklejohn, and Ian Miers. *Updatable and* *Universal Common Reference Strings with Applications to zk-SNARKs*. Cryptology ePrint Archive, Report 2018/280. https://eprint.iacr.org/2018/280. 2018.

[Bün+17] — Benedikt Bünz, Jonathan Bootle, Dan Boneh, Andrew Poelstra, Pieter Wuille, and Greg Maxwell. *Bulletproofs: Short* *Proofs for Confidential Transactions and More*. Cryptology ePrint Archive, Report 2017/1066. https://eprint.iacr.org/2017/1066. 2017.

[Ben+18] — Eli Ben-Sasson, Iddo Bentov, Yinon Horesh, and Michael Riabzev. *Scalable,* *transparent, and post-quantum secure computational integrity*. Cryptology ePrint Archive, Report 2018/046. https://eprint.iacr.org/2018/046. 2018.

[Mal+19] — Mary Maller, Sean Bowe, Markulf Kohlweiss, and Sarah Meiklejohn. *Sonic: Zero-Knowledge SNARKs from Linear-Size Universal* *and Updateable Structured Reference Strings*. Cryptology ePrint Archive, Report 2019/099. https://eprint.iacr.org/2019/099. 2019.