# Detalhes do garbage collection
- hotspot java, utiliza várias técnicas para melhorar a performance do gc, como: evitar fragmentação, localidade/refrência para busca dos dados e compactuação.
- alguns comandos que podemos utilizar.
```
# Habilitar principais otimizações
java -XX:+UseCompressedOops 
     -XX:+UseCompressedClassPointers 
     -XX:+UseStringDeduplication
```
- outro ponto, o tempo de recorrência do gc, não há um cronograma, ele e executado conforme a necessidade de memória, ou seja, com base em eventos.
## STW
```
O Stop-the-World (STW) no Garbage Collector (GC) do Java é um mecanismo importante para garantir a consistência durante a coleta de lixo. Vou explicar como isso funciona:

Mecanismo do STW:
O GC precisa pausar todas as threads da aplicação (application threads) que estão manipulando objetos
Durante o STW, apenas as threads do GC continuam executando
Isso é necessário para evitar que objetos sejam modificados durante a coleta

Como o GC consegue fazer o STW:
Usa um mecanismo chamado "safepoint"
Cada thread da JVM periodicamente checa por requisições de safepoint
Essas checagens são inseridas em pontos específicos do código durante a compilação JIT
Locais comuns para safepoints:

Final de loops
Chamadas de método
Alocação de objetos

Processo:
O GC sinaliza uma requisição de safepoint
As threads continuam executando até atingirem um ponto de safepoint
Quando atingem o safepoint, as threads são suspensas
O GC aguarda todas as threads atingirem um safepoint
Só então inicia a coleta de lixo
Após terminar, as threads são liberadas para continuar

Otimizações modernas:
GCs mais recentes como ZGC e Shenandoah minimizam o STW
Usam técnicas como coleta concorrente
Mantêm STW apenas para operações críticas e muito breves
Mesmo assim, ainda precisam do mecanismo de safepoint

O STW é fundamental para garantir a consistência da heap durante a coleta de lixo, mesmo que seja um dos principais causadores de latência em aplicações Java.
```
## safepoints
```
Os safepoints são pontos específicos no código Java onde é seguro para a JVM pausar a execução das threads. Vou explicar em detalhes:

Definição Técnica:
São locais no código onde o estado da thread é totalmente conhecido pela JVM
Todas as referências de objetos estão em locais conhecidos
O mapeamento entre registradores e variáveis está bem definido
A pilha de execução está em um estado consistente

Onde são inseridos:
Fim de loops
Antes/depois de chamadas de método
Retorno de métodos
Final de blocos de código
Durante alocações de objetos
Em operações de locking/synchronization

Como funcionam:
javaCopy// Exemplo conceitual de como safepoints funcionam
while(condicao) {  // Safepoint inserido no teste do loop
    operacao();    // Safepoint antes da chamada do método
    if(x > 0) {    
        return;    // Safepoint antes do return
    }
}              // Safepoint no final do loop

Por que são necessários:
Permitem que o GC saiba exatamente onde estão todas as referências
Facilitam a pausa sincronizada de threads (STW)
São essenciais para deoptimização de código JIT
Permitem profile sampling preciso
Facilitam o debug da JVM

Comportamento:
A JVM pode requisitar que threads parem em safepoints
Threads checam periodicamente se há requisições de safepoint
Se há uma requisição, a thread para quando atinge o próximo safepoint
A thread só continua quando a requisição de safepoint é liberada

Impacto no desempenho:
Safepoints adicionam overhead mínimo em execução normal
O compilador JIT otimiza as checagens de safepoint
Em loops muito pequenos, podem ser removidos para melhor performance
Loops muito grandes podem ter safepoints adicionais inseridos

Debug e Monitoramento:
bashCopy# Exemplo de comando para ver estatísticas de safepoint
java -XX:+PrintSafepointStatistics -XX:PrintSafepointStatisticsCount=1

Casos especiais:
Código nativo (JNI) não tem safepoints
Algumas operações críticas podem ser marcadas como "no safepoint"
Threads em código nativo precisam retornar ao Java para checar safepoints

É importante notar que safepoints são uma implementação interna da JVM e os desenvolvedores geralmente não precisam se preocupar diretamente com eles, mas entender seu funcionamento ajuda a compreender melhor o comportamento da JVM e do GC.
```

