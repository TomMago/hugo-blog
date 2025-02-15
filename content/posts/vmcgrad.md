---
title: "Gradient estimator in Variational Monte Carlo"
date: 2025-02-12
tags: ["VMC"]
author: "Tom Magorsch"
ShowToc: true
bibFile: content/posts/bib_lit.json
math: true
---

I recently tried to derive the formula for the gradient estimator in Variational Monte Carlo and got confused trying to obtain the common covariance formula $\partial_\Theta\braket{E} = 2\text{Re}\text{Cov}[(\partial_\Theta\log\psi)^*, E_l]$ in the case of complex wavefunctions. At first, to me it seemed like the direct derivative of the estimator was missing (which vanishes for real wavefunctions). Indeed most derivations in the literature seem to assume real wavefunctions, however it turns out this direct part gives a similar term included in the score based part which eventually lets us simplify the estimator to the real part of the covariance formula. Since I found the full derivation non-trivial I will give it here in case anyone else out there is confused as well.

## The setup

Assume we have some Hamiltonian $H$ and a variational wavefunction $\ket{\psi\_\Theta}$ (I will leave out the explicit dependence of $\psi$ on $\Theta$ in the following), where $\Theta$ are some parameters, which I assume to be real. We wish to minimize the energy 
$$E = \frac{\braket{\psi|H|\psi}}{\braket{\psi|\psi}}$$
by optimizing our parameters. To do this efficiently for a large Hilbertspace, we estimate the energy stochastically as 
$$\braket{E}=E\_\chi\left[\frac{H\psi}{\psi}\right],$$
where we sample basis states from the distribution of our wavefunction $$\chi(s) = |\psi(s)|^2/\int dk |\psi(k)|^2.$$
The expectation value can therefore be written as 
$$E\_\chi\left[\frac{H\psi}{\psi}\right] = \frac{1}{\int dk |\psi(k)|^2}\int ds |\psi(s)|^2\frac{H\psi(s)}{\psi(s)}.$$
We denote $E\_l(s)=H\psi(s)/\psi(s)$ as the *local* estimator.

## The gradient estimator

To optimize the parameters of our variational ansatz we need to estimate the gradient of the energy $\partial\_\Theta\braket{E}$.
To do this, one essentially uses a *score based gradient estimator*. We can derive the estimator by pulling the derivative into the expectation value
$$\partial\_\Theta\braket{E} = \int ds \partial\_\Theta (\chi(s)E\_l(s)) = \int ds\left( \chi(s)(\partial\_\Theta\log\chi(s))E\_l(s) + \chi(s)(\partial\_\Theta E\_l(s))\right)$$

In the equality we rewrote the derivative into the score of $\chi$ times $\chi$. In this way we can rewrite the expression as an expectation value over $\chi$ itself:
$$\partial_\Theta\braket{E}=E_\chi[(\partial_\Theta\log\chi) E_l + \partial_\Theta E_l].$$
This is also referred to score based gradient estimator or originally REINFORCE in reinforcement learning.
I will usually refer to the first term in the estimator as "score based" and the second as "direct term". 

Lets look at the two terms seperately.

## Score based term

To calculate the score based term explicitly I will "undo" score based rewriting so we can explicit apply the derivative to our wavefunction.
$$E_\chi[(\partial_\Theta\log\chi)E_l]=\int ds \left(\partial_\Theta \frac{|\psi(s)|^2}{\int dk |\psi(k)|^2}\right)E_l.$$
From the derivative we get two terms, one for the nominator one for the denominator
$$=\frac{1}{\int dk|\psi(k)|^2}\int ds \left[(\partial_\Theta\psi(s)) \psi^* (s)
    + \psi (s)\partial_\Theta\psi^* (s) \right]\frac{H\psi(s)}{\psi(s)}\qquad\qquad\qquad\quad$$
$$ \qquad - \frac{1}{\left[\int dk |\psi(k)|^2\right]^2}\int ds |\psi(s)|^2 \frac{H\psi(s)}{\psi(s)} \int dk \left[(\partial_\Theta\psi (k))\psi^* (k) + \psi (k) (\partial_\Theta\psi^* (k))\right]$$
Now in both terms we can simplify
$$\int ds \psi(s)\partial_\Theta\psi^* (s) = \int ds |\psi(s)|^2\frac{\partial_\Theta\psi^\* (s)}{\psi^\*(s)} = \int ds |\psi(s)|^2 (\partial_\Theta \log\psi(s))^*$$

and similarly for its complex conjugate and since $z^* + z = 2\text{Re}(z)$ we get

$$E_\chi[(\partial_\Theta\log\chi)E_l]=\frac{1}{\int dk|\psi(k)|^2}\int ds |\psi(s)|^2 2\text{Re}(\partial_\Theta\log\psi(s))\frac{H\psi(s)}{\psi(s)}\qquad\qquad\qquad\quad$$

$$\qquad\qquad - \frac{1}{\left[\int dk |\psi(k)|^2\right]^2}\int ds |\psi(s)|^2 \frac{H\psi(s)}{\psi(s)} \int dk |\psi(k)|^2 2\text{Re}(\partial_\Theta\log\psi(k)).$$

We can now again identify expectation values over $\chi$ and get

