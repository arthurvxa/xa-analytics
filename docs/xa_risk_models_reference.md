# XA Risk Models Reference — Version 1.0
**Anexo Técnico — Modelos Quantitativos de Risco do XA Analytics**  
**Escopo:** definições, fórmulas, hipóteses, calibração, backtesting e stress testing.  
**Uso:** referência para estudo, auditoria e evolução do algoritmo.

---

## 0. Convenções e Notação
- \( P_t \): preço no tempo \( t \).  
- \( r_t = \ln(P_t) - \ln(P_{t-1}) \): retorno logarítmico.  
- \( \mu \): média dos retornos.  
- \( \sigma \): desvio-padrão (volatilidade).  
- \( \Sigma \): matriz de covariância.  
- \( \rho \): correlação.  
- \( \alpha \): nível de confiança (ex.: 95% ou 99%).  
- \( z_\alpha \): quantil da Normal padrão (1,645 para 95%; 2,326 para 99%).  
- \( VaR_\alpha \): Value at Risk (Valor em Risco) ao nível \( \alpha \).  
- \( CVaR_\alpha \) ou ES\(_\alpha\): Conditional Value at Risk / Expected Shortfall (Valor em Risco Condicional).  
- \( \lambda \): fator de decaimento (EWMA — Exponentially Weighted Moving Average).  
- \( w \): vetor de pesos do portfólio; \( \mathbf{1} \): vetor de uns.  
- Horizonte de risco (padrão deste anexo): 1 dia útil, salvo menção em contrário.

---

## 1. Retornos, Limpeza e Reamostragem
1.1 **Retorno logarítmico diário**  
\[
r_t = \ln\!\left(\frac{P_t}{P_{t-1}}\right)
\]

1.2 **Tratamento de dados**  
- Ausentes: imputar somente quando imprescindível; preferir exclusão local.  
- Outliers: documentar política (winsorization, robust z-score ou filtros baseados em volatilidade).  
- Ajustes: preferir séries ajustadas por proventos e desdobramentos.

1.3 **Reamostragem temporal**  
- Agregação de retornos: \( r_{t,T} = \sum_{i=1}^{T} r_{t-i+1} \).  
- Regra \(\sqrt{T}\) para volatilidade: usar com cautela (não se aplica bem a séries com volatilidade condicional e caudas gordas).

---

## 2. Volatilidade
2.1 **Histórica (janelas fixas)**  
\[
\hat{\sigma} = \sqrt{\frac{1}{N-1}\sum_{i=1}^N (r_i - \bar{r})^2}
\]

2.2 **EWMA (Exponentially Weighted Moving Average)**  
\[
\sigma_t^2 = (1-\lambda)\, r_t^2 + \lambda\, \sigma_{t-1}^2,\quad \lambda \in [0{,}94,\;0{,}97] \text{ (diário)}
\]
- Responde mais rápido a choques; padrão RiskMetrics.

2.3 **Volatilidade do portfólio**  
\[
\sigma_P = \sqrt{w^\top \Sigma\, w}
\]

---

## 3. Correlação e Covariância
3.1 **Covariância**  
\[
\operatorname{Cov}(X,Y) = \mathbb{E}[(X-\mu_X)(Y-\mu_Y)]
\]

3.2 **Correlação**  
\[
\rho_{X,Y} = \frac{\operatorname{Cov}(X,Y)}{\sigma_X \sigma_Y}
\]

3.3 **Matriz de covariância do portfólio**  
\[
\Sigma = \begin{bmatrix}
\sigma_1^2 & \cdots & \sigma_{1n} \\
\vdots & \ddots & \vdots \\
\sigma_{n1} & \cdots & \sigma_n^2
\end{bmatrix}
\quad\text{e}\quad
\sigma_P = \sqrt{w^\top \Sigma w}
\]

3.4 **Estabilidade e encolhimento (shrinkage)**  
- Preferir shrinkage (Ledoit–Wolf) em universos grandes para reduzir ruído de estimação.

---

## 4. VaR — Value at Risk
4.1 **Paramétrico Normal (delta-normal)**  
\[
VaR_\alpha = z_\alpha \, \sigma_P \, V_P
\]
- Vantagem: simples e rápido.  
- Limitação: normalidade e linearidade; subestima caudas.

4.2 **Cornish–Fisher (ajuste por assimetria e curtose)**  
\[
z_\alpha^{CF} = z_\alpha + \frac{1}{6}(z_\alpha^2-1)\,\gamma_1 + \frac{1}{24}(z_\alpha^3-3z_\alpha)\,\gamma_2 - \frac{1}{36}(2z_\alpha^3-5z_\alpha)\,\gamma_1^2
\]
\[
VaR_\alpha^{CF} = z_\alpha^{CF} \, \sigma_P \, V_P
\]
- \(\gamma_1\): assimetria; \(\gamma_2\): curtose em excesso.

4.3 **Histórico (não paramétrico)**  
- Ordenar perdas diárias; selecionar o quantil \( 1-\alpha \).  
- Vantagem: captura caudas empíricas. Limitação: depende do histórico.

4.4 **Monte Carlo**  
- Simular retornos a partir de processo especificado (ex.: Normal, t-Student, GARCH, cópulas).  
- Calcular distribuição de perdas e extrair quantil.

---

## 5. CVaR / ES — Conditional Value at Risk (Expected Shortfall)
5.1 **Definição**  
\[
CVaR_\alpha = \mathbb{E}[L \mid L \ge VaR_\alpha]
\]

5.2 **Normal (fechada)**  
\[
CVaR_\alpha = \frac{\phi(z_\alpha)}{1-\alpha}\, \sigma_P \, V_P
\]
onde \(\phi\) é a densidade da Normal padrão.

