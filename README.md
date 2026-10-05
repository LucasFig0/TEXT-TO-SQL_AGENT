# CineData Analytics - GenAI Agent (Text-to-SQL)

Agente inteligente de analise de dados cinematograficos capaz de traduzir perguntas em linguagem natural para consultas SQL sobre a Camada Gold (Data Lakehouse) armazenada no banco SQLite cinerocket.db. O agente executa consultas de leitura em tempo real, valida guardrails de seguranca, sintetiza respostas executivas e gera visualizacoes graficas automatizadas.

Projeto desenvolvido como parte do processo seletivo Rocket Lab 2026 da Visagio.

---

## Arquitetura e Decisoes Tecnicas

Para garantir robustez, precisao analitica e aderencia ao limite de 50 requisicoes diarias de modelos gratuitos via OpenRouter, a solucao foi desenhada com os seguintes pilares:

1. Injecao Contextual de Schema (Camada Gold):
   O system prompt recebe o mapeamento completo das 10 tabelas dimensionais (dim_movies, fact_movies_performance, dim_genres, dim_people, dim_companies, dim_reviews, movie_reviews e tabelas bridge de relacionamento N:N), assegurando o uso correto de joins e regras de negocio financeiras (como margem de lucro e filtros de receita valida).

2. Guardrails de Seguranca SQL:
   Validador regex estrito que garante operacoes exclusivas de leitura (SELECT ou WITH), bloqueando qualquer tentativa de comando destrutivo ou de alteracao estrutural (DROP, DELETE, UPDATE, INSERT, ALTER, TRUNCATE, ATTACH).

3. Mecanismo de Auto-Correcao (Self-Healing):
   Caso o SQLite aponte erro de sintaxe ou coluna inexistente, o agente captura a mensagem de erro do motor e envia de volta ao LLM para uma tentativa automatica de correcao imediata.

4. Cache de Consultas em Memoria:
   Perguntas repetidas sao respondidas instantaneamente a partir do cache, consumindo zero chamadas a API do OpenRouter.

5. Visualizacao e Formatacao Executiva:
   Formatacao automatizada de moedas no padrao brasileiro (BRL) e geracao automatica de graficos (Matplotlib/Seaborn) quando os dados retornados contiverem dimensoes categoricas e metricas numericas.

---

## Base de Dados

O banco de dados do projeto (cinerocket.db) possui aproximadamente 554 MB e contem o modelo dimensional completo da camada Gold. Devido ao limite de tamanho de arquivos individuais no Git, ele deve ser obtido no diretorio oficial do Google Drive no seguinte endereco:

https://drive.google.com/drive/folders/19478J9a36_zdiMYd8aGxythohOFWj1zy

---

## Como Executar no Google Colab

1. Abra o Google Colab e carregue o arquivo do notebook (cinedata_agent.ipynb).

2. Obtenha o arquivo do banco de dados:
   Acesse a pasta do Drive indicada acima (https://drive.google.com/drive/folders/19478J9a36_zdiMYd8aGxythohOFWj1zy) e faca o download do arquivo cinerocket.db.

3. Carregue o banco no Colab:
  Via Google Drive montado: Com o arquivo ja estiver salvo no seu Drive pessoal, execute a celula de montagem do Drive presente no notebook:
     from google.colab import drive
     drive.mount('/content/drive')
     !cp "/content/drive/MyDrive/caminho_da_pasta/cinerocket.db" "cinerocket.db"

4. Execute as celulas sequencialmente:
   - A Celula 1 instalara as dependencias necessarias (openai, pandas, tabulate, matplotlib, seaborn).
   - A Celula 2 validara a conexao e inspecionara as 10 tabelas do banco de dados.
   - A Celula 3 solicitara a sua chave da OpenRouter (formato sk-or-v1-...) de forma segura via getpass.
   - As Celulas 4 e 5 inicializam a camada de seguranca, prompts e o pipeline do agente.
   - Celula 6: Bateria de Validacao Pratica das Perguntas de Negocio
   - Celula 7: Formatacao Financeira, Visualizacao de Dados (DataViz) e Chat Interativo
   - Celula 8: Suite de Avaliacao Automatizada (Golden Set / Benchmark)
   - Célula 9: Teste unitário
 
---



## Como Fazer Perguntas para o Agente

O agente oferece duas formas principais de interacao para usuarios e analistas:

### Metodo 1: Chamada Direta via Funcao Python

Voce pode consultar o agente diretamente em qualquer celula de codigo chamando a funcao ask_cinedata com a pergunta desejada:

```python
resultado = ask_cinedata("Quais sao os 5 filmes com maior receita em R$?")

# Visualizar a sintese em linguagem natural:
print(resultado["answer"])



# Visualizar a consulta SQL gerada pelo agente:
print(resultado["sql"])
````


# Acessar a tabela bruta de resultados como DataFrame:
print(resultado["data"])
