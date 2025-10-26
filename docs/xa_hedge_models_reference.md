docs/
├── xa_analytics_master_config_v1.1.2.md     ← documento principal
└── xa_hedge_models_reference.md             ← anexo técnico de modelos de hedge
# XA Hedge Models Reference — Version 1.0
**Anexo Técnico — Modelos Quantitativos de Hedge do XA Analytics**  
**Escopo:** fórmulas, hipóteses, calibração e validação.  
**Uso:** referência para estudo e evolução do algoritmo.

---

## 0. Notação e Convenções

- \( P_t \): preço no tempo \( t \).  
- \( r_t = \ln(P_t) - \ln(P_{t-1}) \): retorno logarítmico.  
- \( \mu \): média dos retornos.  
- \( \sigma \): desvio-padrão (volatilidade).  
- \( \Sigma \): matriz de covariância.  
- \( \rho \): correlação.  
- \( \alpha \): nível de confiança (ex.: 95%).  
- \( z_\alpha \): quantil da Normal padrão no nível \( \alpha \) (ex.: 1,645 para 95%).  
- \( VaR \): Value at Risk (Valor em Risco).  
- \( CVaR \): Conditional Value at Risk (Valor em Risco Condicional).  
- \( \beta \): beta do ativo em relação ao benchmark.  
- \( \Delta \): delta (sensibilidade) de uma opção.  
- \( \lambda \): parâmetro de decaimento exponencial (EWMA).  
- \( \lambda_1 \): maior autovalor (PCA) da matriz de correlação.

---

## 1. Volatilidade EWMA (Exponentially Weighted Moving Average)

**Definição (variância condicional):**
\[
\sigma_t^2 = (1-\lambda)\, r_t^2 + \lambda\, \sigma_{t-1}^2,\quad 0<\lambda<1
\]

- Valores típicos: \( \lambda \in [0{,}94,\;0{,}97] \) para dados diários.  
- **Interpretação:** dá mais peso aos retornos recentes; responde rápido a choques.

**Gatilho de anomalia (exemplo de política):**  
\[
\sigma_t > 2 \times \text{Perc}_{95\%}(\sigma_{\text{hist}})
\]
Se verdadeiro, indica regime de alta volatilidade (elegível a hedge).

---

## 2. Bandas de Bollinger (20 períodos, \( K \) desvios)

- Média móvel (pode-se usar EMA de 20):
\[
\text{MB}_{t} = \text{EMA}_{20}(P_t)
\]
- Desvio (20 períodos):
\[
s_{20,t} = \sqrt{\frac{1}{20-1}\sum_{i=0}^{19}\left(P_{t-i}-\overline{P}_{20,t}\right)^2}
\]
- Bandas:
\[
\text{SUP}_t = \text{MB}_t + K\, s_{20,t},\qquad
\text{INF}_t = \text{MB}_t - K\, s_{20,t}
\]
- **Evento de cauda (exemplo):** fechamento além de \( K=2{,}5 \) desvios → sinal de estresse/extremo.

---

## 3. EMAs e MACD (Momentum/Tendência)

- **EMA** (Exponential Moving Average) de período \( n \):
\[
EMA_t^{(n)} = \alpha\, P_t + (1-\alpha)\, EMA_{t-1}^{(n)},\quad \alpha=\frac{2}{n+1}
\]
- **MACD clássico:**
\[
MACD_t = EMA_t^{(12)} - EMA_t^{(26)},\qquad
\text{Sinal}_t = EMA^{(9)}(MACD_t),\qquad
\text{Hist}_t = MACD_t - \text{Sinal}_t
\]
- **Condição de tendência baixista (exemplo):**  
\( EMA^{(12)} < EMA^{(26)} \) **e** \( MACD_t<0 \) **e** \( \text{Hist}_t<0 \).

---

## 4. Beta-Hedge (Hedge por Beta)

**Objetivo:** neutralizar risco direcional relativo a um benchmark \( B \).

