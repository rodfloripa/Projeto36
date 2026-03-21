<p align="justify"><h1>Artigo: <a href="https://arxiv.org/abs/2501.11730">Previsão de vibração com transformadores para o avanço da segurança ferroviária e da manutenção 4.0</a></h1></p>

<p align="justify">A implementação de um sistema de monitoramento para a <b>Manutenção 4.0</b> utiliza o poder dos Transformers para aprender o comportamento dinâmico "saudável" de uma infraestrutura ferroviária. Ao contrário de métodos estatísticos simples, o modelo de Deep Learning consegue capturar dependências temporais complexas, permitindo distinguir entre o ruído operacional comum e falhas estruturais críticas. Abaixo, detalhamos os blocos fundamentais do código e a interpretação dos resultados.</p>

<p align="justify"><h3>1. Geração de Sinais: O Gêmeo Digital do Trilho</h3></p>

<p align="justify">Para validar o modelo, simulamos a física de um ambiente ferroviário. O <b>Sinal Normal</b> é composto por frequências fundamentais estáveis. Já o <b>Sinal Anômalo</b> introduz transientes não-lineares: impactos súbitos com decaimento exponencial e alta frequência.</p>

```python

# Sinal Normal: Composição de harmónicas (10Hz e 50Hz) + Ruído Gaussiano
normal_signal = 0.6 * np.sin(2 * np.pi * 10 * t) + 0.3 * np.sin(2 * np.pi * 50 * t) + np.random.normal(0, 0.05, n_points)

# Sinal Anómalo: Injeção de impacto transiente de alta frequência (150Hz)
impact = 2.5 * np.exp(-50 * (t_anomaly - t_anomaly[0])) * np.sin(2 * np.pi * 150 * t_anomaly)
anomalous_signal[start:start+duration] += impact
```
<p align="justify"><h3>2. A Arquitetura: Transformer para Séries Temporais</h3></p>

<p align="justify">O coração do projeto é a classe <code>RailVibrationTransformer</code>. Ela utiliza o mecanismo de <b>Self-Attention</b> para analisar uma janela de dados. O modelo aprende a correlação entre todos os pontos da janela simultaneamente para filtrar a normalidade.</p>

```Python

# O mecanismo de atenção permite focar em diferentes frequências do sinal
encoder_layers = nn.TransformerEncoderLayer(d_model=64, nhead=4, batch_first=True)

# A saída linear projeta o resumo do contexto para o próximo valor escalar de aceleração (g)
self.output_fc = nn.Linear(64, 1)
``` 
<p align="justify"><h3>3. O Motor de Detecção: Erro de Reconstrução</h3></p>

<p align="justify">A detecção não ocorre por classificação direta, mas através da métrica de <b>Resíduo</b>. O modelo prevê o próximo ponto; se o sinal for normal, a previsão é precisa. Se for anómalo, o erro dispara.</p>

```Python

# Calculamos a distância absoluta entre o que aconteceu e o que o Transformer esperava
error = torch.abs(predictions - y_test).numpy()

# Definimos um Limiar (Threshold) estatístico baseado no sinal saudável
# 300 pontos é o período de calibração com sinal saudável
threshold = np.mean(error[:300]) + 4 * np.std(error[:300])
```

<p align="justify"><h3>4. Adaptabilidade: Limiar Dinâmico (Rolling Window)</h3></p>

<p align="justify">Em um cenário real, como um trem percorrendo diferentes terrenos  a vibração natural muda drasticamente ao passar por uma ponte, um túnel ou uma curva fechada. Um valor fixo de calibração (como os primeiros 300 pontos) causaria alarmes falsos. Para resolver isso, implementamos uma <b>Janela Móvel (Rolling Window)</b>, onde o "normal" é recalculado constantemente com base no histórico recente:</p>

```Python

# O Threshold se adapta ao terreno (Ex: Ponte vs. Reta)
# Calcula a média e desvio padrão dos últimos 500 pontos
window = 500
threshold_dinamico = erro.rolling(window=window).mean() + 4 * erro.rolling(window=window).std()
```

<p align="justify">Essa técnica de <b>Estatística Adaptativa</b> permite que o Transformer ignore mudanças graduais no ambiente e foque apenas em picos de energia que fogem do padrão estatístico imediato da via. No nosso código simplificado, o valor de 300 serviu como o período de calibração inicial, mas em produção, a janela móvel é o que garante a robustez do sistema.</p>

<p align="justify"><h3>4. Conclusão e Análise das Figuras</h3></p>

<p align="justify">Ao observar os gráficos gerados, a eficácia do método torna-se evidente:</p>

<p align="center">
<img src="https://github.com/rodfloripa/Projeto37/blob/main/fig1.png">
</p>

<p align="justify">
<ul>
<li><b>Figura Superior (Sinal de Aceleração):</b> Mostra que o sinal de vibração é altamente ruidoso. As áreas destacadas indicam onde o comportamento transiente rompe a ciclicidade esperada do sistema.</li>
<li><b>Figura Inferior (Erro Residual):</b> É onde a inteligência reside. O erro permanece baixo durante a operação normal e apresenta picos agudos exatamente nos eventos de falha, cruzando a barreira de segurança.</li>
</ul>
</p>

<p align="justify">Em suma, a aplicação de Transformers para previsão de vibração transforma dados brutos de sensores em <b>informação acionável</b>. Este projeto demonstra que é possível automatizar a vigilância de ativos ferroviários, prevenindo acidentes através da identificação precoce de comportamentos anômalos.</p>