## Tri-Color Marking
```
O Tri-Color Marking é um algoritmo importante usado em Garbage Collection, especialmente em coletores concorrentes. Vou explicar seu funcionamento:

As três cores:
Branco: Objetos potencialmente não alcançáveis (possível lixo)
Cinza: Objetos alcançáveis mas ainda não escaneados
Preto: Objetos alcançáveis e já completamente escaneados

Invariantes do algoritmo:
Forte: Nenhum objeto preto aponta diretamente para um branco
Fraco: Todo objeto branco alcançável por um preto é também alcançável por um cinza

Fases principais:
Inicialização: Todos objetos começam brancos
Marcação inicial: GC roots marcados como cinza
Marcação: Objetos cinzas são processados e suas referências marcadas
Finalização: Objetos brancos são coletados

Desafios em ambientes concorrentes:
Mutador (aplicação) pode modificar referências durante marcação
Precisa manter invariantes mesmo com modificações concurrent
Usa barreiras de escrita/leitura para consistência

Vantagens:
Permite coleta incremental e concorrente
Facilita visualização do progresso do GC
Reduz pause times em GCs modernos

Implementações na prática:
G1 GC usa variação do tri-color
ZGC e Shenandoah usam conceitos similares
CMS também baseia-se neste conceito

Otimizações comuns:
SATB (Snapshot At The Beginning)
Refinamento incremental de cards
Barreiras otimizadas por JIT

Este algoritmo é fundamental para GCs modernos e permite coleta de lixo com baixa latência em aplicações Java.
```

## tempo de pausa
```
O tempo de pausa (pause time) no Garbage Collector é o período em que a aplicação é completamente interrompida (stopped-the-world) para que o GC possa realizar a limpeza da memória.

1. **O que acontece durante a pausa**:
- Todas as threads da aplicação são pausadas
- GC examina e limpa a memória
- Nenhum novo objeto pode ser alocado
- Aplicação fica completamente inativa

2. **Tipos de Pausas**:
- Minor GC (Young Generation)
  - Pausas mais curtas
  - Mais frequentes
  - Geralmente milissegundos

- Full GC (Toda a Heap)
  - Pausas mais longas
  - Menos frequentes
  - Podem durar segundos

3. **Impactos**:
- Latência da aplicação
- Responsividade
- Throughput
- Experiência do usuário

4. **Exemplo Prático**:

Aplicação rodando -> [PAUSA 200ms para GC] -> Aplicação volta a rodar


5. **Fatores que influenciam**:
- Tamanho da heap
- Taxa de alocação de objetos
- Tipo de GC usado
- Configurações da JVM
- Carga da aplicação

Por exemplo: Se você configurar um tempo máximo de pausa de 200ms para o G1GC:
-XX:MaxGCPauseMillis=200

O GC tentará manter as pausas abaixo de 200ms, mas isso não é garantido em todas as situações.
```

