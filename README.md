

# Grupo ArtificialSapiens - Desafio 2

## Integrantes do Grupo

Nome         | E-mail
-------------|---------------------
Lauro A.     | l********@***
Rodrigo D.   | d********@***
Ricardo S.   | s********@***
Luiz M.      | l****@***
Manuella M.  | m*************@***
Wilfred B.   | w****************@***
Roberto F.   | f*******@***
Marcos L. Z. | m********@***



# Arquitetura Proposta da Solução

## Framework escolhida:
N8N localmente via Docker + Docker MySQL + Gemini Flash 2.0 para os agentes


![image](https://github.com/user-attachments/assets/c48e720d-5b45-438c-a401-f2193993ed71)

O objetivo deste projeto é construir um agente capaz de responder a qualquer pergunta sobre as notas fiscais processadas.

** A solução json está no arquivo Solucao.7z cuja senha foi enviada no documento PDF para evitar cópias não autorizadas por outras equipes , após o prazo um novo commit será realizado com o json em claro **

## Motivação

Devido à limitação de janela de contexto dos modelos de linguagem, não é viável enviar todos os dados diretamente para o modelo de IA. Modelos LLMs (Large Language Models) não conseguem processar grandes volumes de dados de uma só vez.

## Abordagem Arquitetural

Para superar essa limitação, a solução adota uma arquitetura híbrida, integrando:

- Um banco de dados relacional (MySQL)
- Um agente especializado em SQL

Esse agente será responsável por:

- Interpretar a pergunta feita pelo usuário
- Construir dinamicamente as queries SQL necessárias para consultar a base de dados
- Resumir e entregar ao modelo de IA apenas as informações relevantes para gerar a resposta

## Papel do Agente SQL

A função principal do agente SQL é montar consultas que recuperem apenas o contexto necessário para cada pergunta.

Exemplo de pergunta:

"Qual empresa teve o maior faturamento?"

O fluxo será:

1. A LLM interpreta a pergunta
2. O agente SQL gera uma query que soma os valores das notas fiscais agrupados por fornecedor
3. Retorna o fornecedor com maior total
4. O resultado é passado ao modelo de IA para compor a resposta final ao usuário

## Benefícios da Arquitetura

Essa abordagem permite realizar operações analíticas como:

- Cálculos
- Somatórios
- Médias
- Agrupamentos
- Comparações

Tarefas que não podem ser feitas apenas via busca semântica com embeddings.

## Solução Técnica Adotada

### Processamento dos Arquivos CSV

Os arquivos contendo os dados das notas fiscais:

- 202401_NFs_Cabecalho.csv
- 202401_NFs_Itens.csv

São processados e carregados em um banco de dados MySQL, permitindo o armazenamento estruturado e relacional dos dados.

### Camada de Interpretação de Perguntas

Quando o usuário faz uma pergunta em linguagem natural, um agente baseado em LLM (Large Language Model) executa as seguintes etapas:

- Interpreta a pergunta
- Gera a SQL correspondente

Exemplo de pergunta:

"Qual fornecedor recebeu o maior montante no mês?"

O agente gera automaticamente uma SQL correspondente para a base.

### Execução e Validação da Query

O comando SQL gerado passa por um módulo em Python que:

- Valida se a query é segura (por exemplo: apenas SELECT)
- Executa a query no banco de dados MySQL
- Obtém o resultado estruturado (JSON, dicionário, etc)

### Formatação da Resposta

O resultado da query pode ser:

- Diretamente exibido
- Ou enviado de volta para a LLM, para que ela gere uma resposta textual natural, contextualizada e bem formatada.

### Resposta ao Usuário

Por fim, o agente exibe a resposta ao usuário em formato de texto descritivo e fácil de entender.



# Preparação do Ambiente de Execução

Para que o agente possa funcionar corretamente, é necessário preparar o ambiente de execução e realizar o processamento inicial dos dados.

O ambiente foi testado em uma máquina Linux, mas o processo também pode ser realizado em Windows utilizando o WSL (Windows Subsystem for Linux) com uma distribuição baseada em Debian, como o Ubuntu.

---

## 1. Instalação do Docker

O Docker será utilizado para simplificar a instalação e o gerenciamento dos serviços necessários (como o MySQL e o n8n).

Atualize os pacotes e instale o Docker:

```bash
sudo apt update
sudo apt install docker.io
```

## 2. Criar uma Rede Docker para Comunicação entre Containers

Para que os containers possam se comunicar entre si (por exemplo, o n8n acessar o MySQL), crie uma rede Docker dedicada:
sudo docker network create n8n-network
Com isso, os containers poderão se localizar e trocar informações internamente.

## 3. Executar o Container do n8n (Orquestrador de Workflows)

O n8n será usado como motor de integração e automação entre os serviços.
Execute o seguinte comando:
```
sudo docker run -d \
  --name n8n \
  --network n8n-network \
  -p 5678:5678 \
  -v ~/.n8n:/home/node/.n8n \
  n8nio/n8n

```

## 4. Executar o Container do MySQL (Banco de Dados)

O MySQL será o repositório para armazenar os dados das notas fiscais.  
Rode o container com:


```
sudo docker run -d&#x20;
\--name mysql-notas&#x20;
\--network n8n-network&#x20;
-e MYSQL\_ROOT\_PASSWORD=root&#x20;
-e MYSQL\_DATABASE=notas\_db&#x20;
-p 3306:3306&#x20;
mysql:8

```

Isso irá:  
- Criar o banco de dados `notas_db`  
- Definir a senha de root como `root`  
- Tornar o MySQL acessível na porta 3306

## 5. Criar a Tabela de Notas Fiscais
Agora vamos criar a estrutura da tabela que irá receber os dados das notas.
Entre no container do MySQL:
sudo docker exec -it mysql-notas mysql -uroot -proot notas_db
Depois, execute o seguinte comando SQL para criar a tabela:
```
CREATE TABLE notas_fiscais (
    CHAVE_ACESSO VARCHAR(50),
    MODELO VARCHAR(100),
    SERIE VARCHAR(20),
    NUMERO VARCHAR(20),
    NATUREZA_OPERACAO VARCHAR(255),
    DATA_EMISSAO DATETIME,
    CPFCNPJ_EMITENTE VARCHAR(20),
    RAZAO_SOCIAL_EMITENTE VARCHAR(255),
    INSCRICAO_ESTADUAL_EMITENTE VARCHAR(50),
    UF_EMITENTE VARCHAR(5),
    MUNICIPIO_EMITENTE VARCHAR(100),
    CNPJ_DESTINATARIO VARCHAR(20),
    NOME_DESTINATARIO VARCHAR(255),
    UF_DESTINATARIO VARCHAR(5),
    INDICADOR_IE_DESTINATARIO VARCHAR(50),
    DESTINO_OPERACAO VARCHAR(100),
    CONSUMIDOR_FINAL VARCHAR(50),
    PRESENCA_COMPRADOR VARCHAR(100),
    NUMERO_PRODUTO INT,
    DESCRICAO_PRODUTO_SERVICO VARCHAR(255),
    CODIGO_NCM VARCHAR(20),
    NCH_SH_TIPO_PRODUTO VARCHAR(255),
    CFOP VARCHAR(10),
    QUANTIDADE DECIMAL(10,2),
    UNIDADE VARCHAR(50),
    VALOR_UNITARIO DECIMAL(10,2),
    VALOR_TOTAL DECIMAL(10,2),
    EVENTO_MAIS_RECENTE VARCHAR(255),
    DATAHORA_EVENTO_MAIS_RECENTE DATETIME,
    VALOR_NOTA_FISCAL DECIMAL(10,2)
);
```
## 6. Acesso ao n8n
Com tudo rodando, acesse a interface web do n8n diretamente no navegador:
http://localhost:5678
Na primeira execução, será necessário configurar as credenciais da API de LLM (Large Language Model). No exemplo usado, foi integrada a API do Gemini (modelo 2.0 Flash), mas nada impede que você ajuste posteriormente para outros provedores, como OpenAI (GPT) ou Anthropic (Claude).


# Processamento de Dados
![image](https://github.com/user-attachments/assets/57d8f633-9b7a-4246-8881-cc76c62bddd8)


O processo de ingestão e transformação dos dados foi realizado utilizando os recursos nativos do n8n, explorando sua interface de workflow visual.
Etapas do Processo:
Upload do Arquivo Compactado


Utilizamos o recurso de formulários do n8n para permitir o upload de um arquivo compactado contendo os dois arquivos CSV de origem. Esses arquivos representam:
Um CSV com informações gerais da nota fiscal (cabeçalho)
Um CSV com os itens de cada nota fiscal
Descompactação


Após o upload, o arquivo é automaticamente descompactado dentro do workflow, utilizando os nós de manipulação de arquivos do n8n.
Combinação dos Arquivos CSV


Para unir os dados das duas fontes, usamos o nó “Merge - Combine”, realizando a junção com base no campo “Chave de Acesso”.
Esse campo é único por nota fiscal e está presente em ambos os arquivos (tanto no cabeçalho quanto nos itens), o que garante uma combinação correta e consistente dos registros.
O resultado é um único conjunto de dados com todas as colunas necessárias, alinhado ao formato da tabela criada previamente no banco de dados MySQL.
Inserção no Banco de Dados


Com os dados já no formato final, foi realizada a conexão ao banco MySQL (que está na mesma rede Docker criada anteriormente) por meio dos nós de integração de banco de dados do n8n.
O workflow executa um batch de inserts, populando a tabela notas_fiscais com os dados consolidados.

# Testes Realizados

## Questão 1
![image](https://github.com/user-attachments/assets/efaa0b2a-ebb9-491e-a358-15fcf360f0d0)
![image](https://github.com/user-attachments/assets/621327ab-1980-4b9e-b1b9-d27e53917fc3)

## Questão 2
![image](https://github.com/user-attachments/assets/180771e9-8ca1-4910-b057-b8f3a77b97ad)
![image](https://github.com/user-attachments/assets/9f984b3f-86db-4626-ba6c-92a9ed404454)

## Questão 3
Relacione TAPETE e preço de venda, quero saber quais TAPETES são vendidos e a variação de preço
![image](https://github.com/user-attachments/assets/d4ff7a5e-ec7f-4b16-a977-75ac145e1634)
![image](https://github.com/user-attachments/assets/d1a3121a-cb7a-4dfa-a0bb-fd92cbd57a3f)



