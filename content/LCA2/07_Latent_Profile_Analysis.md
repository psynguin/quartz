# Latent Profile Analysis (LPA)

## 1. Introduction

Latent Profile Analysis (LPA) is a specific application of finite mixture modeling designed to identify unobserved, mutually exclusive subgroups (latent classes or profiles) within a population based on their responses to a set of observed **continuous** indicators. Like Latent Class Analysis (LCA), LPA is a person-centered analytic approach; however, its handling of continuous data differentiates it mathematically and practically. The primary objective is to unmix a heterogeneous sample into homogeneous latent profiles characterized by distinct patterns of means and variances on the observed variables. 

See [[01_Introduction_to_LCA|Introduction to LCA]] to compare it with the categorical-indicator equivalent.

## 2. LPA vs. LCA: Continuous vs. Categorical Indicators

While both LPA and LCA are structural equation models involving categorical latent variables, they fundamentally differ in the distributional assumptions placed on the observed indicators:

*   **Nature of Indicators:** LCA is utilized when the observed indicators are categorical (binary, ordinal, or nominal). LPA is utilized when the observed indicators are continuous.
*   **Distributional Assumptions:** LCA relies on the Bernoulli or multinomial distributions to model the probability of endorsing an item category. LPA assumes the continuous indicators follow a multivariate normal (Gaussian) distribution within each latent profile.
*   **Parameterization:** 
    *   In LCA, the primary parameters estimated are the **item response probabilities** (the probability of a specific response given class membership).
    *   In LPA, the primary parameters estimated are the **profile-specific means** (the expected value of each continuous indicator within a profile) and **profile-specific variances and covariances**. 
*   **Local Independence:** In LCA, the local independence assumption posits that indicators are statistically independent within each latent class. In LPA, strict local independence implies that the covariance matrix of the continuous indicators within each profile is strictly diagonal (i.e., zero off-diagonal covariances). However, a distinct advantage of LPA (and generalized Gaussian Mixture Models) is the ability to relax this assumption and estimate non-zero covariances between indicators within classes, provided the model remains identified.

## 3. Methodology and Mathematical Framework

### The Model Formulation

Let $y_i = (y_{i1}, y_{i2}, \dots, y_{iP})^\top$ represent a vector of $P$ continuous observed variables for individual $i$. The fundamental equation of LPA defines the marginal probability density function of $y_i$ as a weighted sum (mixture) of $K$ class-specific densities:

$$f(y_i) = \sum_{k=1}^K \pi_k f_k(y_i | \theta_k)$$

Where:
*   $K$ is the total number of latent profiles.
*   $\pi_k$ is the mixing proportion (unconditional prior probability) of belonging to profile $k$, with the constraints that $\pi_k > 0$ and $\sum_{k=1}^K \pi_k = 1$.
*   $f_k(y_i | \theta_k)$ is the profile-specific probability density function parameterized by $\theta_k$.

Because LPA deals with continuous variables, $f_k$ is typically specified as a $P$-dimensional multivariate normal distribution:

$$f_k(y_i | \mu_k, \Sigma_k) = (2\pi)^{-P/2} |\Sigma_k|^{-1/2} \exp\left(-\frac{1}{2}(y_i - \mu_k)^\top \Sigma_k^{-1} (y_i - \mu_k)\right)$$

Here, the parameter vector $\theta_k$ consists of:
*   $\mu_k$: A $P \times 1$ vector of means for the $P$ indicators in profile $k$.
*   $\Sigma_k$: A $P \times P$ variance-covariance matrix for the indicators in profile $k$.

### Model Specifications and Covariance Structures

The flexibility of LPA lies in how the covariance matrix $\Sigma_k$ is specified. By placing different constraints on the variances (diagonal elements) and covariances (off-diagonal elements) across the $K$ profiles, researchers can specify a range of models. Common specifications (drawing heavily from Gaussian Mixture Modeling literature like the *mclust* R package framework) include:

1.  **Class-Invariant, Diagonal (Strict Local Independence):** Variances are constrained to be equal across profiles ($\sigma_{pk}^2 = \sigma_p^2$), and all covariances are constrained to zero.
2.  **Class-Varying, Diagonal:** Variances are freely estimated across profiles ($\sigma_{pk}^2$), but covariances are constrained to zero.
3.  **Class-Invariant, Unstructured:** Variances and covariances are non-zero but constrained to be identical across all profiles ($\Sigma_k = \Sigma$).
4.  **Class-Varying, Unstructured:** Variances and covariances are freely estimated in every profile ($\Sigma_k$). This is the most complex specification and requires very large sample sizes to converge without producing non-positive definite matrices.

### Model Estimation

LPA parameters are estimated via Maximum Likelihood (ML) using the Expectation-Maximization (EM) algorithm. The log-likelihood function to be maximized is:

