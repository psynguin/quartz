# Introduction to Latent Class Analysis (LCA)

## 1. Concept and Theoretical Foundation
Latent Class Analysis (LCA) is a statistical modeling technique belonging to the family of finite mixture models. It is utilized to identify unobserved (latent) subgroups—termed "classes"—within a heterogeneous population based on individuals' responses to a set of observed categorical variables. The foundational premise of LCA is that the observed associations among the manifest variables are entirely explained by the existence of underlying latent classes. Once the latent class membership is accounted for, the observed variables are assumed to be statistically independent, a principle known as *local independence*. LCA is essentially the categorical analog to factor analysis; while factor analysis models continuous observed variables using continuous latent variables, LCA models categorical observed variables using a categorical latent variable. See [[07_Latent_Profile_Analysis|Latent Profile Analysis]] for the continuous analog.

## 2. Methodology and Mathematical Formulation
The methodology of LCA is rooted in probability theory and maximum likelihood estimation. For mathematical depth, see [[02_Mathematics_of_LCA|Mathematics of LCA]].

Let $\mathbf{Y}=(Y_1,Y_2,\dots,Y_J)$ represent a vector of $J$ observed categorical variables (items).
Let $C$ denote the unobserved latent categorical variable with $K$ distinct classes, where $k\in\{1,2,\dots,K\}$.

The unconditional probability of observing a specific response pattern $\mathbf{y}=(y_1,y_2,\dots,y_J)$ is the sum of the probabilities of that pattern across all $K$ classes:
$$P(\mathbf{Y}=\mathbf{y})=\sum_{k=1}^KP(C=k)P(\mathbf{Y}=\mathbf{y}|C=k)$$

Under the assumption of local independence, the probability of the response pattern conditional on class membership is the product of the individual item response probabilities:
$$P(\mathbf{Y}=\mathbf{y}|C=k)=\prod_{j=1}^JP(Y_j=y_j|C=k)$$

Substituting this into the prior equation yields the foundational LCA model equation:
$$P(\mathbf{Y}=\mathbf{y})=\sum_{k=1}^K\gamma_k\prod_{j=1}^J\rho_{j,y_j|k}$$

Here, the model estimates two primary sets of parameters:
1. **Latent Class Probabilities ($\gamma_k$)**: Also known as class prevalences or mixing proportions, representing the unconditional probability that a randomly selected individual belongs to class $k$, where $\gamma_k=P(C=k)$ and $\sum_{k=1}^K\gamma_k=1$.
2. **Item Response Probabilities ($\rho_{j,y_j|k}$)**: The probability of providing response $y_j$ to item $j$ given membership in class $k$, where $\rho_{j,y_j|k}=P(Y_j=y_j|C=k)$.

Model parameters are typically estimated using Maximum Likelihood Estimation (MLE) via the Expectation-Maximization (EM) algorithm (see [[03_LCA_Estimation_Process|LCA Estimation Process]]). The log-likelihood function for a sample of size $N$ with $U$ unique response patterns is defined as:
$$\ln L=\sum_{u=1}^Un_u\ln\left(\sum_{k=1}^K\gamma_k\prod_{j=1}^J\rho_{j,y_{uj}|k}\right)$$
where $n_u$ is the observed frequency of the $u$-th response pattern.

## 3. Use: Model Selection and Fit
In applied research, the true number of latent classes $K$ is unknown. Researchers estimate models with varying values of $K$ (e.g., 2-class, 3-class, 4-class) and compare their fit to select the most parsimonious and interpretable model. Standard criteria include:
- **Information Criteria**: The Akaike Information Criterion ($\text{AIC}=-2\ln L+2p$), the Bayesian Information Criterion ($\text{BIC}=-2\ln L+p\ln(N)$), and the sample-size adjusted BIC ($\text{aBIC}=-2\ln L+p\ln((N+2)/24)$), where $p$ is the number of estimated parameters. Lower values indicate better relative model fit. The BIC is frequently cited as the most robust indicator for class enumeration.
- **Likelihood Ratio Tests (LRT)**: The Lo-Mendell-Rubin adjusted Likelihood Ratio Test (LMR-LRT) and the Bootstrapped Likelihood Ratio Test (BLRT) compare a $K$-class model to a $(K-1)$-class model. A significant $p$-value suggests the $K$-class model fits significantly better.
- **Entropy**: A standardized measure of classification uncertainty ranging from $0$ to $1$. Entropy values approaching $1$ indicate clear delineation of classes. It is computed as:
$$E=1-\frac{\sum_{i=1}^N\sum_{k=1}^K\hat{p}_{ik}\ln(\hat{p}_{ik})}{N\ln(K)}$$
where $\hat{p}_{ik}$ is the estimated posterior probability for individual $i$ belonging to class $k$.
For advanced model selection, read [[04_Advanced_Interpretation_of_LCA|Advanced Interpretation of LCA]].

## 4. Interpretation of the Model
Once the optimal number of classes is selected, the interpretation focuses on the parameter estimates:
- **Class Profiling**: Researchers "name" or describe the classes by examining the item response probabilities ($\rho_{j,y_j|k}$). For instance, a class with high probabilities of endorsing items related to anxiety and depression might be labeled the "Internalizing Profile."
- **Class Assignment**: Individuals are not deterministically assigned to a class but have a probability of belonging to each. Bayes' theorem is used to calculate the posterior probability of class membership for a given response pattern:
$$P(C=k|\mathbf{Y}=\mathbf{y})=\frac{\gamma_k\prod_{j=1}^J\rho_{j,y_j|k}}{\sum_{m=1}^K\gamma_m\prod_{j=1}^J\rho_{j,y_j|m}}$$
Individuals are typically assigned to the class for which their posterior probability is highest (modal assignment).

## 5. Advanced Extensions
In contemporary structural equation modeling frameworks, LCA is often extended to include:
- **Covariates**: Predictors of latent class membership, often analyzed using multinomial logistic regression simultaneously with class estimation. See [[06_Complex_Variations_of_LCA|Complex Variations of LCA]].
- **Distal Outcomes**: Assessing how latent class membership predicts downstream continuous or categorical outcomes. Modern approaches, such as the 3-step approach (BCH or auxiliary variable methods), correct for classification error without allowing the distal outcome to improperly influence the formation of the classes.
- **Multiple Group LCA**: Testing for measurement invariance of the latent classes across different demographic groups.
- **Latent Transition Analysis (LTA)**: The longitudinal extension of LCA, where transitions between latent classes are modeled over time via transition probabilities.

## 6. Reliable Academic Sources
The following are foundational and highly reliable academic references that establish the methodology and application of LCA:
1. **Lazarsfeld, P. F., & Henry, N. W. (1968).** *Latent Structure Analysis.* Houghton Mifflin.
2. **Goodman, L. A. (1974).** "Exploratory latent structure analysis using both identifiable and unidentifiable models." *Biometrika*, 61(2), 215-231.
3. **McCutcheon, A. L. (1987).** *Latent Class Analysis* (Quantitative Applications in the Social Sciences). Sage Publications.
4. **Nylund, K. L., Asparouhov, T., & Muthén, B. O. (2007).** "Deciding on the number of classes in latent class analysis and growth mixture modeling: A Monte Carlo simulation study." *Structural Equation Modeling*, 14(4), 535-569.
5. **Collins, L. M., & Lanza, S. T. (2010).** *Latent Class and Latent Transition Analysis: With Applications in the Social, Behavioral, and Health Sciences.* Wiley.
