# Decisões técnicas

Fomos anotando aqui as decisões do projeto conforme elas apareciam: o que
escolhemos, por que, em qual aula nos apoiamos e, quando deu para medir, o
número que sustentou a escolha. Também registramos o que deu errado no caminho,
porque boa parte do que aprendemos veio daí. Este documento serviu de base para
o README e para o roteiro do vídeo.

---

## D1 — Gemini como modelo de linguagem

Usamos o Gemini, do Google, como LLM do pipeline. A escolha de modelos e
tecnologias ficava a critério da equipe. As aulas usam OpenAI nos exemplos de
código, mas a 1IAST Aula 2 cita o stack do Google Cloud (Vertex AI) entre as
opções de RAG gerenciado. Pesou também o fato de o Gemini ter camada gratuita:
qualquer pessoa consegue reproduzir o trabalho sem pagar nada.

---

## D2 — Qual versão do modelo

Ficamos com o `gemini-3.5-flash`. Antes disso testamos algumas versões com uma
pergunta de abstenção: mostramos duas avaliações que não falavam de cashback e
perguntamos sobre cashback.

| Modelo | O que aconteceu |
|---|---|
| `gemini-2.5-flash` | erro 404: aparece na listagem, mas "no longer available to new users" |
| `gemini-3.5-flash` | recusou corretamente, em 2,1 s |
| `gemini-3.8-flash` | recusou corretamente, mas levou 6,6 s, três vezes mais sem ganho visível |
| `gemini-3.5-flash-lite` | recusou corretamente, em 1,2 s |

Uma surpresa: pedíamos o `gemini-3.5-flash`, mas as mensagens de erro de cota
falavam em `gemini-3.6-flash`. Ou seja, a API redireciona versões por conta
própria, e fixar o nome do modelo no código não garante que o modelo seja
sempre o mesmo. O comportamento pode mudar sem mudarmos nada. Deixamos isso
registrado como limitação de reprodutibilidade.

---

## D3 — Os limites da camada gratuita

Esses limites acabaram mudando a arquitetura. Tiramos os valores do próprio
erro devolvido pela API:

| Limite | Valor | Identificador |
|---|---|---|
| Requisições por minuto | 5 | `GenerateRequestsPerMinutePerProjectPerModel-FreeTier` |
| Requisições por dia | 20 | `GenerateRequestsPerDayPerProjectPerModel-FreeTier` |

Os dois valem por modelo e por projeto.

Na prática, cada pergunta faz de 3 a 4 chamadas ao LLM (HyDE, julgamento de
relevância e geração, mais a reescrita quando precisa). Isso dá umas 5 ou 6
perguntas por dia. Na nossa primeira bateria completa, 7 de 15 perguntas
terminaram; as outras 8 pararam porque a cota do dia acabou, não por erro no
código.

Daí tiramos três conclusões:

1. Rodar o RAGAS só com o Gemini gratuito não era viável, porque ele faz dezenas
   de chamadas por amostra.
2. Quem fosse reproduzir o notebook com uma chave gratuita travaria lá pela
   sexta pergunta.
3. Até para desenvolver ficava caro: cada teste consumia boa parte da cota do dia.

Testamos então, com uma requisição por modelo direto na API REST, quais modelos
têm cota própria:

| Modelo pedido | Resultado | Observação |
|---|---|---|
| `gemini-3.5-flash` | 429 | cota do dia esgotada; o limite é compartilhado com o `gemini-3.6-flash` |
| `gemini-3.5-flash-lite` | ok | cota própria |
| `gemini-3.1-flash-lite` | ok | cota própria |
| `gemini-flash-lite-latest` | ok | respondeu como `3.5-flash-lite`: é só um apelido e divide a mesma cota |
| `gemini-2.5-flash-lite` | 404 | indisponível para contas novas, como o `2.5-flash` |
| `gemini-3.8-flash` | timeout | inconclusivo |

