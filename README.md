# Sistema de Análise de Risco de Carteira (VaR via Monte Carlo)

Aplicação full-stack para montar uma carteira de investimentos (renda fixa +
derivativos), simular milhares de cenários de preços futuros via **Simulação
de Monte Carlo** e calcular o **Value at Risk (VaR)** e o **CVaR (Expected
Shortfall)** da carteira em um horizonte de tempo escolhido pelo usuário.

```
Frontend (React)  →  API REST (C# / .NET 8)  →  SQL Server
                         │
                         └── Motor de Monte Carlo (puro, sem dependências
                             de infraestrutura — testável isoladamente)
```

---

## 1. Arquitetura

O backend segue **Clean Architecture** com os princípios **SOLID**, dividido
em 4 projetos + 1 de testes:

```
backend/
├── src/
│   ├── VaR.Domain          → Entidades (Ativo, Carteira, CarteiraItem, Simulacao...)
│   ├── VaR.Application     → Regra de negócio: interfaces + MonteCarloEngine + CarteiraService
│   ├── VaR.Infrastructure  → EF Core + SQL Server (repositórios, DbContext)
│   └── VaR.Api             → Controllers REST + Swagger/OpenAPI
└── tests/
    └── VaR.Tests           → Testes xUnit do motor de cálculo
```

### Por que essa separação importa (SOLID na prática)

- **`VaR.Domain`** não depende de nada — são apenas classes de dados (POCOs).
- **`VaR.Application`** contém o **motor de cálculo** (`MonteCarloEngine`) e o
  `CarteiraService`. Ambos dependem **apenas de interfaces**
  (`IMonteCarloEngine`, `ICarteiraRepository`, `IAtivoRepository`,
  `ISimulacaoRepository`) — nunca de Entity Framework, SQL Server ou ASP.NET.
  Isso é o **Princípio da Inversão de Dependência (D)** do SOLID: a regra de
  negócio não sabe (nem precisa saber) como os dados são persistidos ou como
  a API é exposta.
- **`VaR.Infrastructure`** implementa as interfaces de repositório usando EF
  Core + SQL Server. Poderia ser trocado por PostgreSQL, MongoDB ou um
  repositório em memória (como fazemos nos testes) sem alterar uma linha do
  motor de cálculo.
- **`VaR.Api`** apenas expõe a regra de negócio via HTTP (Controllers) e faz
  a injeção de dependência (`Program.cs`), conectando as abstrações às
  implementações concretas.

**Consequência prática:** o `MonteCarloEngine` é testado em `VaR.Tests`
**sem subir banco de dados, sem subir API, sem mocks pesados** — é uma classe
pura que recebe parâmetros e devolve resultados determinísticos (dada uma
seed).

---

## 2. Modelagem de dados (SQL Server)

Ver script completo em [`database/schema.sql`](database/schema.sql).

| Tabela              | Responsabilidade                                                        |
|---------------------|---------------------------------------------------------------------------|
| `Ativos`             | Cadastro dos ativos financeiros (renda fixa, ações, derivativos, câmbio, cripto) com retorno/volatilidade anualizados |
| `Historico_Precos`   | Série histórica de cotações, usada para estimar retorno/volatilidade/correlação |
| `Carteiras`          | Carteira montada pelo usuário (nome, valor total)                        |
| `Carteira_Itens`     | Composição da carteira: qual ativo entra e com qual **peso** (0 a 1)     |
| `Simulacoes`         | Histórico de simulações executadas, com VaR, CVaR e parâmetros usados    |

A soma dos pesos (`Carteira_Itens.Peso`) de uma carteira **deve ser igual a
1 (100%)** — essa regra é validada tanto no backend (`CarteirasController` e
`MonteCarloEngine`) quanto no frontend (em tempo real, no `PortfolioBuilder`).

---

## 3. Motor de Cálculo — Simulação de Monte Carlo

Implementado em `VaR.Application/Services/MonteCarloEngine.cs`.

### Modelo estatístico

Cada ativo segue um **Movimento Geométrico Browniano (GBM)**:

```
r = (μ − 0.5·σ²)·dt + σ·√dt·Z
```

