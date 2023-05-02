# d3bdd

## 背景

LWE问题的主要攻击方式分为两种：primal attack与dual attack，尽管在理论上，这两种攻击的复杂度基本相近，但在实际中，primal attack的实现相当简单，且实际复杂度要低于dual attack，所以在CTF中基本大家都会使用primal attack也就是最简单地将LWE转化为uSVP问题直接构造一个格，然后使用LLL或是BKZ解决。但dual attack在最近引入了FFT区分器^[1,2]^以后理论复杂度已略低于primal attack，（不过他们的工作目前还存在一定的问题^[3]^）因此我希望通过题的方式向大家介绍一下这种攻击。

dual attack与primal attack有一个区别在于dual attack是将LWE问题转化为aSVP问题，而不是uSVP问题。而如果一个具有特殊性质的格，它的left kernel有着容易找到且特别短的向量，那么dual attack的复杂度就会远小于primal attack，这是本题主要的出题思路。

题目主要分为两部分，伪随机数生成器与一个理想格上的BDD问题，出题人的预期是希望通过使用不恰当的伪随机数生成器生成的多项式具有易得的不合理短的向量，然后通过dual attack去解决BDD问题。

## 非预期

在比赛中所有做出该题的队伍均没有采用预期的方式解题。

这是由于多项式环的模多项式选取时，使用了$x^{512} - 1$，这个多项式有非常多小因式，其中，特别是$x^{128}+1$与$x^{128}-1$，在这两个因式上的RLWE问题只需要对256维的格进行规约，且由于噪声很小，是能够解得结果的。由此可以通过CRT得到$s \% (x^{256}-1)$的值。虽然$s\%(x^{256}+1)$的值无法求得（格的维数达到了512），但由于$s$的值是flag，是可见且有意义的字符串，因此选手能够通过猜单词以及题目中给出的sha256(flag)来判断flag的正确性。

非预期出现的原因主要是因为模多项式选取问题，应当选取$x^{512}+1$作为模多项式，而预期解对于这两种情况都能够解决。

## 预期解



### dual attack

首先来介绍一下dual attack的流程

dual attack能够判断一对(A,b)是否符合LWE的形式，假如它符合，则
$$
As + e = b \mod{q}
$$
我们可以寻找短向量$u,v$满足
$$
uA = v \mod{q}
$$
将(1)式两边同乘以$u$得
$$
vs + ue = ub\mod{q}
$$
由于s与e都很小，vs与ue也都很小，因此ub也很小。

但假如这对(A,b)不是LWE的实例，则ub应当为均匀分布，期望在q/2附近，通过这个我们能够解决判定-LWE(decision-LWE)问题。

如何使用这样的攻击求得s呢？

