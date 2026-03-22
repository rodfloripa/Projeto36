
<p align="justify"><h3>Entendendo o Artigo  <a href="https://arxiv.org/pdf/2508.12278">CRoC:Detecção de Anomalias em Grafos</a></h1></p>

<p align="justify">O artigo <b>CRoC (Context Refactoring Contrast)</b> propõe uma solução inovadora para um desafio clássico na ciência de dados: como identificar o "estranho no ninho" dentro de uma rede complexa (como redes sociais ou sistemas bancários) quando temos pouquíssimos exemplos do que é uma anomalia. O foco aqui é a <b>Detecção de Anomalias em Grafos (GAD)</b> com supervisão limitada.</p>

<p align="justify"><h3>1. O Problema: A Camuflagem das Anomalias</h3></p>

<p align="justify">Em um grafo, as anomalias são astutas. Um fraudador, por exemplo, não age isoladamente; ele tenta se conectar a usuários legítimos para parecer um "nó normal". Se você analisar apenas as conexões (estrutura) ou apenas o comportamento individual (atributos), ele pode passar despercebido. O CRoC parte da premissa de que a verdadeira anomalia reside na <b>inconsistência entre o nó e o seu contexto</b>.</p>

<p align="justify"><h3>2. A Solução: Refatoração de Contexto</h3></p>

<p align="justify">Imagine que você está tentando validar a identidade de uma pessoa perguntando sobre ela para seus vizinhos. O processo de "Refatoração de Contexto" no CRoC funciona de forma similar:</p>

<p align="justify"><b>Isolamento e Reconstrução:</b> O algoritmo pega um nó e observa sua vizinhança. Ele então "perturba" ou mascara as informações desses vizinhos (refatoração).</p>

<p align="justify"><b>Previsão de Expectativa:</b> O modelo tenta prever como o nó central deveria ser, baseando-se puramente nesse contexto modificado.</p>

<p align="justify"><h3>3. O Mecanismo: Aprendizado Contrastivo</h3></p>

<p align="justify">A "mágica" acontece através de um duelo de representações. O modelo gera duas visões para cada nó:</p>

<p align="center">
<b>Visão Real (Z) vs. Visão de Contexto (Z')</b>
</p>

<p align="justify">Usando uma função de perda chamada <b>InfoNCE</b>, o sistema aprende que, para nós normais, essas duas visões devem estar muito próximas (alta similaridade). Para anomalias, a distância entre o que o nó <i>é</i> e o que o contexto <i>diz que ele deveria ser</i> é muito grande. Esse "contraste" é o que permite ao modelo apontar, com precisão, quem não pertence àquele ambiente.</p>

<p align="justify">Em resumo, o CRoC não tenta apenas decorar padrões; ele aprende a <b>lógica de vizinhança</b>. Se um nó quebra essa lógica, ele é marcado como uma anomalia, mesmo que o modelo nunca tenha visto um exemplo de fraude antes.</p>

Vamos quebrar o código do CRoC em blocos funcionais e explicar o que cada linha "faz na prática".
---
Aqui estão as partes vitais do motor de detecção de anomalias:

<p align="justify"><h3>1. Injeção de Anomalias (O "Crime")</h3></p>

<p align="justify">Para o modelo aprender, precisamos de alvos. Aqui, pegamos nós aleatórios e "sujamos" seus atributos com ruído (valores entre 0 e 1) para que eles não combinem mais com seus vizinhos originais.</p>

```python

#Seleciona 100 índices aleatórios do Cora

anomaly_idx = np.random.choice(data.num_nodes, 100, replace=False)

#Substitui o vetor de características desses nós por lixo (ruído)

data.x[anomaly_idx] = torch.rand(100, data.x.size(1))
```

<p align="justify"><h3>2. O Encoder GCN (O "Observador")</h3></p>

<p align="justify">Esta classe define como o modelo "enxerga" o grafo. A camada <code>GCNConv</code> faz a soma ponderada dos vizinhos, criando um resumo do contexto de cada nó.</p>

```python

#Camada 1: Agrega informações dos vizinhos diretos

self.conv1 = GCNConv(in_dim, hid_dim) 

#Camada 2: Refina a representação (capta vizinhos de 2º nível)

self.conv2 = GCNConv(hid_dim, hid_dim)

#Projetor: Transforma o resumo em um espaço matemático para comparação

self.projector = nn.Linear(hid_dim, hid_dim)
```


<p align="justify"><h3>3. Refatoração de Contexto (A "Dúvida")</h3></p>

<p align="justify">No treinamento, criamos uma versão "mutilada" dos dados (<code>x_ref</code>). O objetivo é ver se o modelo consegue manter o nó no mesmo lugar mesmo se faltarem pedaços da informação.​O data.x é um tensor (uma matriz) com o formato [Número de Nós, Número de Atributos].</p>

```python

#Criamos uma máscara binária (0 ou 1) onde 10% dos dados são zerados

mask = (torch.rand_like(data.x) > 0.1).float()

#Aplicamos a máscara para gerar a visão "refatorada"

x_ref = data.x * mask
```

<p align="justify"><h3>4. Perda Contrastiva InfoNCE (O "Ajuste")</h3></p>

<p align="justify">Esta é a lógica de recompensa. Se o nó original e a sua versão refatorada estiverem próximos, a perda é baixa. Se estiverem longe, o modelo é penalizado.Z1 e Z2 são os 'embeddings' do nó.</p>

```python

#Similaridade entre o real (z1) e o reconstruído (z2)

pos_sim = torch.exp(torch.sum(z1 * z2, dim=-1) / temp)

#Similaridade com TODOS os outros nós do grafo (ruído de fundo)

all_sim = torch.exp(torch.mm(z1, z2.t()) / temp).sum(dim=-1)

#Perda: Queremos maximizar a parte de cima (parceiro real) e minimizar a de baixo (outros nós)

loss = -torch.log(pos_sim / all_sim).mean()
```

<p align="justify"><h3>5. Cálculo do Score Final (A "Sentença")</h3></p>

<p align="justify">Após o treino, comparamos o que o nó <b>é</b> com o que o modelo <b>esperava que ele fosse</b> se não tivesse nenhum atributo próprio (apenas o contexto dos vizinhos).</p>

```python

#Calculamos a distância entre o nó real e o nó "vazio".
#Anomalias terão uma distância (norma) muito maior

scores = torch.norm(z_original - z_contexto_puro, dim=1)
```

<p align="justify">Esses blocos formam o ciclo completo do CRoC: <b>Observar -> Perturbar -> Comparar -> Detectar</b>. </p>
