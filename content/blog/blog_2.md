Title: #002: Different forms and normalizations for spherical harmonics 
Date: 2024-01-29
Summary: 
Tags: Science
Slug: 002

<script src="https://polyfill.io/v3/polyfill.min.js?features=es6"></script>
<script id="MathJax-script" async src="https://cdn.jsdelivr.net/npm/mathjax@3/es5/tex-mml-chtml.js"></script>

Spherical harmonics are a very useful mathematical tool that have utility across many discplines. Different academic fields have different conventions on how these functions are defined, which is prone to confuse (atleast me). In this post, I will attempt to reconcile the canonical description, based on so-called "spherical harmonic" functions ( \\( Y_l^m \\) ), with that used in geophysics which use associated Legendre polynomials ( \\( P_n^m \\) ).

Wikipedia has many good articles on this topic, including one on [spherical harmonics](https://en.wikipedia.org/wiki/Spherical_harmonics). Spherical harmonic functions ( \\( Y_l^m \\) ) are solutions to Laplace's equation : \\( \nabla^2 f = 0\\), where \\( f\\) is some scalar field. If \\( f=A\\) is the scalar magnetic potential, one can retrieve Gauss's law for magnetism. That is, if \\( \mathbf{B} = \nabla A \\), then \\( \nabla^2 
A  = 0\\) implies that \\( \nabla \cdot \mathbf{B} = 0\\).

We will use spherical coordinates \\( r, \theta, \phi\\), where \\( \phi\\) is the azimuthal angle and \\( \theta\\) is the co-latitude. Then,

\begin{equation}
   Y_l^m \ \equiv Y_l^m (\theta, \phi)
\end{equation}

In geomagnetism, spherical harmonics are used to specify the scalar magnetic potential in one of two ways - 

**Representation (1): Using spherical harmonics \\( Y_l^m \\) :**

$$ A(r, \theta, \phi) = a \sum_{l=1}^k \left(\frac{a}{r}\right)^{l+1} \sum_{m=-l}^{l} g_l^m Y_l^m (\theta, \phi) $$ 

**Representation (2): Using associated Legendre polynomials \\( P_n^m \\) (this is more common in geophysics):**
$$ A(r, \theta, \phi) = a \sum_{n=1}^k \left(\frac{a}{r}\right)^{l+1} \sum_{m=0}^{n} [g_n^m \cos(m\phi) + h_n^m \sin(m\phi)] P_n^m (\theta) $$ 

Before proceeding further, we should first discuss the differences between \\( Y_l^m \\) and \\( P_n^m \\). \\( Y_l^m \\) is a function of both \\( \theta \\) and \\( \phi \\), whereas \\( P_n^m \\) is a function of \\( \theta \\) only. \\( P_n^m (\theta) \\) are *associated* Legendre polynomials, which are different from \\( P_n(\theta)\\), which are just plain-ole Legendre polynomials . So, we are talking about three different polynomials here - 

**Legendre Polynomials** \\( P_n(\theta) \\), which are solutions to the equation 

$$ \left(1 - \theta^2\right) \frac{d^2 P_n(\theta)}{d\theta^2} - 2 \theta \frac{dP_n(\theta)}{d\theta} + n(n+1) P_n(\theta) = 0 $$

**Associated Legendre Polynomials** \\( P_n^m (\theta) \\), which are solutions to the equation 

$$ \frac{d}{d\theta} \left[ (1 - x^2) \frac{d}{d\theta} P_n^m(\theta) \right] + \left[n(n + 1) - \frac{m^2}{1 - \theta^2}\right] P_n^m(\theta) = 0 $$

**Spherical Harmonics** \\( Y_l^m (\theta, \phi) \\), which are related to associated Legendre polynomials as 

$$ Y_l^m (\theta, \phi) = N e^{im\phi} P_l^m(\cos\theta) $$

\\( N \\) here is a normalization constant. Different values of N lead to different kinds of normalization that are in use in different academic fields. In our case, we are interested in *Schmidt semi-normalized* harmonics, since this is what planetary magnetic field coefficients \\(g \\) and \\(h \\) are typically defined in. This is when  

$$ Y_l^m (\theta, \phi) = \sqrt{\frac{(l-m)!}{(l+m)!}} e^{im\phi} P_l^m(\cos\theta) $$