Os apelidos `-latest` não aumentam a cota, já que apontam para um modelo que já
está na lista. Por outro lado, a mesma chave tem pelo menos três cotas
independentes, então trocar de modelo Gemini quando um esgota ajuda de verdade.

A principal lição foi que o número de chamadas ao LLM por pergunta é uma decisão
de projeto. Cada etapa que usa o modelo precisa valer o que custa, e foi isso
que nos levou às mitigações da D4.

---

## D4 — Como contornamos os limites

Todas as saídas vêm de padrões vistos nas aulas.

| Mitigação | O que faz | Aula |
|---|---|---|
| Modelo mais leve nas etapas auxiliares | HyDE e reescrita usam um modelo *lite*, que tem cota própria; julgamento e geração ficam no modelo principal | POSTECH 2, redução de custo com modelo menor |
| `.with_fallbacks()` | quando a cota estoura, a cadeia passa sozinha para outro modelo | POSTECH 2: "Se a API retornar Rate Limit, a chain alterna automaticamente para outro modelo" |
| Cache das respostas do LLM | rodar o notebook de novo não gasta cota | POSTECH 4, caching |
| Medir o HyDE | se não ajudasse, sairia e pouparia uma chamada por pergunta | — |
| Primeira camada de abstenção sem LLM | perguntas fora do tema são barradas antes de qualquer chamada | 1IAST 2 |

O último item já se pagou: na bateria, as perguntas fora do tema foram barradas
em 0,3 s sem gastar nenhuma requisição.

---

## D5 — Groq como reserva e como juiz da avaliação

Colocamos o Groq como segundo provedor, com dois papéis: assumir quando as cotas
do Gemini acabam e servir de juiz nas métricas do RAGAS.

O Groq não aparece nas aulas, então vale explicar por que achamos a escolha
defensável:

1. O padrão em si é ensinado. Na POSTECH Aula 2: "Continuidade de Serviço: se a
   API da OpenAI retornar um erro 500 ou Rate Limit, a chain alterna
   automaticamente para um modelo no Azure ou Anthropic." Trocar de provedor por
   limite de taxa é exatamente o nosso caso.
2. A escolha do provedor era livre.
3. A motivação foi um limite que medimos (D3), não gosto pessoal.

Limites que lemos nos cabeçalhos `x-ratelimit-*` das respostas:

| | Gemini (gratuito) | Groq (gratuito) |
|---|---|---|
| Requisições/dia | 20 por modelo | 1.000 por modelo |
| Limite por minuto | 5 requisições | 8.000 tokens |
| Teste de abstenção | ok | ok nos três modelos testados |
| Latência no teste | 2,1 s | 0,2 a 0,7 s |
| Tokens/dia | — | 200.000 por modelo (descobrimos depois) |

Esse último limite nos pegou de surpresa. Os cabeçalhos mostravam requisições por
dia e tokens por minuto, mas o Groq também limita tokens por dia: 200 mil no
`gpt-oss-120b`. O RAGAS faz várias chamadas longas por resposta (extrai
afirmações, verifica cada uma, gera perguntas, julga cada contexto), e estouramos
esse teto bem antes das 1.000 requisições. A API foi clara: "Rate limit reached
... on tokens per day (TPD): Limit 200000". Desde então rodamos o RAGAS num dia
com a cota inteira e não usamos o mesmo modelo para outros testes no mesmo dia.
No fim, são três limites a respeitar: requisições por dia, tokens por minuto e
tokens por dia.

Os modelos disponíveis na nossa conta eram `openai/gpt-oss-120b`,
`openai/gpt-oss-20b`, `qwen/qwen3.8-27b` e `allam-2-7b`. Esperávamos encontrar
modelos Llama, mas não havia. O `gpt-oss` é o modelo de pesos abertos da OpenAI,
o mesmo fornecedor dos exemplos de código das aulas.

Usar o Groq também como juiz do RAGAS tem outra vantagem: avaliar o Gemini com um
modelo de outra família diminui a chance de o juiz favorecer respostas parecidas
com as que ele mesmo daria. É o LLM-as-a-Judge da 1IAST Aula 2, "uso de modelos
de linguagem poderosos para avaliar as respostas de modelos menores": aqui, um
modelo de 120B avaliando um modelo *flash*.

