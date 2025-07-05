O Lynx Bench é um backend de testes de performance integrado, que tem como objetivo providenciar uma grande gama de testes, sob uma mesma interface (tanto de entrada quanto de saída ), e que seja facilmente automatizável.

## Frontend
O Frontend é o módulo que interage diretamente com o usuário, ele é responsável por ler argumentos de linha de comando e configurações em formato JSON, também abrangendo as macros procedurais de configuração de benchmark.

Além da interface direta com o usuário,  o Frontend é responsável por planejar a execução e formatar a saída para um padrão estruturado usando Formatters.

```mermaid
  flowchart TD
    A((lynx-bench))--->B[Ler CLI]
    B-->G
    D((cargo bench))--->E[Ler Macros]
    E--->F[Criar Main]
    F--->G[Criar Grupos de Contadores de Performance]
    G--->H[Inserir Benchmarks e Contadores nos Runners]
    H--->I[Executar]
    A--->C[Ler JSON]
    C-->G
```


### Macros
Existem duas macros principais: 
 - `benchmark!(callback(args?), runner, name?, description?, {domain: datapoints}+))` isso incluíra o Benchmark com todas as outras configurações passadas para a Macro para execução no Runner. O nome e descrição são opcionais, quando não são providenciados, o nome será o nome da função e a descrição será o conjunto de datapoints coletados
 - `benchmark_main!(runners)` cria a função main do benchmark, onde todos os runners providenciados serão realmente criados e executados (precisamos disso pois os runners precisam ser estáticos e thread-safe)
 Ao gerar a macro em tempo de compilação (`cargo bench`) uma função main com estrutura similar a esta:

```rust
  fn main(){
    let datapoints = vec![ValidDomains::CPU(ValidDatapoints::CPU(ValidCPUDatapoints::Instructions))];
    let (mut group, mut counters) = Planner::plan(datapoints);
    let runner = Runner::new(None, vec![PerfTool]);
    runner.add_bench(mass_insertion, "Mass Insertion", "Tests insertion of a high amount of entities", datapoints);
    runner.run()
  }
```

Este código não inclue loops, samples, warmup ou overhead passes, mas serve como estrutura básica para uma main.

### CLI
A interface de linha de comando sobrescreve as definições do código, executando todas as funções de benchmark com um mesmo pacote de datapoints. Não permite customização de ferramentas.

### Planner
Cria um Grupo de contadores de performance, grupo este que pode ser ativado antes do benchmark e desativado após o término deste.

Este design é assumidamente inspirada na forma como o `perf` funciona.

O que esta estrutura faz é:
  1. Recebe os Datapoints requisitados pelo usuário
  2. Cria um grupo de contadores de performance utilizando as ferramentas disponíveis
  3. Retorna o Grupo e seus Contadores dentro de uma estrutura `BenchCollection`

## Formatter
Estes plugins são responsáveis por interpretar os resultados providenciados pelas ferramentas, formatando a saída em algum formato customizado ou embutido (JSON, Raw, HTML, Influx, etc).
Os formatters podem ser selecionados por JSON ou argumentos de linha de comando.

O ponto de encaixe dos formatters é logo após a execução de um batch de testes (chunking feito a cada 64kb de dados). Estas estruturas receberão TestResults diretamente, e devem providenciar uma função `format(&self, chunk: Vec<TestResult>) -> String`. Eles funcionam como Serializadores e a forma mais comum de implementa-los é com o `serde`.

## Runners
Runners são o backbone do sistema, eles agregam ferramentas e recebem informações para processar sobre as funções marcadas para teste. 
Eles funcionam utilizando o fluxo de enable->disable->flush providenciados pelas ferramentas dinâmicas, ou separando partes do código (assembly, binario ou Rust) para enviar às ferramentas estáticas.

O funcionamento dos runners é o seguinte:
  1. Lê os Datapoints requisitados em um Benchmark
  2. Executar o Planner para receber o Grupo de contadores de performance 
  3. Inicia o grupo de contadores
  4. Chama a função do benchmark
  5. Encerra o grupo de contadores
  6. Lê a resposta para dentro de um BenchmarkResult 
  7. Repete até o último benchmark


## Statistics
Esta é outra parte altamente customizável do pipeline, providenciando funções de análise e tratamento de dados, transformando resultados de testes em uma forma matematicamente compreensível (geralmente através de relações, funções ).

## Ferramentas
As ferramentas são quaisquer estruturas em Rust que implemente `DynamicTool` ou `StaticTool`.

Os dois tipos de ferramentas têm pipelines diferentes, porém ambos retornam TestResult 

### Ferramentas estáticas
São ferramentas que analisam código, seja ele Rust, assembly ou binário, e retornam informações sobre ele. Um exemplo disso é o LLVM-MCA, que recebe código em assembly e computa dados deterministicos sobre o pipeline de CPU.

Quando os runners precisam executar um analisador estático, eles primeiro devem verificar que tipo de dados esta ferramenta espera e então enviá-lo para a ferramenta.

## Ferramentas Dinâmicas
Analisam informações de performance de processos em execução. Estas ferramentas são as mais poderosas, porém estão sujeitas a erros por motivos aleatórios (número anormal de threads em execução na hora do benchmark por exemplo).

Quando um runner encontra um desses, segue o pipeline descrito na seção [[#Runners]]