$$\ln L = \sum_{i=1}^N \ln \left( \sum_{k=1}^K \pi_k f_k(y_i | \mu_k, \Sigma_k) \right)$$

The EM algorithm iteratively alternates between calculating the expected class membership probabilities for each individual (E-step) and maximizing the likelihood of the parameters given those expected probabilities (M-step) until convergence is achieved. (This mirrors the EM process in [[02_Mathematics_of_LCA|Mathematics of LCA]]).

## 4. Interpretation of Results

The primary substantive interpretation of an LPA model focuses on three sets of estimates:

### 1. Profile Patterns (Means and Variances)
The $\mu_k$ parameters define the "shape" and "level" of each latent profile. Researchers interpret profiles by examining which indicators have comparatively high, low, or average means. For example, in a psychological study, one profile might exhibit elevated means on all anxiety continuous indicators, representing a "Severe Anxiety" profile. Profile-specific variances ($\Sigma_k$) reveal the dispersion of individuals around the mean within that profile.

### 2. Profile Prevalences
The mixing proportions ($\pi_k$) define the relative size of each profile in the population. It indicates what fraction of the sample is estimated to belong to each latent subgroup. 

### 3. Posterior Class Probabilities
Once the model parameters are estimated, Bayes' theorem is used to compute the posterior probability that individual $i$ belongs to profile $k$, given their observed data vector $y_i$:

$$P(C_i = k | y_i) = \frac{\pi_k f_k(y_i | \mu_k, \Sigma_k)}{\sum_{j=1}^K \pi_j f_j(y_i | \mu_j, \Sigma_j)}$$

Individuals are subsequently assigned to the profile for which they have the highest posterior probability (modal assignment).

## 5. Model Evaluation and Selection

Determining the optimal number of latent profiles $K$ involves estimating a sequence of models (e.g., $K=1, 2, 3, \dots$) and comparing their fit using several criteria (see also [[04_Advanced_Interpretation_of_LCA|Advanced Interpretation of LCA]]):

*   **Information Criteria:** The Akaike Information Criterion ($AIC$), Bayesian Information Criterion ($BIC$), and sample-size adjusted BIC ($SABIC$) are primarily used. Lower values indicate a better balance between model fit and parsimony. The $BIC$ is widely considered the most robust criterion for enumeration in LPA.
*   **Likelihood Ratio Tests (LRTs):** The Lo-Mendell-Rubin LRT ($LMR-LRT$) and the Bootstrapped LRT ($BLRT$) statistically test the improvement in fit between a $K$-profile model and a $(K-1)$-profile model. A significant $p$-value indicates that the $K$-profile model fits significantly better than the one with one fewer profile.
*   **Classification Quality (Entropy):** Entropy summarizes the precision of individual classification based on posterior probabilities, scaled from $0$ to $1$:

    $$E = 1 - \frac{\sum_{i=1}^N \sum_{k=1}^K -\hat{p}_{ik} \ln \hat{p}_{ik}}{N \ln K}$$

    where $\hat{p}_{ik} = P(C_i = k | y_i)$. Entropy values closer to $1.0$ (typically $\ge 0.80$) indicate excellent separation of the latent profiles, whereas lower values suggest excessive overlap and classification ambiguity.

## 6. Practical Coding in R

While `poLCA` handles categorical indicators, analysts primarily use `mclust` or `tidySEM` for continuous LPA. 

### Estimation using `tidySEM`
The `tidySEM` package provides a highly intuitive interface for estimating LPA models with class-invariant diagonal covariance structures (the most common LPA formulation).

```R
library(tidySEM)

# Define continuous indicators
continuous_vars <- clean_data[, c("continuous_var1", "continuous_var2", "continuous_var3")]

# Estimate models from 1 to 5 profiles
lpa_models <- mx_profiles(data = continuous_vars, classes = 1:5)

# View a comprehensive fit table (including BIC, SABIC, Entropy)
fit_table <- table_fit(lpa_models)
print(fit_table)

# Select the best model (e.g., 3-class) and view results
best_lpa <- lpa_models[[3]]
summary(best_lpa)
```

## 7. Advanced Extensions

Like LCA, LPA can be extended into broader structural frameworks:
*   **Predictors:** Including covariates to predict the log-odds of profile membership using multinomial logistic regression (e.g., the 3-step auxiliary variable approach).
*   **Distal Outcomes:** Testing whether latent profile membership predicts downstream continuous or categorical consequences.
*   **Longitudinal Models:** LPA serves as the foundation for Growth Mixture Modeling (GMM), where the latent variables driving the mixture are continuous growth factors (intercepts and slopes) rather than cross-sectional means.

*Sources synthesized for this exposition include standard foundational texts such as McLachlan & Peel's "Finite Mixture Models" (2000), Muthén's foundational papers on latent variable mixture modeling, and applied frameworks established by Pastor et al. (2007) and Oberski (2016).*
