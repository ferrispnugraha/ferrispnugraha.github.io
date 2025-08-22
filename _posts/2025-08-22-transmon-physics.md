---
layout: post
title:  Fundamental transmon physics
date:   2025-08-22 21:00:00
description: Cheat sheet style
tags: physics superconducting
categories: reading
---

Since transmon qubit is a non-linear LC circuit, we expect some deviation of the Hamiltonian from the linear LC oscillator, as seen in Fig. 1. [1]

<figure style="text-align: center;">
  <img src="/assets/img/LC_linear_nonlinear.png" 
       alt="A diagram showing the difference between linear and non-linear LC circuits." 
       style="width: 50%; max-width: 500px;">
  <figcaption style="font-size: 0.9em; ">
    Fig. 1. Linear LC circuit vs nonlinear LC circuit implemented by transmon, image from [2].
  </figcaption>
</figure>

Given that the superconducting phase is $$\phi$$. Since Josephson Junction has the current $$I = I_c \sin \phi$$ while the potential difference is $$V = \frac{\Phi_0}{2\pi} \frac{d\phi}{dt}$$ (here, $$\Phi_0 = \frac{h}{2e}$$ is the magnetic flux quantum), it can be shown that Josephson Junction can be modeled as a non-linear inductor, where

\begin{equation}
L_{\text{eff}} = \frac{\Phi_0}{2\pi I_c} \frac{1}{\cos \phi}
\end{equation}

Hence, the effective transmon energy can be expressed as

\begin{equation}
\hat{H} = 4 E_C (\hat{n} - n_g)^2 - E_J \cos \hat{\phi}
\end{equation}

where $$E_J = \frac{\Phi_0 I_c}{2\pi}$$ giving the energy landscape as shown in Fig. 1. [1]

Rearranging the Hamiltonian in Eq. (2) using the ladder operator $$\hat{b}$$, we obtain

\begin{equation}
\hat{H} = \hbar \omega_q \left( \hat{b}^\dagger \hat{b} + \frac{1}{2} \right) + \frac{\alpha}{2} \hat{b}^\dagger \hat{b}^\dagger \hat{b} \hat{b}
\end{equation}

The second term is not found in linear LC resonator, hence giving $$\alpha = -E_C = -\frac{e^2}{2C_q}$$ the term non-linearity or anharmonicity.

Obtaining the eigenenergies yields:

\begin{equation}
E_m \approx -E_J + \sqrt{8 E_C E_J} \left(m + \frac{1}{2} \right) - \frac{E_C}{12} (6m^2 + 6m + 3)
\end{equation}

Simple calculations from Eq. (4) will provide:

{% raw %}
$$
\begin{align*}
\omega_{01} &= E_1 - E_0 &= \sqrt{8 E_C E_J} - E_C &= \omega_p + \alpha \\
\omega_{12} &= E_2 - E_1 &= \sqrt{8 E_C E_J} - 2E_C &= \omega_{01} + \alpha \\
\omega_{23} &= E_3 - E_2 &= \sqrt{8 E_C E_J} - 3E_C &= \omega_{01} + 2\alpha \\
\omega_{n, n+1} &= E_{n+1} - E_n &= \sqrt{8 E_C E_J} - n E_C &= \omega_{01} + n\alpha
\end{align*}
$$
{% endraw %}

One may also show that $$E_{0m} = m \omega_{01} + \frac{\alpha}{2} m(m-1)$$

The number of states in the transmon is limited by the energy barrier $$2 E_J$$. Hence, we can approximate that

\begin{equation}
N \approx \frac{2 E_J}{\hbar \omega_{01}}
\end{equation}

For $$E_J = 13.6GHz$$ and $$\omega_{01} = 5GHz$$, we obtain $$N \approx 5$$ levels [3].

# References 
[1] J Koch et al, Charge-insensitive qubit design derived from the Cooper pair box, Physical Review Applied 2007. \\
[2] P Krantz et al, A quantum engineer's guide to superconducting qubits, Applied Physics Reviews 6, 021318 (2019) \\
[3] Z Wang et al, High-EJ/EC transmon qudits with up to 12 levels, Physical Review Applied 2025.