## Forwarding Pointers 
```
Os Forwarding Pointers são um mecanismo importante usado durante a compactação de memória no GC. Vou explicar em detalhes:

Conceito básico:
São ponteiros temporários usados durante movimentação de objetos
Permitem atualizar referências para objetos que foram movidos
Mantêm um link entre localização antiga e nova do objeto

Processo em 3 fases:
Fase 1: Marcar objetos vivos e calcular novos endereços
Fase 2: Instalar forwarding pointers nos objetos
Fase 3: Atualizar todas as referências usando os forwarding pointers

Uso prático:
Durante compactação de heap
Em coletores que movem objetos (como G1)
Para manter consistência durante relocação

Benefícios:
Permite movimentação segura de objetos
Facilita atualização de referências
Suporta compactação incremental
Reduz fragmentação de memória

Desafios:
Overhead de memória temporário
Necessidade de sincronização em ambientes paralelos
Complexidade na implementação

Otimizações comuns:
Reutilização de bits no header do objeto
Processamento paralelo de atualizações
Técnicas de compactação sliding

Impacto no desempenho:
Pause time durante instalação dos ponteiros
Overhead de memória durante a compactação
Benefício de redução de fragmentação

Os forwarding pointers são essenciais para implementar GCs que movem objetos e são uma parte fundamental de coletores modernos como G1 e Shenandoah.
```
## shenandoah
- desenvolvido pela redhat
- indicado para reduzir tempos de pausa em heaps grandes

## ZGC
```
O ZGC (Z Garbage Collector) é um coletor de lixo de baixa latência desenvolvido pela Oracle para o Java. Vou explicar suas principais características:

1. Design e Objetivos:
- Projetado para aplicações que requerem baixa latência e trabalham com grandes heaps (terabytes de dados)
- Visa manter pausas de GC abaixo de 10ms, independente do tamanho do heap
- Otimizado para sistemas modernos com muita memória e múltiplos núcleos

2. Principais características:
- Coleta de lixo concorrente, realizando a maior parte do trabalho enquanto a aplicação está em execução
- Usa load barriers e colored pointers para rastreamento de referências
- Compactação de memória concorrente para evitar fragmentação
- Suporte nativo a NUMA (Non-Uniform Memory Access)

3. Vantagens:
- Pausas extremamente curtas, mesmo com heaps grandes
- Escala bem com o aumento do tamanho do heap
- Performance consistente e previsível
- Bom gerenciamento de memória em sistemas NUMA

4. Quando usar:
- Aplicações que necessitam de baixa latência
- Sistemas que trabalham com grandes volumes de dados em memória
- Microserviços e aplicações cloud-native que precisam de performance consistente
- Sistemas de tempo real não críticos

Vale notar que o ZGC está disponível como um coletor de lixo totalmente suportado desde o Java 15, embora tenha sido introduzido experimentalmente no Java 11.

O ZGC continua em desenvolvimento ativo, com melhorias sendo adicionadas em cada nova versão do Java. Por exemplo, recentemente foram adicionadas otimizações para melhor desempenho em containers e ambientes virtualizados.

Para utiliza-lo -XX:+UseZGC -XX:+ZGenerational
```
## G1
```
G1 (Garbage First) é um coletor de lixo moderno da JVM. Vou explicar seus principais aspectos:

Estrutura da Heap:
Dividida em regiões (regions) de tamanho fixo
Cada região pode ser Eden, Survivor, Old ou Humongous
Tamanho padrão da região é calculado baseado no heap (1-32MB)
Regiões Humongous são para objetos grandes (>50% do tamanho da região)

Características principais:
Coleta incremental e paralela
Prioriza regiões com mais garbage ("Garbage First")
Predictable pause times
Compactação automática durante a coleta
SATB (Snapshot At The Beginning) para marcação concorrente

Fases de coleta:
Initial Mark (STW)
Root Region Scan (Concurrent)
Concurrent Mark
Remark (STW)
Cleanup (STW + Concurrent)
Copying (STW)

Remembered Sets (RSet):
Cada região mantém track de referências externas
Permite coleta independente de regiões
Otimiza scanning durante coleta

Collection Sets (CSet):
Conjunto de regiões a serem coletadas
Escolhidas baseadas em efficiency
Balanceia tempo de pausa vs espaço recuperado

Tuning comum:
bashCopy# Exemplos de parâmetros importantes
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
-XX:G1HeapRegionSize=16m
-XX:InitiatingHeapOccupancyPercent=45

Vantagens:
Pause times previsíveis (não há garantia que será acatado)
Bom throughput
Fragmentação reduzida
Melhor escalabilidade em heaps grandes

Desvantagens:
Overhead de memória (RSets)
Complexidade de tuning
Pode ter pause times maiores que CMS em alguns casos

Situações ideais de uso:
Heaps grandes (>4GB)
Requisitos de latência moderados
Aplicações com muitos threads
Sistemas com múltiplos cores

O G1 é o GC default desde Java 9 e é uma excelente escolha para muitas aplicações modernas, especialmente aquelas que precisam de pause times previsíveis em heaps grandes.
```
### g1 old
```
O G1 Old Generation (G1Old) é parte do G1 Garbage Collector, focando especificamente na gerração Old. Vou explicar seus principais aspectos:

Características Principais:


Parte do G1GC (Garbage First)
Gerencia objetos de vida longa
Usa regiões de tamanho fixo
Coleta incremental


Funcionamento:


Objetos sobreviventes da Young Gen são promovidos
Usa RSet (Remembered Sets) para rastreamento
Collection Set (CSet) determina regiões a serem coletadas
Evacuação de regiões selecionadas


Mixed Collections:


Coleta tanto regiões Old quanto Young
Acontece quando Old Gen está muito cheia
Tenta manter dentro dos objetivos de pausa
Seleciona regiões com mais espaço livre primeiro


Configurações Importantes:

bashCopy-XX:G1MixedGCLiveThresholdPercent=85  # Limite para mixed collections
-XX:G1HeapWastePercent=5              # Desperdício tolerável
-XX:G1MixedGCCountTarget=8            # Número alvo de mixed GCs

Situações de Full GC:


Quando não consegue recuperar memória rápido o suficiente
Se a heap ficar muito fragmentada
Quando marking não termina a tempo
Em caso de falha na evacuação


Otimizações:


Coleta regiões com mais garbage primeiro
Balanceia tempo de pausa vs throughput
Usa SATB (Snapshot-At-The-Beginning)
Marking concorrente

O G1Old é crucial para o funcionamento eficiente do G1GC, especialmente em aplicações com grandes heaps e necessidade de baixa latência.
```