5.3 **Histórico**  
- Média das perdas além do quantil \( VaR_\alpha \).

5.4 **Propriedade**  
- ES é coerente (subaditividade); preferível para risco de cauda.

---

## 6. Drawdown e Risco de Calda
6.1 **Drawdown no tempo \( t \)**  
\[
DD_t = 1 - \frac{P_t}{\max_{s\le t} P_s}
\]

6.2 **Máximo Drawdown (MDD)**  
\[
MDD = \max_t (DD_t)
\]

6.3 **Tail Risk**  
- Complementar a VaR/CVaR: análise de quantis extremos, POT (Peaks Over Threshold) — Teoria de Valores Extremos.

---

## 7. Beta, Fatores e PCA
7.1 **Beta (em relação ao benchmark \( B \))**  
\[
\beta = \frac{\operatorname{Cov}(r, r_B)}{\operatorname{Var}(r_B)}
\]

7.2 **Modelo de Fatores (ex.: CAPM e multi-fatores)**  
\[
r = \alpha + \beta_1 f_1 + \cdots + \beta_k f_k + \varepsilon
\]

7.3 **PCA (Principal Component Analysis)**  
- Decomposição de \( \Sigma \) ou da matriz de correlação \( R \) em autovalores \( \lambda_i \) e autovetores.  
- \( \lambda_1 \) elevado sinaliza risco sistêmico e colapso de diversificação.

---

## 8. Liquidez, Slippage e Custos
8.1 **Spread Bid–Ask médio**  
\[
\text{Spread} = \frac{\text{Ask} - \text{Bid}}{\text{Mid}}
\]

8.2 **Amihud Illiquidity (ILR)**  
\[
ILR = \frac{1}{N}\sum_{t=1}^{N}\frac{|r_t|}{\text{Volume}_t}
\]
- Maior ILR ⇒ pior liquidez.

8.3 **Slippage esperado**  
- Modelar como função do spread, profundidade do livro, urgência (market vs. limit), e fração do volume.

---

## 9. Agregação de Risco do Portfólio
9.1 **Retorno esperado**  
\[
\mu_P = w^\top \mu
\]

9.2 **Variância e volatilidade**  
\[
\sigma_P^2 = w^\top \Sigma w,\quad \sigma_P = \sqrt{w^\top \Sigma w}
\]

9.3 **Contribuição marginal de risco (MRC)**  
\[
MRC_i = \frac{\partial \sigma_P}{\partial w_i} = \frac{(\Sigma w)_i}{\sigma_P}
\]
- Útil para alocação por paridade de risco (risk parity).

---

## 10. Escalonamento Temporal
- Para retornos independentes e identicamente distribuídos (IID) e Normal:
\[
\sigma_{T} \approx \sigma_{1}\sqrt{T}, \quad VaR_{T} \approx VaR_{1}\sqrt{T}
\]
- Atenção: com volatilidade condicional (GARCH) e caudas gordas, estas aproximações degradam. Validar empiricamente.

---

## 11. Diagnósticos de Não-Normalidade
11.1 **Assimetria (skewness)**  
\[
\gamma_1 = \frac{\mathbb{E}[(r-\mu)^3]}{\sigma^3}
\]

11.2 **Curtose (excesso)**  
\[
\gamma_2 = \frac{\mathbb{E}[(r-\mu)^4]}{\sigma^4} - 3
\]

11.3 **Jarque–Bera**  
\[
JB = \frac{N}{6}\left(\gamma_1^2 + \frac{\gamma_2^2}{4}\right)
\]
- Rejeita Normalidade para \( p\text{-valor} < 0{,}05 \).

---

## 12. Backtesting de VaR
12.1 **Kupiec POF (Proportion of Failures)**  
- Compara violações observadas com a taxa esperada \( 1-\alpha \).

12.2 **Christoffersen (Independence e Conditional Coverage)**  
- Verifica independência temporal das violações.

12.3 **Basel Traffic Light**  
- Classifica em verde/âmbar/vermelho conforme número de violações em janela (ex.: 250 dias).

---

## 13. Stress Testing
- Choques **paramétricos**: \( \pm k\sigma \) (3 a 6 desvios).  
- Cenários **históricos**: 2008, 2020 etc.  
- **Hipotéticos**: quebras de correlação, secas de liquidez, gaps de abertura.

---

## 14. Calibração e Governança
14.1 **Parâmetros sugeridos**  
- EWMA: \( \lambda \in [0{,}94, 0{,}97] \).  
- Janelas: 250 dias (aprox. 1 ano útil) para VaR/CVaR histórico.  
- PCA: 60 dias para regime; ajustar por liquidez.  
- Nível de confiança: 95% para operação; 99% para capital regulatório.

14.2 **Revisão periódica**  
- Recalibrar por ativo/classe; registrar alterações (Git + hash SHA-256).  
- Aderir a WORM para logs e resultados.

---

## 15. Integração com Políticas de Hedge
- **Hedge é acionado** quando métricas de risco quebram limites (ex.: \( \sigma \), VaR, CVaR, \( \lambda_1 \)).  
- **Dimensionamento**: por metas de VaR/CVaR, beta, correlação e volatilidade-alvo.  
- **Restrições**: limites diários, por ativo, por custo e por slippage.

---

## 16. Referências Sugeridas
- J.P. Morgan, **RiskMetrics Technical Document** (1996).  
- P. Jorion, **Value at Risk**.  
- J. Hull, **Options, Futures, and Other Derivatives**.  
- Basel Committee on Banking Supervision, **Market Risk Framework**.  
- Artigos de Kupiec (1995) e Christoffersen (1998) sobre backtesting de VaR.

---

**Fim do Anexo Técnico — XA Risk Models Reference v1.0**
