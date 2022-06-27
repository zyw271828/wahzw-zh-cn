# 摘要

不管是原始的论文 [Bit+11]; [Par+13] 还是原理讲解 [Rei16]; [But16]; [But17]; [Gab17]，其实市面上已经有大量关于 zk-SNARK 的学习资源了。zk-SNARK 由大量的可变模块组成，所以对很多人来说它依然像一个黑盒子一样很难懂。这些资料对 zk-SNARK 中的一些技术难题部分做出了解释，但由于缺少了对应的其它环节的解释，大家还是很难通过这些资料了解到 zk-SNARK 的全貌。

当我第一次了解到 zk-SNARK 技术是如何将这些东西完美地融合在一起的时候，就被数学之美震撼到了，并且随着我发现的维度越多，好奇心就越强烈。在这篇文章中，我主要就基于一些实例简洁明了地阐明 zk-SNARK，并对这里面的很多问题做出了解释，并利用这种方式分享了我的经验，进而让更多人也能够欣赏到这项最先进的技术以及它的创新之处，最终欣赏到数学之美。

这篇文章的主要贡献是比较简洁明了的解释了其中相当复杂的技术，这些简单的解释对于在不具备任何与之相关的先决知识（比如密码学和高等数学）的前提下理解 zk-SNARK 是很有必要的。文章中并不仅仅只解释 zk-SNARK 是如何工作的，还解释了为什么这样就可以工作，以及它是怎么来的。