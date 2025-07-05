O Lynx Bench é um backend de testes de performance integrado, que tem como objetivo providenciar uma grande gama de testes, sob uma mesma interface (tanto de entrada quanto de saída ), e que seja facilmente automatizável.

## Resumo Estrutural
O `lynx-bench` é separado em camadas modulares, organizadas da seguinte forma:
```mermaid
	flowchart LR
		A[CommandProcessor] ---> B[Planner] ---> C[Gatherer] ---> D[Interpreter] ---> E[Statistics] ---> F[Formatter] ---> G[Dispatcher]
```
Existem alguns pontos de customização fixos neste pipeline, são eles:
 - `Planner` pode receber `datapoints` e `domains` customizados
 - `Gatherer` pode receber ferramentas customizadas
 - `Interpreter` pode receber interpretadores customizados
 - `Statistics` é extensível com novos métodos, mas dentro de um mesmo framework
 - `Formatter` pode receber formatadores customizados
 - `Dispatcher` é parte do `Formatter`
## CommandProcessor
Responsável por duas coisas:
 1. Interpretar requisições de benchmark
	 1. Requisitados por CLI (`lynx-bench`)
	 2. Requisitados por JSON (`lynx-bench`)
	 3. Em caso de execução por `cargo bench` ou diretamente no código, esta etapa é ignorada
 2. Interpretar configurações do pipeline
	 1. Arquivo JSON de configuração (`lynx-bench`)
	 2. Chamadas de API de registro de módulos (diretamente no código)
### Customização
Existe um objeto de configuração padrão passado por todo o pipeline:
```rust
	pub struct Bench {
		pub name: String,
		pub description: String,
		pub function: fn(),
		pub perf_counter_group: Option<PerfCounterGroup>
	}

	pub struct PipelineConfiguration {
		pub datapoints: HashMap<DomainName, Vec<Datapoint>>,
		pub dynamic_tools: Vec<Box<dyn DynamicTools>>,
		pub static_tools: Vec<Box<dyn StaticTools>>,
		pub interpreters: Vec<Box<dyn Interpreter>>,
		pub formatters: Vec<Box<dyn Formatter>>,
		pub benches: Vec<Bench>
	}
```
Especifiquemos então os dois pipelines de configuração possíveis:
  1. Arquivo JSON:
	  1. Datapoints são definidos como um map no JSON
	  2. Ferramentas, Interpretadores e Formatadores são bibliotecas dinâmicas (compiladas como `cdylib` para compatibilidade com C)
  2. Chamadas de API: Configuração criada com um Builder

O Builder das chamadas de API tem este formato:
```rust
	impl Builder {
		fn new() -> Self;
		fn add_datapoint(&mut self, domain: DomainName, datapoint: Datapoint) -> &mut self;
		fn append_datapoints(&mut self, domain: DomainName, datapoints: Vec<Datapoint>) -> &mut self;
		fn add_dylib_static_tool(&mut self, tool: Box<dyn StaticTool>) -> &mut self;
		fn add_static_tool(&mut self, tool: Box<dyn StaticTool>) -> &mut self;
		fn add_dylib_dynamic_tool(&mut self, tool: Box<dyn DynamicTool>) -> &mut self;
		fn add_dynamic_tool(&mut self, tool: Box<dyn DynamicTool>) -> &mut self;
		fn add_dylib_interpreter(&mut self, tool: Box<dyn Interpreter>) -> &mut self;
		fn add_interpreter(&mut self, tool: Box<dyn Interpreter>) -> &mut self;
		fn add_dylib_formatter(&mut self, tool: Box<dyn Formatter>) -> &mut self;
		fn add_formatter(&mut self, tool: Box<dyn Formatter>) -> &mut self;
	}
```
Como pode ser notado, existem métodos separados para inserção de módulos dinâmicos, isto é necessário para que o usuário tenha ciência daquilo que está inserindo, considerando que módulos de biblioteca dinâmica são menos eficientes, incentivando o uso de módulos estáticos.

## Planejamento de Execução
Existe uma bifurcação clara no planejamento de execução:
 - O binário `lynx-bench`  deve compilar cada arquivo de benchmark presente em "benches" e executar cada binário gerado em um processo diferente
 - Usando `cargo bench` para teste direto do código, o controle de execução é definido pelo próprio `cargo bench`
