# 深度学习调参手册（Deep Learning Tuning Playbook）

翻译by 我，持续更新。

翻译自[Deep Learning Tuning Playbook](https://github.com/google-research/tuning_playbook)。基于2023/01/20的最新版本(SHA：890cd5387477a5ea1b3a7a40fe4ad1d077f54151)。

由Google Research和Harvard University研究人员联合出品。

**翻译说明：**

非完全直译。对于一些常用的词汇，比如epoch，step等，选择不翻译。

对于一些难以翻译又不得不翻译的，选择意译的方式。

---

*非官方支持的Google产品*

作者：**Varun Godbole<sup>&dagger;</sup>, George E. Dahl<sup>&dagger;</sup>, Justin Gilmer<sup>&dagger;</sup>, Christopher J. Shallue<sup>&Dagger;</sup>, Zachary Nado<sup>&dagger;</sup>**


 <sup>&dagger;</sup>Google Research, Brain Team

 <sup>&Dagger;</sup>Harvard University

## 目录

-   [这个文档是为谁设计的？](#这个文档是为谁设计的？)
-   [为什么要写这个调参指南?](#为什么要写这个调参指南?)
-   [开始新项目前的指南](#开始新项目前的指南)
    -   [选择模型架构](#选择模型架构)
    -   [选择优化器](#选择优化器)
    -   [选择batchsize](#选择batchsize)
    -   [选择初始配置](#选择初始配置)
-   [提升模型性能的科学方法](#提升模型性能的科学方法)
    -   [The incremental tuning strategy](#the-incremental-tuning-strategy)
    -   [Exploration vs exploitation](#exploration-vs-exploitation)
    -   [Choosing the goal for the next round of experiments](#choosing-the-goal-for-the-next-round-of-experiments)
    -   [Designing the next round of experiments](#Designing-the-next-round-of-experiments)
    -   [Determining whether to adopt a training pipeline change or
        hyperparameter
        configuration](#Determining-whether-to-adopt-a-training-pipeline-change-or-hyperparameter-configuration)
    -   [After exploration concludes](#After-exploration-concludes)
-   [确定每次训练的steps数](#确定每次训练的steps数)
    -   [当训练不受计算限制时如何决定该训练多久](#当训练不受计算限制时如何决定该训练多久)
    -   [当训练受计算限制时如何决定该训练多久](#当训练受计算限制时如何决定该训练多久)
-   [Additional guidance for the training pipeline](#Additional-guidance-for-the-training-pipeline)
    -   [Optimizing the input pipeline](#Optimizing-the-input-pipeline)
    -   [Evaluating model performance](Evaluating-model-performance)
    -   [Saving checkpoints and retrospectively selecting the best checkpoint](#Saving-checkpoints-and-retrospectively-selecting-the-best-checkpoint)
    -   [Setting up experiment tracking](#Setting-up-experiment-tracking)
    -   [Batch normalization implementation details](#Batch-normalization-implementation-details)
    -   [Considerations for multi-host pipelines](#Considerations-for-multi-host-pipelines)
-   [常见问题解答](#常见问题解答)
-   [致谢](#致谢)
-   [引用](#引用)
-   [贡献](#contributing)

## 这个文档是为谁设计的？

本文档面向对深度学习**模型性能最优化**感兴趣的工程师和研究人员（个人和团队）。我们假设你已经掌握了机器学习和深度学习概念的基本知识。

我们的重点是超参数调参过程。我们触及了深度学习培训的其他方面，例如pipeline应用和优化，但是我们对这些方面的处理并不完整。

我们假设机器学习问题是一个有监督的学习问题或类似的问题（例如自监督）。尽管如此，本文档中的一些规定也可能适用于其他类型的问题。

## 为什么要写这个调参指南?

目前，要让深度神经网络在实践中表现得很好，需要涉及大量的辛劳和尝试。更糟糕的是，人们使用深度学习来获得良好结果的实际方法很少被记录下来。论文为了呈现一个更清晰的故事，往往掩盖了导致最终结果的过程。而研究商业问题的机器学习工程师很少有时间退一步，概括他们的过程。教科书倾向于回避实际指导，优先考虑基本原则，即使它们的作者在应用工作中有必要的经验，可以提供有用的建议。在准备创建本文档时，我们找不到任何全面的尝试来真正解释如何使用深度学习获得良好的结果。相反，我们在博客文章和社交媒体上找到了一些建议的片段，在研究论文的附录中发现了一些技巧，偶尔会有关于某个特定项目或pipeline的案例研究，还有很多困惑。深度学习专家和不太熟练的从业者使用表面上相似的方法，但是所取得的结果之间却存在着巨大的鸿沟。与此同时，这些专家欣然承认，他们所做的一些事情可能并不完全合理。随着深度学习的成熟并对世界产生更大的影响，社区需要更多的资源来涵盖有用的诀窍，包括所有对获得更好效果至关重要的实际细节。

我们是一个由五名研究人员和工程师组成的团队，他们在深度学习领域工作了多年，其中一些人早在2006年就开始了。我们已经将深度学习应用于从语音识别到天文学的很多问题，并在此过程中学到了很多东西。这份文档源于我们自己训练神经网络的经验，教授新的机器学习工程师，并就深度学习的实践为我们的同事提供建议。尽管看到深度学习从少数学术实验室实践的机器学习方法发展为数十亿人使用的产品技术是令人欣慰的，但深度学习作为一门工程学科仍处于起步阶段，我们希望这份文件鼓励其他人帮助系统化该领域的实验规程。

这份文档的出现是为了明确我们自己的深度学习方法，因此它代表了作者在写作时的观点，而不是任何形式的客观事实。我们自己在超参数调优方面的挣扎使它成为我们指南的一个特别重点，但我们也涵盖了我们在工作中遇到的其他重要问题(或看到的错误)。我们的意图是让这项工作成为一份活的文件，随着我们观念的改变而成长和发展。例如，关于debug和减少训练错误的材料在两年前是不可能写出来的，因为它是基于最近的结果和正在进行的调查。不可避免地，我们的一些建议需要不断更新，以说明新的结果和改进的工作流程。我们不知道最佳的深度学习配方，但在社区开始写下并讨论不同的过程之前，我们无法指望找到它。为此，我们鼓励那些对我们的建议有异议的读者提出替代建议，并提供令人信服的证据，这样我们就可以更新诀窍。我们也很乐意看到可能有不同建议的替代指南和诀窍，这样我们就可以作为一个社区，共同努力，从而实现最佳的实践。最后，任何标有🤖表情符号的区域都是我们想做更多研究的地方。

只有在尝试写完这本playbook之后，我才完全清楚在深度学习从业者的工作流程中可以找到多少有趣而被忽视的研究问题。

## 开始新项目前的指南

我们在调参过程中做出的许多决定可以在项目开始时做，只有在情况发生变化时偶尔会重新审视。

我们的指南提出了以下假设:

-   已经完成了足够的问题制定（problem formulation），数据清洁等的基本工作，以使在模型架构和训练配置上花费时间是有意义的。
-   已经有了一个pipeline,可以进行训练和评估。并容易为各种感兴趣的模型进行训练和预测。
-   选定并应用了合适的metrics。这些应该尽可能地代表将在部署环境中测量的内容。

### 选择模型架构

***概要:*** *当开始一个新项目时，请尝试使用已经work的模型。*

-   选择一个成熟的、广泛使用的模型架构。 这样可以更好的在后续构建自定义模型。
-   模型架构常有很多超参数来确定模型的大小和其他细节。比如layers的数量和宽度，比如激活函数的类型。

    -   因此，选择模型架构实际上代表着选择一类模型，模型大家族内不同的模型代表着不同的一组超参数。
    -   我们将在[选择初始配置](#选择初始配置)和[提升模型性能的科学方法](#提升模型性能的科学方法)两章考虑模型超参数选择的问题。

-   如果可能，尝试找到一篇 解决了与手头问题尽可能相近的问题 的论文，并把复现论文中的模型作为起点。

### 选择优化器

***概要:*** *从最受欢迎的优化器开始，用于解决手头问题。*

-   在所有类型的机器学习问题和模型架构中，没有优化器是“最佳”的。 甚至比较优化器性能是困难的。
    [参考论文：《comparing the performance of optimizers is a difficult task》](https://arxiv.org/abs/1910.05446).
    🤖
-   我们推荐使用成熟的、广泛使用的优化器，尤其是在开始一个新项目的时候。
    -   理想情况下，选择用于相同类型问题下的最广泛使用的优化器。
-   注意关注所选择优化器的***所有***超参数。
    -   具有更多超参数的优化器可能需要更多的调参经历来找到最佳配置。
    -   这点很重要，尤其是在项目的开始阶段我们可能，而忽略了优化器的超参数，把它看成了无用且讨厌的参数（[nuisance parameters](#identifying-scientific-nuisance-and-fixed-hyperparameters)）。
    -   在项目的初始夹断，最好是从一个简单的优化器开始，比如使用固定动量的SGD或固定 $\epsilon$、 $\beta_{1}$、$\beta_{2}$的Adam。然后再切换到更通用的优化器。
-   我们喜欢的成熟的优化器包括（但不限于）以下：
    -   [带动量的SGD](#what-are-the-update-rules-for-all-the-popular-optimization-algorithms)
        (我们喜欢 Nesterov变种的。)
    -   [Adam 和NAdam](#what-are-the-update-rules-for-all-the-popular-optimization-algorithms),
        它们比带动量的SGD更通用。请注意，Adam有四个可调的超参数，而且[它们都很重要](https://arxiv.org/abs/1910.05446)！
        -   可看
            [Adam的超参数应该如何调整？](#Adam的超参数应该如何调整？)

### 选择batchsize

***概要:*** *batch size 影响训练速度，不应用于直接调整验证集的性能。通常，理想的batch size是硬件所能达到的最大的batch size。*

-   batch size 是确定训练时间和计算资源消耗的关键因素。
-   增加batch size的大小通常会减少训练时间。 这可能是非常有益的，因为它，例如：
    -   允许在固定时间间隔内更彻底地调整超参数，从而可能导致更好的最终模型。
    -   减少开发周期的延迟，从而更频繁地测试新想法。
-   增加batch size大小可能会减少，增加或不改变资源消耗。
-   batch size不应被视为针对验证集性能的超参数。
    -   只要所有超参数都经过了良好的调参(特别是学习率和正则化的参数)，并且训练步骤足够多，那么使用任何batch size大小都应该能够获得相同的最终性能 (见
        [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).
    -   请见 [Why shouldn't the batch size be tuned to directly improve
        validation set
        performance?](#why-shouldnt-the-batch-size-be-tuned-to-directly-improve-validation-set-performance)

#### 确定可行的batchsize大小和估计训练的吞吐量


<details><summary><em>[点击展开]</em></summary>


<br>

-   对于给定的模型和优化器，可用硬件通常会支持一定范围的batch size大小。限制因素通常是GPU等加速器的内存（accelerator memory）。
-   不幸的是，如果不运行或至少编译完整的训练程序，就很难计算出哪些batch size大小适合内存。
-   最简单的解决方案通常是以不同的batch size大小(例如增加2的幂)运行少量步骤的训练作业（ job），直到其中一个作业超过可用内存。
-   对于每个批次大小，我们应该训练足够长的时间，以获得*训练吞吐量（training throughput）*的可靠估计。

<p align="center">training throughput = (# examples processed per second)</p>

<p align="center">或者,同样的, the <em>time per step</em>.</p>

<p align="center">time per step = (batch size) / (training throughput)</p>

-   当加速器尚未饱和时，如果batch size翻倍，训练的吞吐量也应该翻倍(或至少接近翻倍)。
    同样的, 随着batch size 增加，the time per step 是恒定的 (或几乎恒定) 
-   如果不是这种情况，那么训练pipeline就会出现瓶颈，比如计算节点之间的I/O或同步。在继续之前，这可能值得诊断和纠正。
-   如果训练吞吐量只增加到某个最大的batch size，那么我们应该只考虑最大的batch size，即，使硬件支持更大的batch size。
    -   使用更大的batch size的所有好处都假定训练吞吐量增加。如果不能，修复瓶颈或使用较小的batch size。
    -   **梯度累加（Gradient accumulation）** 是一种不需要额外硬件资源就可以增加batch size的训练技巧，能模拟出大于硬件所能支持的batch size。它不提供任何吞吐量优势。在实际工作中一般应避免使用。
-   这些步骤可能需要在每次模型或优化器更改时重复执行(例如，不同的模型架构可能允许更大的batch size)..

</details>

#### 选择batchsize来最小化训练时间

<details><summary><em>[点击展开]</em></summary>


<br>


<p align="center">Training time = (time per step) x (total number of steps)</p>

-   对于可行的batch size，我们可以认为time per step是常数。这是正确的，当没有并行运算且所有的训练瓶颈都被诊断和纠正时。
    (查看[上一节内容](#确定可行的batchsize大小和估计训练的吞吐量)来识别训练瓶颈). 在实际中，增加batch size通常会产生一些开销。
-   随着batch size的增加，达到固定性能目标所需的总step数通常会减少(假设在batch size改变时重新调优所有相关的超参数;;[Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).
    -   例， 将batch size 大小加倍可能会使所需的step数减半。这被称为**perfect scaling**.
    -   Perfect scaling 适用于所有的batch size，知道临界batch size。超过临界batch size后，收益逐渐减少。
    -   最终，增加batch size的大小不再减少训练step的数量（但永远不会增加）。
-   因此，使训练时间最小的batch size通常是最大的batch size，仍然可以减少所需的训练step数。
    -   batch size 大小取决于数据集、模型和优化器。如何计算它是一个开放问题，除了通过实验为每个新问题找到它。🤖
    -   在比较batch size大小时，请注意样本预算/epoch预算（在固定训练样本演示数量的同时运行所有实验（experiments））和step预算（在确定训练step数量的情况下运行所有实验）之间的区别。
        -   将batch size的大小与epoch预算进行），即使当较大的batch size大小仍然可以通过减少所需的训练step数量来提供有意义的加速。
    -   通常，可用硬件支持的最大batch size大小将小于临界的batch size大小。因此，一个好的经验法则（不进行任何实验）是尽可能使用最大的batch size。
-   如果最终增加了训练时间，那么使用更大的batch size是没有意义的。

</details>

#### 选择batchsize以最小化资源消耗

<details><summary><em>[点击展开]</em></summary>


<br>


-   与增大batch size 相关的资源成本有两种类型:
    1.  *前期成本*,例如，购买新的硬件或者在训练pipeline上应用多GPU/多TPU训练方式重新运行。
    2.  *使用成本*, 例如根据团队的资源预算计费、云服务提供商计费、电力/维护成本。
-   如果增加batch size需要大量的前期成本，那么最好推迟增加batch size，直到项目成熟，这样更容易评估成本效益权衡。实施多主机并行训练程序可能会引入[bugs](#considerations-for-multi-host-pipelines)和[subtle issues](#batch-normalization-implementation-details)，因此最好从更简单的pipeline开始。（另一方面，当需要进行大量调参实验时，在训练过程的早期，大幅加快训练时间可能非常有益）。
-   我们将总使用成本（可能包括多种不同的成本）称为“资源消耗（量）”。我们可以将资源消耗量分解为以下几个部分：

<p align="center">资源消耗量 = (每个step的资源消耗量) x (总的steps的数量)</p>

-   增加batch size 往往允许我们[减少总的steps的数量](#选择batchsize来最小化训练时间)。资源消耗量是增加还是减少取决于每个step的消耗量如何变化。
    -   增加batch size大小可能会减少资源消耗。例如，如果具有较大batch size大小的每个step可以在与较小batch size大小相同的硬件上运行（每个step的时间仅略有增加），则每个step的资源消耗的任何增加都可能被step数量的减少所抵消。
    -   增加batch size大小可能不会改变资源消耗。例如，如果将batch size大小增加一倍，则所需的steps数减少一半，使用的GPU数量增加一倍时，总消耗量（以GPU时为单位）不会改变。
    -   增加batch size大小可能会*增加*资源消耗。例如，如果增加batch size大小需要升级硬件，则每一step的消耗增加可能会超过step数的减少。

</details>

#### 更改batchsize大小需要重新调整大多数超参数

<details><summary><em>[点击展开]</em></summary>


<br>


-   大多数超参数的最佳值对batch size大小敏感。因此，更改batch szie的大小通常需要重新启动调参过程。
-   与batch size大小影响最相关的超参数是优化器超参数（例如，学习率、动量）和正则化的超参数，因此，对于每个batch size大小，对它们进行单独调整是个重要的过程。
-   在项目开始时选择batch size大小时，请记住这一点：如果以后需要切换到不同的batch size大小，则为新的batch size重新调整所有内容可能会很困难、耗时且成本高昂。

</details>

#### batch norm如何和batch size相互影响

<details><summary><em>[点击展开]</em></summary>


<br>


-   batch norm是复杂的，一般来说，应该使用不同于梯度计算的批次大小来计算统计数据。有关详细讨论，请参阅[batch norm部分](#batch-normalization-implementation-details)。

</details>

### 选择初始配置

-   在开始超参数调参之前，我们必须确定起始点。这包括指定（1）模型配置（例如层数）、（2）优化器超参数（例如学习率）和（3）训练step的数量。
-   确定此初始配置将需要一些手动配置的训练运行和试错。
-   我们的指导原则是找到一种简单、相对快速、相对低的资源消耗配置，以获得“合理”的结果。
    -   “简单”意味着尽可能避免一些花里胡哨的东西；这些总是可以稍后添加。即使一些花里胡哨的东西在未来证明是有用的，但在初始配置中添加它们也有可能会导致浪费时间在调整无用的功能和/或导致不必要的复杂性。
        -   例如，在添加梯度衰减学习率之前，先从恒定学习率开始。
    -   选择一个快速且消耗最少资源的初始配置将使超参数调整更加有效。
        -   例如，先从一个小模型开始。
    -   “合理的”性能取决于问题本身，但至少意味着经过训练的模型在验证集上的表现比随机机会要好得多(尽管它可能坏到不值得部署)。
-   选择训练step涉及到平衡以下的紧张关系（tension）:
    -   一方面，训练更多的step可以提高性能，使超参数调优更容易 (见 [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).
    -   另一方面，训练step更少意味着每次训练运行更快，使用更少的资源，通过减少周期之间的时间和允许更多的实验并行运行来提高调参效率。
        此外，如果一开始选择了一个不必要的大步骤预算，那么在接下来的过程中可能很难改变它，例如，针对step数进行调整的学习率策略。

## A scientific approach to improving model performance

For the purposes of this document, the ultimate goal of machine learning
development is to maximize the utility of the deployed model. Even though many
aspects of the development process differ between applications (e.g. length of
time, available computing resources, type of model), we can typically use the
same basic steps and principles on any problem.

Our guidance below makes the following assumptions:

-   There is already a fully-running training pipeline along with a
    configuration that obtains a reasonable result.
-   There are enough computational resources available to conduct meaningful
    tuning experiments and run at least several training jobs in parallel.

### The incremental tuning strategy

***概要:*** *Start with a simple configuration and incrementally make
improvements while building up insight into the problem. Make sure that any
improvement is based on strong evidence to avoid adding unnecessary complexity.*

-   Our ultimate goal is to find a configuration that maximizes the performance
    of our model.
    -   In some cases, our goal will be to maximize how much we can improve the
        model by a fixed deadline (e.g. submitting to a competition).
    -   In other cases, we want to keep improving the model indefinitely (e.g.
        continually improving a model used in production).
-   In principle, we could maximize performance by using an algorithm to
    automatically search the entire space of possible configurations, but this
    is not a practical option.
    -   The space of possible configurations is extremely large and there are
        not yet any algorithms sophisticated enough to efficiently search this
        space without human guidance.
-   Most automated search algorithms rely on a hand-designed *search space* that
    defines the set of configurations to search in, and these search spaces can
    matter quite a bit.
-   The most effective way to maximize performance is to start with a simple
    configuration and incrementally add features and make improvements while
    building up insight into the problem.
    -   We use automated search algorithms in each round of tuning and
        continually update our search spaces as our understanding grows.
-   As we explore, we will naturally find better and better configurations and
    therefore our "best" model will continually improve.
    -   We call it a *launch* when we update our best configuration (which may
        or may not correspond to an actual launch of a production model).
    -   For each launch, we must make sure that the change is based on strong
        evidence – not just random chance based on a lucky configuration – so
        that we don't add unnecessary complexity to the training pipeline.

At a high level, our incremental tuning strategy involves repeating the
following four steps:

1.  Identify an appropriately-scoped goal for the next round of experiments.
2.  Design and run a set of experiments that makes progress towards this goal.
3.  Learn what we can from the results.
4.  Consider whether to launch the new best configuration.

The remainder of this section will consider this strategy in much greater
detail.

### Exploration vs exploitation

***概要:*** *Most of the time, our primary goal is to gain insight into the
problem.*

-   Although one might think we would spend most of our time trying to maximize
    performance on the validation set, in practice we spend the majority of our
    time trying to gain insight into the problem, and comparatively little time
    greedily focused on the validation error.
    -   In other words, we spend most of our time on "exploration" and only a
        small amount on "exploitation".
-   In the long run, understanding the problem is critical if we want to
    maximize our final performance. Prioritizing insight over short term gains
    can help us:
    -   Avoid launching unnecessary changes that happened to be present in
        well-performing runs merely through historical accident.
    -   Identify which hyperparameters the validation error is most sensitive
        to, which hyperparameters interact the most and therefore need to be
        re-tuned together, and which hyperparameters are relatively insensitive
        to other changes and can therefore be fixed in future experiments.
    -   Suggest potential new features to try, such as new regularizers if
        overfitting is an issue.
    -   Identify features that don't help and therefore can be removed, reducing
        the complexity of future experiments.
    -   Recognize when improvements from hyperparameter tuning have likely
        saturated.
    -   Narrow our search spaces around the optimal value to improve tuning
        efficiency.
-   When we are eventually ready to be greedy, we can focus purely on the
    validation error even if the experiments aren't maximally informative about
    the structure of the tuning problem.

### Choosing the goal for the next round of experiments

***概要:*** *Each round of experiments should have a clear goal and be
sufficiently narrow in scope that the experiments can actually make progress
towards the goal.*

-   Each round of experiments should have a clear goal and be sufficiently
    narrow in scope that the experiments can actually make progress towards the
    goal: if we try to add multiple features or answer multiple questions at
    once, we may not be able to disentangle the separate effects on the results.
-   Example goals include:
    -   Try a potential improvement to the pipeline (e.g. a new regularizer,
        preprocessing choice, etc.).
    -   Understand the impact of a particular model hyperparameter (e.g. the
        activation function)
    -   Greedily maximize validation error.

### Designing the next round of experiments

***概要:*** *Identify which hyperparameters are scientific, nuisance, and
fixed hyperparameters for the experimental goal. Create a sequence of studies to
compare different values of the scientific hyperparameters while optimizing over
the nuisance hyperparameters. Choose the search space of nuisance
hyperparameters to balance resource costs with scientific value.*

#### Identifying scientific, nuisance, and fixed hyperparameters

<details><summary><em>[点击展开]</em></summary>


<br>

-   For a given goal, all hyperparameters will be either **scientific
    hyperparameters**, **nuisance hyperparameters**, or **fixed
    hyperparameters**.
    -   Scientific hyperparameters are those whose effect on the model's
        performance we're trying to measure.
    -   Nuisance hyperparameters are those that need to be optimized over in
        order to fairly compare different values of the scientific
        hyperparameters. This is similar to the statistical concept of
        [nuisance parameters](https://en.wikipedia.org/wiki/Nuisance_parameter).
    -   Fixed hyperparameters will have their values fixed in the current round
        of experiments. These are hyperparameters whose values do not need to
        (or we do not want them to) change when comparing different values of
        the scientific hyperparameters.
        -   By fixing certain hyperparameters for a set of experiments, we must
            accept that conclusions derived from the experiments might not be
            valid for other settings of the fixed hyperparameters. In other
            words, fixed hyperparameters create caveats for any conclusions we
            draw from the experiments.
-   For example, if our goal is to "determine whether a model with more hidden
    layers will reduce validation error", then the number of hidden layers is a
    scientific hyperparameter.
    -   The learning rate is a nuisance hyperparameter because we can only
        fairly compare models with different numbers of hidden layers if the
        learning rate is tuned separately for each number of layers (the optimal
        learning rate generally depends on the model architecture).
    -   The activation function could be a fixed hyperparameter if we have
        determined in prior experiments that the best choice of activation
        function is not sensitive to model depth, or if we are willing to limit
        our conclusions about the number of hidden layers to only cover this
        specific choice of activation function. Alternatively, it could be a
        nuisance parameter if we are prepared to tune it separately for each
        number of hidden layers.
-   Whether a particular hyperparameter is a scientific hyperparameter, nuisance
    hyperparameter, or fixed hyperparameter is not inherent to that
    hyperparameter, but changes depending on the experimental goal.
    -   For example, the choice of activation function could be a scientific
        hyperparameter (is ReLU or tanh a better choice for our problem?), a
        nuisance hyperparameter (is the best 5-layer model better than the best
        6-layer model when we allow several different possible activation
        functions?), or a fixed hyperparameter (for ReLU nets, does adding batch
        normalization in a particular position help?).
-   When designing a new round of experiments, we first identify the scientific
    hyperparameters for our experimental goal.
    -   At this stage, we consider all other hyperparameters to be nuisance
        hyperparameters.
-   Next, we convert some of the nuisance hyperparameters into fixed
    hyperparameters.
    -   With limitless resources, we would leave all non-scientific
        hyperparameters as nuisance hyperparameters so that the conclusions we
        draw from our experiments are free from caveats about fixed
        hyperparameter values.
    -   However, the more nuisance hyperparameters we attempt to tune, the
        greater the risk we fail to tune them sufficiently well for each setting
        of the scientific hyperparameters and end up reaching the wrong
        conclusions from our experiments.
        -   As described
            [below](#striking-a-balance-between-informative-and-affordable-experiments),
            we could counter this risk by increasing the computational budget,
            but often our maximum resource budget is less than would be needed
            to tune over all non-scientific hyperparameters.
    -   We choose to convert a nuisance hyperparameter into a fixed
        hyperparameter when, in our judgment, the caveats introduced by fixing
        it are less burdensome than the cost of including it as a nuisance
        hyperparameter.
        -   The more a given nuisance hyperparameter interacts with the
            scientific hyperparameters, the more damaging it is to fix its
            value. For example, the best value of the weight decay strength
            typically depends on the model size, so comparing different model
            sizes assuming a single specific value of the weight decay would not
            be very insightful.
-   Although the type we assign to each hyperparameter depends on the
    experimental goal, we have the following rules of thumb for certain
    categories of hyperparameters:
    -   Of the various optimizer hyperparameters (e.g. the learning rate,
        momentum, learning rate schedule parameters, Adam betas etc.), at least
        some of them will be nuisance hyperparameters because they tend to
        interact the most with other changes.
        -   They are rarely scientific hyperparameters because a goal like "what
            is the best learning rate for the current pipeline?" doesn't give
            much insight – the best setting could easily change with the next
            pipeline change anyway.
        -   Although we might fix some of them occasionally due to resource
            constraints or when we have particularly strong evidence that they
            don't interact with the scientific parameters, we should generally
            assume that optimizer hyperparameters must be tuned separately to
            make fair comparisons between different settings of the scientific
            hyperparameters, and thus shouldn't be fixed.
            -   Furthermore, we have no *a priori* reason to prefer one
                optimizer hyperparameter value over another (e.g. they don't
                usually affect the computational cost of forward passes or
                gradients in any way).
    -   In contrast, the *choice* of optimizer is typically a scientific
        hyperparameter or fixed hyperparameter.
        -   It is a scientific hyperparameter if our experimental goal involves
            making fair comparisons between two or more different optimizers
            (e.g. "determine which optimizer produces the lowest validation
            error in a given number of steps").
        -   Alternatively, we might make it a fixed hyperparameter for a variety
            of reasons, including (1) prior experiments make us believe that the
            best optimizer for our problem is not sensitive to current
            scientific hyperparameters; and/or (2) we prefer to compare values
            of the scientific hyperparameters using this optimizer because its
            training curves are easier to reason about; and/or (3) we prefer to
            use this optimizer because it uses less memory than the
            alternatives.
    -   Hyperparameters introduced by a regularization technique are typically
        nuisance hyperparameters, but whether or not we include the
        regularization technique at all is a scientific or fixed hyperparameter.
        -   For example, dropout adds code complexity, so when deciding whether
            to include it we would make "no dropout" vs "dropout" a scientific
            hyperparameter and the dropout rate a nuisance hyperparameter.
            -   If we decide to add dropout to our pipeline based on this
                experiment, then the dropout rate would be a nuisance
                hyperparameter in future experiments.
    -   Architectural hyperparameters are often scientific or fixed
        hyperparameters because architecture changes can affect serving and
        training costs, latency, and memory requirements.
        -   For example, the number of layers is typically a scientific or fixed
            hyperparameter since it tends to have dramatic consequences for
            training speed and memory usage.
-   In some cases, the sets of nuisance and fixed hyperparameters will depend on
    the values of the scientific hyperparameters.
    -   For example, suppose we are trying to determine which optimizer out of
        Nesterov momentum and Adam results in the lowest validation error. The
        scientific hyperparameter is the `optimizer`, which takes values
        `{"Nesterov_momentum", "Adam"}`. The value
        `optimizer="Nesterov_momentum"` introduces the nuisance/fixed
        hyperparameters `{learning_rate, momentum}`, but the value
        `optimizer="Adam"` introduces the nuisance/fixed hyperparameters
        `{learning_rate, beta1, beta2, epsilon}`.
    -   Hyperparameters that are only present for certain values of the
        scientific hyperparameters are called **conditional hyperparameters**.
    -   We should not assume two conditional hyperparameters are the same just
        because they have the same name! In the above example, the conditional
        hyperparameter called `learning_rate` is a *different* hyperparameter
        for `optimizer="Nesterov_momentum"` versus `optimizer="Adam"`. Its role
        is similar (although not identical) in the two algorithms, but the range
        of values that work well in each of the optimizers is typically
        different by several orders of magnitude.

</details>

#### Creating a set of studies

<details><summary><em>[点击展开]</em></summary>


<br>


-   Once we have identified the scientific and nuisance hyperparameters, we
    design a "study" or sequence of studies to make progress towards the
    experimental goal.
    -   A study specifies a set of hyperparameter configurations to be run for
        subsequent analysis. Each configuration is called a "trial".
    -   Creating a study typically involves choosing the hyperparameters that
        will vary across trials, choosing what values those hyperparameters can
        take on (the "search space"), choosing the number of trials, and
        choosing an automated search algorithm to sample that many trials from
        the search space. Alternatively, we could create a study by specifying
        the set of hyperparameter configurations manually.
-   The purpose of the studies is to run the pipeline with different values of
    the scientific hyperparameters, while at the same time **"optimizing away"**
    (or "optimizing over") the nuisance hyperparameters so that comparisons
    between different values of the scientific hyperparameters are as fair as
    possible.
-   In the simplest case, we would make a separate study for each configuration
    of the scientific parameters, where each study tunes over the nuisance
    hyperparameters.
    -   For example, if our goal is to select the best optimizer out of Nesterov
        momentum and Adam, we could create one study in which
        `optimizer="Nesterov_momentum"` and the nuisance hyperparameters are
        `{learning_rate, momentum}`, and another study in which
        `optimizer="Adam"` and the nuisance hyperparameters are `{learning_rate,
        beta1, beta2, epsilon}`. We would compare the two optimizers by
        selecting the best performing trial from each study.
    -   We can use any gradient-free optimization algorithm, including methods
        such as Bayesian optimization or evolutionary algorithms, to optimize
        over the nuisance hyperparameters, although
        [we prefer](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
        to use quasi-random search in the
        [exploration phase](#exploration-vs-exploitation) of tuning because of a
        variety of advantages it has in this setting.
        [After exploration concludes](#after-exploration-concludes), if
        state-of-the-art Bayesian optimization software is available, that is
        our preferred choice.
-   In the more complicated case where we want to compare a large number of
    values of the scientific hyperparameters and it is impractical to make that
    many independent studies, we can include the scientific parameters in the
    same search space as the nuisance hyperparameters and use a search algorithm
    to sample values of *both* the scientific and nuisance hyperparameters in a
    single study.
    -   When taking this approach, conditional hyperparameters can cause
        problems since it is hard to specify a search space unless the set of
        nuisance hyperparameters is the same for all values of the scientific
        hyperparameters.
    -   In this case,
        [our preference](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
        for using quasi-random search over fancier black-box optimization tools
        is even stronger, since it ensures that we obtain a relatively uniform
        sampling of values of the scientific hyperparameters. Regardless of the
        search algorithm, we need to make sure somehow that it searches the
        scientific parameters uniformly.

</details>

#### Striking a balance between informative and affordable experiments

<details><summary><em>[点击展开]</em></summary>


<br>


-   When designing a study or sequence of studies, we need to allocate a limited
    budget in order to adequately achieve the following three desiderata:
    1.  Comparing enough different values of the scientific hyperparameters.
    2.  Tuning the nuisance hyperparameters over a large enough search space.
    3.  Sampling the search space of nuisance hyperparameters densely enough.
-   The better we can achieve these three desiderata, the more insight we can
    extract from our experiment.
    -   Comparing as many values of the scientific hyperparameters as possible
        broadens the scope of the insights we gain from the experiment.
    -   Including as many nuisance hyperparameters as possible and allowing each
        nuisance hyperparameter to vary over as wide a range as possible
        increases our confidence that a "good" value of the nuisance
        hyperparameters **exists** in the search space for each configuration of
        the scientific hyperparameters.
        -   Otherwise, we might make unfair comparisons between values of the
            scientific hyperparameters by not searching possible regions of the
            nuisance parameter space where better values might lie for some
            values of the scientific parameters.
    -   Sampling the search space of nuisance hyperparameters as densely as
        possible increases our confidence that any good settings for the
        nuisance hyperparameters that happen to exist in our search space will
        be found by the search procedure.
        -   Otherwise, we might make unfair comparisons between values of the
            scientific parameters due to some values getting luckier with the
            sampling of the nuisance hyperparameters.
-   Unfortunately, improvements in *any* of these three dimensions require
    either increasing the number of trials, and therefore increasing the
    resource cost, or finding a way to save resources in one of the other
    dimensions.
    -   Every problem has its own idiosyncrasies and computational constraints,
        so how to allocate resources across these three desiderata requires some
        level of domain knowledge.
    -   After running a study, we always try to get a sense of whether the study
        tuned the nuisance hyperparameters well enough (i.e. searched a large
        enough space extensively enough) to fairly compare the scientific
        hyperparameters (as described in greater detail
        [below](#extracting-insight-from-experimental-results)).

</details>

### Extracting insight from experimental results

***概要:*** *In addition to trying to achieve the original scientific goal of
each group of experiments, go through a checklist of additional questions and,
if issues are discovered, revise the experiments and rerun them.*

-   Ultimately, each group of experiments has a specific goal and we want to
    evaluate the evidence the experiments provide toward that goal.
    -   However, if we ask the right questions, we will often find issues that
        need to be corrected before a given set of experiments can make much
        progress towards their original goal.
        -   If we don’t ask these questions, we may draw incorrect conclusions.
    -   Since running experiments can be expensive, we also want to take the
        opportunity to extract other useful insights from each group of
        experiments, even if these insights are not immediately relevant to the
        current goal.
-   Before analyzing a given set of experiments to make progress toward their
    original goal, we should ask ourselves the following additional questions:
    -   [Is the search space large enough?](#identifying-bad-search-space-boundaries)
        -   If the optimal point from a study is near the boundary of the search
            space in one or more dimensions, the search is probably not wide
            enough. In this case, we should run another study with an expanded
            search space.
    -   [Have we sampled enough points from the search space?](#not-sampling-enough-points-in-the-search-space)
        -   If not, run more points or be less ambitious in the tuning goals.
    -   What fraction of the trials in each study are **infeasible** (i.e.
        trials that diverge, get really bad loss values, or fail to run at all
        because they violate some implicit constraint)?
        -   When a very large fraction of points in a study are **infeasible**
            we should try to adjust the search space to avoid sampling such
            points, which sometimes requires reparameterizing the search space.
        -   In some cases, a large number of infeasible points can indicate a
            bug in the training code.
    -   [Does the model exhibit optimization issues?](#how-can-optimization-failures-be-debugged-and-mitigated)
    -   [What can we learn from the training curves of the best trials?](#examining-the-training-curves)
        -   For example, do the best trials have training curves consistent with
            problematic overfitting?
-   If necessary, based on the answers to the questions above, refine the most
    recent study (or group of studies) to improve the search space and/or sample
    more trials, or take some other corrective action.
-   Once we have answered the above questions, we can move on to evaluating the
    evidence the experiments provide towards our original goal (for example,
    [evaluating whether a change is useful](#detecting-whether-a-change-is-useful-with-isolation-plots)).

#### Identifying bad search space boundaries

<details><summary><em>[点击展开]</em></summary>


<br>


-   A search space is suspicious if the best point sampled from it is close to
    its boundary. We might find an even better point if we expanded the search
    range in that direction.
-   To check search space boundaries, we like to plot completed trials on what
    we call **basic hyperparameter axis plots** where we plot the validation
    objective value versus one of the hyperparameters (e.g. learning rate). Each
    point on the plot corresponds to a single trial.
    -   The validation objective value for each trial should usually be the best
        value it achieved over the course of training.

<p align="center" id="figure-1">
<img src="assets/bad_search_space.png" width="49%" alt="Example of bad search space boundaries">
<img src="assets/good_search_space.png" width="49%" alt="Example of good search space boundaries">
</p>


<p align="center"><b>Figure 1:</b> Examples of bad search space boundaries and acceptable search space boundaries.</p>

-   The plots in [Figure 1](#figure-1) show the error rate (lower is better)
    against the initial learning rate.
-   If the best points cluster towards the edge of a search space (in some
    dimension), then the search space boundaries might need to be expanded until
    the best observed point is no longer close to the boundary.
-   Often, a study will include "infeasible" trials that diverge or get very bad
    results (marked with red Xs in the above plots).
    -   If all trials are infeasible for learning rates greater than some
        threshold value, and if the best performing trials have learning rates
        at the edge of that region, the model [may suffer from stability issues
        preventing it from accessing higher learning
        rates](#how-can-optimization-failures-be-debugged-and-mitigated).

</details>

#### Not sampling enough points in the search space

<details><summary><em>[点击展开]</em></summary>


<br>


-   In general,
    [it can be very difficult to know](#how-many-trials-are-needed-to-get-good-results-with-quasi-random-search)
    if the search space has been sampled densely enough. 🤖
-   Running more trials is of course better, but comes at an obvious cost.
-   Since it is so hard to know when we have sampled enough, we usually sample
    what we can afford and try to calibrate our intuitive confidence from
    repeatedly looking at various hyperparameter axis plots and trying to get a
    sense of how many points are in the "good" region of the search space.

</details>

#### Examining the training curves

<details><summary><em>[点击展开]</em></summary>


<br>


***概要:*** *Examining the training curves is an easy way to identify common
failure modes and can help us prioritize what actions to take next.*

-   Although in many cases the primary objective of our experiments only
    requires considering the validation error of each trial, we must be careful
    when reducing each trial to a single number because it can hide important
    details about what’s going on below the surface.
-   For every study, we always look at the **training curves** (training error
    and validation error plotted versus training step over the duration of
    training) of at least the best few trials.
-   Even if this is not necessary for addressing the primary experimental
    objective, examining the training curves is an easy way to identify common
    failure modes and can help us prioritize what actions to take next.
-   When examining the training curves, we are interested in the following
    questions.
-   Are any of the trials exhibiting **problematic overfitting?**
    -   Problematic overfitting occurs when the validation error starts
        *increasing* at some point during training.
    -   In experimental settings where we optimize away nuisance hyperparameters
        by selecting the "best" trial for each setting of the scientific
        hyperparameters, we should check for problematic overfitting in *at
        least* each of the best trials corresponding to the settings of the
        scientific hyperparameters that we’re comparing.
        -   If any of the best trials exhibits problematic overfitting, we
            usually want to re-run the experiment with additional regularization
            techniques and/or better tune the existing regularization parameters
            before comparing the values of the scientific hyperparameters.
            -   This may not apply if the scientific hyperparameters include
                regularization parameters, since then it would not be surprising
                if low-strength settings of those regularization parameters
                resulted in problematic overfitting.
        -   Reducing overfitting is often straightforward using common
            regularization techniques that add minimal code complexity or extra
            computation (e.g. dropout, label smoothing, weight decay), so it’s
            usually no big deal to add one or more of these to the next round of
            experiments.
        -   For example, if the scientific hyperparameter is "number of hidden
            layers" and the best trial that uses the largest number of hidden
            layers exhibited problematic overfitting, then we would usually
            prefer to try it again with additional regularization instead of
            immediately selecting the smaller number of hidden layers.
        -   Even if none of the "best" trials are exhibiting problematic
            overfitting, there might still be a problem if it occurs in *any* of
            the trials.
            -   Selecting the best trial suppresses configurations exhibiting
                problematic overfitting and favors those that do not. In other
                words, it will favor configurations with more regularization.
            -   However, anything that makes training worse can act as a
                regularizer, even if it wasn't intended that way. For example,
                choosing a smaller learning rate can regularize training by
                hobbling the optimization process, but we typically don't want
                to choose the learning rate this way.
            -   So we must be aware that the "best" trial for each setting of
                the scientific hyperparameters might be selected in such a way
                that favors "bad" values of some of the scientific or nuisance
                hyperparameters.
-   Is there high step-to-step variance in the training or validation error late
    in training?
    -   If so, this could interfere with our ability to compare different values
        of the scientific hyperparameters (since each trial randomly ends on a
        "lucky" or "unlucky" step) and our ability to reproduce the result of
        the best trial in production (since the production model might not end
        on the same "lucky" step as in the study).
    -   The most likely causes of step-to-step variance are batch variance (from
        randomly sampling examples from the training set for each batch), small
        validation sets, and using a learning rate that’s too high late in
        training.
    -   Possible remedies include increasing the batch size, obtaining more
        validation data, using learning rate decay, or using Polyak averaging.
-   Are the trials still improving at the end of training?
    -   If so, this indicates that we are in the
        ["compute bound" regime](#determining-the-number-of-steps-for-each-training-run)
        and we may benefit from
        [increasing the number of training steps](#Deciding-how-long-to-train-when-training-is-compute-bound)
        or changing the learning rate schedule.
-   Has performance on the training and validation sets saturated long before
    the final training step?
    -   If so, this indicates that we are in the
        ["not compute-bound"](#determining-the-number-of-steps-for-each-training-run)
        regime and that we may be able to
        [decrease the number of training steps](#deciding-how-long-to-train-when-training-is-not-compute-bound).
-   Although we cannot enumerate them all, there are many other additional
    behaviors that can become evident from examining the training curves (e.g.
    training loss *increasing* during training usually indicates a bug in the
    training pipeline).

</details>

#### Detecting whether a change is useful with isolation plots

<details><summary><em>[点击展开]</em></summary>


<br>


<p align="center" id="figure-2">
<img src="assets/isolation_plot.png" width="49%" alt="Isolation plot that investigates the best value of weight decay for ResNet-50
trained on ImageNet.">
</p>


<p align="center"><b>Figure 2:</b> Isolation plot that investigates the best value of weight decay for ResNet-50 trained on ImageNet.</p>

-   Often, the goal of a set of experiments is to compare different values of a
    scientific hyperparameter.
    -   For example, we may want to determine the value of weight decay that
        results in the best validation error.
-   An **isolation plot** is a special case of the basic hyper-parameter axis
    plot. Each point on an isolation plot corresponds to the performance of the
    *best* trial across some (or all) of the nuisance hyperparameters.
    -   In other words, we plot the model performance after "optimizing away"
        the nuisance hyperparameters.
-   An isolation plot makes it easier to perform an apples-to-apples comparison
    between different values of the scientific hyperparameter.
-   For example, [Figure 2](#figure-2) reveals the value of weight decay that
    produces the best validation performance for a particular configuration of
    ResNet-50 trained on ImageNet.
    -   If our goal is to determine whether to include weight decay at all, then
        we would compare the best point from this plot against the baseline of
        no weight decay. For a fair comparison, the baseline should also have
        its learning rate equally well tuned.
-   When we have data generated by (quasi)random search and are considering a
    continuous hyperparameter for an isolation plot, we can approximate the
    isolation plot by bucketing the x-axis values of the basic hyperparameter
    axis plot and taking the best trial in each vertical slice defined by the
    buckets.

</details>

#### Automate generically useful plots

<details><summary><em>[点击展开]</em></summary>


<br>

-   The more effort it is to generate plots, the less likely we are to look at
    them as much as we should, so it behooves us to set up our infrastructure to
    automatically produce as many of them as possible.
-   At a minimum, we automatically generate basic hyperparameter axis plots for
    all hyperparameters that we vary in an experiment.
-   Additionally, we automatically produce training curves for all trials and
    make it as easy as possible to find the best few trials of each study and
    examine their training curves.
-   There are many other potential plots and visualizations we can add that can
    be useful. Although the ones described above are a good starting point, to
    paraphrase Geoffrey Hinton, "Every time you plot something new, you learn
    something new."

</details>

### Determining whether to adopt a training pipeline change or hyperparameter configuration

***概要:*** *When deciding whether to make a change to our model or training
procedure or adopt a new hyperparameter configuration going forward, we need to
be aware of the different sources of variation in our results.*

-   When we are trying to improve our model, we might observe that a particular
    candidate change initially achieves a better validation error compared to
    our incumbent configuration, but find that after repeating the experiment
    there is no consistent advantage. Informally, we can group the most
    important sources of variation that might cause such an inconsistent result
    into the following broad categories:
    -   **Training procedure variance**, **retrain variance**, or **trial
        variance**: the variation we see between training runs that use the same
        hyperparameters, but different random seeds.
        -   For example, different random initializations, training data
            shuffles, dropout masks, patterns of data augmentation operations,
            and orderings of parallel arithmetic operations, are all potential
            sources of trial variance.
    -   **Hyperparameter search variance**, or **study variance**: the variation
        in results caused by our procedure to select the hyperparameters.
        -   For example, we might run the same experiment with a particular
            search space, but with two different seeds for quasi-random search
            and end up selecting different hyperparameter values.
    -   **Data collection and sampling variance**: the variance from any sort of
        random split into training, validation, and test data or variance due to
        the training data generation process more generally.
-   It is all well and good to make comparisons of validation error rates
    estimated on a finite validation set using fastidious statistical tests, but
    often the trial variance alone can produce statistically significant
    differences between two different trained models that use the same
    hyperparameter settings.
-   We are most concerned about study variance when trying to make conclusions
    that go beyond the level of an individual point in hyperparameters space.
    -   The study variance depends on the number of trials and the search space
        and we have seen cases where it is larger than the trial variance as
        well as cases where it is much smaller.
-   Therefore, before adopting a candidate change, consider running the best
    trial N times to characterize the run-to-run trial variance.
    -   Usually, we can get away with only recharacterizing the trial variance
        after major changes to the pipeline, but in some applications we might
        need fresher estimates.
    -   In other applications, characterizing the trial variance is too costly
        to be worth it.
-   At the end of the day, although we only want to adopt changes (including new
    hyperparameter configurations) that produce real improvements, demanding
    complete certainty that something helps isn't the right answer either.
-   Therefore, if a new hyperparameter point (or other change) gets a better
    result than the baseline (taking into account the retrain variance of both
    the new point and the baseline as best we can), then we probably should
    adopt it as the new baseline for future comparisons.
    -   However, we should only adopt changes that produce improvements that
        outweigh any complexity they add.

### After exploration concludes

***概要:*** *Bayesian optimization tools are a compelling option once we’re
done exploring for good search spaces and have decided what hyperparameters even
should be tuned at all.*

-   At some point, our priorities will shift from learning more about the tuning
    problem to producing a single best configuration to launch or otherwise use.
-   At this point, there should be a refined search space that comfortably
    contains the local region around the best observed trial and has been
    adequately sampled.
-   Our exploration work should have revealed the most essential hyperparameters
    to tune (as well as sensible ranges for them) that we can use to construct a
    search space for a final automated tuning study using as large a tuning
    budget as possible.
-   Since we no longer care about maximizing our insight into the tuning
    problem, many of
    [the advantages of quasi-random search](#why-use-quasi-random-search-instead-of-more-sophisticated-black-box-optimization-algorithms-during-the-exploration-phase-of-tuning)
    no longer apply and Bayesian optimization tools should be used to
    automatically find the best hyperparameter configuration.
    -   If the search space contains a non-trivial volume of divergent points
        (points that get NaN training loss or even training loss many standard
        deviations worse than the mean), it is important to use black box
        optimization tools that properly handle trials that diverge (see
        [Bayesian Optimization with Unknown Constraints](https://arxiv.org/abs/1403.5607)
        for an excellent way to deal with this issue).
-   At this point, we should also consider checking the performance on the test
    set.
    -   In principle, we could even fold the validation set into the training
        set and retraining the best configuration found with Bayesian
        optimization. However, this is only appropriate if there won't be future
        launches with this specific workload (e.g. a one-time Kaggle
        competition).

## 确定每次训练的steps数

-   有两种类型的工作负载：受计算限制（compute-bound）的工作负载和不受计算约束的工作负载。
-   训练时**计算受限**是指训练过程是受限于我们愿意等待的时间，而不是受限于我能有多少训练数据或其他因素。
    -   In this case, if we can somehow train longer or more efficiently, we should see a lower training loss and, with proper tuning, an improved validation loss.
    -   In other words, *speeding up* training is equivalent to *improving* training and the "optimal" training time is always "as long as we can afford."
    -   That said, just because a workload is compute-limited doesn't mean training longer/faster is the only way to improve results.
-   When training is **not compute-bound**, we can afford to train as long as we would like to, and, at some point, training longer doesn't help much (or even causes problematic overfitting).
    -   In this case, we should expect to be able to train to very low training loss, to the point where training longer might slightly reduce the training loss, but will not meaningfully reduce the validation loss.
    -   Particularly when training is not compute-bound, a more generous training time budget can make tuning easier, especially when tuning learning rate decay schedules, since they have a particularly strong interaction with the training budget.
        -   In other words, very stingy training time budgets might require a learning rate decay schedule tuned to perfection in order to achieve a good error rate.
-   Regardless of whether a given workload is compute-bound or not, methods that increase the variance of the gradients (across batches) will usually result in slower training progress, and thus may increase the number of training steps required to reach a particular validation loss. High gradient variance can be caused by:
    -   Using a smaller batch size
    -   Adding data augmentation
    -   Adding some types of regularization (e.g. dropout)

### 当训练不受计算限制时如何决定该训练多久

-   我们的主要目标是确保我们训练的时间足够长，以使模型达到最佳效果，同时避免在训练step的数量上过度浪费。
-   有疑问的时候，宁可多训练一点。当训练时间较长时，应确保性能不下降。回顾性(可选)的checkpoin应被正确选择，checkpoint足够频繁。
-   永远不要在研究中调整 `max_train_steps` 。选择一个值并将其用于所有试验。从这些试验中，绘制回顾检查点选择发现的训练steps，以优化`max_train_steps`的选择。
    -   例如，如果最佳step总是出现在训练过程的前10%达到，那么最大训练step数就太高了。
    -   或者，如果最好的step总是出现在训练过程的最后的25%中，我们可能可以在增加训练时间和重新调整学习率衰减策略中受益。
-   当模型架构或数据发生变化时(例如添加数据增强)，理想的训练的step数也会发生变化。
-   下面我们将描述如何根据使用恒定学习率“完全拟合”训练集所需的step数，为`max_train_steps`选择初始候选值。
    -   注意，我们并没有以精确或数学定义良好的方式使用短语“完美拟合训练集”。
        它只是一个非正式的描述语，表示非常低的训练损失。
        -   例如，当训练损失为log loss。没有正则化项时，我们可能会看到训练损失会一直在缓慢减小，直到达到浮点极限（floating point limits），因为网络权值无约束地增长，并且模型在训练集上的预测变得越来越有"自信“。在这种情况下，我们可能会说，当训练集中的错误分类为0是，模型“完全拟合”训练集。
    -   如果训练过程中梯度噪声（gradient noise ）的数量增加，我们发现`max_train_steps`的起始值可能需要增加。
        -   例如，如果在模型中引入数据增强或dropout等正则化方法。
    -   如果训练过程以某种方式改进，可能会减少`max_train_steps`。
        -   例如，使用更好的优化器或更好的学习率更新策略。

#### 使用学习率扫描来为max_train_steps选择初始候选值的算法

<details><summary><em>[点击展开]</em></summary>


<br>

-   This procedure assumes it is possible to not only "perfectly" fit the
    training set, but to do so using a constant learning rate schedule.
-   If it is possible to perfectly fit the entire training set, then there must
    exist a configuration (with some value of `max_train_steps`) that perfectly
    fits the training set; find any such configuration and use its value of
    `max_train_steps` as a starting point `N`.
-   Run a constant learning rate sweep (i.e. grid search the learning rate)
    without data augmentation and without regularization where each trial trains
    for `N` steps.
-   The number of steps required for the fastest trial in the sweep to reach
    perfect training performance is our initial guess for `max_train_steps`.
-   **NOTE:** Bad search spaces can make it possible to engage in
    self-deception.
    -   For example, if all the learning rates in a study are too small, we
        might incorrectly conclude that a very large value of `max_train_steps`
        is necessary.
    -   At a minimum, we should check that the optimal learning rate in the
        study is not at the boundary of the search space.

</details>

### 当训练受计算限制时如何决定该训练多久

-   In some cases, training loss keeps improving indefinitely and our patience
    and computational resources become the limiting factors.
-   If training loss (or even validation loss) keeps improving indefinitely,
    should we always train as long as we can afford? Not necessarily.
    -   We might be able to tune more effectively by running a larger number of
        shorter experiments and reserving the longest "production length" runs
        for the models we hope to launch.
    -   As the training time for trials approaches our patience limit, tuning
        experiments become more relevant for our potential launch candidates,
        but we can complete fewer of them.
    -   There are probably many questions we can answer while only training for
        ~10% of the production length, but there is always a risk that our
        conclusions at this time limit will not apply to experiments at 20% of
        the production length, let alone 100%.
-   Tuning in multiple rounds with increasing, per-trial training step limits is
    a sensible approach.
    -   We can do as many rounds as we want, but usually 1-3 are the most
        practical.
    -   Essentially, try to obtain as much understanding of the problem as
        possible using trials with a very quick turnaround time, trading off
        tuning thoroughness with relevance to the final, longest runs.
    -   Once a given per-trial time limit has generated useful insights, we can
        increase the training time and continue tuning, double-checking our
        conclusions from the shorter runs as needed.
-   As a starting point, we recommend two rounds of tuning:
    -   Round 1: Shorter runs to find good model and optimizer hyperparameters.
    -   Round 2: Very few long runs on good hyperparameter points to get the
        final model.
-   The biggest question going from `Round i` &rarr; `Round i+1` is how to
    adjust learning rate decay schedules.
    -   One common pitfall when adjusting learning rate schedules between rounds
        is using all the extra training steps with too small of a learning rate.

#### Round 1

<details><summary><em>[点击展开]</em></summary>


<br>

-   Unfortunately, there is no guarantee that good hyperparameters found in
    short, incomplete training are still good choices when training length is
    significantly increased. However, for some kinds of hyperparameters, they
    are often correlated enough for Round 1 to be useful.
-   What hyperparameter values found in shorter runs do we expect to transfer to
    longer training runs? For all of this, we need more research. But based on
    what we know so far, here are the authors’ suspicions in order of decreasing
    probability of transferring:
    -   Very likely to transfer
        -   Early training instability can be resolved in the first round of
            tuning using a smaller number of training steps. Perhaps these
            hyperparameters are the closest thing to a sure bet for transfer
            that we have.
            -   Warmup length
            -   Initialization
    -   Likely to transfer
        -   Model architecture - A dramatic win in the model architecture will
            usually transfer, but there are probably many counterexamples.
    -   Might transfer
        -   Optimization algorithm/optimizer hyperparameters - We think this
            would "loosely" transfer. It’s definitely weaker than the things
            above it.
        -   Data augmentation
        -   Regularization
            -   If it isn't possible to perfectly fit the training set, the
                model might be in a regime where regularization is unlikely to
                help very much.
    -   Unlikely to transfer
        -   Learning rate schedule: unlikely to transfer perfectly.
            -   [This paper](https://arxiv.org/abs/2203.15556) suggests that
                even decay schedule transfers, but we don't believe this is true
                in general. Example: Tuning sqrt decay on small # of training
                steps then extending to large # will result in the majority of
                training occurring at overly small steps.
                -   One can likely do "good enough" with most schedules in the
                    limit of extreme training budget, but noticeable performance
                    improvements can likely be seen if it is tuned.
            -   [Understanding Short-Horizon Bias in Stochastic
                Meta-Optimization](https://arxiv.org/abs/1803.02021) describes
                the dangers of trying to pick learning rates myopically.

</details>

#### Round 2

<details><summary><em>[点击展开]</em></summary>


<br>

-   Run the best hyperparameter configuration from Round 1.
-   **(Speculation)** 🤖 Use the extra steps to extend the period of training at a high learning rate.
    -   E.g. if linear schedule then keep the length of the decay fixed from Round 1 and extend the period of constant lr in the beginning.
    -   For cosine decay, just keep the base lr from Round 1 and extend `max_train_steps` as in [Chinchilla paper](https://arxiv.org/abs/2203.15556).
-   More rounds might make sense for teams with very mature modeling and tuning pipelines and very long and expensive production training runs, but they will often be overkill.
    -   We've described how to transfer from Step 1 &rarr; Step 2. If we didn't care about analysis time and if making efficient use of compute was the overriding concern, then the ideal would be to exponentially increase the length of training runs (and thus the end-to-end time to complete a study) over many different rounds of tuning.
        -   At each round we systematically ensure our choices continue to hold up.
        -   New ideas go through a pipeline that progressively derisks them using increasingly long-running experiments from Step i to Step i+1.

</details>

## training pipeline的额外指导

### Optimizing the input pipeline

***概要:*** *The causes and interventions of input-bound pipelines are highly task-dependent; use a profiler and look out for common issues.*

-   Use an appropriate profiler to diagnose input-bound pipelines. For example,[Perfetto](https://jax.readthedocs.io/en/latest/profiling.html) for JAX or [TensorFlow profiler](https://www.tensorflow.org/guide/profiler) for TensorFlow.
-   Ultimately, the specific causes and interventions will be highly task-dependent. Broader engineering considerations (e.g. minimizing disk footprint) may warrant worse input pipeline performance.
-   Common causes:
    -   Data are not colocated with the training process, causing I/O latency (this might happen when reading training data over a network).
    -   Expensive online data preprocessing (consider doing this once offline
        and saving).
    -   Unintentional synchronization barriers that interfere with data pipeline
        prefetching. For example, when synchronizing metrics between the device
        and host in CommonLoopUtils
        ([link](https://github.com/google/CommonLoopUtils/blob/fea2518ada8814a78e1492023fd9f00edb0b0568/clu/metrics.py#L291)).
-   Common tips:
    -   Instrument input pipeline to prefetch examples (e.g.[tf.data.Dataset.prefetch](https://www.tensorflow.org/guide/data_performance#prefetching))
    -   Remove unused features/metadata from each as early in the pipeline as possible.
    -   Increase the replication of the number of jobs generating examples for the input pipeline. For example, by using the
        [tf.data service](https://www.tensorflow.org/api_docs/python/tf/data/experimental/service).

### Evaluating model performance

***概要:*** *Run evaluation at larger batch sizes than training. Run evaluations at regular step intervals, not regular time intervals.*

#### Evaluation settings

<details><summary><em>[点击展开]</em></summary>


<br>

-   There are several settings in which we can evaluate the performance of our
    models.
    -   **Online evaluation** - metrics are collected when the model is serving
        predictions in a production environment.
    -   **Offline evaluation** - metrics are collected when the model is run on
        offline train/validation/test sets that are representative of the
        production environment.
    -   **Periodic evaluations** - metrics are collected during model training
        that might either be a proxy for the offline evaluation, and/or on a
        subset of the data used in offline evaluation.
-   Online evaluation is the gold standard, but is often impractical during the
    model development phase.
-   Depending on the problem, offline evaluation can be fairly involved and
    computationally expensive.
-   Periodic evaluations are the most practical and economical choice, but may
    not fully represent the production environment.
    -   Our goal during periodic evaluation is to use an expedient proxy of the
        offline evaluation, without sacrificing the reliability of the signal we
        get during training.

</details>

#### Setting up periodic evaluations

<details><summary><em>[点击展开]</em></summary>


<br>

-   We run periodic evaluations during training to monitor its progress in real
    time, to
    [facilitate retrospective model checkpoint selection](#saving-checkpoints-and-retrospectively-selecting-the-best-checkpoint),
    and so that we can
    [examine the training curves at the end of training](#examining-the-training-curves).
-   The simplest configuration is to perform both training and periodic
    evaluations within the same compute instance, periodically alternating
    between training and evaluation.
    -   In this case, the batch size used to perform evaluations should be *at
        least* as large as the batch size used for training because model
        activations don't need to be maintained during evaluation, lowering the
        computational requirements per example.
-   Periodic evaluations should be done at regular step intervals, not time
    intervals.
    -   Evaluating based on time intervals can make it harder to interpret the
        training curves, especially when training may suffer from preemptions of
        the training jobs, network latency issues, etc.
-   Periodicity in valid/test metrics (when using a shuffled
    train/validation/test split) can indicate implementation bugs such as test
    data having overlap with training data, or training data not being properly
    shuffled. Evaluating at regular step intervals can make these issues easier
    to catch.
-   Partial batches can occur when the evaluation sets are not divisible by the
    batch size. Ensure that the padded examples are correctly weighed to prevent
    the loss function from being biased by them. Often, these padded examples
    can be given a weight of zero.
-   Save sufficient information per evaluation to support offline analysis.
    Ideally, we would save predictions on a selection of individual examples
    since they can be invaluable for debugging.
    -   Generating artifacts like
        [SavedModels](https://www.tensorflow.org/guide/saved_model) make it easy
        to do ad-hoc model inspection after evaluation jobs finish.

</details>

#### Choosing a sample for periodic evaluation

<details><summary><em>[点击展开]</em></summary>


<br>

-   The periodic evaluation job might not run fast enough to compute metrics on
    the full offline evaluation set in a reasonable amount of time. This often
    necessitates sampling data for periodic evaluation.
-   We consider the following factors when constructing a sampled dataset:
    -   <ins>Sample size</ins>
        -   Check that the performance computed on the sampled dataset used by
            the periodic job matches the performance on the whole offline
            evaluation set, i.e. there is no skew between the sampled set and
            the full dataset.
        -   The dataset used for periodic evaluation should be small enough that
            it’s easy to generate model predictions over its entirety, but large
            enough that improvements to the model can be accurately measured
            (i.e. not overwhelmed by label noise).
        -   It should be large enough to accommodate multiple such evaluations
            across trials in sequence, and still produce accurate estimates.
            That is, to avoid adaptively "fitting" to the validation set over
            time, in a way that doesn't generalize to a held-out test set.
            However, this consideration is rarely a practical concern.
    -   <ins>Imbalanced datasets</ins>
        -   For imbalanced datasets, performance on rare classes of examples
            will often be noisy.
        -   For datasets with a small number of examples in a class label, log
            the number of examples predicted correctly to get more insight into
            accuracy improvements (.05 sensitivity improvement sounds exciting,
            but was it just one more example correct?).

</details>

### Saving checkpoints and retrospectively selecting the best checkpoint

***概要:*** *Run training for a fixed number of steps and retrospectively
choose the best checkpoint from the run.*

-   Most deep learning frameworks support
    [model checkpointing](https://flax.readthedocs.io/en/latest/api_reference/flax.training.html).
    That is, the current state of the model is periodically preserved on disk.
    This allows the training job to be resilient to compute instance
    interruptions.
-   The best checkpoint is often not the last checkpoint, particularly when the
    validation set performance does not continue to increase over time but
    rather fluctuates about a particular value.
-   Set up the pipeline to keep track of the N best checkpoints seen so far during training. At the end of training, model selection is then a matter of choosing the best checkpoint seen during training. We call this **retrospective optimal checkpoint selection**.
-   Supporting prospective early stopping is usually not necessary, since we’re
    pre-specifying a trial budget and are preserving the N best checkpoints seen
    so far.

### Setting up experiment tracking

***概要:*** *When tracking different experiments, make sure to note a number
of essentials like the best performance of a checkpoint in the study, and a
short description of the study.*

-   We've found that keeping track of experiment results in a spreadsheet has
    been helpful for the sorts of modeling problems we've worked on. It often
    has the following columns:
    -   Study name
    -   A link to wherever the config for the study is stored.
    -   Notes or a short description of the study.
    -   Number of trials run
    -   Performance on the validation set of the best checkpoint in the study.
    -   Specific reproduction commands or notes on what unsubmitted changes were
        necessary to launch training.
-   Find a tracking system that captures at least the information listed above
    and is convenient for the people doing it. Untracked experiments might as
    well not exist.

### Batch normalization implementation details

***概要:*** *Nowadays batch norm can often be replaced with LayerNorm, but in
cases where it cannot, there are tricky details when changing the batch size or
number of hosts.*

-   Batch norm normalizes activations using their mean and variance over the
    current batch, but in the multi-device setting these statistics are
    different on each device unless explicitly synchronized.
-   Anecdotal reports (mostly on ImageNet) say calculating these normalizing
    statistics using only ~64 examples actually works better in practice (see
    Ghost Batch Norm from [this paper](https://arxiv.org/abs/1705.08741)).
-   Decoupling the total batch size and the number of examples used to calculate
    batch norm statistics is particularly useful for batch size comparisons.
-   Ghost batch norm implementations do not always correctly handle the case
    where the per-device batch size > virtual batch size. In this case we'd
    actually need to subsample the batch on each device in order to get the
    proper number of batch norm statistic examples.
-   Exponential moving averages used in test mode batch norm are just a linear
    combination of training statistics, so these EMAs only need to be
    synchronized before saving them in checkpoints. However, some common
    implementations of batch norm do not synchronize these EMAs and only save
    the EMA from the first device.

### Considerations for multi-host pipelines

***概要:*** *for logging, evals, RNGs, checkpointing, and data sharding,
multi-host training can make it very easy to introduce bugs!*

-   Ensure the pipeline is only logging and checkpointing on one host.
-   Make sure before evaluation or checkpointing is run, the batch norm
    statistics are synchronized across hosts.
-   It is critical to have RNG seeds that are the same across hosts (for model
    initialization), and seeds that are different across hosts (for data
    shuffling/preprocessing), so make sure to mark them appropriately.
-   Sharding data files across hosts is usually recommended for improved
    performance.

## 常见问题解答

### What is the best learning rate decay schedule family?

<details><summary><em>[点击展开]</em></summary>


<br>

-   It’s an open problem. It’s not clear how to construct a set of rigorous
    experiments to confidently answer what the "best" LR decay schedule is.
-   Although we don't know the best schedule family, we're confident that it’s
    important to have some (non-constant) schedule and that tuning it matters.
-   Different learning rates work best at different times during the
    optimization process. Having some sort of schedule makes it more likely for
    the model to hit a good learning rate.

</details>

### Which learning rate decay should I use as a default?

<details><summary><em>[点击展开]</em></summary>
<br>


-   Our preference is either linear decay or cosine decay, and a bunch of other
    schedule families are probably good too.

</details>

### Why do some papers have complicated learning rate schedules?

<details><summary><em>[点击展开]</em></summary>
<br>


-   It’s not uncommon to see papers with complicated piecewise learning rate
    (LR) decay schedules.
-   Readers often wonder how the authors arrived at such a complicated study.
-   Many complicated LR decay schedules are the result of tuning the schedule as
    a function of the validation set performance in an ad hoc way:
    1.  Start a single training run with some simple LR decay (or a constant
        learning rate).
    2.  Keep training running until the performance seems to stagnate. If this
        happens, pause training. Resume it with a perhaps steeper LR decay
        schedule (or smaller constant learning rate) from this point. Repeat
        this process until the conference/launch deadline.
-   Blithely copying the resulting *schedule* is generally not a good idea since
    the best particular schedule will be sensitive to a host of other
    hyperparameter choices.
    -   Better to copy the *algorithm* that produced the schedule, although this
        is rarely possible when arbitrary human judgment produced the schedule.
-   This type of validation-error-sensitive schedule is fine to use if it can be
    fully automated, but human-in-the-loop schedules that are a function of
    validation error are brittle and not easily reproducible, so we recommend
    avoiding them.
    -   Before publishing results that used such a schedule, please try to make
        it fully reproducible.

</details>

### How should Adam’s hyperparameters be tuned?

<details><summary><em>[点击展开]</em></summary>
<br>


-   As discussed above, making general statements about search spaces and how
    many points one should sample from the search space is very difficult. Note
    that not all the hyperparameters in Adam are equally important. The
    following rules of thumb correspond to different "budgets" for the number of
    trials in a study.
    -   If < 10 trials in a study, only tune the (base) learning rate.
    -   If 10-25 trials, tune learning rate and $\beta_1$.
    -   If 25+ trials, tune the learning rate, $\beta_1$ and $\epsilon$.
    -   If one can run substantially more than 25 trials, additionally tune
        $\beta_2$.

</details>

### Why use quasi-random search instead of more sophisticated black box optimization algorithms during the exploration phase of tuning?

<details><summary><em>[点击展开]</em></summary>


-   Quasi-random search (based on
    [low-discrepancy sequences](https://en.wikipedia.org/wiki/Low-discrepancy_sequence))
    is our preference over fancier black box optimization tools when used as
    part of an iterative tuning process intended to maximize insight into the
    tuning problem (what we refer to as the "exploration phase"). Bayesian
    optimization and similar tools are more appropriate for the exploitation
    phase.
-   Quasi-random search based on randomly shifted low-discrepancy sequences can
    be thought of as "jittered, shuffled grid search", since it uniformly, but
    randomly, explores a given search space and spreads out the search points
    more than random search.
-   The advantages of quasi-random search over more sophisticated black box
    optimization tools (e.g. Bayesian optimization, evolutionary algorithms)
    include:
    1.  Sampling the search space non-adaptively makes it possible to change the
        tuning objective in post hoc analysis without rerunning experiments.
        -   For example, we usually want to find the best trial in terms of
            validation error achieved at any point in training. But the
            non-adaptive nature of quasi-random search makes it possible to find
            the best trial based on final validation error, training error, or
            some alternative evaluation metric without rerunning any
            experiments.
    2.  Quasi-random search behaves in a consistent and statistically
        reproducible way.
        -   It should be possible to reproduce a study from six months ago even
            if the implementation of the search algorithm changes, as long as it
            maintains the same uniformity properties. If using sophisticated
            Bayesian optimization software, the implementation might change in
            an important way between versions, making it much harder to
            reproduce an old search. It isn’t always possible to roll back to an
            old implementation (e.g. if the optimization tool is run as a
            service).
    3.  Its uniform exploration of the search space makes it easier to reason
        about the results and what they might suggest about the search space.
        -   For example, if the best point in the traversal of quasi-random
            search is at the boundary of the search space, this is a good (but
            not foolproof) signal that the search space bounds should be
            changed. [This section](#identifying-bad-search-space-boundaries)
            goes into more depth. However, an adaptive black box optimization
            algorithm might have neglected the middle of the search space
            because of some unlucky early trials even if it happens to contain
            equally good points, since it is this exact sort of non-uniformity
            that a good optimization algorithm needs to employ to speed up the
            search.
    4.  Running different numbers of trials in parallel versus sequentially will
        not produce statistically different results when using quasi-random
        search (or other non-adaptive search algorithms), unlike with adaptive
        algorithms.
    5.  More sophisticated search algorithms may not always handle infeasible
        points correctly, especially if they aren't designed with neural network
        hyperparameter tuning in mind.
    6.  Quasi-random search is simple and works especially well when many tuning
        trials will be running in parallel.
        -   Anecdotally[^3], it is very hard for an adaptive algorithm to beat a
            quasi-random search that has 2X its budget, especially when many
            trials need to be run in parallel (and thus there are very few
            chances to make use of previous trial results when launching new
            trials).
        -   Without expertise in Bayesian optimization and other advanced black
            box optimization methods, we might not achieve the benefits they
            are, in principle, capable of providing. It is hard to benchmark
            advanced black box optimization algorithms in realistic deep
            learning tuning conditions. They are a very active area of current
            research, and the more sophisticated algorithms come with their own
            pitfalls for inexperienced users. Experts in these methods are able
            to get good results, but in high-parallelism conditions the search
            space and budget tend to matter a lot more.
-   That said, if our computational resources only allow a small number of
    trials to run in parallel and we can afford to run many trials in sequence,
    Bayesian optimization becomes much more attractive despite making our tuning
    results harder to interpret.

[^3]: Ben Recht and Kevin Jamieson

    [pointed out](http://www.argmin.net/2016/06/20/hypertuning/) how strong
    2X-budget random search is as a baseline (the
    [Hyperband paper](https://jmlr.org/papers/volume18/16-558/16-558.pdf)
    makes similar arguments), but it is certainly possible to find search
    spaces and problems where state-of-the-art Bayesian optimization
    techniques crush random search that has 2X the budget. However, in our
    experience beating 2X-budget random search gets much harder in the
    high-parallelism regime since Bayesian optimization has no opportunity to
    observe the results of previous trials.

</details>

### Where can I find an implementation of quasi-random search?

<details><summary><em>[点击展开]</em></summary>
<br>


-   We use
    [this implementation](https://github.com/mlcommons/algorithmic-efficiency/blob/main/algorithmic_efficiency/halton.py)
    that generates a Halton sequence for a given search space (intended to
    implement a shifted, scrambled Halton sequence as recommended in
    https://arxiv.org/abs/1706.03200).
-   If a quasi-random search algorithm based on a low-discrepancy sequence is
    not available, it is possible to substitute pseudo random uniform search
    instead, although this is likely to be slightly less efficient.
    -   In 1-2 dimensions, grid search is also acceptable, although not in
        higher dimensions (see
        [Bergstra & Bengio, 2012](https://www.jmlr.org/papers/v13/bergstra12a.html)).

</details>

### How many trials are needed to get good results with quasi-random search?

<details><summary><em>[点击展开]</em></summary>
<br>


<p align="center">
<img src="assets/have_we_sampled_enough.png" width="49%" alt="A box plot showing the importance of sampling enough">
</p>


<p align="center"><b>Figure 3:</b> A ResNet-50 was tuned on ImageNet with 100
trials. Via bootstrapping, different amounts of tuning budget were simulated.
Box plots of the best performances for each trial budget are plotted above.


-   There is no way to answer this question in general, but we can look at
    specific examples.
-   As the Figure 3 shows, the number of trials in a study can have a
    substantial impact on the results.
    -   Notice how large the interquartile ranges are when 6 trials were
        sampled, versus when 20 trials were sampled.
    -   Even with 20 trials, it is likely that the difference between especially
        lucky and unlucky studies will be larger than the typical variation
        between re-trains of this model on different random seeds, with fixed
        hyperparameters, which for this workload might be around +/- 0.1% on a
        validation error rate of \~23%.

</details>

### How can optimization failures be debugged and mitigated?

<details><summary><em>[点击展开]</em></summary>
<br>



***概要:*** *If the model is experiencing optimization difficulties, it’s
important to fix them before trying other things. Diagnosing and correcting
training failures is an active area of research.*

<p align="center">
<img src="assets/stride_instability.png" width="80%" alt="Changing the strides in a single residual block in a WideResnet results in training instability.">
</p>



<p align="center"><b>Figure 4:</b> Changing the strides in a single residual block (2x2 -> 1x1) in a WideResnet results in training instability. This does not degrade performance at low learning rates, but high learning rates no longer train well due to the instability. Applying 1000 steps of learning rate warmup resolves this particular instance of instability, allowing stable training at max learning rate of .1.</p>

#### Identifying unstable workloads

-   Any workload will become unstable if the learning rate is too large.
    Instability is only an issue when it forces us to use a learning rate that’s
    too small.
-   There are at least two types of training instability worth distinguishing:
    1.  Instability at initialization/early in training.
    2.  Sudden instability in the middle of training.
-   We can take a systematic approach to identifying stability issues in our
    workload.
    1.  Do a learning rate sweep and find the best learning rate lr*.
    2.  Plot training loss curves for learning rates just above lr*.
    3.  If the learning rates > lr* show loss instability (loss goes up not down
        during periods of training), then it is likely that fixing the
        instability will result in better training.
-   Log the L2 norm of the full loss gradient during training, outlier values
    can result in spurious instability in the middle of training. This can
    inform how to pick gradient/update clipping.

**NOTE:** Some models show very early instability followed by a recovery that
results in slow but stable training. **Common evaluation schedules can miss
these issues by not evaluating frequently enough!**

To check for this, we can train for an abbreviated run of just \~500 steps using
`lr = 2 * current best`, but evaluate every step.

<p align="center">
<img src="assets/more_frequent_evals.png" width="80%" alt="Illustration of the value of more frequent evaluations at the start of
training.">
</p>


<p align="center"><b>Figure 5:</b> Illustration of the value of more frequent evaluations at the start of training. Useful if there’s a suspicion that the model suffers from early training instability.</p>

#### Potential fixes for common instability patterns

-   Apply learning rate warmup
    -   Best for early training instability.
-   Apply gradient clipping
    -   Good for both early and mid training instability, may fix some bad inits
        that warmup cannot.
-   Try a new optimizer
    -   Sometimes Adam can handle instabilities that Momentum can’t. This is an
        active area of research.
-   We can ensure that we’re using best practices/initializations for our model
    architecture (examples below).
    -   Add residual connections and normalization if the model doesn't contain
        it already.
-   Normalization should be the last operation before the residual. E.g. x +
    Norm(f(x)).
-   Norm(x + f(x)) known to cause issues.
-   Try initializing residual branches to 0 (e.g.
    [ReZero init](https://arxiv.org/abs/2003.04887)).
-   Lower the learning rate
    -   This is a last resort.

#### Learning rate warmup

<p align="center">
<img src="assets/instability_during_warmup.png" width="80%" alt="An example of instability during a warmup period (note the horizontal axis log
scale).">
</p>


<p align="center"><b>Figure 6:</b> An example of instability during a warmup period (note the horizontal axis log scale). 40k steps of warmup was needed for successful training in this case.</p>

##### When to apply learning rate warmup

<p align="center">
<img src="assets/axis_model_with_instability.png" width="49%" alt="Axis plot for model with instability">
</p>


<p align="center"><b>Figure 7a:</b> An example of a hyperparameter axis plot for a model exhibiting training instability. The best learning rate is at the edge of what is feasible. An "infeasible" trial is defined as one that either produces NaNs or uncharacteristically high values of the loss.</p>

<p align="center">
<img src="assets/loss_model_with_instability.png" width="49%" alt="Loss curve for model with instability">
</p>


<p align="center"><b>Figure 7b:</b> The training loss of a model trained with a learning rate where we see instability.</p>

-   Figure 7a shows a hyperparameter axis plot that indicates a model
    experiencing optimization instabilities, because the best learning rate is
    right at the edge of instability.
-   Figure 7b shows how this can be double-checked by examining the training
    loss of a model trained with a learning rate either 5x or 10x larger than
    this peak. If that plot shows a sudden rise in the loss after a steady
    decline (e.g. at step \~10k in the figure above), then the model likely
    suffers from optimization instability.

##### How to apply learning rate warmup

<p align="center">
<img src="assets/beneficial_effect_warmup.png" width="80%" alt="Beneficial effect of warmup on training instabilities">
</p>


<p align="center"><b>Figure 8:</b> Beneficial effect of learning rate warmup on addressing training instabilities.</p>

-   Using the section immediately above, we assume that the practitioner has
    already identified the learning rate at which the model becomes unstable.
    This is the `unstable_base_learning_rate`.
-   Warmup involves prepending a learning rate schedule that ramps up the
    learning rate from 0 to some stable `base_learning_rate`, that is at least
    one order of magnitude larger than `unstable_base_learning_rate`. The
    default would be to try a `base_learning_rate` that’s 10x
    `unstable_base_learning_rate`. Although note that it’d be possible to run
    this entire procedure again for something like 100x
    `unstable_base_learning_rate`. The specific schedule is:
    -   Ramp up from 0 to `base_learning_rate` over `warmup_steps`.
    -   Train at a constant rate for `post_warmup_steps`.
-   Our goal is to find the shortest number of `warmup_steps` that allows us to
    access peak learning rates that are much higher than
    `unstable_base_learning_rate`.
-   So for each `base_learning_rate`, we need to tune `warmup_steps` and
    `post_warmup_steps`. It’s usually fine to set `post_warmup_steps` to be
    `2*warmup_steps`.
-   Warmup can be tuned independently of an existing decay schedule.
    `warmup_steps` should be swept at a few different orders of magnitude. For
    example, an example study could try [10, 10<sup>3</sup>, 10<sup>4</sup>,
    10<sup>5</sup>]. The largest feasible point shouldn't be more than 10% of
    `max_train_steps`.
-   Once a `warmup_steps` that doesn't blow up training at `base_learning_rate`
    has been established, it should be applied to the baseline model.
    Essentially, we prepend this schedule onto the existing schedule, and use
    the optimal checkpoint selection discussed above to compare this experiment
    to the baseline. For example, if we originally had 10,000 `max_train_steps`
    and did `warmup_steps` for 1000 steps, the new training procedure should run
    for 11,000 steps total.
-   If long `warmup_steps` are required for stable training (>5% of
    `max_train_steps`), `max_train_steps` may need to be increased to account
    for this.
-   There isn't really a "typical" value across the full range of workloads.
    Some models only need 100 steps, while others (particularly transformers)
    may need 40k+.

#### Gradient clipping

<p align="center">
<img src="assets/gradient_clipping.png" width="80%" alt="Gradient clipping on early training instabilities">
</p>


<p align="center"><b>Figure 9:</b> Illustration of gradient clipping correcting early training instability.</p>

-   Gradient clipping is most useful when large or outlier gradient issues
    occur.
-   Clipping can fix either early training instability (large gradient norm
    early), or mid training instabilities (sudden gradient spikes mid training).
-   Sometimes longer warmup periods can correct instabilities that clipping does
    not: see [this section above](#How-to-apply-learning-rate-warmup).
    -   🤖 What about clipping during warmup?
-   The ideal clip thresholds are just above the "typical" gradient norm.
-   Here’s an example of how gradient clipping could be done:
    -   If the norm of the gradient $\left | g \right |$ is greater than the
        gradient clipping threshold $\lambda$, then do ${g}'= \lambda \times \frac{g}{\left | g \right |}$ where ${g}'$ is the new gradient.
-   Log the unclipped gradient norm during training. By default, generate:
    -   A plot of gradient norm vs step
    -   A histogram of gradient norms aggregated over all steps
-   Choose a gradient clipping threshold based on the 90th percentile of
    gradient norms.
    -   The threshold will be workload dependent, but 90% is a good starting
        point. If it doesn't work, this threshold can be tuned.
    -   🤖 What about some sort of adaptive strategy?
-   If we try gradient clipping and the instability issues remain, we can try it
    harder (i.e. make the threshold smaller).
-   Extremely aggressive gradient clipping is in essence a strange way of
    reducing the learning rate. If we find ourselves using extremely aggressive
    clipping, we probably should just cut the learning rate instead.
-   We would usually consider having >50% of the updates getting clipped somehow
    as "extremely aggressive".
-   If we need to do extremely aggressive gradient clipping to deal with our
    instability issues, then we might as well reduce the learning rate.

</details>

### Why do you call the learning rate and other optimization parameters hyperparameters? They are not parameters of any prior distribution.

<details><summary><em>[点击展开]</em></summary>
<br>


-   It is true that the term "hyperparameter" has a precise
    [meaning](https://en.wikipedia.org/wiki/Hyperparameter) in Bayesian machine
    learning and referring to the learning rate and most of the other parameters
    we tune in deep learning as "hyperparameters" is an abuse of terminology.
-   We would prefer to use the term "metaparameter" for learning rates,
    architectural parameters, and all the other things we tune in deep learning,
    since it avoids the potential for confusion that comes from misusing the
    word "hyperparameter" (confusion that is especially likely when discussing
    Bayesian optimization where the probabilistic response surface models have
    their own true hyperparameters).
-   Unfortunately, although potentially confusing, the term hyperparameter has become
    extremely common in the deep learning community.
-   Therefore, for a document, such as this one, intended for a wide audience
    that includes many people who are unlikely to be aware of this technicality,
    we made the choice to contribute to one source of confusion in the
    field in hopes of avoiding another.
-   That said, we might make a different choice when publishing a research
    paper, and we would encourage others to use "metaparameter" instead in most
    contexts.

</details>

### Why shouldn't the batch size be tuned to directly improve validation set performance?

<details><summary><em>[点击展开]</em></summary>
<br>


-   Changing the batch size *without changing any other details of the training pipeline* will often affect the validation set performance.
-   However, the difference in validation set performance between two batch sizes typically goes away if the training pipeline is optimized independently for each batch size.
-   The hyperparameters that interact most strongly with the batch size, and therefore are most important to tune separately for each batch size, are the optimizer hyperparameters (e.g. learning rate, momentum) and the regularization hyperparameters.
    - Smaller batch sizes introduce more noise into the training algorithm due to sample variance, and this noise can have a regularizing effect. Thus, larger batch sizes can be more prone to overfitting and may require stronger regularization and/or additional regularization techniques.
-   In addition, [the number of training steps may need to be adjusted](#choosing-the-batch-size-to-minimize-training-time) when changing the batch size.
-   Once all these effects are taken into account, there is currently no convincing evidence that the batch size affects the maximum achievable validation performance (see [Shallue et al. 2018](https://arxiv.org/abs/1811.03600)).

</details>

### 所有流行的优化算法的更新规则是什么？

<details><summary><em>[点击展开]</em></summary>
<br>


#### Stochastic gradient descent (SGD)

$$\theta_{t+1} = \theta_{t} - \eta_t \nabla \mathcal{l}(\theta_t)$$

#### Momentum

$$v_0 = 0$$

$$v_{t+1} = \gamma v_{t} + \nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - \eta_t v_{t+1}$$

#### Nesterov

$$v_0 = 0$$

$$v_{t+1} = \gamma v_{t} + \nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - \eta_t( \gamma v_{t+1} + \nabla \mathcal{l}(\theta_{t})$$

#### RMSProp

$$v_0 = 1 \text{,} m_0 = 0$$

$$v_{t+1} = \rho v_{t} + (1 - \rho) \nabla \mathcal{l}(\theta_t)^2$$

$$m_{t+1} = \gamma m_{t} + \frac{\eta_t}{\sqrt{v_{t+1} + \epsilon}}\nabla \mathcal{l}(\theta_t)$$

$$\theta_{t+1} = \theta_{t} - m_{t+1}$$

#### ADAM

$$m_0 = 0 \text{,} v_0 = 0$$

$$m_{t+1} = \beta_1 m_{t} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)$$

$$v_{t+1} = \beta_2 v_{t} + (1 - \beta_2) \nabla \mathcal{l}(\theta_t)^2$$

$$b_{t+1} = \frac{\sqrt{1 - \beta_2^{t+1}}}{1 - \beta_1^{t+1}}$$

$$\theta_{t+1} = \theta_{t} - \alpha_t \frac{m_{t+1}}{\sqrt{v_{t+1}} + \epsilon} b_{t+1}$$

#### NADAM

$$m_0 = 0 \text{,} v_0 = 0$$

$$m_{t+1} = \beta_1 m_{t} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)$$

$$v_{t+1} = \beta_2 v_{t} + (1 - \beta_2) \nabla \mathcal{l} (\theta_t)^2$$

$$b_{t+1} = \frac{\sqrt{1 - \beta_2^{t+1}}}{1 - \beta_1^{t+1}}$$

$$\theta_{t+1} = \theta_{t} - \alpha_t \frac{\beta_1 m_{t+1} + (1 - \beta_1) \nabla \mathcal{l} (\theta_t)}{\sqrt{v_{t+1}} + \epsilon} b_{t+1}$$

</details>

## 致谢

-   我们要感谢Max Bileschi，Roy Frostig，Zelda Mariet，Stan Bileschi，Mohammad Norouzi，Chris Dubois和Charles Sutton阅读了手稿并提供了宝贵的反馈。
-   我们使用了了一些最初由Naman Agarwal生产的其他联合研究机构的实验数据。
-   我们要感谢Will Chen在文档的介绍中的宝贵建议。
-   我们还要感谢Rohan Anil的有帮助的讨论。

## 引用

```
@misc{tuningplaybookgithub,
  author = {Varun Godbole and George E. Dahl and Justin Gilmer and Christopher J. Shallue and Zachary Nado},
  title = {Deep Learning Tuning Playbook},
  url = {http://github.com/google/tuning_playbook},
  year = {2023},
  note = {Version 1.0}
}
```

## 贡献

-   This is not an officially supported Google product.

-   We'd love to hear your feedback!

    -   If you like the playbook, please [leave a star](https://docs.github.com/en/get-started/exploring-projects-on-github/saving-repositories-with-stars#starring-a-repository)! Or email
        deep-learning-tuning-playbook \[at\] googlegroups.com. Testimonials help
        us justify creating more resources like this.
    -   If anything seems incorrect, please file an issue to start a discussion.
        For questions or other messages where an issue isn't appropriate, please
        open a new discussion topic on GitHub.

-   As discussed in the preamble, this is a living document. We anticipate
    making periodic improvements, both small and large. If you’d like to be
    notified, please watch our repository (see [instructions](https://docs.github.com/en/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications#configuring-your-watch-settings-for-an-individual-repository)).

-   Please don't file a pull request without first coordinating with the authors
    via the issue tracking system.

### 贡献者许可协议

Contributions to this project must be accompanied by a Contributor License
Agreement (CLA). You (or your employer) retain the copyright to your
contribution; this simply gives us permission to use and redistribute your
contributions as part of the project. Head over to
<https://cla.developers.google.com/> to see your current agreements on file or
to sign a new one.

You generally only need to submit a CLA once, so if you've already submitted one
(even if it was for a different project), you probably don't need to do it
again.

### Code Reviews

All submissions, including submissions by project members, require review. We
use GitHub pull requests for this purpose. Consult
[GitHub Help](https://help.github.com/articles/about-pull-requests/) for more
information on using pull requests.

### Community Guidelines

This project follows
[Google's Open Source Community Guidelines](https://opensource.google/conduct/).