Mesmo com uma cota 50 vezes menor, mantivemos o Gemini como principal, porque foi
a escolha da equipe (D1) e a arquitetura não depende dela: o provedor é só
configuração. Se a cota do Gemini atrapalhar, inverter a ordem é mudar duas
linhas em `config.py`.

Como ficaram as cadeias:

    geração e julgamento:  gemini-3.5-flash -> gemini-3.1-flash-lite -> groq gpt-oss-120b -> groq gpt-oss-20b
    HyDE e reescrita:      gemini-3.5-flash-lite -> groq gpt-oss-20b
    juiz do RAGAS:         groq gpt-oss-120b

---

## Decisões da construção da base e da busca

### D6 — Sem chunking
O comentário mediano tem 53 caracteres e o maior tem 208. A 1IAST Aula 2 explica
que o chunking existe porque "documentos extensos raramente cabem inteiros na
janela de contexto", e avisa que "chunks curtos demais podem fragmentar contexto
essencial". Aqui cada avaliação já é a menor unidade que faz sentido sozinha.

### D7 — Dois pré-processamentos
Seguimos a Documento Aula 2, que sugere manter "duas versões do pipeline: uma
agressiva para BoW/TF-IDF, outra leve para modelos contextuais". Nos dois casos
não removemos negações: entre "chegou" e "não chegou" está a reclamação inteira.

### D8 — Duplicatas viram contagem
Juntamos 4.228 avaliações repetidas, mas guardamos quantas vezes cada uma
apareceu. "Relatado por 47 clientes" entra no contexto do LLM como sinal de que
o problema se repete.

Um limite que encontramos depois: o filtro de tamanho mínimo (12 caracteres) não
olha o conteúdo. Uma "avaliação" com 19 asteriscos (`*******************`,
repetida por 5 clientes) passou e virou o `doc_000000`. Varremos a base e ela é o
único documento sem nenhuma letra (os outros 10 com até 8 letras são legítimos,
como "Muito bom .." e "Tudo ok, 100 %."). Tirá-la mudaria a numeração de todos os
`doc_id` e obrigaria a treinar os tópicos de novo. Por 1 documento em 33.332,
preferimos manter e deixar anotado. Numa próxima versão, a regra seria exigir
pelo menos uma letra.

### D9 — Embeddings `paraphrase-multilingual-MiniLM-L12-v2`
A Documento Aula 4 recomenda "Sentence-BERT em português", e esse modelo aparece
no código da Aula 6. Ele indexou os 33.332 documentos em 141 s; o que usávamos
antes levava 13 minutos.

### D10 — ChromaDB
É o banco do hands-on da POSTECH Aula 3 e permite filtrar por metadado antes da
busca, o que a 1IAST Aula 2 chama de "filtragem pré-recuperação".

### D11 — Busca híbrida com pesos medidos
BM25 e busca densa, fundidas por RRF com c=60 (1IAST Aula 2). Para os pesos,
testamos várias combinações em vez de chutar:

| denso/léxico | 1,0/0,0 | 0,9/0,1 | 0,7/0,3 | 0,5/0,5 | 0,0/1,0 |
|---|---|---|---|---|---|
| precisão média | 0,800 | 0,833 | **0,867** | 0,733 | 0,700 |

### D12 — O Cross-Encoder ordena, mas não serve para medir confiança
A ideia inicial era usar a nota do Cross-Encoder para decidir quando recusar.
Medimos e não funcionou: "Quais são os sintomas da gripe?" teve nota maior que
quatro perguntas legítimas. O modelo foi treinado para avaliar se um trecho
responde à pergunta, e perguntas agregadas não são respondidas por nenhuma
avaliação sozinha. Ele ficou no pipeline só para ordenar os resultados, e aí
ajudou bastante: a precisão em "elogios ao atendimento" foi de 0,4 para 0,8.

