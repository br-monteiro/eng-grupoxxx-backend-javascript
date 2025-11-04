# XXX Currency Converter

[![Unit Tests](https://github.com/br-monteiro/eng-grupoxxx-backend-javascript/actions/workflows/unit-tests.yml/badge.svg)](https://github.com/br-monteiro/eng-grupoxxx-backend-javascript/actions/workflows/unit-tests.yml)

### Dependências
 - Docker v20.10.21
 - Docker Compose v2.12.2
 - Node.js v18.12.1

### Desafio técnico
... O desafio proposto foi expor o valor das mercadorias na moeda corrente do cliente.
Precisamos de uma solução tecnológica em que os clients (frontend, app, outras aplicações backend,dentre outros) possam consultar o valor em outras moedas. ...

### Diagrama

<p align="center">
  <img src="docs/converter-diagram.jpg">
</p>

### Solução
A solução para o desafio proposto foi desenvolver uma REST API que retorna os valores convertidos para as moedas suportadas pela aplicação.

Basicamente o fluxo segue os seguintes passos:
1. Cliente (que pode ser outra API, aplicação desktop, aplicação mobile, etc) realiza uma request do tipo `GET` informando a moeda base e o valor a ser convertido.
2. A requisição é processada pela rota padrão, resgistrada no `Gateway API`, que por sua vez valida se todos os dados necessário estão presentes na requisição, então repassa-os para o `Converter`.
3. O papel do `Converter`, como seu nome sugere, é converter o valor informado para outras moedas, dado uma moeda base e os valores das cotações. Para realizar esta conversão o `Converter` consulta as cotações da moeda base para as demais moedas usando o `RateManager`.
4. Quando uma cotação (rate) é solicitada ao `RateManager`, o mesmo segue alguns fluxos para obter a cotação: primeiro é verificado se há o valor em cache (gerenciado por `RateCacheManager`; O TTL dos registros é de **300 segundos** por padrão em **produção**), em caso de HIT no cache, retorna o valor da cotação cacheada. Em caso de cache MISS, então o `RateManager` segue com o fluxo e solicita ao pool de `RateProvider` (que basicamente são componentes especializados em consulmir as cotações de APIs externas). Aqui foi decidido por usar um pool de `RateProvider` para tentar amenizar os casos de falha de requisição.

### Componentes do Sistema
A aplicação é composta por alguns componentes com responsabilidades diferentes. Abaixo é possível ver alguns detalhes sobre os principais.

#### CurrenciesContainer
O papel do `CurrenciesContainer` é basicamente servir como wrap para as moedas (currency) suportadas para conversão. Este componente fornece métodos usados pela aplicação para saber se uma há uam determinada currency no container, retornar uma determinada currency, retornar todos as identificações das currencies, etc.

Para adicionar uma currency no container, basta usar o metodo `setCurrency(...)`. Abaixo é possível ver um exemplo de código:

```javascript
const currenciesContainer = new CurrenciesContainer()

currenciesContainer.setCurrency(new AbstractCurrency('BRL'))
```

#### AbstractCurrency
O componente `AbstractCurrency` tem o papel de representar uma moeda na aplicação. Este componente precisa que o símbolo da modea seja fornecido no momento da instanciação do objeto (**BRL**, **USD**, ...). Outro papel importante é que ele também possui o metodo `convert`, que é chamado pelo `Converter` no momento da conversão.

Esta decisão foi tomada levando em consideração que podemos ter uma regra de conversão diferente para uma determinada moeda. Abaixo é possível ver um exemplo de como poderia ficar a implementação.

```javascript
class CurrencyBRL extends AbstractCurrency {
  constructor(id) {
    super(id)
  }

  convert(value, rate) {
    return (value * rate) * 1.2 // adiciona 20%
  }
}
```
>Isto não é comum de acontencer, mas podemos usar caso necessário.


#### RateProvider
Outo componente bem importante na aplicação é o `RateProvider`. Este componente serve como class base para implementação reais dos provedores de cotação. Esses são especialistas em fazer a requisição para as APIs externas.

Esta class possui dois principais métodos que devem ser implementados pelas classes filhas:
1. `fetch(...)` que sabe como fazer a requisição para a API de contações.
2. `resultAdapter(...)` que sabe como tratar o resultado obtido pela requisição.

Ambos devem retornar uma `Promise` que será **resolvida** em caso de sucesso ou **rejeitada** em cada de falha.

O methodo deve ter a saída seguindo o seguite `typedef`:

```javascript
/**
 * @typedef CurrencyRateMap
 * @property { string } base
 * @property { Date } rateDate
 * @property { Map<string, number> } rates
 */
```

#### RateProvider implmentation
Para esta aplicação foi implementado dois providers `FixerRateProvider` que usa API de cotações da [Fixer](https://apilayer.com/marketplace/fixer-api#documentation-tab). Também foi implementado um `FakeRateProvider` que basicamente esta consumindo um JSON stático apenas pra fins de testes da aplicação e simulação de um segundo provider:<br>
https://api.npoint.io/1ba6866e9c3737ae0613

### Rotas
As rotas suportadas pelo sistema são as seguites
##### GET /api/v1/convert/:currency/:value
Esta rota retorna `status code 200` e os valores convertidos quando a conversão é realizada com sucesso

```bash
curl --request GET 'http://my-service-converter.com/api/v1/convert/BRL/700'

```

Exemplo do objeto retornado em caso de sucesso:
```json
{
  "status": "success",
  "rates": {
    "USD": 110.66,
    "EUR": 104.91,
    "INR": 9147.83
  }
}

```

Esta rota retorna `status code 404` quando é solicitada uma conversão para uma moeda não suportada pela aplicação.

```bash
curl --request GET 'http://my-service-converter.com/api/v1/convert/EDINHO/150'

```

Esta rota retorna `status code 400` quando o valor fornecido para conversão não é numérico.

```bash
curl --request GET 'http://my-service-converter.com/api/v1/convert/USD/abc123'

```

Esta roda retorna `status code 500` quando não foi possível, por algum motivo, realizar a conversão.

### Monitoramento
Em uma aplicação real de produção é primordial que se tenha monitoramento dos recursos, das exceções, erros, etc relacionados à aplicação. Recomenda-se uma biblioteca ou serviço dedicado à isso, por exemplo:

1. Prometheus + Grafana
2. DataDog
3. Sentry
4. NewReLic

Nesta aplicação, para evistar custos e complexidade, foi usado a lib [express-status-monitor](https://www.npmjs.com/package/express-status-monitor)<br>

Esta biblioteca fornce alguns dados relevantes para o monitoramento da aplicação, porém não é possível setar alertas e montar dashboards mais elaborados.
É possível visualizar esses dados através da rota `/status`.

![Monitoring Page](http://i.imgur.com/AHizEWq.gif)

### Usando Local
Para rodar aplicação local, use o comando:
```bash
NODE_ENV=production FIXER_APIKEY=<apikey-value> docker compose up
```

Note que é preciso fornecer algumas variáveis de ambiente: `NODE_ENV` e `FIXER_APIKEY`.<br>

Os valores suportados para `NODE_ENV` são `development` e `production`. Caso `NODE_ENV` seja omitido, então a aplicação carrega como `development`.<br>

Para obter um valor válido para `FIXER_APIKEY`, é necessário realizar um registor no [site da Fixer](https://apilayer.com/marketplace/fixer-api).

É possível rodar a aplicação usando apenas `FakeRateProvider`, porém, como mencionado anteriormente, este provider apenas consulta um JSON stático em:<br>
https://api.npoint.io/1ba6866e9c3737ae0613

```bash
docker compose up
```

### Tests
Os testes unitários podem ser executados diretamente do container docker. Basta rodar os comandos:

```bash
docker build -t xxx-currency-converter:latest .
```
depois:
```bash
docker run --rm xxx-currency-converter:latest npm test
```

## LAUS DEO ∴