$$E_\chi[(\partial_\Theta\log\chi)E_l] = E_\chi[2\text{Re}(\partial_\Theta\log\psi)E_l] - E_\chi[E_l]E_\chi[2\text{Re}(\partial_\Theta\log\psi)].$$

Therefore we see, that the score based term alone already gives us a formula in covariance form

$$E_\chi[(\partial_\Theta\log\chi)E_l] = \text{Cov}[2\text{Re}(\partial_\Theta\log\psi), E_l]$$

In the case of a real $\psi$, also $E_l$ will be real, so then we get the common form 

$$E_\chi[(\partial_\Theta\log\chi)E_l] = 2\text{Re} \text{Cov}[\partial_\Theta\log\psi, E_l],$$

where $F=\text{Cov}[\partial_\Theta\log\psi, E_l]$  is usually called the force.
Note that if $\psi$ is not real, we cant just pull the $\text{Re}$ outside since $\text{Re}(z\cdot w)\neq\text{Re}(z)\cdot\text{Re}(w)$.
This is an important difference between the expression for the score based term and the common covariance formula from the literature.

## Direct term

Now lets come to the direct term $E_\chi[\partial_\Theta E_l]$. In many derivations it seems like this term is just skipped or in some cases even calculated wrongly. I think the confusion arises because in the case of real values $\psi$ the term is zero and many people therefore just dont mention it at all, even though to me it was often not clear that they implicitly assume a real $\psi$. 

In the following I will suppress the explicit evaluation of $\psi$ at $s$ and just denote $\psi\equiv\psi(s)$ as it should be clear from the context.

So lets have a look: Applying the derivative to the local estimator we get two terms:
$$E_\chi[\partial_\Theta E_l] = \frac{1}{\int |\psi|^2}\int |\psi|^2 \left[\frac{H\partial_\Theta\psi}{\psi} - \frac{H\psi}{\psi^2}\partial_\Theta\psi\right],$$
here the second term can again be rewritten using the score
$$\frac{1}{\int |\psi|^2}\int |\psi|^2 \frac{H\psi}{\psi^2}\partial_\Theta\psi=E_\chi[(\partial_\Theta\log\psi)E_l].$$

The first term on the other hand can be rewritten by shifting the hamiltonian:

$$\frac{1}{\int |\psi|^2}\int |\psi|^2 \frac{H\partial_\Theta\psi}{\psi}=\frac{1}{\int |\psi|^2}\int \psi^* \left[H\partial_\Theta\psi\right]=\frac{1}{\int |\psi|^2}\int (H\psi)^* \partial_\Theta\psi.$$
Here the last equality can be seen by writing it more explicitly e.g. 
$$\int ds \braket{s|\psi}^* \braket{s|H|\partial_\Theta\psi}=\braket{\psi|H|\partial_\Theta\psi}=\int ds\braket{s|H^\dagger|\psi}^*\braket{s|\partial_\Theta\psi},$$
where we used $\int ds \ket{s}\bra{s}=1$. To obtain the above expression we then have to use hermicity $H^\dagger = H$. This is important, since if you might be interested in a non-hermitian operator things will be different.

Coming back to out term we can now rewrite 
$$\frac{1}{\int |\psi|^2}\int (H\psi)^* \partial_\Theta\psi=\frac{1}{\int |\psi|^2}\int |\psi|^2\left(\frac{H\psi}{\psi}\right)^* \frac{\partial_\Theta\psi}{\psi}=E_\chi[(\partial_\Theta\log\psi)E^*_l].$$

Putting it all together we have

$$E_\chi[\partial_\Theta E_l]=E_\chi[(\partial_\Theta\log\psi)(E^\*_l-E_l)]=E\_\chi[(\partial\_\Theta\log\psi)(-2i\text{Im}(E_l))].$$

So we can already see that in the case of real $\psi$ this is actually zero. However for complex valued $\psi$ we have to consider it


## Full gradient estimator

Now putting together the score based and direct terms we obtain

$$\partial_\Theta\braket{E} = E_\chi[2\text{Re}(\partial_\Theta\log\psi)E_l]+E\_\chi[(\partial\_\Theta\log\psi)(-2i\text{Im}(E_l))] - E_\chi[E_l]E_\chi[2\text{Re}(\partial_\Theta\log\psi)].$$

And now we can see how the direct term simplifies together with the score based term by using that for complex numbers $z$ and $w$ we have $\text{Re}(z)w -iz\text{Im}(w)=\text{Re}(z^*\cdot w)$ 

$$\partial_\Theta\braket{E} = E_\chi[2\text{Re}\left((\partial_\Theta\log\psi)^*E_l\right)] - E_\chi[E_l]E_\chi[2\text{Re}(\partial_\Theta\log\psi)].$$

This expression we can finally rewrite into the well known covariance formula everyone is using (noting that $E_\chi[E_l]$ is always real)

$$\partial\_\Theta\braket{E}=2\text{Re}[\text{Cov}((\partial\_\Theta\log\psi)^\*, E\_l)].$$

So there you have it, everything comes out correctly, you just have to take into account the additional term correctly for complex valued wavefunctions. In the case of complex valued parameters $\Theta$, things look a bit different, but you still get a covariance in the end. 