- **Beta pela regressão:**
\[
\beta = \frac{\operatorname{Cov}(r_{\text{ativo}}, r_B)}{\operatorname{Var}(r_B)}
\]
- **Quantidade de contratos futuros (noção simplificada):**
\[
Q_{\text{hedge}} = \frac{V_{\text{posição}} \times \beta}{V_{\text{contrato}}}
\]
onde \( V_{\text{posição}} \) é o nocional a proteger e \( V_{\text{contrato}} \) o nocional de um contrato futuro/índice.

**Meta:** aproximar a variação líquida da carteira a zero frente ao benchmark.

---

## 5. Hedge por Volatilidade (Target-Vol)

**Redimensionamento de exposição para alvo de volatilidade \( \sigma_{\text{alvo}} \):**
\[
w_{\text{novo}} = w_{\text{antigo}} \times \frac{\sigma_{\text{alvo}}}{\sigma_{\text{atual}}}
\]
- Se \( \sigma_{\text{atual}} = 2 \times \sigma_{\text{alvo}} \Rightarrow w_{\text{novo}} = 0{,}5 \).  
- Pode-se compensar diferença via posição inversa (futuro/ETF inverso) até atingir o alvo.

---

## 6. VaR (Value at Risk) — Paramétrico Normal

**Perda máxima esperada em 1 dia, com confiança \( \alpha \):**
\[
VaR_{\alpha} = z_{\alpha}\, \sigma_P\, V_P
\]
onde \( \sigma_P \) é a volatilidade diária da posição/carteira e \( V_P \) o valor nocional.

**Hedge para reduzir VaR ao alvo \( VaR_{\alpha}^{\text{alvo}} \):**  
Se \( \Delta VaR = VaR_{\alpha}^{\text{atual}} - VaR_{\alpha}^{\text{alvo}} > 0 \), dimensionar hedge tal que a contribuição de risco do instrumento compense \( \Delta VaR \) (via beta/vol ou simulação incremental).

**Validação:** Teste de cobertura de VaR (Kupiec, 1995).

---

## 7. CVaR (Conditional VaR) — Cauda da Distribuição

**Definição (discreta):**
\[
CVaR_{\alpha} = \mathbb{E}[L \mid L \geq VaR_{\alpha}]
\]
ou, para distribuição Normal:
\[
CVaR_{\alpha} = \frac{\phi(z_{\alpha})}{1-\alpha}\, \sigma_P\, V_P
\]
onde \( \phi \) é a densidade da Normal padrão.

**Meta:** reduzir \( CVaR_{\alpha} \) em pelo menos uma fração \( \kappa \) (ex.: 25%).  
Hedge com **opções de venda (PUT)** é indicado para reduzir cauda.

---

## 8. Delta-Hedge (Portfólios com Opções)

**Delta total da carteira:**
\[
\Delta_{\text{carteira}} = \sum_i N_i \Delta_i
\]
**Neutralização (no subjacente):**
\[
Q_{\text{hedge}} = -\Delta_{\text{carteira}}
\]
- Se \( \Delta_{\text{carteira}} = +250 \Rightarrow \) vender 250 unidades do subjacente (ou equivalente em futuros).

---

## 9. Hedge por Correlação (Cross-Asset)

Quando não há derivativo direto do ativo, usa-se um correlato \( H \).

**Tamanho do hedge (forma prática):**
\[
Q_{\text{hedge}} = \frac{V_{\text{posição}} \cdot \rho_{A,H} \cdot \sigma_A}{\sigma_H \cdot V_{\text{contrato}}}
\]
- Quanto maior a correlação \( \rho_{A,H} \), mais eficiente o hedge.

---

## 10. PCA e Regime Sistêmico (λ₁-Hedge)

**Matriz de correlação \( R \)** da carteira → autovalores \( \lambda_1 \ge \lambda_2 \ge \dots \).

- **Indicador sistêmico:** aumento de \( \lambda_1 \) acima de \( +30\% \) da média de 60 dias.  
- **Leitura:** diversificação colapsando; risco concentrado no 1º componente.  
- **Ação:** reduzir exposição ao fator 1 (ex.: vender índice/ETF amplo).

