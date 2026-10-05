# CineData Analytics - Agente GenAI (Text-to-SQL)
 
Agente de análise de dados cinematográficos que traduz perguntas em linguagem natural para consultas SQL sobre a **Camada Gold** do Data Lakehouse, armazenada no banco SQLite `cinerocket.db`. O agente executa consultas somente de leitura, valida o SQL gerado, sintetiza uma resposta executiva e gera gráficos automaticamente.
 
Projeto desenvolvido como parte do processo seletivo **Rocket Lab 2026** da Visagio.
 
---
 
## Sumário
 
- [Visão geral do fluxo](#visão-geral-do-fluxo)
- [Arquitetura e decisões técnicas](#arquitetura-e-decisões-técnicas)
- [Base de dados](#base-de-dados)
- [Pré-requisitos](#pré-requisitos)
- [Como executar no Google Colab](#como-executar-no-google-colab)
- [Como fazer perguntas ao agente](#como-fazer-perguntas-ao-agente)
- [Perguntas de exemplo](#perguntas-de-exemplo)
- [Avaliação (Golden Set)](#avaliação-golden-set)
- [Limitações conhecidas](#limitações-conhecidas)
- [Tecnologias](#tecnologias)
---
 
## Visão geral do fluxo
 
```
Pergunta em português
        |
        v
  Cache em memória ----(hit)----> resposta imediata
        |
      (miss)
        v
  LLM gera o SQL (prompt com schema + regras de negócio)
        |
        v
  Validação (somente SELECT / WITH) e execução no SQLite
        |
        |--(erro)--> 1 tentativa de autocorreção pelo LLM
        v
  LLM sintetiza a resposta executiva a partir dos dados reais
        |
        v
  Retorno: SQL, DataFrame, resposta e gráfico automático
```
 
---
 
## Arquitetura e decisões técnicas
 
A solução foi desenhada para ser robusta, precisa e compatível com o limite de cerca de 50 requisições diárias dos modelos gratuitos do OpenRouter.
 
1. **Injeção contextual de schema (Camada Gold).**
   O prompt de sistema recebe o mapeamento das 10 tabelas do modelo dimensional (`dim_movies`, `fact_movies_performance`, `dim_genres`, `dim_people`, `dim_companies`, `dim_reviews`, `movie_reviews` e as tabelas *bridge* N:N de gêneros, pessoas e produtoras). Junto com o schema, o prompt traz regras de negócio: qual coluna usar para "receita", "orçamento", "lucro" e "margem de lucro", filtros de valores nulos ou zerados em cálculos financeiros e o padrão de *joins* para cada relacionamento.
2. **Guardrails de segurança SQL.**
   Antes de executar, o SQL passa por uma validação que:
   - remove blocos de markdown (```` ```sql ````) e normaliza o `;` final;
   - bloqueia palavras-chave de escrita ou de alteração estrutural: `DROP`, `DELETE`, `UPDATE`, `INSERT`, `ALTER`, `TRUNCATE`, `EXEC`, `CREATE`, `REPLACE`, `ATTACH`, `DETACH` e `PRAGMA`;
   - exige que a consulta comece com `SELECT` ou `WITH` (para CTEs).
   A validação é feita por expressões regulares, então é conservadora: pode bloquear consultas legítimas que contenham essas palavras (por exemplo, um título de filme com a palavra "Drop").
3. **Autocorreção (self-healing).**
   Se o SQLite retornar erro (sintaxe ou coluna inexistente), o agente envia a consulta e a mensagem de erro de volta ao LLM e executa a versão corrigida. É feita **uma** tentativa de correção por pergunta.
4. **Cache de consultas em memória.**
   Perguntas repetidas (comparadas em minúsculas e sem espaços nas pontas) são respondidas a partir do cache, sem consumir chamadas à API. O cache vale apenas durante a sessão do notebook.
5. **Visualização e formatação executiva.**
   Valores monetários são formatados no padrão brasileiro (R$ mil, mi, bi). Quando o resultado tem uma coluna de texto (rótulo) e uma coluna numérica, o agente gera automaticamente um gráfico de barras horizontais com Matplotlib e Seaborn.
6. **Seleção automática de modelo gratuito.**
   O notebook lista os modelos do OpenRouter com sufixo `:free` e usa o primeiro que responder a um teste de conexão.
---
 
## Base de dados
 
O arquivo `cinerocket.db` (aproximadamente 554 MB) contém o modelo dimensional completo da Camada Gold, em formato SQLite. Como ele ultrapassa o limite de tamanho de arquivos do Git, não está neste repositório e deve ser baixado da pasta oficial do Google Drive:
 
**https://drive.google.com/drive/folders/19478J9a36_zdiMYd8aGxythohOFWj1zy**
 
<!-- TODO: adicionar aqui o link do repositório do pipeline ETL (Bronze/Silver/Gold) e uma frase explicando como a Camada Gold foi exportada para o SQLite. -->
 
---
 
## Pré-requisitos
 
- Uma conta Google, para usar o Google Colab e o Google Drive.
- Uma **chave de API do OpenRouter** (formato `sk-or-v1-...`):
  1. Crie uma conta em [openrouter.ai](https://openrouter.ai).
  2. Acesse **Keys** nas configurações da conta e crie uma nova chave.
  3. Os modelos com sufixo `:free` não geram custo, mas têm limite diário de requisições (cerca de 50 por dia no plano gratuito).
> **Segurança:** nunca escreva a chave dentro do código nem a compartilhe. O notebook a solicita via `getpass`, mas, se você salvar o notebook com a saída da célula visível, a chave pode ficar gravada no arquivo. Antes de compartilhar ou publicar, use **Editar > Limpar todas as saídas** no Colab, ou guarde a chave nos *Secrets* do Colab.
 
---
 
## Como executar no Google Colab
 
1. Abra o Google Colab e carregue o notebook `agente_text_to_sql.ipynb`.
2. Baixe o `cinerocket.db` na [pasta do Google Drive](https://drive.google.com/drive/folders/19478J9a36_zdiMYd8aGxythohOFWj1zy) e salve-o no seu Drive pessoal.
3. Na célula de montagem do Drive, ajuste o caminho para o local onde você salvou o arquivo:
```python
   from google.colab import drive
   drive.mount('/content/drive')
 
   !cp "/content/drive/MyDrive/<caminho_da_pasta>/cinerocket.db" "cinerocket.db"
```
 
4. Execute as células em ordem:
   | Célula | O que faz |
   |--------|-----------|
   | 1 | Instala as dependências (`openai`, `pandas`, `tabulate`) e faz os imports. |
   | Montagem do Drive | Copia o `cinerocket.db` para o ambiente do Colab. |
   | 2 | Verifica se o arquivo existe e extrai o schema das tabelas para o prompt. |
   | 3 | Solicita a chave do OpenRouter (via `getpass`) e testa os modelos gratuitos. |
   | 4 | Define os guardrails de segurança, a execução no SQLite e o cache. |
   | 5 | Define o prompt de sistema e o pipeline do agente (`ask_cinedata`). |
   | 6 | Valida o agente com perguntas de negócio. |
   | 7 | Formatação monetária, gráficos automáticos e modo interativo. |
   | 8 | Suíte de avaliação automatizada por categoria (Golden Set). |
   | 9 | Teste unitário com uma pergunta específica. |
---
 
## Como fazer perguntas ao agente
 
### Método 1: chamada direta em Python
 
```python
resultado = ask_cinedata("Quais são os 5 filmes com maior receita em R$?")
 
# Resposta executiva em linguagem natural
print(resultado["answer"])
 
# SQL gerado pelo agente
print(resultado["sql"])
 
# Tabela bruta de resultados (DataFrame)
print(resultado["data"])
```
 
O dicionário retornado traz as chaves `status` (`"success"` ou `"error"`), `question`, `sql`, `data`, `answer` e, em caso de falha, `error`.
 
### Método 2: modo interativo
 
Depois de executar a célula 7, chame:
 
```python
ask_cinedata_interactive()
```
 
Digite perguntas livremente e use `sair` para encerrar. A cada pergunta o agente mostra o SQL executado, uma prévia da tabela, a resposta executiva e o gráfico, quando aplicável.
 
---
 
## Perguntas de exemplo
 
| Categoria | Pergunta |
|-----------|----------|
| Bilheteria e finanças | Quais são os 5 filmes com maior receita em R$? |
| Bilheteria e finanças | Quais os 5 filmes com maior margem de lucro, entre os que possuem receita e orçamento informados? |
| Gêneros e produtoras | Qual a quantidade de filmes por gênero? |
| Gêneros e produtoras | Qual a produtora com maior lucro total em R$? |
| Popularidade e engajamento | Quais são os 5 filmes com maior divergência absoluta entre a nota TMDB e a nota IMDb? |
| Elenco e equipe | Quais são os diretores com maior nota média no IMDb, considerando apenas diretores com no mínimo 5 filmes? |
| Elenco e equipe | Qual a dupla ator e diretor que mais trabalhou junta e quantos filmes fizeram? |
| Avaliações dos usuários | Quais são os 5 filmes mais avaliados pelos usuários da plataforma? |
 
<!-- TODO: adicionar aqui uma captura de tela (ou tabela) com o resultado real de 1 ou 2 perguntas, incluindo o gráfico gerado. -->
 
---
 
## Avaliação (Golden Set)
 
A célula 8 executa uma bateria de perguntas cobrindo as categorias de negócio do enunciado e gera um relatório consolidado com categoria, pergunta, status, número de linhas retornadas e o início do SQL gerado.
 
Atualmente o critério de sucesso é **técnico**: a consulta foi gerada, passou nos guardrails, executou sem erro e retornou ao menos uma linha. A avaliação não compara o resultado com uma resposta de referência.
 
---
 
## Limitações conhecidas
 
- **Qualidade variável dos modelos gratuitos.** O modelo é o primeiro `:free` que responde, então a qualidade do SQL pode mudar entre execuções.
- **Limite de requisições.** Com cerca de 50 requisições por dia, muitas perguntas novas seguidas podem esgotar a cota. O cache ajuda apenas em perguntas repetidas.
- **Autocorreção limitada.** Há uma única tentativa de correção por pergunta.
- **Guardrail por palavras-chave.** Pode gerar falsos positivos (veja a seção de arquitetura).
- **Avaliação sem gabarito.** O Golden Set verifica execução, não correção semântica dos resultados.
- **Síntese baseada em amostra.** A resposta executiva usa as 10 primeiras linhas do resultado, e a tabela exibida mostra as 5 primeiras.
---
 
## Tecnologias
 
- Python, SQLite
- OpenRouter (API compatível com o SDK da OpenAI) com modelos gratuitos
- Pandas, Tabulate
- Matplotlib e Seaborn
- Google Colab