### gc flutuante
- quando objetos mortos sao mantigos vivos por causa de referência de outros objetos mortos.
### g1 coleta mista
- thread do gc são executadas simultaneamente com as threads do aplicativo.
- throughput da app e reduzido nesse momento.

### rsets
```
O Reset (R-Set ou Remembered Set) é uma estrutura de dados importante usada no gerenciamento de memória da JVM, especialmente relacionada ao Garbage Collection.
Principais aspectos dos RSets:

Função Principal:


Mantém registro de referências entre diferentes gerações de objetos
Rastreia quais objetos na geração mais velha apontam para objetos na geração mais nova
Evita a necessidade de varrer toda a heap durante coletas de lixo menores


Como Funciona:


Divide a heap em "cards" (pequenas regiões de memória)
Marca cards que contêm referências para objetos em gerações mais novas
Quando um objeto é atualizado, o card correspondente é marcado como "dirty"


Benefícios:


Melhora a performance do GC
Reduz a pausa do GC
Permite coletas parciais eficientes
Otimiza o tempo de varredura da memória


Uso Prático:


Fundamental para GC geracional
Importante em collectors como G1 e CMS
Ajuda a manter a eficiência em aplicações com grandes heaps


Custo:


Consome memória adicional
Requer processamento para manutenção
Pequeno overhead durante a execução do programa

O RSet é uma otimização crucial que permite que o Garbage Collector trabalhe de forma mais eficiente, focando apenas nas áreas relevantes da memória em vez de ter que examinar toda a heap.
```