### D13 — Abstenção em três camadas
| Camada | O que verifica | Como |
|---|---|---|
| 1. Escopo | a pergunta é sobre avaliações? | similaridade do bi-encoder, sem LLM |
| 2. Suficiência | o que foi recuperado sustenta uma resposta? | o LLM julga cada evidência, reescreve a busca ou desiste |
| 3. Geração | a resposta cita o que foi recuperado? | campo `evidencia_suficiente` e validação das citações |

O limiar da camada 1 é 0,48 (dá para mudar em `LIMIAR_ESCOPO`).

Chegamos a esse valor depois de errar. A primeira calibração usou 32 perguntas e
fixou 0,53, que aceitava 16 de 16 legítimas e barrava 13 de 16 fora do tema.
Só que, na bateria de validação, a pergunta "Quais problemas aparecem com maior
frequência nas avaliações?" foi recusada como fora do escopo, com 0,510. Perguntas
genéricas ("problemas", "avaliações") são menos parecidas com avaliações
concretas ("o relógio veio quebrado") do que perguntas específicas, e a nossa
calibração não tinha esse tipo de pergunta.

Refizemos a calibração com 50 perguntas, agora incluindo as 18 perguntas de
negócio que o sistema precisava responder. As duas com menor nota ficaram em
0,509 e 0,510. Com 0,48:

| | 0,53 (antes) | 0,48 (agora) |
|---|---|---|
| Perguntas de negócio aceitas | 16/18 | 18/18 (folga de 0,029) |
| Fora do tema barradas na camada 1 | 13/16 | 8/16 |

O preço é a camada 1 barrar menos. Testamos uma a uma as cinco perguntas fora do
tema que passaram a atravessá-la (cotação do dólar, Copa do Mundo, previsão do
tempo, passagem aérea e imposto de renda), e todas foram barradas na camada 2: o
juiz não acha avaliação relevante e o sistema não responde. O custo é uma chamada
a mais ao LLM.

Achamos a troca justa. Recusar uma pergunta legítima é um erro que o usuário vê
e vai contra o objetivo do sistema; deixar uma pergunta fora do tema seguir até a
camada 2 custa uma chamada e termina na mesma recusa. A camada 1 é um filtro
barato, não a única defesa.

O que ficou de aprendizado: calibrar com as perguntas que vão ser feitas de
verdade. A amostra que achávamos representativa não tinha perguntas genéricas de
frequência.

### D14 — O primeiro gabarito de avaliação não servia
No começo avaliávamos a busca usando o título da avaliação como consulta e o
texto como resposta esperada. Deu Recall@25 de 4,5%. Fomos olhar e vimos por quê:
"Caneta tinteiro crown" (título) → "Satisfeito, recebi antes do prazo" (texto). O
título fala do produto e o texto fala da experiência; estávamos medindo uma
tarefa impossível, e não a qualidade da busca. Trocamos por um gabarito baseado
em metadados, que não aparecem no texto.

---

## Perguntas de frequência

### D15 — O prompt só deixa falar em frequência quando há contagem

Na primeira bateria, o modelo escreveu que "agilidade na entrega é o aspecto mais
frequentemente citado" depois de ver 10 avaliações de 33 mil. Na segunda, falou
em "reclamações recorrentes sobre os Correios" com base em duas.

Por isso a regra 5 do prompt da rota RAG proíbe "recorrente", "frequente",
"comum", "maioria", "principal", "muitos" e "predominante", a não ser que a
avaliação traga `relatada_por` com o número de clientes, e nesse caso o número
tem que aparecer. No lugar, sugerimos "há relatos de" ou "aparecem casos de".

Isso resolve o exagero, mas ainda não responde à pergunta de frequência. Para
isso fizemos a D16.

### D16 — Tópicos com BERTopic