我们可以爆破一部分的s，例如我们令s = (s1 | s2)，且爆破s1的取值
$$
As + e = A_1s_1 + A_2s_2+e = b\\
A_2s_2 + e = b- A_1s_1 = b'
$$
当我们猜对s1的时候，$(A_2 , b')$是一个LWE实例，反之则不是，由此我们可以通过爆破的方式求出完整的s。

### 理想格

如何找到这样的u呢？首先我们需要先观察格A的形状。

如果模多项式是$x^{512} +1$是这样的
$$
\left(
    \begin{matrix}
    	a_0 & -a_{n-1} & -a_{n-2} & \cdots & -a_1\\
    	a_1 & a_{0} & -a_{n-1} & \cdots & -a_2 \\
    	\vdots & \vdots& \vdots & \ddots & \vdots\\
    	a_{n-1} & a_{n-2} & a_{n-3} & \cdots & a_0
    \end{matrix}
\right)
$$
如果是$x^{512}-1$是这样的
$$
\left(
    \begin{matrix}
    	a_0 & a_{n-1} & a_{n-2} & \cdots & a_1\\
    	a_1 & a_{0} & a_{n-1} & \cdots & a_2 \\
    	\vdots & \vdots& \vdots & \ddots & \vdots\\
    	a_{n-1} & a_{n-2} & a_{n-3} & \cdots & a_0
    \end{matrix}
\right)
$$
基本差别不大。它有一个很重要的特点是它的每一列是循环的，我们可以找到很多相同的样式。

如上下为$a_{n-1} , a_{n}$的，在前两行中，只有在对角线附近的第二列的$a_{n-1} , a_0$不符合，其他的均符合这样的格式。

也就是说，如果我找到了一个向量$u = (u_1 , u_2)$，对于所有的n，满足
$$
u_1a_{n-1} + u_2a_{n} = 0
$$
则它乘以A的前两行，几乎能让结果的每一维都为0，并且有部分维不是0（这也相当重要，否则是无法枚举成功的）。因此我们的目标是找到满足类似于(7)式这样的u，即可使用dual attack完成题目。

### 伪随机数生成器

我们可以发现，这个伪随机数生成器在模m下是一个线性的生成器，即
$$
a_{n+17}=\sum_{i=0}^{16}a_{n+i}f_i\mod{m}
$$
若令$f_{17} = -1$，则
$$
\sum_{i=0}^{17}a_{n+i}f_i = 0 \mod{m}
$$
因此这个f乘以A的前17维已经能够全为0了，但由于题目中的f是随机选取的，值很大，不满足u很小的条件。

对于这个问题，可以多取几个n，将(9)式进行求和，即
$$
\sum_{j=0}^k(c_j\sum_{i=0}^{17}a_{j+i}f_i) = 0 \mod{m}\\
\sum_{i=0}^{k+17}a_i(\sum_{j=max(0,i-k)}^{min(17,i)}f_jc_{i-j}) = \sum_{i=0}^{k+17}a_if_i' \mod{m}
$$
其中，$c_i$是可以任选的，因此我们可以构造如下格来求解$f'$
$$
\left(
    \begin{matrix}
    	mI\\
  		F
    \end{matrix}
\right)
$$
其中F为
$$
F = \left(
    \begin{matrix}
    	f_0 & f_{1} & \cdots & f_{17}\\
    	    & f_{0} & f_{1} & \cdots & f_{17} \\
    	& & \ddots &  & \\
    	& & & f_0 & f_1&\cdots&f_{17}
    \end{matrix}
\right)
$$
（如果将此处的f看作是线性反馈序列的特征多项式，这里的格的构造会更显然）

在本题条件下，k取到约80左右能够有比较良好的效果。但这也会意味着我们需要爆破80bit的s，这是不可行的。然而题目中的s并不是一个随机值，有长达9个字节，72bit的flag头`antd3ctf{`（巧的是这次的flag头真的很长），那么爆破量就不大了。（这也是我将flag放在s里的原因，由于lwe一般不会把明文放s里，ggh会，所以一时不知道取什么名字，索性取个宽泛的bdd）

### m不是q

在这里，还会遇到一个重要的问题，上面的计算是在模m的前提下成立的，而LWE的模是q，m%q也相当大，如果不解决这个问题，上面得到的向量是没法使用的。

将上面式子中的mod m去掉可以得到
$$
f'A = gm
$$
g是一个向量，且由于A是小于m的，因此g的规模只会略大于f'，最多是f‘的k倍（k是f'的维度），但实际中基本达不到这么大。

同时，在去年d3ctf的crypto赛题leak_dsa中，我们用到了一个trick，即能够通过乘以一个值来平衡一个过大的项。

构造格
$$
\left(
    \begin{matrix}
		q & 0\\
		m & 1
    \end{matrix}
\right)
$$
对它进行规约可以得到向量(q1,q2)满足
$$
q_2m = q_1 \mod{q}
$$
理论上，这样得到的q_1,q_2大约会在$\sqrt{q}$附近，而本题中的m我稍微多random了几次，选了个比较小的，这样得到的q_1 = 5049,q_2 = -6683，会比期望结果小很多，后续dual attack中的成功率会更高一些，但如果得到的结果大小完全就是$\sqrt{q}$，使用dual attack在理论上也是能够做的，但可能就需要获取更多的向量，更高的维度或者是使用筛法/FFT区分器等技术了。

把上面的式子全部带回最初的式子中
$$
A_2s_2 + e = b'\mod{q}\\
f'A_2s_2 + f'e = mgs_2 + f'e = f'b'\mod{q}\\
q_1gs_2 + q_2f'e = q_2f'b'\mod{q}
$$
最终，在实验中，若猜对了s_1，等式右边的计算结果约为q/100，是随机选取的1/50左右，这里还可以使用多个f'向量将得到的结果相加作为最终分数（也可以使用FFT，不过过于麻烦，并没有实现）。

并且，由于这里有部分的概率原因，所以我额外给了sha256(flag)方便选手检查得到的flag是否正确。（结果方便了猜单词）

## 参考文献

[1] Q. Guo and T. Johansson, ‘Faster Dual Lattice Attacks for Solving LWE with Applications to CRYSTALS’, in *Advances in Cryptology – ASIACRYPT 2021*, M. Tibouchi and H. Wang, Eds., in Lecture Notes in Computer Science, vol. 13093. Cham: Springer International Publishing, 2021, pp. 33–62. doi: [10.1007/978-3-030-92068-5_2](https://doi.org/10.1007/978-3-030-92068-5_2).

[2] MATZOV. (2022). “Report on the Security of LWE: Improved Dual Lattice Attack”. Zenodo. https://doi.org/10.5281/zenodo.6412487

[3] L. Ducas and L. N. Pulles, ‘Does the Dual-Sieve Attack on Learning with Errors even Work?’. https://eprint.iacr.org/2023/302