## ihop
```
O iHOP (Inline Hood Optimization Program) é uma técnica de otimização da JVM HotSpot que faz parte do JIT (Just-In-Time) Compiler.
Principais características:

Funcionamento Básico:


Otimiza chamadas de método inline
Evita overhead de chamadas de método
Melhora o desempenho do código em tempo de execução


Otimizações:


Inlining de métodos pequenos
Eliminação de dispatch virtual
Redução de overhead de chamadas
Melhora o cache locality


Benefícios:


Melhor performance
Redução do uso de memória
Menos chamadas de método
Código mais eficiente


Quando é usado:


Em métodos chamados frequentemente
Em métodos pequenos
Em hot spots da aplicação
Durante otimizações JIT


Limitações:


Não funciona com todos os tipos de métodos
Pode aumentar o tamanho do código
Tem limites de profundidade de inlining
Requer análise em tempo de execução

O iHOP é uma das muitas otimizações que a JVM faz automaticamente para melhorar o desempenho das aplicações Java.
```

## alocação
- quantidade de memoria utilizada pelo objeto

## tempo de vida do objeto
- muito dificil de medir ou estimar
- o tempo de vida (lifecycle) de um objeto é o período entre sua criação (instanciação) e o momento em que ele se torna elegível para coleta pelo Garbage Collector, ou seja, quando não há mais referências alcançáveis a ele.
## pico de alocacao
```
Causas comuns:


Processamento em lote de muitos dados de uma vez
Cache sendo preenchido ou recarregado
Upload de arquivos grandes
Operações de bulk insert no banco de dados
Geração de relatórios grandes
Respostas HTTP com muito conteúdo


Impactos:


Pode disparar GCs mais frequentes
Maior pressão na Young Generation
Pode causar promoção prematura para Old Gen
Risco de OutOfMemoryError se muito intenso
Aumento no tempo de pausa do GC
Degradação de performance da aplicação


Como identificar:


Monitoramento de métricas de alocação
Análise de GC logs
Ferramentas como JVisualVM ou JFR
Aumento repentino no uso de memória
GCs mais frequentes que o normal


Soluções comuns:


Implementar processamento em lotes menores
Usar paginação
Controlar tamanho máximo de uploads/downloads
Object pooling para objetos grandes
Otimizar algoritmos que geram muitas alocações
Ajustar tamanho das generations do GC
```
## stw
```
O STW (Stop-The-World) é um momento durante a execução do Garbage Collector onde todas as threads da aplicação são pausadas para que o GC possa executar seu trabalho.
Detalhando o funcionamento:

O que acontece durante STW:


Todas as threads da aplicação são pausadas
Nenhum novo objeto pode ser alocado
Nenhuma referência pode ser atualizada
Apenas as threads do GC continuam executando


Quando ocorre:


Durante operações críticas do GC que precisam de consistência
Na fase de marcação de objetos vivos
Durante compactação/movimentação de objetos
Quando precisa atualizar referências


Impactos:


Causa latência na aplicação
Pode ser problemático para aplicações que precisam de resposta rápida
O tempo de pausa varia conforme tamanho do heap e quantidade de objetos


Em diferentes coletores:


ParallelOld: STW durante todo o processo de coleta
CMS: Tenta minimizar STW fazendo parte do trabalho concorrentemente
G1: Usa STW mais curtos mas frequentes
ZGC: Tenta manter STW abaixo de 10ms


Monitoramento:


GC logs mostram duração dos STW
Pode ser monitorado via JMX
Ferramentas como JVisualVM mostram pausas
```

## EDEN
- aonde os objetos novos ficam

## TLABS
- thread local allocation buffers
- é uma técnica de otimização importante utilizada pela JVM para melhorar a performance de alocação de objetos. Cada thread recebe seu próprio buffer de memória para alocação de objetos, o que reduz a contenção entre threads durante a alocação.
- quando uma thread precisa alocar um novo objeto, a JVM utiliza o TLAB (Thread Local Allocation Buffer) que é uma área específica da heap reservada para cada thread.
- O processo funciona assim:
  - Cada thread recebe seu próprio TLAB na Young Generation (Eden Space)
  - Quando a thread cria um novo objeto, ele é inicialmente alocado no TLAB dessa thread
  - Isso evita sincronização entre threads durante a alocação (melhor performance)
