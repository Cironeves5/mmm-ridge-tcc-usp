# Marketing Mix Modeling (MMM): a contribuição da mídia sobre as vendas

**Modelagem de mix de marketing com regressão regularizada, adstock e curvas de saturação**

> 🚧 **Projeto em desenvolvimento.** Trabalho de Conclusão de Curso do MBA em Ciência de Dados do ICMC/USP, sob orientação da Profa. Cibele Russo. As fases de preparação, modelagem e avaliação estão em construção, e os resultados apresentados aqui são preliminares.

# 1. Problema de Negócio

Saber quanto cada canal de mídia contribui para as vendas sempre foi uma pergunta central do marketing, e ela ficou mais difícil de responder nos últimos anos. Navegadores como Safari e Firefox passaram a bloquear cookies de terceiros por padrão, a App Tracking Transparency da Apple, em vigor desde o iOS 14.5 (2021), passou a exigir a autorização do usuário para o rastreamento entre aplicativos, e leis de proteção de dados como a LGPD e o GDPR restringiram o uso de informações pessoais. Com isso, as plataformas perderam boa parte da capacidade de identificar de onde vem cada acesso e cada compra.

Mesmo quando o rastreamento funciona, os modelos de atribuição mais usados, como o de último clique, representam mal o comportamento do consumidor. Uma pessoa pode ver um anúncio no display, encontrar a marca no jornal dias depois e só então buscar o produto no Google e comprar. Na atribuição por último clique, a busca paga leva todo o crédito pela venda, e os canais que despertaram o interesse ficam invisíveis, sobretudo os offline, que nunca geram um clique.

Sem uma rastreabilidade confiável, as empresas perdem a base para dizer qual canal gera retorno e como distribuir o orçamento. É nesse contexto que o Marketing Mix Modeling volta a ganhar espaço. Por ser uma regressão sobre séries temporais agregadas (vendas e investimento por semana, por exemplo), o MMM estima a contribuição de cada canal sem depender de dados individuais de usuário nem dos relatórios das próprias plataformas, e ainda coloca a mídia online e a offline na mesma base de comparação.

## Os desafios de estimação

Para cumprir esse papel, o modelo precisa lidar com três características da resposta à mídia.

- **Carryover (adstock).** O efeito de um anúncio continua nas semanas seguintes à veiculação, por memória de marca, e o modelo precisa estimar quanto desse efeito permanece a cada semana.
- **Saturação.** O retorno de cada canal é decrescente, porque a audiência de conversão mais fácil se esgota primeiro, e o modelo precisa identificar a partir de que nível o investimento adicional passa a render pouco.
- **Multicolinearidade.** Quando os canais são acionados juntos, em campanhas integradas, as séries de investimento se movem ao mesmo tempo e fica difícil separar o efeito de cada uma.

Nenhuma dessas características tem um valor universal. O carryover e a saturação mudam conforme o canal, o mercado, a região e a categoria de produto, e a multicolinearidade depende de como cada empresa planeja a veiculação. Por isso, os parâmetros precisam ser estimados a partir do histórico de cada negócio, e a qualidade dessa estimação define o quanto o modelo é confiável para orientar decisões de orçamento.

# 2. Objetivo e Perguntas de Pesquisa

**Objetivo geral.** Desenvolver um modelo de MMM capaz de quantificar a contribuição incremental de cada canal de mídia sobre as vendas e estimar o retorno sobre o investimento.

- **P1.** Qual a participação de cada canal de mídia nas vendas do período analisado?
- **P2.** Quais canais apresentam maior efeito de carryover e quais respondem de forma mais imediata?
- **P3.** Em que nível de investimento cada canal atinge a saturação?
- **P4.** Dado o orçamento observado, qual alocação alternativa entre canais maximizaria as vendas esperadas?

# 3. Ferramentas Utilizadas

**Linguagem e ambiente.** Python no Google Colab.

**Bibliotecas.**
- **pandas e numpy** para a manipulação das séries semanais.
- **matplotlib e seaborn** para a análise exploratória.
- **statsmodels** para a decomposição sazonal, a ACF, a PACF e o teste de Durbin-Watson.
- **dtaidistance e tslearn** para a similaridade temporal por Dynamic Time Warping (DTW).
- **phik** para a associação entre variáveis numéricas e binárias.
- **Prophet** para a extração do componente de tendência.
- **scikit-learn** para a padronização, a regressão Ridge e a busca do parâmetro de regularização.
- **Nevergrad** (planejado) para a busca dos hiperparâmetros de adstock e saturação, como no Robyn da Meta.

**Metodologia.** CRISP-DM, com o notebook organizado nas seis fases do processo.

# 4. Descrição dos Dados