Now, we can make use of of the property (see the [wiki](https://en.wikipedia.org/wiki/Associated_Legendre_polynomials))

$$ P_l^{-m} (x) = (-1)^m \frac{(l-m)!}{(l+m)!} P_l^m(x) $$

to prove that 

$$ Y_l^{-m} (\theta, \phi) = (-1)^m \bar{Y}_l^m (\theta, \phi) $$

where \\( \bar{Y}_l^m \\) is the complex conjugate of \\( Y_l^m \\).

Now, we enter another avenue of confusion. The above functions \\( Y_l^m \\) are complex valued spherical harmonics. It turns out, that there are also *real* spherical harmonics ([wiki](https://en.wikipedia.org/wiki/Spherical_harmonics#Real_form)) that are defined differently. Real spherical harmonics are denoted as \\( Y_{lm}\\) and are **not** equivalent to the real part of \\( Y_l^m \\) ( \\( Y_{lm}\neq \mathcal{R} \\{ Y_l^m \\} \\)). They are given as 

For \\( m < 0 \\)
$$ Y_{lm} = \frac{i}{\sqrt{2}} \left( Y_l^m - (-1)^m Y_l^{-m}\right) $$ 
For \\( m > 0 \\)
$$ Y_{lm} = \frac{1}{\sqrt{2}} \left( Y_l^m + (-1)^m Y_l^{-m}\right) $$ 
For \\( m = 0 \\)
$$ Y_{lm} = Y_l^0 $$ 

Hence, for a real valued scalar potential \\( A \\), it is more precise to denote Repr (1) in terms of real values spherical harmonics and real coefficients,

$$ A(r, \theta, \phi) = a \sum_{l=1}^k \left(\frac{a}{r}\right)^{l+1} \sum_{m=-l}^{l} g_{lm} Y_{lm} (\theta, \phi) $$ 

By substituting the expression for \\( Y_l^m \\) in terms of \\( P_l^m\\), we get (considering Schmidt semi-normalization),

For \\( m < 0 \\)
$$ Y_{lm} = \sqrt{\frac{(l-|m|)!}{(l+|m|)!}} \sqrt{2} (-1)^m P_l^{|m|} \sin(|m|\phi) $$ 
For \\( m > 0 \\)
$$ Y_{lm} = \sqrt{\frac{(l-m)!}{(l+m)!}} \sqrt{2} (-1)^m P_l^m \cos(m\phi) $$ 
For \\( m = 0 \\)
$$ Y_{l0} = P_l^0  $$ 

Which we can use to write the vector potential as,


$$ A(r, \theta, \phi) = a \sum_{l=1}^k \left(\frac{a}{r}\right)^{l+1} \sum_{m=0}^{l} \sqrt{\frac{(l-m)!}{(l+m)!}} P_l^m \left[ (-1)^m  \cos(m\phi) g_l^m + (-1)^{-m} g_l^{-m}\sin(m\phi) \right] [\sqrt{2} \text{ if } m \neq 0]$$ 

This is very similar to Repr. (2). Although we have convolved the negative \\(m\\) values into the sum, \\( g_l^m\\) and \\( g_l^{-m}\\) remain different unique coefficients. Comparing the two, we can set the two sets of Legendre coefficients (I use \\( n \\) to refer to coefficients in Repr. (2) and \\(l\\) to refer those in (1). But they are equal : \\( n = l \\) ).

For \\(m \neq 0 \\),
$$ g_n^m = \sqrt{\frac{2(l-m)!}{(l+m)!}} (-1)^m g_l^m $$
$$ h_n^m = \sqrt{\frac{2(l-m)!}{(l+m)!}} (-1)^{-m} g_l^{-m} $$

For \\( m = 0 \\),
$$ g_n^0 = g_l^0$$
$$ h_n^0= \text{N/A or 0} $$

Hence, the two sets of coefficients used in Repr. (1) and (2) are related, as they should be. The above equations may change for different normalizations. If someone hands you a set of coefficients, you would be well served by asking them exactly how these coefficients were defined (i.e. what expression was used to fit them), and what normalization they use. 

# Beware using libraries like `scipy`:

The Python `scipy` package has methods to calculate associated Legendre polynomials and spherical harmonics. But keep in mind that these are calculated with the normalization (see [`scipy.special.sph_harm`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.special.sph_harm.html)). 

$$ Y_l^m (\theta, \phi) = \sqrt{\frac{2l + 1}{4\pi}\frac{(l-m)!}{(l+m)!}} e^{im\phi} P_l^m(\cos\theta) $$

which is different from Schmidt semi-normalization. Also note that these are *complex* spherical harmonics. 

The associated Legendre polynomials can also be calculated using [`scipy.special.lpmv`](https://docs.scipy.org/doc/scipy/reference/generated/scipy.special.lpmv.html), but they are also not normalized. MATLAB does the normalized variants using the `norm='sch'` argument in the [`legendre`](https://www.mathworks.com/help/matlab/ref/legendre.html) function.  

Typically, the coefficients \\(g_n^m\\) and \\( h_n^m\\) are provided in units of nT under the assumption that the Legendre polynomials are Schmidt semi-normalized. But IMO, the Legendre polynomials are what they are, it is really the coefficients where normalization should be applied. The choice to not normalize the coefficients was taken by the geophysics community early on to compare the relative contribution of different degrees to the field. The responsibility for proper normalization was then passed onto the unfortunate scientist.

A good reference for this topic (in addition to Wikipedia) is the book by James R. Wertz that is available [here](https://link.springer.com/book/10.1007/978-94-009-9907-7) - . I encourage you to read the appendices.

If I find the time sometime later, I will write about the old Fortran libraries used by `scipy` and other packages to calculate these coefficients. They are particularly opaque and fast.