- Quando o TLAB de uma thread fica cheio:
  - A thread recebe um novo TLAB
  - O TLAB antigo se torna parte normal do Eden Space
  - Se não houver espaço para um novo TLAB, ocorre uma coleta de lixo (GC)
- A alocação no TLAB é mais rápida porque:
  - Não precisa de sincronização entre threads
  - É uma operação simples de ponteiro
  - Reduz contenção na heap compartilhada 
- e outro ponto, quando thread recebe um objeto, ela aponta para outro endereço vazio na memória.

## exemplo simplificado como funciona o gc
```
Considere uma aplicação processando uma lista de pedidos:

Estado Inicial da Heap:

CopyEden (vazio)
S0 (vazio)
S1 (vazio)
Old Gen (alguns objetos antigos)

Aplicação começa a criar objetos:

CopyEden:
- Pedido#1
- Pedido#2
- Pedido#3
- Itens do Pedido
- Objetos temporários

Eden fica cheio, dispara Minor GC:


JVM marca objetos vivos no Eden
Pedidos #1, #2, #3 ainda têm referências (estão vivos)
Objetos temporários não têm referências (são lixo)
Objetos vivos são movidos para S0
Eden é limpo

CopyEden (vazio)
S0:
  - Pedido#1 (age=1)
  - Pedido#2 (age=1)
  - Pedido#3 (age=1)
S1 (vazio)

Mais alocações acontecem:

CopyEden:
  - Pedido#4
  - Pedido#5
  - Mais objetos temporários
S0:
  - Pedidos anteriores
S1 (vazio)

Novo Minor GC:


Objetos vivos do Eden vão para S1
Objetos sobreviventes de S0 também vão para S1
Age dos objetos é incrementado

CopyEden (vazio)
S0 (vazio)
S1:
  - Pedido#1 (age=2)
  - Pedido#2 (age=2)
  - Pedido#3 (age=2)
  - Pedido#4 (age=1)
  - Pedido#5 (age=1)

Se objetos sobrevivem a várias coletas (threshold, geralmente 15):


São promovidos para Old Generation
Exemplo: após mais GCs, Pedidos #1, #2, #3 vão para Old Gen

CopyEden: (novos objetos)
S0/S1: (objetos mais novos)
Old Gen:
  - Pedido#1
  - Pedido#2
  - Pedido#3
Pontos importantes:

Minor GC: coleta apenas Young Gen (Eden + Survivors)
Major GC: coleta Old Gen
Full GC: coleta toda a heap
Survivor spaces (S0 e S1) nunca são usados ao mesmo tempo
Objetos grandes podem ir direto para Old Gen
Promoção prematura pode ocorrer se Survivor ficar cheio

Este é um exemplo simplificado, mas mostra o ciclo básico de vida dos objetos na heap. 
```
## gc open8j
```
O OpenJ9 é um coletor de lixo alternativo desenvolvido inicialmente pela IBM e depois disponibilizado como projeto open source através da Eclipse Foundation. Uma de suas características mais interessantes é o sistema de Arraylets.

Sobre os Arraylets do OpenJ9:

1. Conceito Básico:
- Arraylets são uma forma especial de representar arrays grandes no heap
- Em vez de alocar um array grande contíguo, o OpenJ9 divide em pedaços menores chamados "arraylets"
- Cada arraylet tipicamente tem 2MB de tamanho por padrão

2. Funcionamento:
- Um array grande é dividido em uma "spine" (espinha) e múltiplos arraylets
- A spine contém ponteiros para os arraylets individuais
- Os arraylets são alocados de forma não contígua na memória
- O acesso aos elementos é feito através de indireção via spine

3. Vantagens:
- Reduz fragmentação de memória
- Permite alocação mais eficiente de arrays muito grandes
- Menor impacto nas pausas do GC
- Melhor utilização da memória em sistemas NUMA
- Reduz a necessidade de contiguidade no heap

4. Configuração:
- Pode ser controlado através de parâmetros como:
  - `-Xmca`: Define o tamanho máximo de um array contíguo
  - `-Xmco`: Controla o overhead máximo permitido para arraylets
  - `-Xmca:size`: Define o tamanho do arraylet

5. Casos de Uso Ideais:
- Aplicações que manipulam arrays muito grandes
- Sistemas com restrições de memória contígua
- Ambientes onde a fragmentação de memória é um problema
- Aplicações Big Data que precisam de grandes estruturas de dados

6. Comparação com outros GCs:
- Diferente do ZGC e G1 que focam em regiões
- Abordagem única para tratamento de arrays grandes
- Complementa outras estratégias de gerenciamento de memória do OpenJ9

7. Performance:
- Reduz significativamente o impacto de OutOfMemoryErrors causados por fragmentação
- Overhead mínimo para acesso a elementos do array
- Melhor escalabilidade para arrays muito grandes
- Pode reduzir o tempo total de GC em aplicações com muitos arrays grandes

O sistema de Arraylets é uma das características que torna o OpenJ9 particularmente eficiente em cenários onde há manipulação de grandes estruturas de dados em memória, especialmente em ambientes com restrições de recursos ou necessidade de otimização de memória.
```

