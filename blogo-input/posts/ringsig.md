# bLSAG ring signatures overview
*2022-07-20*

> Note: I’m not a mathematician, I’m just an amateur on math. These notes are just an attempt to try to sort the notes that I took while learning abut bLSAG.

<br>
bLSAG: Back's Linkable Spontaneous Anonymous Group signatures

- signer ambiguity
- linkability
- unforgeability

### Setup
Let $G$ be the generator of an EC group.
We use a hash function $\mathcal{H}_p$, which maps to curve points in EC, and a normal hash $\mathcal{H}_n$, which maps to $\mathbb{Z}_p$.
Signer's key pair: $k_{\pi}$, s.t. $K_{\pi} = k_{\pi} \cdot G \in \mathcal{R}$, with secret index $\pi$.
Set of Public Keys: $\mathcal{R} = \{ K_1, K_2, \ldots, K_n \}$

```python
def new_key():
    k = F.random_element()
    K = g * k # g is the generator of the EC group
    return K
```

### Signature
1. compute key image: $\tilde{K} = k_{\pi} \mathcal{H_p} ( K_{\pi}) \in G$
    ```python
    key_image = k * hashToPoint(K)
    ```

2. Generate $\alpha \in^R \mathbb{Z}_p$, and $r_i \in^R \mathbb{Z}_p$, for $i \in \{1, 2, \ldots, n \}$, with $i \neq \pi$
    - $r_i$ is used for the fake responses
    ```python
        a = F.random_element()
        r = [None] * len(R)
        for i in range(0, len(R)):
            if i==pi:
                continue

            r[i] = mod(F.random_element(), p)
    ```

3. Compute $c_{\pi + 1} = \mathcal{H}_n ( m, [\alpha G], [\alpha \mathcal{H}_p(K_{\pi})])$

    ```python
    c[pi1] = hash(R, m, a * g, hashToPoint(R[pi]) * a, p)
    ```

4. for $i=\pi + 1, \pi +2, \ldots, n, 1, 2, \ldots, \pi -1$, calculate, replacing $n+1 \rightarrow 1$
    $$
    c_{i+1} = \mathcal{H}_n (m, [r_i G + c_i K_i], [r_i \mathcal{H}_p (K_i) + c_i \tilde{K}])
    $$
    - Notice that (from step 3 & 4):<br>
    $\alpha \mathcal{H}_p (K_{\pi}) = r_{\pi} \mathcal{H}_p (K_{\pi}) + c_{\pi} \cdot (\tilde{K})$,<br>
    where $\tilde{K}= k_{\pi} \mathcal{H_p} ( K_{\pi})$, so:<br>
    $\alpha \mathcal{H}_p (K_{\pi}) = r_{\pi} \mathcal{H}_p (K_{\pi}) + c_{\pi} \cdot (k_{\pi} \mathcal{H}_p(K_{\pi}))$<br>
    which is equal to,<br>
    $\alpha \cdot \mathcal{H}_p (K_{\pi}) = (r_{\pi} + c_{\pi} \cdot k_{\pi}) \cdot \mathcal{H}_p(K_{\pi})$<br>
    From where we can see: $\alpha = r_{\pi} + c_{\pi} \cdot k_{\pi}$<br>
    which we can rearrange to
    $r_{\pi} = \alpha - c_{\pi} \cdot k_{\pi}$.<br><br>
    
    ```python
    for j in range(0, len(R)-1):
        i = mod(pi1+j, len(R))
        i1 = mod(pi1+j +1, len(R))

        c[i1] = hash(R, m, r[i] * g + c[i] * R[i],
                r[i] * hashToPoint(R[i]) + c[i] * key_image, p)
    ```
    
6. Define $r_{\pi} = \alpha - c_{\pi} k_{\pi} \mod{p}$

```python
r[pi] = mod(a - c[pi] * k, p)
```

Signature: $\sigma(m) = (c_1, r_1, \ldots, r_n)$, with key image $\tilde{K}$ and ring $\mathcal{R}$.
- $len(\sigma(m)) = 1+n$

```python
return [c[0], r]
```


<br><br><br>

#### Step by step (simplified):

<div style="overflow:auto;">
<div style="width: 60%; float:left; height: 360px; overflow-y:scroll;">
    
<img src="img/posts/ring-sig/step00.png" style="width:100%;" />
<img src="img/posts/ring-sig/step00.png" style="width:100%;" />
<img src="img/posts/ring-sig/step01.png" style="width:100%;" />
<img src="img/posts/ring-sig/step02.png" style="width:100%;" />
<img src="img/posts/ring-sig/step03.png" style="width:100%;" />
<img src="img/posts/ring-sig/step04.png" style="width:100%;" />
<img src="img/posts/ring-sig/step05.png" style="width:100%;" />
<img src="img/posts/ring-sig/step06.png" style="width:100%;" />
    
</div>


<div style="width: 40%; float:right; margin-top:80px;">
    
    <ul>
<li>Generate $r_i \in^R \mathbb{Z_p}$</li>
<li>Compute $c_{i+1}$ from $r_i$</li>
<li>Link $r_{\pi}$ with $c_{\pi}$</li>
</ul>

</div>
</div>

*You can scroll down the images through the step-by-step diagrams.*

<br>

It reminds in some way to the approach to close a box like the one in the picture:
![](img//posts/ring-sig/box-closed.png)


<br><br><br>
### Verification
1. check $p \tilde{K} \stackrel{?}{=} 0$
    - to ensure that $\tilde{K} \in G$ (and not in a cofactor group of $G$)
2. for $i = 1, 2, \ldots, n$, replacing $n+1 \rightarrow 1$
    $$
    c'_{i+1} = \mathcal{H}_n (m, [r_i G + c_i K_i], [r_i \mathcal{H}_p (K_i) + c_i \tilde{K}])
    $$
3. check $c_1 \stackrel{?}{=} c'_i$
    ```python
    c[0] = c1
    for j in range(0, len(R)):
        i = mod(j, len(R))
        i1 = mod(j+1, len(R))
        c[i1] = hash(R, m, r[i] * g + c[i] * R[i],
                     r[i] * hashToPoint(R[i]) + c[i] * key_image, p)

    assert c1 == c[0]
    ```

<br><br>

## Links
Toy implementation:

- Sage: https://github.com/arnaucube/math/blob/master/ring-signatures.sage
- Rust: https://github.com/arnaucube/ring-signatures-rs

Resources:

- *"Zero to Monero"* - https://web.getmonero.org/library/Zero-to-Monero-2-0-0.pdf
(section *"3.4 Back’s Linkable Spontaneous Anonymous Group (bLSAG) signatures"*)