**Porque executar em processos diferentes**: Isto permite que executemos vários benchmarks em paralelo, sem que a execução de um contamine a execução do outro. Por exemplo, testes de inserção de entidades vão ocupar grande parte do bandwidth de memória do núcleo, portanto executá-los em paralelo pode gerar alteração entre os benchmarks. Dentro de cada função testada, o usuário pode definir multithreading, porém o funcionamento correto é de responsabilidade do usuário.

A segunda responsabilidade do Planejador de execução é fornecer o grupo de contadores de performance correto para o próximo estágio. Para tanto, o planejador deve verificar:
 1. Quais são os `datapoints` requisitados
 2. Encontrar as ferramentas que fornecem estes `datapoints`
 3. Requisitar que as ferramentas criem o grupo de contadores corretos
 Esta etapa deve modificar os benches definidos na configuração, adicionando o `PerfCounterGroup` gerado pelas ferramentas à entrada do benchmark.

## Leitura de Contadores de performance
Esta parte do pipeline executa realmente os testes requisitados. Para isso, ele segue esta receita:
 1. Ler um benchmark da lista de benchmarks
 2. Executar `enable` seguido de `disable` do `PerfCounterGroup` 3 vezes, coletando o overhead de execução da ferramenta e armazenando em um `PerfCounterGroupOverhead`
 3. Executar `enable` seguido da função de benchmark, seguido de `disable` do `PerfCounterGroup` 2 vezes para warmup (descartando o resultado)
 4. Executar `enable` seguido da função de benchmark, seguido de `disable` do `PerfCounterGroup` em um loop, coletando os dados de benchmark
 5. Coletar informações de benchmark e dispô-los com três métricas: benchmark_results_clean (descontado o overhead), benchmark_results_raw (sem descontar o overhead), overhead
Este processo deve ser executado para cada função de benchmark registrado pelo usuário.

## Interpretação de Resultados
O usuário pode fornecer métodos de interpretação customizados, que podem alterar a estrutura de resultados de benchmark, adicionando informações extras, interpretadas de forma holística, que não podem ser adquiridas antes no pipeline. Isto é útil, por exemplo, para relacionar duas métricas (`instructions` e `cycle` por exemplo), gerando uma análise mais complexa dos resultados.
Esta parte deve retornar o `BenchmarkResult` já tratado:
```rust
	pub enum ResultValue {
		String(String),
		Integer(u64),
		BigInteger(u128),
		FloatingPoint(f64),
		Boolean(bool),
	}
	pub struct Sample(Vec<ResultValue>);
	
	pub struct Result(pub Datapoint, pub Samples);

	pub struct BenchmarkResult {
		pub raw: HashMap<DomainName, Result>,
		pub clean: HashMap<DomainName, Result>,
		pub overhead: HashMap<Counter, f64>
	}
```

## Estatística
Esta parte do pipeline recebe o `BenchmarkResult` já tratado e cria uma estrutura interpretada e matematicamente modelada dos resultados, chamado `FinalResult`:
```rust
	pub struct Metrics {
		pub name: String,
		pub value: ResultValue
	}
	
	pub struct AnalysisResult(pub Datapoint, pub Metrics);
	pub struct FinalResult {
		pub raw: HashMap<DomainName, AnalysisResult>,
		pub clean: HashMap<DomainName, AnalysisResult>,
		pub overhead: HashMap<Counter, f64>
	}
```
A interpretação desta estrutura é extensível, o usuário pode requisitar que certas métricas sejam coletadas, podendo especificar funções de análise customizadas:
```rust
	pub struct Statistics {
		pub single_datapoint_analysis: Vec<fn(Vec<Sample>)>,
		pub holistic_analysis: Vec<fn(BenchmarkResults)>
	}
	pub struct HolisticStatistics {
		pub single_datapoint_anaylsis: Vec<fn(HashMap<Bench, Vec<Sample>>),
		pub holistic_analysis: Vec<fn(Vec<BenchmarkResult>),
		pub holistic_recursive: Vec<fn(Vec<FinalResult>)
	}
```
O usuário pode configurar um pipeline de análise de todos os Benchmarks de forma holística (por padrão desabilitado), que interpreta os resultados de todos os benchmarks, assim como pode fornecer ao `Statistics` normal funções de análise de datapoint (ex.: max, min, avg, moda, etc) e funções de análise holística (capaz de relacionar diversas métricas coletadas).