- `μ` = retorno esperado anualizado do ativo
- `σ` = volatilidade anualizada do ativo
- `dt` = horizonte da simulação em anos (dias úteis / 252)
- `Z` = choque aleatório com distribuição normal padrão N(0,1)

Para carteiras com múltiplos ativos, os choques `Z` são **correlacionados**
via **Decomposição de Cholesky** sobre a matriz de correlação entre os
ativos (por padrão, assume-se independência — matriz identidade — mas a
interface já suporta informar uma matriz de correlação real calculada a
partir de `Historico_Precos`).

O retorno da carteira em cada cenário é a soma ponderada pelos pesos:

```
r_carteira = Σ (peso_i × r_i)
```

### Cálculo do VaR e CVaR

1. Geram-se **N cenários** (ex.: 10.000) de retorno da carteira.
2. Ordena-se a distribuição de retornos simulados.
3. **VaR** = o retorno no percentil `(1 − nível de confiança)` — ex.: para
   95% de confiança, é o retorno no percentil 5% (a pior cauda de 5% dos
   cenários).
4. **CVaR (Expected Shortfall)** = a média de todos os retornos que caem
   **na cauda além do VaR** — representa a perda média esperada nos piores
   cenários, sendo sempre ≥ VaR.

### Por que Monte Carlo (e não VaR paramétrico ou histórico puro)?

- Permite modelar **ativos não-lineares** (derivativos) de forma mais
  flexível que o VaR paramétrico (que assume normalidade fechada).
- Não depende de uma janela histórica fixa como o VaR histórico — os
  parâmetros (`μ`, `σ`) podem ser recalibrados e a simulação já reflete
  isso no próximo run.
- É facilmente extensível para incluir correlação entre ativos, o que já
  está modelado na engine via Cholesky.

---

## 4. API REST (C# / .NET 8)

Documentação interativa via **Swagger** disponível em `/swagger` quando a
API roda em ambiente de desenvolvimento.

| Método | Rota                          | Descrição                                            |
|--------|-------------------------------|-------------------------------------------------------|
| GET    | `/api/ativos`                 | Lista os ativos cadastrados                            |
| POST   | `/api/ativos`                 | Cadastra um novo ativo                                  |
| PUT    | `/api/ativos/{id}`            | Atualiza um ativo                                       |
| DELETE | `/api/ativos/{id}`            | Remove (logicamente) um ativo                           |
| GET    | `/api/carteiras`              | Lista as carteiras                                      |
| GET    | `/api/carteiras/{id}`         | Detalha uma carteira (com ativos e pesos)                |
| POST   | `/api/carteiras`              | Cria uma carteira (valida soma de pesos = 100%)          |
| PUT    | `/api/carteiras/{id}`         | Atualiza composição da carteira                          |
| DELETE | `/api/carteiras/{id}`         | Remove uma carteira                                      |
| POST   | `/api/simulacoes/simular`     | **Executa a Simulação de Monte Carlo** e retorna VaR/CVaR + histograma |

### Exemplo de requisição — `POST /api/simulacoes/simular`

```json
{
  "carteiraId": 1,
  "numeroCenarios": 10000,
  "horizonteDias": 21,
  "nivelConfianca": 0.95
}
```

### Exemplo de resposta

```json
{
  "simulacaoId": 42,
  "carteiraId": 1,
  "valorCarteira": 100000.00,
  "varAbsoluto": 4230.15,
  "varPercentual": 0.042301,
  "cVaR": 5870.40,
  "retornoMedioSimulado": 0.011200,
  "desvioPadraoSimulado": 0.025400,
  "piorCenario": -0.098700,
  "melhorCenario": 0.087600,
  "histograma": [
    { "retornoMinimo": -0.10, "retornoMaximo": -0.095, "frequencia": 12 }
  ]
}
```

---

## 5. Frontend (React)

Dashboard de página única com dois painéis:

- **Montagem da carteira** (`PortfolioBuilder`): adicionar ativos
  cadastrados no backend, definir peso (%) de cada um (com validação em
  tempo real de que a soma é 100%), valor total investido e os parâmetros
  da simulação (número de cenários, horizonte em dias, nível de confiança).
