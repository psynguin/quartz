# The Step-by-Step Process of Estimation of Latent Class Analysis (LCA)

Latent Class Analysis (LCA) is a finite mixture modeling technique used to uncover unobserved (latent) heterogeneity within a population. By analyzing cross-sectional patterns of categorical responses, LCA groups individuals into a set of mutually exclusive and exhaustive latent classes. This report outlines the theoretical foundation, mathematical estimation via the Expectation-Maximization (EM) algorithm, and the practical step-by-step workflow researchers use to estimate these models.

For deeper mathematical details, see [[02_Mathematics_of_LCA|Mathematics of LCA]]. For advanced interpretation techniques, check [[04_Advanced_Interpretation_of_LCA|Advanced Interpretation of LCA]].

## 1. Mathematical Formulation of LCA
Let $N$ be the total sample size and $J$ be the number of observed binary indicator variables. Let $Y_{ij}$ represent the response of individual $i$ to item $j$, where $Y_{ij}\in\{0,1\}$. 
Let $C_i$ represent the unobserved latent class variable for individual $i$, consisting of $K$ classes ($k=1,\dots,K$).

The LCA model is defined by two primary sets of parameters:
1. **Latent Class Probabilities ($\gamma_k$)**: The unconditional probability that a randomly selected individual belongs to class $k$. These sum to one: $\sum_{k=1}^K\gamma_k=1$.
2. **Item Response Probabilities ($\rho_{j|k}$)**: The conditional probability that an individual in class $k$ endorses item $j$, formally $P(Y_{ij}=1|C_i=k)$.

A core assumption of LCA is **local independence**, meaning that within any given latent class, the observed indicators are statistically independent of one another. Therefore, the probability of observing a specific response pattern $\mathbf{y}_i=(y_{i1},\dots,y_{iJ})$ conditional on class membership is the product of the individual item probabilities:
$$P(\mathbf{Y}_i=\mathbf{y}_i|C_i=k)=\prod_{j=1}^J\rho_{j|k}^{y_{ij}}(1-\rho_{j|k})^{1-y_{ij}}$$

The marginal probability of observing response pattern $\mathbf{y}_i$ across all classes is:
$$P(\mathbf{Y}_i=\mathbf{y}_i)=\sum_{k=1}^K\gamma_k\prod_{j=1}^J\rho_{j|k}^{y_{ij}}(1-\rho_{j|k})^{1-y_{ij}}$$

## 2. Parameter Estimation via Maximum Likelihood
The parameters are estimated using Maximum Likelihood Estimation (MLE). The likelihood function for the entire sample is:
$$L=\prod_{i=1}^N\left(\sum_{k=1}^K\gamma_k\prod_{j=1}^J\rho_{j|k}^{y_{ij}}(1-\rho_{j|k})^{1-y_{ij}}\right)$$