Perguntas como "quais problemas aparecem com maior frequência?" ou "quais
reclamações de entrega são mais frequentes?" pedem contagem, e o RAG não conta:
ele traz exemplos. A Documento Aula 6 mostra esse uso: "BERTopic é usado para
indexar e organizar a base de conhecimento" em pipelines de RAG.

Aplicamos o pipeline da aula (embeddings → UMAP → HDBSCAN → c-TF-IDF) às 33.332
avaliações, reaproveitando os embeddings que já estavam no ChromaDB. Cada
avaliação ganhou um tópico, gravado como metadado no Chroma.

Testamos seis configurações, sempre com a entrada ordenada por `doc_id` (D18):

| Método | Tam. mínimo | Tópicos | Sem tópico | c_v | Maior tópico |
|---|---|---|---|---|---|
| **eom** | **50** | **85** | **44,2%** | **0,622** | **6,4%** |
| eom | 100 | 34 | 31,3% | 0,558 | 27,0% |
| eom | 200 | 10 | 16,0% | 0,485 | 43,4% |
| leaf | 50 | 100 | 53,4% | 0,642 | 2,6% |
| leaf | 100 | 52 | 56,8% | 0,606 | 4,1% |
| leaf | 200 | 21 | 53,1% | 0,556 | 7,6% |

Escolhemos eom/50 por três motivos:

1. A coerência de 0,622 fica acima do 0,5 que a aula indica como interpretável.
2. Os tópicos ficam equilibrados: o maior tem 6,4% das avaliações. No eom/100 um
   único tópico fica com 27%, e no eom/200, com 43%.
3. O resultado praticamente não muda com a ordem da entrada (D18).

O leaf/50 tem coerência um pouco maior (0,642), mas deixa 9 pontos percentuais a
mais de avaliações sem tópico. Para contar frequência, cobertura importa mais que
dois centésimos de coerência, porque avaliação sem tópico fica fora da conta.

O HDBSCAN deixa 44,2% das avaliações sem tópico. Elas só voltam para um tópico se
a similaridade com o centróide for de pelo menos 0,70. Olhando os casos na
fronteira, deu para entender o corte:

| Similaridade | Avaliação | Tópico mais próximo | |
|---|---|---|---|
| 0,71 | "1 dos produtos (o cavalo) não veio" | produto não entregue | ✓ |
| 0,70 | "Entregou antes do prazo" | antes do prazo | ✓ |
| 0,68 | "A barraca veio com falhas na costura" | peça faltando | ✗ |

Com 0,70, as avaliações sem tópico caem de 44,2% para 17,5%. As que sobram
continuam na conta como "não atribuídas", não desaparecem.

Os nomes dos 85 tópicos foram gerados pelo `gpt-oss-120b` (Groq), em lotes. Como
um lote não vê o outro, o modelo repetiu nomes: "Produto não recebido" aparecia
em três tópicos diferentes. Uma segunda passada encontra os nomes repetidos e
pede nomes que diferenciem os tópicos com base nos exemplos. No fim não sobrou
nenhum nome repetido.

Uma curiosidade dos dados: há um tópico chamado "Loja Lannister". A Olist
anonimizou os vendedores com nomes de casas de *Game of Thrones*, então não é
erro.

### D17 — Na rota de frequência, o LLM escolhe e o código conta

Perguntas de frequência seguem um caminho próprio, com uma regra fixa: o modelo
de linguagem nunca escreve número.

    1. o código separa as avaliações pela nota: reclamação = nota 1 ou 2,
       elogio = nota 4 ou 5 (pelo metadado, não pela leitura do texto)
    2. o código monta a tabela de tópicos com as contagens reais, somando as
       repetições que a deduplicação juntou
    3. para cada tópico, busca exemplos dentro dele, os mais próximos da pergunta
    4. o LLM escolhe quais tópicos respondem à pergunta e escreve a síntese, sem números
    5. o código coloca as contagens

Toda resposta passa por três verificações automáticas: nenhum tópico fora da
tabela, nenhuma citação fora dos exemplos mostrados e nenhum número no texto do
modelo.

