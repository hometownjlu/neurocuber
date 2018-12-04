# NeuroCuber

NeuroCuber: training NeuroSAT to make cubing decisions for hard SAT problems.

## Overview

In previous work, we showed that the neural network architecture _NeuroSAT_ can learn to solve SAT problems even when trained only as a classifier to predict satisfiability (see [paper](https://openreview.net/forum?id=HJMC_iA5tm) and [code](https://github.com/dselsam/neurosat) for more details).
We found this result fascinating, but as we discuss at the end of the paper, it is not clear if this finding will lead to improvements to state-of-the-art (SOTA) SAT solvers.

The main reason for pessimism is that CDCL solvers are already extremely fast and reliable on many classes of important SAT problems. Moreover, many of the problems that are hard for CDCL are so large that they would not even fit in GPU memory. In this work, we do not even try to compete with CDCL solvers on the problems that they excel at, nor do we try to solve the problems that cause CDCL solvers strife due to their size. Instead, we restrict our focus to the (not as common) classes of SAT problems that have been shown to benefit from expensive, hard-engineered branching heuristics, and that are nonetheless moderately sized.

Specifically, we consider the [cube-and-conquer](https://www.cs.utexas.edu/~marijn/publications/cube.pdf) (CnC) paradigm, whereby a "cuber" puts a lot of effort into choosing the first several variables to cube (i.e. branch) on, and a CDCL solver "conquers" the subproblems produced by the cuber. This approach is relatively new and is SOTA on many classes of very hard SAT problems. As an example, this approach was recently used to solve the long-open [Boolean Pythagorean Triples Problem](https://www.nature.com/news/two-hundred-terabyte-maths-proof-is-largest-ever-1.19990). Note that the goal for CnC is to conquer _all_ subproblems, i.e. to either prove _unsat_ or to find all satisfying assignments.

In this work, we investigate whether we can use the NeuroSAT architecture to learn a better cuber.

## Measuring progress

### Baseline cubers

As always, there are choices to be made in determining appropriate baselines to compare to. SOTA cubers such as [march](https://github.com/marijnheule/CnC) work by selecting some subset of variables that seem promising and then looking ahead in the search tree to see how much the problems simplify when these variables are set. During the lookahead, they also perform many sophisticated extensions beyond simply selecting the next variable to cube on. For example, they collect failed literals, and also (locally) learn new clauses that are only entailed given the current trail. If NeuroCuber chose variables to cube on without performing such a lookahead, we would lose these benefits. Ultimately we may seek a hybrid approach, whereby NeuroCuber tells a traditional lookahead-based cuber which variables to bother looking ahead for, and also how to evaluate the resulting simplified states. However, as a starting point, we only compare NeuroCuber to the baseline cubers based on the quality of their cubing decisions as opposed to how good of a solver one can build around the core cuber. Specifically, we compare to the implementation of _march_cu_ in [z3](https://github.com/Z3Prover/z3).

### Estimating the size of the cube tree

SOTA cubers work by producing an incremental-CNF (iCNF) file, which includes the original problem along with a sequence of cubes (i.e. conjunctions of literals) whose disjunction is valid. The conquerer is an incremental CDCL solver that (in the single-threaded case) proceeds by pushing the next cube, closing the branch, and then popping the cube. Since conflict clauses can persist across cubes, the time it takes the conquerer to solve all the cubes is not a simple function of the time it takes it to solve each of the cubes individually from scratch. However, we want to be able to estimate NeuroSAT's ability efficiently without needing to solve entire SAT problems, especially during early stages of the project. To do so, we make the following simplifications:

1. We sample paths through the cube tree independently, and construct an unbiased estimator of the number of total number of nodes in the cube tree. In 1975, Knuth introduced [a simple, unbiased estimator for the size of a search tree](https://pdfs.semanticscholar.org/94ce/5bdf77af8693df0d525010850ab6faf7e290.pdf). For binary trees such as ours, this method reduces to averaging the exponentiated lengths of random paths in the tree. Yet as Knuth discusses, this method may have extremely high variance. The solution he suggests and that we adopt in this work is to use importance sampling. Specifically, in addition to training NeuroSAT to make cubing decisions, we train it to predict which branch is the harder one, and use these predictions for importance sampling.

2. We fix a (non-incremental) conquerer, and continue to add literals to a cube until the conquerer can solve it. We use z3's CDCL solver for this, and include hyperparameters for the maximum number of conflicts and restarts per solve attempt.

### SAT problems

We consider three different regimes of train and test problems (in decreasing order of generality):

1. Train on a lot of SAT problems and test on a lot of unseen SAT problems. We could do this by collecting all SAT problems in competitions that CnC solvers were the best at, and training upto a particular year and testing on the following years.

2. Find a distribution of similar SAT problems, train on some of them and test on the rest. In this regime, we need to be careful that the cost of training can be amortized, i.e. that there are enough challenging test problems left that it is worth the cost of training.

3. Find one SAT problem that is so challenging that the cost of training from scratch can be amortized, and then train and test on it. One candidate is the [Schur Number Five problem](https://arxiv.org/abs/1711.08076), which takes rough 14 CPU years to solve using SOTA.