# PGR profile guided recompilation
```
O PGR é uma técnica onde o compilador JIT utiliza informações de perfil coletadas durante a execução do programa para tomar decisões mais inteligentes sobre otimizações. Essas informações incluem:

1. Frequência de execução de métodos
2. Padrões de desvios (branches) mais comuns
3. Tipos mais frequentes em sites polimórficos
4. Distribuição de valores
5. Comportamento de exceções

Com base nesses dados de perfil, o compilador pode:
- Decidir quais métodos devem ser compilados
- Realizar otimizações mais agressivas em caminhos críticos
- Fazer especulações mais precisas sobre tipos e fluxos de execução
- Determinar quando é benéfico recompilar um método com otimizações diferentes
```

# compiladores hotspot
```
HotSpot JVM possui dois compiladores principais: C1 (Client Compiler) e C2 (Server Compiler). Vou explicar as características de cada um:

C1 (Client Compiler):
- Também conhecido como Client HotSpot Compiler
- Otimizado para inicialização rápida e menor consumo de memória
- Realiza otimizações mais simples e rápidas
- Ideal para aplicações desktop e programas que precisam iniciar rapidamente
- Usa menos recursos de memória durante a compilação
- Gera código nativo mais rapidamente, mas com otimizações menos agressivas

C2 (Server Compiler):
- Também conhecido como Server HotSpot Compiler ou Opto
- Foca em maior desempenho a longo prazo
- Realiza otimizações mais sofisticadas e agressivas
- Ideal para aplicações server-side que rodam por longos períodos
- Requer mais memória e tempo durante a compilação
- Gera código nativo mais otimizado, mas leva mais tempo para compilar

Na JVM moderna, existe o modo "tiered compilation" que usa ambos os compiladores em conjunto:
1. Primeiro o código começa sendo interpretado
2. Partes quentes são compiladas pelo C1 para obter ganhos rápidos
3. Se o código continua muito utilizado, é recompilado pelo C2 para otimizações mais agressivas
```
# code cache
```
O Code Cache é uma área de memória especial na JVM onde é armazenado o código nativo gerado pelos compiladores JIT (C1 e C2). Veja os principais aspectos:

Características principais:
- Armazena o código compilado para execução nativa
- Tem tamanho limitado e configurável
- Quando fica cheio, pode causar descompilação de métodos
- É dividido em segmentos para diferentes tipos de código compilado

Segmentação do Code Cache:
1. Non-method code: Para stubs e adaptadores
2. Profiled code: Código compilado pelo C1 (nível mais baixo de otimização)
3. Non-profiled code: Código compilado pelo C2 (maior nível de otimização)

Configurações importantes:
- InitialCodeCacheSize: Tamanho inicial
- ReservedCodeCacheSize: Tamanho máximo
- CodeCacheExpansionSize: Incremento quando precisa crescer

Problemas comuns:
- Code Cache cheio pode causar degradação de performance
- Fragmentação pode reduzir espaço útil disponível
- Métodos muito grandes podem não caber no cache
```
- comandos para alterar o tamanho
```
-XX:InitialCodeCacheSize=32m
-XX:ReservedCodeCacheSize=256m
-XX:CodeCacheExpansionSize=64k
```
## dicas para aumentar o desempenho de forma simples
- primeiro habilite os logs
```
-XX:+PrintCompilation      # Log básico de compilação
-XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation    # Log detalhado em arquivo

java -XX:+PrintCompilation -XX:+UnlockDiagnosticVMOptions -XX:+LogCompilation -XX:+PrintInlining MeuPrograma
```
- verifique se os métodos importantes estão sendo compiladas ou executados de forma interpretada.
- aumente o cache e verifica se o número de métodos compilados e maior
- importe em uma ferramenta ou interprete