**Fonte.** Base pública do projeto [mmm_stan](https://github.com/sibylhe/mmm_stan), com o histórico de um varejista dos Estados Unidos.

**Período e granularidade.** 209 semanas, de 03/08/2014 a 29/07/2018.

**Canais de mídia.** Cada canal aparece em duas formas, o investimento (quanto foi gasto) e a exposição (quantas impressões o gasto comprou).

| Canal | Tipo | Investimento registrado |
|---|---|---|
| Mala direta, folheto e jornal | Offline | Sim |
| Rádio e TV | Offline | Sim |
| Áudio streaming, vídeo, mídia social, display e Google Ads | Digital | Sim |
| E-mail, SMS e afiliados | Própria (owned) | Não, entram como controle e sem ROI |

**Demais variáveis.**

| Grupo | Conteúdo |
|---|---|
| Variável resposta | Vendas semanais |
| Contexto de negócio | Descontos, promoções e número médio de lojas abertas na semana |
| Macroeconomia | Confiança do consumidor e preço da gasolina |
| Calendário | Feriados e datas comerciais (one-hot) e sazonalidade por período e por semana |

# 5. Desenvolvimento da Solução

## 5.1 Especificação do modelo

As vendas de cada semana são modeladas como a soma de uma linha de base, das contribuições de mídia transformadas e dos controles.

$$
\text{vendas}_t = \beta_0 + \sum_{m} \beta_m \, \text{Hill}\big(\text{Adstock}(x_{m,t})\big) + \sum_{c} \gamma_c \, z_{c,t} + \varepsilon_t
$$

**Adstock geométrico.** A parcela α do efeito de uma semana passa para a seguinte.

$$
A_t = x_t + \alpha \, A_{t-1}, \qquad 0 \le \alpha < 1
$$

**Saturação de Hill.** K é o ponto de meia saturação e S controla a inclinação da curva.

$$
\text{Hill}(A_t) = \frac{A_t^{S}}{K^{S} + A_t^{S}}
$$

A estimação usa regressão **Ridge com restrição de não negatividade** nos coeficientes, escolhida pela multicolinearidade entre os canais e alinhada à prática de mercado dos MMMs frequentistas.

## 5.2 Principais achados da análise exploratória

1. **Sazonalidade anual forte.** A ACF mostra picos nos lags 52 e 104, e novembro e dezembro concentram uma parcela das vendas bem acima das demais semanas. Os controles de calendário são necessários para que esses picos não sejam atribuídos à mídia.
2. **A verba acompanha a demanda.** A maioria dos canais concentra investimento no fim de ano, junto com as vendas, o que reforça a multicolinearidade e abre a hipótese de causalidade reversa.
3. **TV, display e Google Ads têm o formato mais próximo do das vendas.** Pela distância DTW (janela de 8 semanas), esses três canais formam o grupo mais parecido com a série de vendas, enquanto vídeo, mala direta, áudio streaming, e-mail, jornal e SMS ficam mais distantes.
4. **Google Ads provavelmente registra cliques.** O custo por unidade é compatível com custo por clique, e não por mil impressões, o que exige cuidado para não superestimar o efeito de um canal de fundo de funil.
5. **O mix muda ao longo do tempo.** O jornal perde investimento a partir de 2016 e quase desaparece em 2018, enquanto mídia social e display crescem. A mídia social também tem investimento sem impressões no primeiro ano da série.

![Série de vendas e decomposição sazonal](imagens/decomposicao_vendas.png)

![Similaridade DTW entre canais e vendas](imagens/dtw_canais.png)

## 5.3 Preparação dos dados e busca de hiperparâmetros

Antes de entrar na regressão, cada canal passa por duas transformações que representam a resposta à mídia.

```
Impressões → Adstock (α) → Saturação de Hill (K, S) → Variável explicativa na Ridge (λ)
```

Os parâmetros dessas transformações, a memória do adstock (α), o ponto de meia saturação (K) e a inclinação da curva (S), além da penalização da Ridge (λ), não são observados nos dados e não têm um valor universal. Eles mudam de mercado para mercado, de empresa para empresa e de canal para canal, conforme a categoria de produto, a região e a forma como a empresa planeja a veiculação. Como não existe um valor correto conhecido de antemão, os parâmetros precisam ser estimados a partir do histórico do próprio negócio.

A estimação é feita por combinação. Cada conjunto candidato de parâmetros gera uma matriz de variáveis transformadas, a Ridge é ajustada sobre ela e o resultado é avaliado no período de validação. Combinações diferentes podem produzir modelos com erro alto ou baixo, e também decomposições mais ou menos plausíveis, então a busca precisa olhar para os dois aspectos.

Para definir os candidatos, o trabalho se apoia nas faixas recomendadas pelo Robyn, da Meta, para cada grupo de mídia, e compara três caminhos.

| Caminho | Estratégia | Característica |
|---|---|---|
| 1 | Grid search com S = 1 | Combina valores prefixados de α, K e λ. É o caminho mais simples, mas não permite saturação em formato de S |
| 2 | Grid search com S livre | Inclui a inclinação na grade, ao custo de um número muito maior de combinações |
| 3 | Nevergrad (TwoPointsDE) | Algoritmo evolutivo usado pelo Robyn, que gera novos candidatos a partir dos melhores em vez de testar todas as combinações |

Os dois primeiros caminhos escolhem a combinação pelo menor erro de previsão. O Nevergrad, seguindo o Robyn, otimiza duas métricas ao mesmo tempo.

- **NRMSE** (raiz do erro quadrático médio normalizada), que mede o quanto as vendas previstas se aproximam das vendas reais.
- **DECOMP.RSSD**, que mede a distância entre a parcela do efeito atribuída a cada canal e a parcela do investimento que ele recebeu. A métrica evita soluções que ajustam bem a curva de vendas, mas concentram todo o crédito em poucos canais e zeram os demais, o que seria difícil de defender diante da área de negócios.

Os três caminhos são comparados no período de teste, e a escolha final considera o ajuste e a plausibilidade da decomposição.

## 5.4 Modelagem

- **Tendência.** Extraída com o Prophet e incluída como controle.
- **Validação temporal.** As últimas 52 semanas ficam reservadas para teste, e o treino usa as 157 anteriores.
- **Padronização.** Ajustada apenas no treino e aplicada ao teste.
- **Regularização.** O λ da Ridge é escolhido por busca em grade, com coeficientes restritos a valores não negativos.

## 5.5 Avaliação

| Critério | Métrica | Referência |
|---|---|---|
| Ajuste | R² e NRMSE fora da amostra | R² alto e NRMSE baixo |
| Plausibilidade | Coeficientes de mídia não negativos e adstock dentro das faixas da literatura | Qualitativo |
| Coerência da decomposição | DECOMP.RSSD entre a divisão do efeito e a do investimento | Quanto menor, mais coerente |
| Resíduos | Durbin-Watson | Próximo de 2 |

**Versão de referência (preliminar).** Uma primeira especificação, com hiperparâmetros fixos (α = 0,3, S = 1,2 e K na mediana de cada canal), serve de ponto de partida para a busca.

| Métrica | Treino | Teste |
|---|---|---|
| R² | 0,68 | 0,52 |
| NRMSE | 0,10 | 0,12 |
| DECOMP.RSSD | 0,37 | |
| Durbin-Watson (série completa) | 2,32 | |

A decomposição por canal, o ROAS e as curvas de mROAS dessa versão estão em revisão e serão publicados junto com o modelo final.

# 6. Status do Projeto

| Fase CRISP-DM | Status |
|---|---|
| 1. Entendimento do negócio | ✅ Concluída |
| 2. Entendimento dos dados (EDA) | 🔄 Em finalização (variáveis categóricas e síntese) |
| 3. Preparação dos dados | 🔄 Em andamento (adstock e saturação) |
| 4. Modelagem | 🔄 Versão de referência ajustada, busca de hiperparâmetros pendente |
| 5. Avaliação | ⏳ Planejada |
| 6. Implantação (simulação de orçamento) | ⏳ Planejada |

# 7. Escopo e Limitações

- O trabalho se concentra na abordagem **frequentista**. O MMM bayesiano fica como trabalho futuro, embora compartilhe a mesma estrutura de adstock, saturação e decomposição.
- O modelo estima efeitos a partir de dados observacionais e **sob as premissas da especificação**. A validação causal dos resultados exige experimentos, como estudos de geolift.
- As simulações de orçamento valem dentro da faixa de investimento observada no histórico e sob a hipótese de continuidade das condições de mercado. Elas estimam a venda associada a um mix de mídia e não preveem a venda de uma semana específica.

# 8. Como Reproduzir

1. Abra (https://colab.research.google.com/drive/13WtGNoEjU4wO7Hn7ZbPYLYL68NgsSPud?usp=sharing) no Google Colab.
2. Execute as células na ordem. A base é carregada diretamente do repositório público do mmm_stan, sem download manual.

# 9. Referências

- CHAN, D.; PERRY, M. *Challenges and Opportunities in Media Mix Modeling*. Google Inc., 2017.
- FOGARTY, D. J.; BHADURI, S.; SRINIVASAN, R. Marketing mix modelling 4.0: the superiority of agentic, Bayesian optimised marketing mix modelling over traditional approaches. *Applied Marketing Analytics*, v. 11, n. 3, p. 256-266, 2025.
- HANSSENS, D. M.; PARSONS, L. J.; SCHULTZ, R. L. *Market Response Models: Econometric and Time Series Analysis*. 2. ed. Boston: Kluwer Academic, 2001.
- JIN, Y. et al. *Bayesian Methods for Media Mix Modeling with Carryover and Shape Effects*. Google Inc., 2017.
- META. *An Analyst's Guide to MMM*. Documentação do Robyn.
- HE, S. *mmm_stan*. Repositório GitHub, 2021. Disponível em https://github.com/sibylhe/mmm_stan.

# 10. Licença

- **Dados.** Base do repositório mmm_stan, publicada sob licença MIT.
- **Código.** MIT.

# Autor

**Ciro Neves** · [LinkedIn](https://www.linkedin.com/in/SEU-PERFIL) · [GitHub](https://github.com/Cironeves5) 
