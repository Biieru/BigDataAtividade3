# Análise das Cotações de Moedas Estrangeiras (2023–2025) via API do Banco Central com MongoDB Atlas

Este projeto tem como objetivo consumir dados de **cinco moedas estrangeiras** diretamente da API do **Banco Central do Brasil (BCB)**, referente ao período de **01/01/2023 até 09/01/2025**. Os dados são processados com `pandas` e armazenados no **MongoDB Atlas** para consultas e análises posteriores.

## 🪙 Moedas analisadas
- **Dólar Americano (USD)**

## 🔗 Acesso rápido
- 🔍 Google Colab, material de estudo: [Abrir no Colab](https://colab.research.google.com/drive/14f1k1rZvMabKVuy_gvVQ9BruGF_MxN50?usp=sharing)
- 🌐 API BCB (cotação por data):  
  [`CotacaoMoedaDia`](https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoMoedaDia(moeda=@moeda,dataCotacao=@dataCotacao)?@moeda='USD'&@dataCotacao='01-01-2023'&$top=100&$format=json&$select=paridadeCompra,paridadeVenda,cotacaoCompra,cotacaoVenda,dataHoraCotacao,tipoBoletim)
- 🌐 API BCB (cotação por período):  
  [`CotacaoMoedaPeriodo`](https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoMoedaPeriodo(moeda=@moeda,dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)?@moeda='USD'&@dataInicial='01-01-2023'&@dataFinalCotacao='09-01-2025'&$top=1000&$format=json&$select=paridadeCompra,paridadeVenda,cotacaoCompra,cotacaoVenda,dataHoraCotacao,tipoBoletim)
- 🍃 MongoDB Atlas: [Acesse aqui](https://cloud.mongodb.com/)

## 🎯 Objetivos do projeto
- Acessar os dados da API PTAX do Banco Central do Brasil
- Consultar a cotação de 5 moedas estrangeiras no intervalo de 2023 até 2025
- Coletar variáveis relevantes: `paridadeCompra`, `paridadeVenda`, `cotacaoCompra`, `cotacaoVenda`, `dataHoraCotacao` e `tipoBoletim`
- Converter os dados JSON para `DataFrames` usando `pandas`
- **Armazenar os dados no MongoDB Atlas em formato de documentos JSON**
- **Criar banco de dados e coleções organizadas por moeda**
- Automatizar e modularizar todo o processo de ETL (Extract, Transform, Load)

## 🛠️ Tecnologias utilizadas
- **Python** - Linguagem principal
- **pandas** - Manipulação e análise de dados
- **requests** - Requisições HTTP para API
- **pymongo** - Driver oficial MongoDB para Python
- **MongoDB Atlas** - Banco de dados NoSQL na nuvem
- **json** - Manipulação de dados JSON

## 🧾 Dados coletados
Cada documento no MongoDB contém as seguintes informações:
- `paridadeCompra` — Paridade de compra da moeda
- `paridadeVenda` — Paridade de venda da moeda  
- `cotacaoCompra` — Valor da cotação de compra
- `cotacaoVenda` — Valor da cotação de venda
- `dataHoraCotacao` — Data e hora da cotação (formato ISODate)
- `tipoBoletim` — Tipo de boletim (ex: "Abertura", "Intermediário", "Fechamento")
- `data_insercao` — Timestamp da inserção no banco
- `moeda` — Código da moeda (USD, EUR, JPY, GBP, CHF)

## ⚡ Instalação e Configuração

### 1. Dependências
```bash
pip install requests pandas pymongo python-dotenv
```

### 2. MongoDB Atlas Setup
1. Criar conta no [MongoDB Atlas](https://cloud.mongodb.com/)
2. Criar um cluster gratuito
3. Configurar usuário e senha do banco
4. Obter string de conexão: `Connect > Connect your application`
5. Adicionar seu IP na whitelist de acesso

### 3. Configuração das Credenciais
```python
# Substitua pela sua string de conexão
MONGO_URI = "mongodb+srv://<username>:<password>@<cluster>.mongodb.net/?retryWrites=true&w=majority"
```

## 📌 Exemplo de uso

### Extração para uma moeda específica:
```python
import requests
import pandas as pd
from pymongo import MongoClient
import json

# Configuração MongoDB
MONGO_URI = "sua_string_de_conexao_aqui"
client = MongoClient(MONGO_URI)

# Parâmetros da consulta
moeda = "USD"
data_inicial = "01-01-2023" 
data_final = "09-01-2025"

# URL da API
url = f"""https://olinda.bcb.gov.br/olinda/servico/PTAX/versao/v1/odata/CotacaoMoedaPeriodo(moeda=@moeda,dataInicial=@dataInicial,dataFinalCotacao=@dataFinalCotacao)?@moeda='{moeda}'&@dataInicial='{data_inicial}'&@dataFinalCotacao='{data_final}'&$top=1000&$format=json&$select=paridadeCompra,paridadeVenda,cotacaoCompra,cotacaoVenda,dataHoraCotacao,tipoBoletim"""

# Requisição à API
resposta = requests.get(url)

if resposta.status_code == 200:
    dados = resposta.json()
    
    # Processar dados
    df = pd.json_normalize(dados['value'])
    df['data_insercao'] = pd.Timestamp.now()
    df['moeda'] = moeda
    
    # Inserir no MongoDB
    db = client['db_cambio']
    colecao = db[f'cotacao_{moeda.lower()}_periodo']
    
    registros = df.to_dict(orient='records')
    resultado = colecao.insert_many(registros)
    
    print(f"✅ {len(resultado.inserted_ids)} registros inseridos para {moeda}")
else:
    print(f"❌ Erro na API: Status {resposta.status_code}")
```

### Pipeline completo para todas as moedas:
```python
def processar_todas_moedas():
    moedas = ['USD', 'EUR', 'JPY', 'GBP', 'CHF']
    
    for moeda in moedas:
        print(f"🔄 Processando {moeda}...")
        # Código de extração e inserção aqui
        print(f"✅ {moeda} concluído!")
    
    print("🎉 Todas as moedas processadas com sucesso!")

# Executar pipeline
processar_todas_moedas()
```

## 🔍 Consultas no MongoDB

### Consultar cotações recentes:
```python
# Últimas 10 cotações do USD
cotacoes_recentes = colecao.find({"moeda": "USD"}).sort("dataHoraCotacao", -1).limit(10)

# Cotações de fechamento apenas
cotacoes_fechamento = colecao.find({"tipoBoletim": "Fechamento"})

# Cotações de um período específico
from datetime import datetime
inicio = datetime(2023, 1, 1)
fim = datetime(2023, 12, 31)

cotacoes_2023 = colecao.find({
    "dataHoraCotacao": {"$gte": inicio, "$lte": fim}
})
```

### Agregações úteis:
```python
# Cotação média mensal
pipeline_media_mensal = [
    {
        "$group": {
            "_id": {
                "ano": {"$year": "$dataHoraCotacao"},
                "mes": {"$month": "$dataHoraCotacao"}
            },
            "cotacao_media": {"$avg": "$cotacaoVenda"}
        }
    }
]

resultado = colecao.aggregate(pipeline_media_mensal)
```

## 🚀 Executando o projeto

1. **Configure suas credenciais MongoDB**
2. **Execute o pipeline completo:**
   ```python
   python pipeline_moedas_mongodb.py
   ```
3. **Ou execute individualmente via Jupyter:**
   ```bash
   jupyter notebook extractMongoDBusdPeriodo.ipynb
   ```

## 📊 Benefícios do MongoDB
- **Flexibilidade**: Schema flexível para diferentes estruturas de dados
- **Escalabilidade**: Fácil expansão horizontal 
- **Consultas ricas**: Agregações complexas e consultas geoespaciais
- **Replicação**: Alta disponibilidade com replica sets
- **Índices**: Performance otimizada para consultas específicas
- **Atlas**: Gerenciamento na nuvem sem configuração de infraestrutura

## 🤝 Contribuindo
Sinta-se à vontade para contribuir com melhorias:
1. Fork do repositório
2. Criar branch para feature (`git checkout -b feature/nova-funcionalidade`)
3. Commit das mudanças (`git commit -am 'Adiciona nova funcionalidade'`)
4. Push para branch (`git push origin feature/nova-funcionalidade`)  
5. Criar Pull Request

## 📄 Licença
Este projeto está sob a licença MIT. Veja o arquivo `LICENSE` para mais detalhes.

---
**Desenvolvido utilizando Python, MongoDB Atlas e a API do Banco Central do Brasil**