- **Resultados** (`SimulationResults` + `HistogramChart`): cartões com VaR,
  CVaR, retorno médio e valor da carteira, além de um **histograma
  (Recharts)** da distribuição de retornos simulados, destacando em
  vermelho a cauda de cenários usada no cálculo do VaR.

```
frontend/src/
├── App.jsx                    → orquestra o fluxo: criar carteira → simular
├── api/varApi.js              → client HTTP para a API C#
└── components/
    ├── PortfolioBuilder.jsx   → formulário de montagem da carteira
    ├── AssetRow.jsx           → linha de ativo + peso
    ├── SimulationResults.jsx  → cartões de métricas
    └── HistogramChart.jsx     → gráfico de distribuição (Recharts)
```

---

## 6. Como rodar o projeto

### Pré-requisitos
- .NET SDK 8
- Node.js 18+
- SQL Server (local, Docker ou Azure SQL)

### Banco de dados
```bash
# Execute o script contra sua instância do SQL Server
sqlcmd -S localhost -i database/schema.sql
```

### Backend
```bash
cd backend
# ajuste a connection string em src/VaR.Api/appsettings.json
dotnet restore
dotnet build
dotnet run --project src/VaR.Api
# Swagger em https://localhost:5001/swagger
```

### Testes
```bash
cd backend
dotnet test tests/VaR.Tests
```

### Frontend
```bash
cd frontend
npm install
# crie um .env com REACT_APP_API_URL=https://localhost:5001/api
npm start
# http://localhost:3000
```

---

## 7. Uso de IA (Claude) no projeto

Conforme o diferencial proposto no escopo, o Claude foi utilizado em duas
frentes específicas — sempre com **revisão humana** do resultado antes do
commit:

### 7.1. Geração dos testes unitários (`VaR.Tests`)

O `MonteCarloEngine` é uma classe estatística e, por isso, seus testes
exigem pensar em casos de borda que não são óbvios em um CRUD comum:
reprodutibilidade via seed, relação entre nível de confiança e VaR,
CVaR ≥ VaR, validação de pesos, etc. Usamos o Claude para:

1. **Mapear os casos de teste relevantes** para um motor de risco
   (ex.: "o VaR a 99% deve ser maior ou igual ao VaR a 90% para a mesma
   carteira?", "o CVaR deve sempre ser ≥ VaR?").
2. **Gerar o esqueleto inicial** dos testes em xUnit + FluentAssertions,
   incluindo `[Theory]`/`[InlineData]` para variar níveis de confiança.
3. **Revisar tolerâncias estatísticas** — como simulações de Monte Carlo
   têm variância entre execuções, o Claude ajudou a definir quando usar
   `Seed` fixa (garantindo reprodutibilidade) versus quando comparar
   apenas a *direção* de uma relação (ex. `varAcoes > varRendaFixa`) em vez
   de valores exatos, reduzindo *flakiness* dos testes.

Todos os testes gerados foram executados e ajustados manualmente antes de
serem incorporados ao projeto — a IA acelerou o mapeamento de cenários e a
escrita boilerplate, mas a validação final do comportamento estatístico foi
humana.

### 7.2. Documentação técnica (Swagger/OpenAPI)

Os comentários XML (`///`) nos Controllers (`AtivosController`,
`CarteirasController`, `SimulacoesController`) foram redigidos com apoio do
Claude para descrever, de forma clara e em português, o propósito de cada
endpoint, seus parâmetros e regras de negócio (ex.: "a soma dos pesos deve
ser 1"). Esses comentários são consumidos automaticamente pelo Swashbuckle
para gerar a documentação interativa do Swagger, mantendo a documentação
sempre sincronizada com o código.

---

## 8. Possíveis evoluções

- Calcular a matriz de correlação real entre ativos a partir de
  `Historico_Precos` (a engine já aceita esse parâmetro).
- Adicionar autenticação/autorização (JWT) e vincular carteiras a usuários.
- Persistir e comparar o histórico de simulações de uma mesma carteira ao
  longo do tempo (série temporal de VaR).
- Suporte a precificação de opções (Black-Scholes) como insumo adicional
  para o motor de Monte Carlo em carteiras com derivativos não-lineares.
