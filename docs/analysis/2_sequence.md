# Sequences

## Limits of Sequences

A *sequence* is a function whose domain is a set of the form
$\{  n \in \mathbb{Z} : n \ge m \}$; $m$ is usually 1 or 0.

The *limit* of a sequence $(s_n)$ is a real number that the values
$s_n$ are close to for latge values of $n$.

> Definition 7.1
A sequence $(s_n)$ of real numbers is said to **converge** to the real number
$s$ provided that for each $\epsilon > 0$ there exists a number $N$ such that
$n>1$ implies $\mid s_{n} - s < \epsilon \mid$.

If $(s_n)$ *converges* to $s$, we will write $lim_{n \rightarrow \infin} s_n = s$, or $s_n \rightarrow s$. The number $s$ is called the limit of the sequence $(s_n)$. A sequence that does not converge to some real number is said to *diverge*.

**Limits are unique.**

## A Discustion about Proofs

* Prove $lim \frac{1}{n^2} = 0$.