Because the class membership is unobserved, directly maximizing this likelihood is complex. Researchers and software (e.g., Mplus, Latent GOLD, R's poLCA—see [[05_Coding_LCA_in_R|Coding LCA in R]]) typically utilize the **Expectation-Maximization (EM) algorithm**.

The EM algorithm alternates between two steps until convergence:
*   **Expectation Step (E-step)**: Using current parameter estimates, the posterior probability ($\pi_{ik}$) that individual $i$ belongs to class $k$ is calculated via Bayes' Theorem:
    $$\pi_{ik}=\frac{\gamma_k\prod_{j=1}^JP(Y_{ij}=y_{ij}|C_i=k)}{\sum_{m=1}^K\gamma_m\prod_{j=1}^JP(Y_{ij}=y_{ij}|C_i=m)}$$
*   **Maximization Step (M-step)**: The parameter estimates are updated to maximize the likelihood using the posterior probabilities from the E-step:
    $$\hat{\gamma}_k=\frac{1}{N}\sum_{i=1}^N\pi_{ik}$$
    $$\hat{\rho}_{j|k}=\frac{\sum_{i=1}^N\pi_{ik}y_{ij}}{\sum_{i=1}^N\pi_{ik}}$$

Because the EM algorithm can converge on local maxima rather than the global maximum, researchers must use **multiple random starting values** during estimation to ensure the true maximum likelihood is found (Goodman, 1974; Nylund-Gibson & Choi, 2018).

## 3. The Practical Step-by-Step Workflow for Researchers
In practice, estimating an LCA involves a sequential modeling approach:

### Step 1: Model Specification and Enumeration
Researchers do not know the true number of classes ($K$) a priori. The process begins by estimating a baseline 1-class model (which assumes no latent heterogeneity and implies item associations are zero). The researcher then systematically estimates models with an increasing number of classes ($K=2$, $K=3$, $K=4$, etc.).

### Step 2: Model Estimation with Multiple Starts
For each $K$-class model, the software runs the EM algorithm. To prevent local maxima, researchers typically specify hundreds of random starting values. A model is considered well-estimated if the best (highest) log-likelihood value is replicated multiple times across these random starts.

### Step 3: Model Selection and Fit Evaluation
Because increasing $K$ always improves the raw likelihood, researchers compare the relative fit of models using several criteria:
*   **Information Criteria**: The Bayesian Information Criterion (BIC), Sample-Size Adjusted BIC (aBIC), and Akaike Information Criterion (AIC) introduce a penalty for adding more parameters. A lower BIC or aBIC indicates a better-fitting, more parsimonious model.
*   **Likelihood Ratio Tests (LRT)**: The Bootstrapped Likelihood Ratio Test (BLRT) and Lo-Mendell-Rubin (LMR-LRT) statistically compare a $K$-class model to a $(K-1)$-class model. A significant $p$-value ($p<0.05$) indicates the $K$-class model provides a statistically superior fit.
*   **Entropy**: Ranging from $0$ to $1$, entropy evaluates how cleanly the model separates individuals into classes (values closer to $1$ indicate clearer classification). While not used to decide the number of classes, it assesses classification utility.

### Step 4: Interpretability and Class Naming
A model must be theoretically interpretable. Researchers examine the estimated item response probabilities ($\rho_{j|k}$) for the chosen $K$-class model. By mapping which items have high versus low endorsement probabilities within a given class, researchers assign a substantive label to each class.

### Step 5: Class Assignment (Three-Step Approach)
Once the optimal model is selected, individuals are assigned to the class for which they have the highest posterior probability (modal assignment). This assignment can then be used in secondary analyses, such as predicting distal outcomes or adding covariates, often applying advanced bias-correction methods (e.g., the BCH or 3-step method) to account for assignment uncertainty. See [[06_Complex_Variations_of_LCA|Complex Variations of LCA]].

## 4. Short Example: Substance Use in Adolescents
**Scenario**: A researcher administers a survey to high school students with 3 binary (Yes/No) questions about substance use in the past month: Alcohol ($Y_1$), Tobacco ($Y_2$), and Marijuana ($Y_3$). 

**Step 1 & 2 (Estimation)**: The researcher tests 1-class to 4-class models using ML via the EM algorithm with 500 random starts.
**Step 3 (Selection)**: The fit statistics show the BIC drops significantly from the 2-class to the 3-class model, but rises for the 4-class model. The BLRT for the 3-class model is significant ($p=0.01$), whereas it is non-significant for the 4-class model ($p=0.45$). The 3-class model is selected.
**Step 4 (Interpretation)**: The researcher analyzes the item response probabilities ($\rho_{j|k}$) for the 3 classes:
*   **Class 1 ($\gamma_1=0.60$)**: Low probabilities across all items ($\rho_{1|1}=0.05$, $\rho_{2|1}=0.02$, $\rho_{3|1}=0.01$). **Label**: "Abstainers" (60% of the sample).
*   **Class 2 ($\gamma_2=0.25$)**: High probability for Alcohol ($\rho_{1|2}=0.85$), but low for Tobacco ($\rho_{2|2}=0.10$) and Marijuana ($\rho_{3|2}=0.05$). **Label**: "Alcohol-Only Users" (25% of the sample).
*   **Class 3 ($\gamma_3=0.15$)**: High probabilities across all three items ($\rho_{1|3}=0.90$, $\rho_{2|3}=0.75$, $\rho_{3|3}=0.80$). **Label**: "Poly-Substance Users" (15% of the sample).

**Step 5 (Assignment)**: An individual who answered Yes to Alcohol and Marijuana but No to Tobacco ($1,0,1$) has their posterior probabilities ($\pi_{ik}$) calculated. Suppose $\pi_{i1}=0.01$, $\pi_{i2}=0.15$, and $\pi_{i3}=0.84$. This individual is modally assigned to Class 3 (Poly-Substance Users).

### References
*   Collins, L. M., & Lanza, S. T. (2010). Latent Class and Latent Transition Analysis: With Applications in the Social, Behavioral, and Health Sciences. Wiley.
*   Goodman, L. A. (1974). Exploratory latent structure analysis using both identifiable and unidentifiable models. *Biometrika*, 61(2), 215-231.
*   Nylund-Gibson, K., & Choi, A. Y. (2018). Ten frequently asked questions about latent class analysis. *Translational Issues in Psychological Science*, 4(4), 440-461.