O roteamento é feito por regra, e não pelo LLM. Classificar a intenção com o
modelo custaria uma chamada por pergunta, com uma cota de 20 por dia (D3). Na
prática é uma Router Chain (POSTECH Aula 2) feita como aresta condicional do
grafo.

Na linguagem das aulas, isso é a indexação hierárquica da 1IAST Aula 2: "resumos
indexados em um nível superior e chunks detalhados em um nível inferior,
combinando visão macro e micro". Os tópicos são o nível macro (quanto) e as
avaliações, o micro (o quê).

De quebra, a rota de frequência faz uma única chamada ao LLM, enquanto a rota RAG
faz de 3 a 4.

### D18 — A ordem da entrada muda os tópicos

A primeira construção deu 84 tópicos. Depois que gravamos o tópico de cada
avaliação no Chroma, reconstruímos e saíram 85, com outros nomes, usando a mesma
semente do UMAP.

O motivo: o Chroma passou a devolver os documentos em outra ordem, e o UMAP é
sensível à ordem da entrada. Mesma semente com ordem diferente dá resultado
diferente.

Passamos a ordenar os documentos por `doc_id` antes do treino. Rodamos duas vezes
seguidas e as atribuições saíram iguais nas 33.332 avaliações.

Isso também nos fez corrigir uma conclusão. Na ordem antiga, o eom/100 tinha dado
3 tópicos, um deles com 98% das avaliações, e chegamos a descrever isso como
"colapso". Com a entrada ordenada, o mesmo eom/100 dá 34 tópicos. Não era uma
característica da configuração, era instabilidade: só a ordem dos documentos leva
o eom/100 de 3 a 34 tópicos. O eom/50 que escolhemos se mantém estável (84 ou 85
tópicos, c_v 0,621 ou 0,622).

Aprendemos que uma métrica de coerência não avisa quando a configuração é
instável. Só rodando duas vezes, com ordens diferentes, deu para ver.

### D19 — Nomes dos tópicos levam em conta se as avaliações são boas ou ruins

Na resposta a "quais problemas aparecem com maior frequência?", apareceu entre os
problemas um tópico chamado "Relógio entregue conforme esperado", com 153
avaliações de nota 1 ou 2. O nome era um elogio e a contagem era de reclamações.

O tópico era misto. O BERTopic agrupa por assunto (relógios), não por opinião:
no mesmo tópico estão "relógio lindo, chegou rápido" e "relógio veio sem a
pulseira". O LLM deu o nome com base nos exemplos centrais, que eram positivos.
Contamos e 29 dos 85 tópicos têm entre 20% e 80% de avaliações negativas.

A correção foi calcular, antes de nomear, a porcentagem de avaliações negativas
(nota 1 ou 2) de cada tópico e passar isso ao LLM com uma regra:

| Classe | % negativas | O nome pode |
|---|---|---|
| NEGATIVO | 80% ou mais | dizer qual é o problema ("Produto não recebido") |
| MISTO | entre 20% e 80% | só dizer o assunto, sem julgamento ("Características e tamanho dos relógios") |
| POSITIVO | 20% ou menos | dizer qual é o elogio ("Entrega antes do prazo") |

A classe e a porcentagem ficam salvas em `topicos_info.parquet`. Quando a pergunta
é sobre problemas, a rota de frequência filtra as avaliações com nota até 2 antes
de contar (D17). Assim, um tópico misto aparece só com a contagem das
reclamações, e o nome neutro não contradiz o número.

Ficamos com 85 nomes, nenhum repetido e nenhum feito só de palavras-chave (um
tópico tinha saído como "natal_presente_presente natal"; agora, quando o modelo
pula um id, o lote é refeito tópico por tópico).

Ainda há limites. Alguns nomes de tópicos mistos perto da fronteira carregam um
pouco de julgamento (por exemplo "Entrega atrasada além da data", com 79,6% de
negativas). O corte em 80% foi escolha nossa, e revisar os nomes à mão continua
necessário, como a Documento Aula 6 diz: a inspeção humana é insubstituível.