---

## 11. Controle Estatístico de Processo (EWMA-Control e CUSUM)

**EWMA-Control (desvio suavizado):**
\[
z_t = \gamma\, x_t + (1-\gamma)\, z_{t-1},\quad 0<\gamma<1
\]
- Disparar alarme se \( |z_t| > L \), com \( L \) calibrado por taxa de falso-alarme.

**CUSUM (soma cumulativa):**
\[
C_t = \max\{0,\; C_{t-1} + (x_t - k)\}
\]
- Alarme quando \( C_t > h \).  
- \( k \): drifte mínimo detectável; \( h \): limiar de decisão.

**Uso:** detectar mudanças de nível/deriva antes de grandes quebras.

---

## 12. Regras de Dimensionamento (Hierarquia de Hedge)

1. **Primeiro**, estabilizar volatilidade (EWMA/Target-Vol).  
2. **Depois**, ajustar direção (EMA/MACD, Bandas).  
3. **Se sistêmico**, usar PCA/λ₁ e hedge de índice.  
4. **Por fim**, calibrar por **VaR/CVaR** até metas.  
5. **Com opções**, neutralizar **Delta** se necessário.

---

## 13. Calibração e Parâmetros

- **EWMA \(\lambda\)**: 0,94–0,97 (diário).  
- **Bandas**: 20 períodos; \( K = 2{,}0 \) a \( 2{,}5 \) para eventos de cauda.  
- **EMAs**: 12, 26, 50, 100, 200 (curto → longo).  
- **VaR/CVaR**: \( \alpha = 95\% \) (ou 99% conforme política).  
- **PCA janela**: 60 dias (ajustável por liquidez).  
- **Cusum/EWMA-Control**: \( k \) e \( h \) calibrados por taxa de falso positivo.

**Boas práticas:** recalibrar parâmetros por ativo/classe, validando em janela *rolling*.

---

## 14. Validação de Risco (Backtest e Estatística)

- **Kupiec (Proportion of Failures Test):** avalia a taxa de violações de VaR observada vs. esperada.  
- **Christoffersen (Independence Test):** verifica independência temporal das violações.  
- **Redução de CVaR:** estimar por *bootstrap* (IC 95%).  
- **Stress testing:** choques de \( 3\sigma \) a \( 6\sigma \); gaps; secas de liquidez.  
- **A/B Shadow:** executar o hedge em modo “sombra” antes de liberar capital (quando viável).

---

## 15. Limites Operacionais (Guardrails)

- **Slippage máximo:** abortar se > 0,20% (ajustável).  
- **Liquidez mínima:** volume/spread definidos por ativo/bolsa.  
- **Fracionamento:** TWAP/VWAP para ordens grandes.  
- **Cooldown:** aguardar \( N \) minutos antes de reforçar hedge.  
- **Teto diário:** hedges novos \(\le\) 10% do capital/dia.  
- **Teto por ativo:** \(\le\) 80% da exposição líquida.  
- **Custo de PUT:** \(\le\) 0,30% do capital/dia.

---

## 16. Observações de Compliance

- **Finalidade do hedge:** mitigação de risco (não especulação).  
- **Logs imutáveis:** WORM + hash SHA-256 + timestamp UTC.  
- **Pré-aprovação por política:** a automação é permitida **dentro** dos limites documentados.  
- **Conformidade:** CVM 505/2011 (mecanismos de contenção), SEC 15c3-5, LGPD art. 20 (§1º), EU AI Act (rastreabilidade/segurança).

---

## 17. Leituras Sugeridas (para estudo)

- **RiskMetrics (J.P. Morgan, 1996):** EWMA e VaR paramétrico.  
- **Bollinger, J.:** *Bollinger on Bollinger Bands*.  
- **PCA em Finanças:** aplicação de fatores principais em risco de carteiras.  
- **Backtesting de VaR:** testes de Kupiec e Christoffersen.  
- **Gestão de cauda:** CVaR e opções de proteção (PUTs).

---

**Fim do Anexo Técnico — XA Hedge Models Reference v1.0**