### caso aumentando o cache não aumentou o número de métodos compilados
```
Existem algumas razões possíveis para isso acontecer. Vamos analisar passo a passo:

1. Verificar se os métodos são "quentes" o suficiente:
   - A JVM só compila métodos que são frequentemente executados
   - Se os métodos não atingirem o threshold de compilação, não serão compilados mesmo com cache maior

2. Checar as configurações de Tiered Compilation:

-XX:+TieredCompilation            # Habilitar/verificar se está ativo
-XX:CompileThreshold=10000        # Ajustar threshold de compilação
-XX:Tier4InvocationThreshold=15000 # Threshold para C2


3. Verificar se não há descompilação acontecendo:

-XX:+PrintCompilation -XX:+PrintCodeCache    # Para monitorar


4. Verificar se os métodos são elegíveis para compilação:
   - Métodos muito grandes podem ser ignorados
   - Métodos com muitas exceções também podem ser ignorados

Para diagnosticar melhor, você poderia:
1. Ativar logs detalhados de compilação
2. Verificar se não há outros limites sendo atingidos
3. Monitorar o comportamento em runtime

```
## aot
```
O AOT (Ahead-of-Time) Compilation é uma abordagem diferente de compilação que apresenta algumas vantagens e desvantagens em relação ao JIT. Vou explicar os principais aspectos:

Funcionamento do AOT:
1. O código é compilado para código nativo antes da execução
2. A compilação acontece durante o processo de build
3. O resultado é um executável específico para a plataforma alvo
4. Não requer compilação em tempo de execução

Principais vantagens:
- Inicialização mais rápida do aplicativo
- Menor consumo de memória em runtime
- Previsibilidade de performance
- Melhor para dispositivos com recursos limitados
- Ideal para microserviços e aplicações nativas em nuvem

Desvantagens:
- Binários específicos para cada plataforma
- Perda de algumas otimizações dinâmicas do JIT
- Maior tamanho do executável final
- Menos flexibilidade para otimizações em runtime

No contexto Java:
- GraalVM Native Image é uma das principais implementações
- Projeto CRaC (Coordinated Restore at Checkpoint)
- Spring Native para aplicações Spring Boot
- Quarkus oferece suporte nativo a AOT
- OpenJ9 também possui recursos AOT

Casos de uso ideais:
- Aplicações serverless
- Microserviços
- Aplicativos CLI
- Sistemas embarcados
- Ambientes com restrições de recursos

O AOT não substitui completamente o JIT - cada um tem seu lugar e caso de uso ideal. Em muitos cenários modernos, especialmente com containers e cloud native, o AOT tem ganhado mais relevância.
```

## comperação entre os gcs
![Captura de tela de 2025-02-04 20-54-30](https://github.com/user-attachments/assets/33be21ed-1935-442e-9787-c2c1ac9db3fb)