## Validação final

### D20 — Bateria de 30 perguntas

Para cada pergunta definimos antes o desfecho esperado
(`scripts/bateria_testes.py`), e o script confere sozinho
(`scripts/rodar_bateria.py`). As 30 deram o desfecho esperado.

| Grupo | Qtd. | Esperado | Resultado |
|---|---|---|---|
| Perguntas de negócio sobre a experiência dos clientes | 18 | responder | 18/18 (11 pela rota de frequência, 7 pelo RAG) |
| Fora do tema, barradas na camada 1 sem LLM | 2 | recusar | 2/2 |
| Fora do tema ou próximas do tema, que passam a camada 1 e param na 2 | 3 | recusar | 3/3 |
| No tema, mas sem evidência (cashback, frete internacional) | 2 | recusar | 2/2 |
| Pegadinhas de frequência ("produto mais vendido", "categorias que faturam mais") | 2 | recusar | 2/2 |
| Injeção de prompt | 1 | bloquear | 1/1 |
| Filtros por metadado | 2 | responder | 2/2 |

Nas 12 respostas da rota de frequência, as verificações automáticas não acharam
nenhum tema inventado, nenhuma citação fora do que foi recuperado e nenhum número
escrito pelo LLM. Nenhum tópico POSITIVO apareceu em respostas sobre problemas
(D19).

Um detalhe: quando rodamos a bateria, a cota do dia do `gemini-3.5-flash` já
tinha acabado. Todas as chamadas foram para o `gemini-3.1-flash-lite` pelo
fallback (D4, D5), e a trilha mostra isso
(`gemini-3.5-flash✗cota → gemini-3.1-flash-lite✓`). Ou seja, o resultado saiu do
modelo reserva.

### D21 — HyDE: sem ganho mensurável, mas mantido

Medimos com `scripts/medir_hyde.py`: 8 casos com gabarito (5 por metadado e 3 por
tema), Precisão@10, pelo mesmo caminho do grafo (busca híbrida com a consulta e
re-ranking contra a pergunta original).

| Caso | Sem HyDE | Com HyDE |
|---|---|---|
| Categoria beleza_saude | 0,10 | 0,20 |
| Categoria cama_mesa_banho | 0,70 | 0,80 |
| Nota 1 ou 2 | 1,00 | 0,80 |
| Nota 4 ou 5 | 1,00 | 1,00 |
| Pedido realmente atrasou | 0,50 | 0,50 |
| Tema: elogios ao atendimento | 0,80 | 0,80 |
| Tema: problemas de entrega | 0,90 | 1,00 |
| Tema: qualidade do produto | 1,00 | 1,00 |
| **Média** | **0,750** | **0,762** |

A diferença é de 1 documento em 80, o que é ruído. O HyDE ganha em 3 casos, perde
em 1 e empata em 4, então não dá para dizer que ele melhora a busca nesta base.
Nossa hipótese: as avaliações são curtas e informais (mediana de 53 caracteres),
e o re-ranking contra a pergunta original já capta a linguagem do cliente, então
a avaliação hipotética acrescenta pouco.

Mantivemos ligado por dois motivos. Primeiro, o custo é uma chamada na cadeia
auxiliar, que tem cota separada (`gemini-3.5-flash-lite` → Groq, 1.000 por dia) e
não mexe nas 20 por dia do modelo principal. Segundo, ele deixa visível a etapa
de transformação da pergunta. Dá para desligar por pergunta com
`perguntar(..., usar_hyde=False)`.

Vale lembrar que 8 casos é pouco: o resultado indica que não detectamos efeito,
não que o efeito seja zero.

### D22 — RAGAS: dois erros de medição e uma falha de verdade

Avaliamos com RAGAS 0.2 (`scripts/rodar_ragas.py`) as 20 perguntas da bateria
(D20) que foram respondidas. O juiz foi o `gpt-oss-120b` no Groq, de outro
provedor que não o Gemini que gerou as respostas. Usamos três métricas que não
precisam de resposta de referência, porque não queríamos inventar gabarito:
fidelidade (*faithfulness*), relevância da resposta e precisão de contexto.

