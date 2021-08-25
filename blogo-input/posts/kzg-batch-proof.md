## Batch proof in KZG Commitments
*2021-08-14*

> **Warning**: I want to state clearly that I'm not a mathematician, I'm just an amateur on math studying in my free time, and this article is just an attempt to try to sort the notes that I took while reading about the KZG Commitments.

Last week I posted some *[notes on KZG Commitmens](https://arnaucube.com/blog/kzg-commitments.html)*, which overviews the scheme for single proofs. The current notes, try to overview the *batch proof* iteraton on KZG Commitments, and the *vector commitment* usage of it. Again (as in the previous post), big thanks to [Dankrad Feist](https://dankradfeist.de/ethereum/2020/06/16/kate-polynomial-commitments.html) and [Alin Tomescu](https://alinush.github.io/2020/05/06/kzg-polynomial-commitments.html) for their articles which helped me to follow a bit the reasoing behind this, and again, I recommend spending the time reading their articles instead of this current notes.


#### Batch proof
The benefit of *batch proof* is that allows us to proof multiple points while the proof size remains constant to one $\mathbb{G}_1$ point.
Let $(z_0, y_0), (z_1, y_1), ..., (z_k, y_k)$ be the points that we want to proof, and that we have a polynomial $p(x)$ that goes through the points.
The *commitment* to the polynomial stands the same than for single proofs: $c=[p(\tau)]_1$.

For the evaluation proof, while in the single proofs we compute $q(x) = \frac{p(x)-y}{x-z}$, we will replace $y$ and $x-z$ by the following two polynomials.
The constant $y$ is replaced by a polynomial that has roots at all the points that we want to prove. This is achieved by computing the [Lagrange interpolation](https://en.wikipedia.org/wiki/Lagrange_polynomial) for the given set of points:

$$
I(x) = \sum_{j=0}^k y_j l_j(x)\newline
where \space\space\space l_j(x) = \prod\_{0\leq m \leq k} \frac{x-x_m}{x_j - x_m}
$$

And the $x-z$, which was to ensure that $q(x)$ had a root at $z$, now, as we want to ensure that $q(x)$ has roots at all the points of the commitment, we will use the *zero polynomial*:
$$
Z(x) = \prod_{i=0}^{k} x-z_i =\newline
=(x-z_0)(x-z_1)...(x-z_k)
$$

This polynomial ensures that when $x=z_i$ ($z_i$ being one of our points), the polynomial evaluation will be zero.

Now we can put $I(x)$ and $Z(x)$ in place, obtaining $q(x)=\frac{p(x)-I(x)}{Z(x)}$. And the batch proof evaluation is obtained by $\pi=[q(\tau)]_1$.

The verification is quite similar than what we did for single proofs, but using the mentioned $z(x)$ and $I(x)$:
$$
\hat{e}(\pi, [Z(\tau)]_2) == \hat{e}(c - [I(\tau)]_1, H)
$$

Which, as we did with the single proofs in the previous post, we can unroll it and see that:
$$
\hat{e}(\pi, [Z(\tau)]_2) == \hat{e}(c - [I(\tau)]_1, H)\newline
\Rightarrow \hat{e}([q(\tau)]_1, [Z(\tau)]_2) == \hat{e}([p(\tau)]_1 - [I(\tau)]_1, H)\newline
\Rightarrow [q(\tau) \cdot Z(\tau)]_T == [p(\tau) - I(\tau)]_T
$$
From where we see that is the equation $q(x)\cdot Z(x)=p(x)-I(x)$, which can be expressed as $q(x) = \frac{p(x) - I(x)}{Z(x)}$, evaluated at $\tau$ from the trusted setup, which is not known: $q(\tau) = \frac{p(\tau) - I(\tau)}{Z(\tau)}$.

#### Vector commitments
As mentioned earlier, this scheme can be used as a *vector commitment* scheme.

A vector commitment allows a prover to commit to a vector and later proof that a certain value belongs to that vector. As a traditional example, we can think of *Merkle Trees*, where we can commit to a vector of values, which are placed in the tree leafs. Then we compute the *root* which acts as the commitment. Then we can provide a proof that a certain leaf belongs to the vector for the commitment (*root*) that we showed earlier.
The problem with Merkle Trees is that the proof size grows linearly with the size of the tree, as it contains the siblings from the leaf to the root. Here is where *KZG Commitments* can be benefitial, as the proof always stands with the same size, one $\mathbb{G}_1$ point, no matter of how many points we are batching.

We can use KZG Commitments as a vector commitment scheme by mapping the *vector* as a batch of points that build the polynomial, so when commiting to the polynomial we are commiting to the vector. Then, it's a matter of using the *batch proof* approach explained above in order to proof multiple elements of the vector in a single proof that can be verified later.

#### Final note
I'm fascinated by this scheme and its potential. One next rabbit hole (related to KZG Commitments) to look at, would be the [Plonk](https://vitalik.ca/general/2019/09/22/plonk.html) and other similar constructions. Also, another use case that is getting attention in the Ethereum community is the [*Verkle Trees*](https://vitalik.ca/general/2021/06/18/verkle.html).

As in the previous notes, in order to try to put in practice the concepts, I've added the *batch proof* logic at https://github.com/arnaucube/kzg-commitments-study.
