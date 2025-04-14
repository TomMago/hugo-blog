---
title: "Quarkonium Suppression Poster"
layout: "single"
type: "page"
robots: "noindex, nofollow"
url: "/QuarkoniumSuppressionPoster"
---

Hey! Looks like you checked out my poster.

First and foremost here are some links to the relevant papers:

- Original derivation of the master equation from pNRQCD: [1711.04515](https://arxiv.org/pdf/1711.04515), [1612.07248](https://arxiv.org/pdf/1612.07248)
- Derivation of the Lindblad Equation at Next to Leading order in $E/(\pi T)$: [2205.10289](https://arxiv.org/abs/2205.10289)
- Phenomenological results: [2302.11826](https://arxiv.org/abs/2302.11826) [2403.15545](https://arxiv.org/abs/2403.15545)

Before going into more detail here are some simulation results for the evolution of the Bottomonium in the QGP for a time of $2$fm$/c$.

<style>
  .top-row {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 10px;
    flex-wrap: nowrap; /* prevent wrapping */
  }

  .top-row img {
    max-width: 100%;
    height: auto;
    flex-grow: 1; /* allow image to scale down to fit */
    max-width: 100px; /* limit max width of image */
  }

  .top-label {
    min-width: 50px;
    text-align: center;
    white-space: nowrap; /* prevent label from breaking */
  }

  .gif-row {
    display: flex;
    justify-content: center;
    align-items: center;
    gap: 20px;
    flex-wrap: wrap; /* allow bottom gifs to stack on small screens */
    margin-top: 40px;
  }

  .gif-container {
    display: flex;
    justify-content: center;
    align-items: center;
    flex: 1 1 200px;
    height: 200px;
  }

  .gif-row img {
    max-width: 100%;
    height: auto;
  }

  .full-container{
    text-align: center; 
    height:450px;
  }

  @media (max-width: 500px) {
    .top-row {
      gap: 5px;
    }
    .top-label {
      font-size: 14px;
    }
    .top-row img {
      max-width: 40%; /* let image scale down further */
    }
    .topimg{
        width: 40%;
    }
    .gif-container {
        display: flex;
        justify-content: center;
        align-items: center;
        flex: 1 1 300px;
        height: 300px;
        margin-bottom:50px;
    }
    .full-container{
        text-align: center; 
        height:850px;
    }     

  }
</style>

<div class="full-container">

  <!-- Top: labels + gif (stay in one line even on mobile) -->
  <div class="top-row">
    <div class="top-label">$t=0$ fm$/c$</div>
    <img src="../time_progress.gif" alt="Time Progress" style="max-width: 300px;" class="topimg">
    <div class="top-label">$t=2$ fm$/c$</div>
  </div>

  <!-- Bottom: two gifs that wrap on mobile -->
  <div class="gif-row">
    <div class="gif-container">
      <img src="../position_space.gif" alt="Position Space">
    </div>
    <div class="gif-container">
      <img src="../angular_momentum.gif" alt="Angular Momentum">
    </div>
  </div>

</div>

The first plot shows the density matrix evolution of the relative radial coordinate of the quarkonium. Here we pick as an initial state a very  localized gaussian, since the production of the bottomonium is very local. The second plot shows the evolution of the angular momentum distribution. We start with an $S$-wave bottomonium ($l=0$), and see that over time the average angular momentum increases and the probability of $S$-wave state decreases. This is exactly the dissociation of the quarkonium.

<style>
.math-responsive {
  font-size: 1em;
}

@media (max-width: 768px) {
  .math-responsive {
    font-size: 0.8em;
  }
}
</style>

Now let me give some more details about the simulation. 

## The Lindblad equation


The Lindblad equation
<span class="math-responsive">
$$\frac{d}{dt}\rho(t)=-i[H,\rho(t)]+\sum_n C^n_i \rho(t)C^{n\dagger}_i - \frac{1}{2}\\{C^{n\dagger}_iC^n_i,\rho(t)\\}$$
</span>

at next to leading order in $E/(\pi T)$ is given by the Hamiltonian
<span class="math-responsive">
$$H=
\begin{pmatrix} 
h_s+ h_T & 0 \\\\
0 & h_o+\frac{N^2_c-2}{2(N^2_c-1)}h_T
\end{pmatrix},$$
</span> 
with 
<span class="math-responsive">
$$h_T = \frac{r^2}{2}\gamma +\frac{\kappa}{4MT}\\{r_i,p_i\\},$$
</span>
and the vacuum hamiltonian
<span class="math-responsive">
$$h_{s(o)}=\frac{p^2}{M}+V_{s(o)}(r).$$
</span>
The $2\times 2$ matrix reflects the structure in color space, i.e. color singlet and color octet. The potentails $V_{s(o)}(r)$ can be derived from pNRQCD. In pNRQCD they are matching coefficients, which can be calculated systematically from full QCD. 

The two collapse operators are given by 

<span class="math-responsive">
$$
C_i^0 = \sqrt{\frac{\kappa}{N^2_c-1}}\begin{pmatrix}
        0 & K_i+\frac{\Delta V_{os}}{4T}r_i\\
        \sqrt{N^2_c-1}\left(K_i+\frac{\Delta V_{so}}{4T}r_i\right) & 0
    \end{pmatrix},
$$
$$
C_i^1 = \sqrt{\frac{\kappa(N^2_c - 4)}{2(N^2_c-1)}}\begin{pmatrix}
        0 & 0\\
        0 & K_i\end{pmatrix},
$$
</span>
with 
<span class="math-responsive">
$$K_i = r_i+\frac{ip_i}{2MT},$$
</span>
Here we sum over the three dimensions $i=x,y,z$.

The coefficients $\kappa$ and $\gamma$ are called transport coefficients and they are are property of the medium. They have a field theoretic definition in terms of correlations functions of the chromo-electric field. As such they are non-perturbative objects and inherintly hard to calculate.

Now to solve the equation in three dimensions, the whole equation is projected on spherical harmonics. In this way  the denstiy matrix $\rho(\vec{r},\vec{r}^\prime)$ becomes $\rho(r,l,m,r^\prime,l^\prime,m^\prime)$. Now assuming that our problem is spherically summetric, the density matrix is diagonal in $l$ and $m$ and also does not depend on $m$. There we end up with $\rho_l(r,r^\prime)$. Furthermore $\rho$ also depends on the color $c$. 

With this setup we can solve the Lindblad equation using the quantum trajectories method and obtain the plots you saw above. In the simulation we assume that the temperature evolution is quasi static and during the solution at every timestep we insert the repsective temperature value. This is neccessary since the QGP cools down and we have a temperature evolution depending on time $T(t)$. In the simulation above we used a simple model for the temperature called  Bjorken evolution . In phenomenological applications however we resert to hydrodynamics simulations of the QGP.

## Nuclear Modification Factor

To connect to phenomenology we calculate the Nuclear Modification Factor $R_{AA}$. Essentially this quantity gives the ratio of Bottomonia measure in the heavy ion collission compared to proton-proton collissions. The idea here is, that in proton-proton collsions, due to the lower densities, there is no QGP, and therefore, if we measure less Bottomonia in the heavy ion collisions it must be due to the effect o the QGP. We calculate the $R_{AA}$ from our simulation results by first calculating the Bottomonium wave functions as eigenfunctions $\ket{i}$ of the hamiltonian
<span class="math-responsive"> 
$$h_s=\frac{p^2}{M}+V_s(r)+\frac{l(l+1)}{Mr^2}.$$
</span>
We then project the density matrix $\rho(t)$ given by the Lindblad solution on these eigenfunction and interpret the ratio
<span class="math-responsive">
$$p_{1S}(t) = \frac{\braket{1S|\rho(t)|1S}}{\braket{1S|\rho(0)|1S}}$$
</span>
as the survival probability of a state, in this case the $1S$ Bottomonium. The suppression would then by obtained from this survival probability. However, we have to take into account that, after surviving the QGP, higher exited states like the $3S$ or $2P$ might decay into lower states, effectively altering the number of Bottomonia in a specific state. We account for this by introducting an _exited state feed down_. The feed down is applied using a matrix $F$, which consists of the branching fractions betwen the different states. This matrix acts on a vector or quarkonium states
<span class="math-responsive">
$$\vec{\sigma} = \\{\Upsilon(1S), \Upsilon(2S), \chi_{b0}(1P), \chi_{b1}(1P), \chi_{b2}(1P),\allowbreak\Upsilon(3S)$$
$$,\chi_{b0}(2P),\allowbreak\chi_{b1}(2P),\chi_{b2}(2P)\\}.$$
</span>
Using the feeddown matrix we can effectively convert between an  experimental cross section $\vec{\sigma}\_\mathrm{exp}$ which is the measurement in the detector, and the direct production cross section $\vec{\sigma}\_\mathrm{direct}$ which is the cross section before taking into account the feed down of the exited states. The relation between the two is 
<span class="math-responsive">
$$\vec{\sigma}\_\mathrm{exp}=F\cdot\vec{\sigma}\_\mathrm{direct}.$$
</span>
Including the feeddown we can calculate the Nuclar Modification Factor as 
<span class="math-responsive">
$$ R^i\_{AA} = \frac{(F\cdot S\cdot\vec{\sigma}\_\text{direct})^i}{\sigma^i\_\text{exp}}.$$
</span>
Here the $S$ is a matrix containing the survival probabilities $p_i$ for the different states on its diagonal. Note that here we take the survival probabilities at the time at which the QGP hadronizes.
This equation amounts to 
1. Taking the direct production cross section from proton-proton collision
2. Applying the dissociation in the QGP through the survival probabilities in $S$
3. Applying the feed down on the way to the detector after the QGP through $F$

Note that in practice we calculate the direct production cross section from the experimental ones by
<span class="math-responsive">
$$\vec{\sigma}\_\mathrm{direct}=F^{-1}\cdot\vec{\sigma}\_\mathrm{exp}.$$
</span> 
In this way we can calculate the $R_{AA}$ for different states directly from the simulation results of the Lindblad equation for $\rho(t)$. 
![adsf](../RAA.png)
In this plot we compare the Nuclear Modification Factor against experimental data. On the $x$-axis we have the number of participaing partons. They denote how much the colliding nuclei overlap during the collisions. The $R_{AA}$ depends on the $N_\mathrm{part}$ because if more partons participate the event will have a higher total energy and the plasma will be hotter. A hotter plasma consequently leads to more suppression. In our simulations we include this by solving the Lindblad equation for diferent temperature profiles, which were all obtained from hydrodynamics simulations for different $N_\mathrm{part}$.