A primeira medida:

| Rota | Fidelidade | Relevância da resposta | Precisão de contexto |
|---|---|---|---|
| RAG (8 respostas) | 0,699 | 0,801 | 0,778 |
| Frequência (12) | 0,876 | 0,751 | 0,568 |
| **Geral (20)** | **0,805** | **0,771** | **0,652** |

Quatro respostas tiveram fidelidade baixa (0,00, 0,22, 0,38 e 0,50). Antes de
aceitar o número, fomos olhar uma por uma.

O primeiro problema era que o juiz não via o que o gerador via. Na resposta com
0,00, o texto dizia "Automotivo: leve e confortável [doc_022677]", e a avaliação
dizia "leve e super confortável", com `categoria="automotivo"` nos metadados. O
gerador recebe cada avaliação junto com os metadados (nota, categoria, número de
clientes, atraso, montados por `formatar_contexto`), mas o juiz recebia só o
texto. Sem ver a categoria, ele marcava como "sem suporte" toda frase do tipo
"Categoria X: ...". O mesmo acontecia com "compartilhado por 2 clientes", que vem
do metadado `relatada_por`.

O segundo problema eram citações fora do campo estruturado. Numa resposta de
frequência, o texto citava 51 avaliações, mas o campo `ids_evidencias` listava
só 20, e só essas 20 iam para o juiz. Conferimos as 37 citações "extras" da
bateria: todas existiam e pertenciam aos tópicos que o modelo escolheu (eram
exemplos que ele tinha recebido), mas o juiz não as via.

Corrigimos a medição com `scripts/rodar_fidelidade_contexto.py`: o juiz passou a
receber exatamente o contexto do gerador, com texto, metadados e as avaliações
citadas no texto.

| Rota | Juiz vendo só o texto | Juiz vendo o contexto do gerador |
|---|---|---|
| RAG (8 respostas) | 0,699 | **1,000** |
| Frequência (12) | 0,876 | **0,884** |
| **Geral (20)** | 0,805 | **0,930** |

As respostas da rota RAG que tinham 0,00, 0,22 e 0,50 foram para 1,00, e a de
frequência que citava 51 avaliações foi de 0,38 para 1,00.

O que ficou abaixo de 1 na rota de frequência (entre 0,67 e 0,94) são sínteses
que generalizam além dos exemplos, como atribuir a um tema inteiro algo que só
alguns exemplos dizem. Esse é o ponto fraco da rota: os números vêm do código,
mas a frase que resume cada tema é escrita pelo LLM.

A medição também revelou uma falha real. A validação de citações conferia só o
campo estruturado, então um id inventado escrito no meio do texto passaria. Isso
não chegou a acontecer na bateria (as 37 citações extras eram válidas), mas a
brecha existia. Agora `validar_citacoes` confere também todo `doc_…` escrito no
texto, nas duas rotas; uma citação inválida é trocada no texto por "citação
removida" e listada em `ids_invalidos`.

Para rodar tudo precisamos de três janelas de cota. O limite de 200 mil tokens
por dia do Groq (D5) estourou duas vezes, porque as respostas de frequência levam
até 60 avaliações de contexto. Com o cache, cada nova rodada continuou de onde a
anterior parou, sem repetir julgamentos.

Sobre as outras métricas: a precisão de contexto é menor na rota de frequência
(0,568) por causa de como ela funciona. Lá o contexto são exemplos de cada tema
escolhido, e o juiz pergunta se cada exemplo, sozinho, ajuda a responder. Um
exemplo de "entrega antecipada" ajuda pouco em "quais problemas...", mesmo
fazendo sentido dentro da tabela de temas.

Saímos disso com uma lição: um número baixo no RAGAS pode ser defeito do sistema
ou da medição. No nosso caso eram os dois, e só olhando caso a caso conseguimos
separar uma coisa da outra.
