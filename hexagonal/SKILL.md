---
name: araujo-hexagonal-microservice
description: >-
  Padrão de engenharia dos microsserviços Java/Spring do ecossistema Araújo/AraujoApp, consolidado
  a partir do ms-produto-service e da lib app-commons. Cobre
  arquitetura hexagonal (ports & adapters; core/application/infra), convenções de nomenclatura e
  estrutura de pacotes, borda interface-first (codegen OpenAPI do app-commons), configuração
  centralizada no application.yml, caching resiliente e integrações HTTP via app-commons,
  observabilidade (@AraujoTrace), redução de código morto, docs/ e CHANGELOG por projeto, testes
  (ArchUnit/contrato/Postgres embedded) e disciplina de commits pequenos e fatiados. Use SEMPRE que
  estiver criando, refatorando ou revisando um serviço backend Araújo — ao adicionar ou mover um
  endpoint, client HTTP, cache, use case, port ou adapter; ao decidir o nome de uma classe ou onde
  uma configuração deve morar; ao organizar commits e PRs; ou ao avaliar aderência arquitetural —
  mesmo que o usuário não diga "hexagonal" explicitamente. Estes são padrões de referência, não
  dogma: adapte às peculiaridades de cada serviço e documente os desvios.
---

# Padrão de microsserviço hexagonal Araújo

> Documento consolidado a partir do pacote de skill `araujo-hexagonal-microservice` (`SKILL.md` + 13 arquivos em `references/`). Os links entre arquivos viraram âncoras internas.

## Sumário

- [Pilares de decisão](#pilares-de-decisão)
- [Princípios (as regras de ouro)](#princípios-as-regras-de-ouro)
- [Estrutura de pacotes](#estrutura-de-pacotes)
- [Como usar este guia (roteiro por tarefa)](#como-usar-este-guia-roteiro-por-tarefa)
- [Anti-padrões (sinais de que algo saiu do trilho)](#anti-padrões-sinais-de-que-algo-saiu-do-trilho)

1. [Arquitetura hexagonal (ports & adapters)](#1-arquitetura-hexagonal-ports--adapters)
2. [Casos de uso e CQRS (marker interfaces do app-commons — 5.7.0+)](#2-casos-de-uso-e-cqrs-marker-interfaces-do-app-commons--570)
3. [Nomenclatura e clean code](#3-nomenclatura-e-clean-code)
4. [Borda interface-first e contrato HTTP](#4-borda-interface-first-e-contrato-http)
5. [Configuração centralizada no `application.yml`](#5-configuração-centralizada-no-applicationyml)
6. [Caching e resiliência](#6-caching-e-resiliência)
7. [Integrações externas, observabilidade e segurança](#7-integrações-externas-observabilidade-e-segurança)
8. [Desempenho e queries](#8-desempenho-e-queries)
9. [Catálogo do app-commons (qual módulo usar para quê)](#9-catálogo-do-app-commons-qual-módulo-usar-para-quê)
10. [Fluxo de trabalho e commits pequenos](#10-fluxo-de-trabalho-e-commits-pequenos)
11. [Documentação viva e CHANGELOG](#11-documentação-viva-e-changelog)
12. [Testes](#12-testes)
13. [Revisão e armadilhas de verificação](#13-revisão-e-armadilhas-de-verificação)

---

Este guia consolida o padrão de engenharia dos microsserviços Java/Spring do ecossistema
**AraujoApp**. Ele foi destilado de uma refatoração longa e verificada no `AD.JAV.ms-produto-service`
(a implementação de referência) e da biblioteca compartilhada **`app-commons`** (o padrão vive na
lib e evolui além de qualquer serviço — hoje `5.7.0`).

> **A stack (versão de Java, Spring Boot, Spring Framework) segue o app-commons vigente — não a
> fixe.** O BOM `app-commons-bom` dita a baseline; se hoje é Java 21 / Spring Boot 4, amanhã pode
> ser Java 25 / Spring Boot 4.1. A regra estável não é "use a versão X", é **"acompanhe a versão que
> o app-commons segue"** (BOM + `CHANGELOG.md` da lib). Ao subir a lib, leia o guia de migração
> correspondente antes.

> **Isto é um guia, não um dogma.** Cada serviço tem peculiaridades (um domínio diferente, uma
> integração legada, uma restrição de performance). Quando você precisar divergir, **diverja com
> consciência e registre o porquê** — em `docs/` e no `CHANGELOG.md` do próprio serviço. Um desvio
> documentado é engenharia; um desvio silencioso é dívida. O objetivo do padrão é que qualquer
> pessoa do time abra qualquer serviço e reconheça o mapa.

## Pilares de decisão

Toda decisão de design pesa, nesta ordem:

1. **Não quebrar o existente.** Preservar o contrato com o app cliente — **status HTTP + corpo +
   projeção de campos** — é inegociável. Compatibilidade retroativa vem antes de elegância. Mudar
   comportamento observável é decisão de produto: coordenada, testada e documentada, nunca de carona.
2. **Arrume a casa quando passar por ela.** Regra do escoteiro: deixe o código que você tocou um
   pouco melhor — mate um dead code adjacente, corrija um nome, adicione o log que faltava. É
   oportunista, não heróico: sem refactor gigante de carona num fix (fatie —
   ver [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos)).
3. **Os quatro pilares — pese-os explicitamente em toda escolha:**
   - **Observabilidade** — dá para ver o que aconteceu em produção? (`@AraujoTrace`, log estruturado
     no nível certo, **sem PII**).
   - **Segurança** — segredo fora do código, PII mascarada, sem baixar a guarda de auth/validação.
   - **Resiliência** — degrada em vez de cair? (cache L1+L2, circuit breaker, fallback, timeout são).
   - **Desempenho** — resolve na query, evita N+1, cacheia o caro, não bloqueia num timeout alto.

   Uma solução que ignora um dos quatro está incompleta, mesmo que "funcione" no caminho feliz.

## Princípios (as regras de ouro)

1. **`core` e `application` nunca importam `infra`.** O núcleo de negócio não conhece Spring, JPA,
   Redis, HTTP nem tipos gerados. Isso é o que torna o domínio testável e durável — e é **travado
   por ArchUnit**, então não é opinião, é build vermelho. Veja
   [arquitetura-hexagonal](#1-arquitetura-hexagonal-ports--adapters).
2. **Ports & adapters.** A borda fala com **input ports**; a infra implementa **output ports**. O
   domínio depende de interfaces, nunca de implementações. Services de domínio ficam sem anotação
   Spring e são registrados como `@Bean` no `CoreBeansConfig`.
3. **A borda é interface-first.** Cada controller `implements` a interface gerada pelo
   `app-commons-codegen-openapi` a partir do OpenAPI. O contrato é o spec; o wire (`*Request`/
   `*Response`, `model.*`) é gerado — não escrito à mão. Veja
   [interface-first-e-contrato](#4-borda-interface-first-e-contrato-http).
4. **Nome por capacidade, não por mecanismo.** `ParametroPort` (lê parâmetros), não
   `ParametroRepositoryPort` (a leitura vem de uma lib, não de um repositório). **Versão só na
   borda**: `V1`/`V2` aparecem apenas no controller; service = capacidade, repository = projeção.
   Sem sufixo `DTO` no domínio. Veja [nomenclatura-e-clean-code](#3-nomenclatura-e-clean-code).
5. **Mapper é tradução pura.** Se uma classe "mapper" injeta Service/Port/Repo ou compõe/deriva
   dados, ela é um **Assembler/Service**, não um mapper. Traduza tipos; não orquestre.
6. **Menos código é melhor.** Delete dead code sem dó (query, DTO, service, entidade, cache que
   ninguém chama). Colapse duplicação. Um `DELETE` limpo é uma das melhores contribuições — o
   diff que remove 300 linhas vale mais que o que adiciona 30. Cada tipo a menos é uma tradução a
   menos para manter.
7. **`application.yml` é a fonte única de configuração.** URLs, credenciais, timeouts, TTLs,
   headers, feature-flags de infra — tudo declarado no yml (com `${ENV}` e perfis por ambiente),
   nunca hardcoded e espalhado pelo código. Veja
   [configuracao-centralizada](#5-configuração-centralizada-no-applicationyml).
8. **Resiliência por padrão.** Cache resiliente (`app-commons-resilient-cache`: L1+L2, anti-stampede,
   circuit breaker) e clients HTTP via `app-commons-restclient`. Leituras degradam (miss → fonte);
   escritas propagam. Adapters **nunca retornam `null`**. Veja
   [caching-e-resiliencia](#6-caching-e-resiliência) e
   [integracoes-e-observabilidade](#7-integrações-externas-observabilidade-e-segurança).
9. **Documentar é parte do "pronto".** Toda implementação atualiza a documentação viva na **mesma
   entrega**: o fluxo afetado em `docs/` e uma entrada no `CHANGELOG.md`. Código que muda
   comportamento sem tocar os docs está **incompleto** — não é um passo opcional para depois. Todo
   projeto tem `docs/` e `CHANGELOG.md`. Veja [docs-e-changelog](#11-documentação-viva-e-changelog).
10. **Refatore em fatias pequenas, verificáveis, com rede de testes antes.** Commits atômicos que
    compilam e passam. Veja [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos).

## Estrutura de pacotes

```
br.com.araujo.apparaujo
├── core                     # domínio puro (POJOs), zero framework
│   ├── domain               # entidades de domínio, enums, comandos, resultados
│   │   ├── enums
│   │   └── pbm / ...         # subpacotes por subdomínio
│   └── services             # services de domínio SEM anotação Spring
│       └── validator
├── application              # casos de uso + contratos (ports)
│   ├── usecases             # *UseCase (orquestram domínio + ports)
│   └── boundary
│       ├── input            # *InputPort  (o que a borda chama)
│       └── output           # *Port       (o que a infra implementa)
└── infra                    # tudo que fala com framework/mundo externo
    ├── config               # CoreBeansConfig, *Config, *Properties
    ├── entrypoint/web        # *ControllerImpl (interface-first) + mapper de borda
    ├── external             # clients HTTP + mappers de integração
    └── persistence          # adapter/, repository/, model/ (entidades JPA), cache/
```

Regras da estrutura (verificadas por ArchUnit — ver [testes](#12-testes)):
- Entidades JPA moram em `infra.persistence.model` como `*Model` (não existe pacote
  `persistence.entity`). O domínio usa **nome puro** (POJO), sem sufixo `Model`.
- `application.usecases` não importa repositórios Spring Data, `EntityManager` nem `model.*` gerado.
- Controllers não injetam repositórios; clients HTTP (`infra.external`) não dependem de `persistence`.

> Nota sobre legado: no ms-produto o domínio ainda vive sob `core.domain.dto` (nome de pacote
> herdado) mesmo sendo POJO puro. O **alvo** é `core.domain` sem `dto`. Ao criar um serviço novo,
> já nasça no alvo.

## Como usar este guia (roteiro por tarefa)

| Vou... | Leia primeiro |
|---|---|
| Montar/organizar as camadas, criar um port/adapter/service de domínio | [arquitetura-hexagonal](#1-arquitetura-hexagonal-ports--adapters) |
| Criar um caso de uso, separar leitura/escrita (CQRS: `Query`/`Command`/`AggregateUseCase`, 5.7.0+) | [use-cases-cqrs](#2-casos-de-uso-e-cqrs-marker-interfaces-do-app-commons--570) |
| Nomear uma classe, decidir mapper vs assembler, matar código morto | [nomenclatura-e-clean-code](#3-nomenclatura-e-clean-code) |
| Adicionar/mover um endpoint, mexer no contrato HTTP | [interface-first-e-contrato](#4-borda-interface-first-e-contrato-http) |
| Decidir onde uma config vai, adicionar um client/cache no yml | [configuracao-centralizada](#5-configuração-centralizada-no-applicationyml) |
| Adicionar/tunar cache, lidar com Redis/estoque/circuit breaker | [caching-e-resiliencia](#6-caching-e-resiliência) |
| Integrar um sistema externo (HTTP), instrumentar observabilidade, tratar segredo/PII | [integracoes-e-observabilidade](#7-integrações-externas-observabilidade-e-segurança) |
| Otimizar uma query/fluxo, decidir cache vs query, caçar N+1 | [desempenho-e-queries](#8-desempenho-e-queries) |
| Escolher a lib certa do app-commons | [catalogo-app-commons](#9-catálogo-do-app-commons-qual-módulo-usar-para-quê) |
| Planejar a refatoração, fatiar em commits, montar o PR | [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos) |
| Escrever/estruturar `docs/` ou o `CHANGELOG.md` | [docs-e-changelog](#11-documentação-viva-e-changelog) |
| Escrever testes (unit, contrato, arquitetura, cache) | [testes](#12-testes) |
| Revisar um PR, verificar antes de aprovar, fugir do "verde local" enganoso | [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação) |

## Anti-padrões (sinais de que algo saiu do trilho)

- Um `import ...infra...` dentro de `core`/`application` → o build ArchUnit deveria pegar; se não,
  falta o guard.
- Um controller chamando um repositório ou montando SQL.
- Um "mapper" com um `@Autowired`/construtor recebendo um Service.
- `V1`/`V2` no nome de um service, port, adapter ou repository (a versão é da borda).
- URL, timeout, TTL, token ou header como constante/`@Value` solto no meio de um client.
- Um adapter retornando `null` em erro de leitura (deveria degradar para vazio) ou engolindo uma
  falha de escrita.
- Um `@Cacheable`/cache artesanal com `RedisTemplate` cru em vez do `app-commons-resilient-cache`.
- Trocar o tipo/projeção de um `*Response` existente (resumido↔completo) — some/aparece campo que o
  app espera; quebra o cliente **silenciosamente** (compila e "passa" se o snapshot for cúmplice).
- Um `boolean` de `*Response` que serializa `null` — o parser do app espera `true`/`false`.
- Um PR gigante e monolítico que mistura refactor de arquitetura, feature nova e rename.
- Código novo sem entrada no `CHANGELOG.md` nem menção em `docs/` quando muda um fluxo.

> Antes de recomendar algo do app-commons por memória, **confirme na versão atual da lib**
> (README do módulo + `CHANGELOG.md`, ou o bytecode com `javap`). A lib evolui rápido e a
> implementação de referência costuma estar um passo atrás dela.


---

# Referências


## 1. Arquitetura hexagonal (ports & adapters)

<sub>`references/arquitetura-hexagonal.md`</sub>

A regra de ouro: **`core` e `application` não importam `infra`**. O núcleo de negócio não conhece
Spring, JPA, Redis, HTTP nem tipos gerados. Isso é o que torna o domínio testável em isolamento e
resistente a troca de framework — e é travado por ArchUnit (build vermelho, não opinião).

### As três camadas

- **`core`** — domínio puro. Entidades como POJOs, enums, comandos, resultados e **services de
  domínio**. Zero anotação de framework.
- **`application`** — casos de uso (`*UseCase`) que orquestram domínio + ports, e o **boundary**:
  `input` (o que a borda chama) e `output` (o que a infra implementa). Também sem `infra`.
- **`infra`** — tudo que fala com o mundo: controllers, adapters de persistência/HTTP, entidades
  JPA (`*Model`), cache, config.

### Ports

- **Input port** (`application.boundary.input`, sufixo `InputPort`): a fronteira que o controller
  invoca. Ex.: `BuscarColecaoProdutosInputPort`. O use case implementa o input port.
- **Output port** (`application.boundary.output`, sufixo `Port`): a fronteira que a infra
  implementa. Ex.: `ProdutoRepositoryPort`, `ParametroPort`, `PbmIntegradorClientPort`. O domínio
  depende **da interface**, nunca da implementação.

```
Controller ──chama──▶ InputPort ◀──implementa── UseCase ──chama──▶ OutputPort ◀──implementa── Adapter
   (infra)          (application)               (application)       (application)              (infra)
```

### Services de domínio ficam sem Spring — `CoreBeansConfig`

Services de `core.services` **não** têm `@Service`/`@Component` (isso os acoplaria ao framework).
Eles são POJOs com construtor, registrados como `@Bean` numa única `@Configuration` na infra:

```java
@Configuration
public class CoreBeansConfig {

    @Bean
    public ParametroService parametroService(ParametroPort parametroPort) {
        return new ParametroService(parametroPort);   // recebe o port, não o adapter
    }

    @Bean
    public DepartamentoService departamentoService(DepartamentoRepositoryPort repository,
                                                   DepartamentoCachePort departamentoCache,
                                                   ProdutoServicePort produtoService,
                                                   BuscaDepartamentoSalesforceService buscaDepartamentoSalesforce) {
        return new DepartamentoService(repository, departamentoCache, produtoService, buscaDepartamentoSalesforce);
    }
}
```

O Spring resolve os `@Bean` de port pelos adapters `@Component` que os implementam. Use cases que
dependem só de ports tendem a virar `@Bean` aqui também; os que precisam de component-scan podem ser
`@Service` — mas o `@Service` fica na **classe do use case em `application`**, que continua sem
importar `infra`.

### Adapters: implementam o output port falando com o mundo

O adapter é a única camada que conhece JPA/Redis/HTTP. Ele traduz do tipo de infra (`*Model`,
resposta HTTP) para **domínio** na saída.

```java
@Component
@RequiredArgsConstructor
public class ParametroAdapter implements ParametroPort {   // nome por capacidade, não *RepositoryAdapter
    private final ParamsService paramsService;             // função de lib (app-commons-jpa-param)

    @Override
    public String buscarPorNome(ParametroEnum parametro) {
        return paramsService.get(parametro.name(), "");
    }
}
```

#### Convenção de erro nos adapters (importante)

Todo método de adapter tem `try-catch` e **não declara `throws`**. Adapters **nunca retornam
`null`**. A regra distingue **erro de infra** (encapsula) de **erro de negócio** (propaga):

- **Erro de infra** (Redis/DB lento ou fora, timeout, serialização) → **encapsula**: leitura degrada
  para vazio (`Optional.empty()` / `List.of()` / objeto vazio do builder) e loga em `warn`.
- **Erro de negócio** (`BaseException`, que carrega o `HttpStatus`) → **propaga sem mascarar**. Nunca
  transforme um 404/409 de regra em `empty`/500.

```java
@Override
public Optional<Cliente> buscarPorId(Long id) {
    try {
        return jpaRepository.findById(id).map(mapper::toModel);
    } catch (BaseException e) {
        throw e;                              // negócio sobe preservando o status
    } catch (Exception e) {
        log.warn("[ADAPTER] Erro ao buscar cliente por id: {}", id, e);
        return Optional.empty();              // infra encapsula, nunca null
    }
}
```

Se o adapter **não lança** nenhuma exceção de negócio, **não** deixe um `catch (BaseException)`
morto (cheiro de cópia-cola). Assinaturas por operação:

- **Consulta** → `Optional<T>` / `Optional<List<T>>` (ou `List<T>` vazia).
- **Escrita que devolve agregado** (`salvar`) → `Optional<T>`; **quem decide o erro é o use case**,
  via `.orElseThrow(() -> new XxxException(...))`. Escrita **não** engole a falha silenciosamente
  (um `salvarTodos` que só loga e segue esconde save perdido).
- **Comando** (`remover`) → `void`, com re-throw de exceção customizada em falha.

#### Resolver na query, não no adapter

Filtro/junção/agregação vai na `@Query` (JPQL/nativa) ou em método derivado — não em loop/`stream`
Java sobre o resultado. Trazer o mundo para a memória e filtrar em Java é P1 de performance e mascara
o que o banco resolveria melhor. O adapter monta a query e traduz o resultado; a regra de seleção é
do banco.

#### Transação mora no adapter/repository, não no use case

A demarcação `@Transactional` fica na camada de persistência (classe ou método do
repository/adapter), porque é um detalhe de infraestrutura. Ex.: `@Transactional(readOnly = true)`
na classe do repositório de leitura. O use case orquestra domínio e não deveria saber que existe
transação.

### Entidades JPA são `*Model` em `infra.persistence.model`

O domínio usa **nome puro** (POJO): `Produto`, `Campanha`. A entidade JPA correspondente é
`ProdutoModel`, `CampanhaModel`, em `infra.persistence.model`. Um `*ModelMapper` (MapStruct) traduz
`Model ↔ domínio` **dentro do adapter**. Não existe pacote `persistence.entity`, e o `Model` **não
vaza** para `application`/`core` (isso, sim, é travado por ArchUnit).

> ⚠️ **O naming `*Model` NÃO é validado pelo ArchUnit hoje.** A regra `ENTITIES_DEVEM_TER_SUFIXO` da
> lib valida `*Entity` no pacote `..infra.persistence.entity..` — que a convenção do time (`*Model` em
> `..persistence.model..`; repos de referência: ms-auth, ms-operacao) **não usa**.
> A regra fica órfã, então o sufixo `Model` depende de **review humano**. Quando a lib divergir do
> padrão do time, **o padrão do time prevalece** e a regra da lib é sinalizada para alinhamento
> (a `app-commons-archunit` deveria aceitar `..persistence.model..`).

> **Deletar um `*Model` é mudança de schema.** Em teste, o schema vem das entidades (`ddl-auto`); se
> a tabela é compartilhada, apagar o `Model` pode fazer o schema vir de outra entidade com menos
> colunas, quebrando seeds/SQL nativo **só** na pipe. Antes de deletar: `grep` por `FROM <tabela>`
> no SQL nativo e `INSERT INTO <tabela>` nos seeds de teste. Ver
> [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação).

### O guard ArchUnit (o que trava tudo isso)

Todo serviço tem um teste de arquitetura. O `app-commons-archunit` já traz regras reutilizáveis
(`HexagonalArchitectureRules`, `CodingStandardsRules`) — prefira-as a reescrever à mão. O núcleo
mínimo, se você escrever local:

```java
@AnalyzeClasses(packages = "br.com.araujo.apparaujo", importOptions = ImportOption.DoNotIncludeTests.class)
class HexagonalPurityArchTest {
    @ArchTest static final ArchRule core_nao_depende_de_infra =
        noClasses().that().resideInAPackage("br.com.araujo.apparaujo.core..")
                   .should().dependOnClassesThat().resideInAPackage("br.com.araujo.apparaujo.infra..");

    @ArchTest static final ArchRule application_nao_depende_de_infra =
        noClasses().that().resideInAPackage("br.com.araujo.apparaujo.application..")
                   .should().dependOnClassesThat().resideInAPackage("br.com.araujo.apparaujo.infra..");
}
```

Detalhes das regras e da estratégia de ratchet em [testes](#12-testes).

### Onde a lógica mora (heurística rápida)

- Regra de negócio que vale para o domínio, independente de origem de dado → **service de domínio**.
- Orquestração de vários ports para atender um caso de uso → **use case**.
- Tradução `Model`/HTTP ↔ domínio, montagem de SQL, chamada de lib → **adapter**.
- Tradução `domínio` ↔ `model.*` (wire gerado) → **mapper de borda** em `entrypoint/web/mapper`.

Se você está prestes a colocar um `if` de regra de negócio dentro de um adapter, pare: ele
provavelmente pertence ao domínio.


---


## 2. Casos de uso e CQRS (marker interfaces do app-commons — 5.7.0+)

<sub>`references/use-cases-cqrs.md`</sub>

A partir do `app-commons` **5.7.0**, os casos de uso têm um contrato-base e uma separação semântica
explícita entre **leitura** e **escrita** (CQRS), via interfaces marker em `app-commons-core`
(pacote `br.com.araujo.apparaujo.usecases`). Isso torna a intenção do use case parte do **tipo** —
o compilador e o leitor sabem se uma operação lê ou modifica estado.

> A implementação de referência (ms-produto) está em 5.6.2 e **predata** esses markers — seus use
> cases ainda são `*UseCase` "puros". Ao subir para 5.7.0+ ou criar serviço novo, adote os markers.

### As quatro interfaces

| Interface | Semântica | Assinatura |
|---|---|---|
| `UseCase<I, O>` | Contrato-base de qualquer caso de uso | `O execute(I input)` |
| `Query<I, O>` | **Leitura** — idempotente, **sem efeito colateral** | `extends UseCase<I, O>` (marker) |
| `Command<I, O>` | **Escrita** — modifica estado; retorna o recurso ou `Void` | `extends UseCase<I, O>` (marker) |
| `AggregateUseCase` | Orquestra **várias** operações (query+command) sobre o mesmo agregado | marker sem genéricos |

`Query` e `Command` só existem para diferenciar a intenção; ambas herdam o `execute` de `UseCase`.

### Como aplicar (operação única)

O **contrato** (input port que o controller chama) estende o marker CQRS certo; o **`*UseCase`** o
implementa com o `execute`:

```java
// Leitura — o contrato é uma Query
public interface BuscarProdutoPorIdQuery extends Query<Long, Produto> {}

@Service
@RequiredArgsConstructor
public class BuscarProdutoPorIdUseCase implements BuscarProdutoPorIdQuery {
    private final ProdutoRepositoryPort repository;
    @Override
    public Produto execute(Long id) {                 // sem efeito colateral
        return repository.buscarPorId(id).orElseThrow(() -> new ProdutoNaoEncontradoException(id));
    }
}

// Escrita — o contrato é um Command
public interface CadastrarClienteCommand extends Command<CadastroClienteComando, Cliente> {}

@Service
@RequiredArgsConstructor
public class CadastrarClienteUseCase implements CadastrarClienteCommand {
    @Override
    public Cliente execute(CadastroClienteComando comando) { /* modifica estado */ }
}
```

O `execute(I): O` unifica a assinatura: `I` é o **input** (um id, um `Pageable`, um objeto de
domínio) e `O` é o resultado de domínio. O controller traduz `model.*` → `I` e domínio → `Response`
(interface-first continua igual — ver [interface-first-e-contrato](#4-borda-interface-first-e-contrato-http)).

### `Query`/`Command` (marker) vs objeto de parâmetro (`*Query`/`*Comando`)

Cuidado com a colisão de nome: são coisas diferentes.

- **`Query<I,O>` / `Command<I,O>`** são markers do **use case** (comportamento).
- **O input `I`** que o use case recebe é um **objeto de domínio** — pode ser um id, um `Pageable`,
  ou um value object de parâmetros (`BuscaPorTermoQuery`) / um comando de escrita
  (`TermoAceiteComando`). Esse objeto é *dado*, não implementa o marker.

Em CQRS isso é coerente: um **objeto de query** (`BuscaPorTermoQuery`, dado) é passado para um
**handler de query** (`... extends Query<BuscaPorTermoQuery, Resultado>`, comportamento). Só não
faça um objeto de dado implementar `Query<I,O>`/`Command<I,O>`.

### Agregados ricos — `AggregateUseCase`

Quando um agregado tem várias operações que fazem parte do mesmo contexto de negócio e **não** cabem
num único `execute` (ex.: Pedido: buscar, mudar fase, cancelar, marcar integrado), use um input port
multi-método + o marker `AggregateUseCase` (sem genéricos) na implementação:

```java
public class GerenciarPedidoUseCase implements GerenciarPedidoInputPort, AggregateUseCase {
    public void mudarFasePedido(String idPedido, FasePedido fase) { ... }        // command
    public List<String> buscarFilaPendentesSincronizacao(Pageable p) { ... }     // query
}
```

O marker documenta que ali convivem leitura e escrita **de propósito** — não é um use case que
deveria ser fatiado em `Query` + `Command` separados. Reserve-o para agregados de verdade; a maioria
dos use cases é uma `Query` **ou** um `Command`.

### Por que isso importa

- **Legibilidade e revisão:** o tipo já diz se a operação pode ter efeito colateral. Um `Query` que
  escreve é um bug de design visível.
- **Consistência de assinatura:** todo use case fala `execute(I): O`, então testes, decorators
  (cache, trace) e composição ficam uniformes.
- **CQRS incremental:** não exige event sourcing nem barramento — é só a disciplina de separar
  leitura de escrita, que já paga em clareza.


---


## 3. Nomenclatura e clean code

<sub>`references/nomenclatura-e-clean-code.md`</sub>

A nomenclatura é a primeira coisa que outra pessoa lê. Nomes consistentes fazem qualquer serviço do
ecossistema parecer o mesmo mapa. As regras abaixo são as que o time convergiu.

### Sufixos por papel

| Papel | Sufixo | Pacote | Exemplo |
|---|---|---|---|
| Input port | `InputPort` | `application.boundary.input` | `BuscarColecaoProdutosInputPort` |
| Output port | `Port` | `application.boundary.output` | `ProdutoRepositoryPort`, `ParametroPort` |
| Caso de uso | `UseCase` | `application.usecases` | `BuscarProdutosPorTermoUseCase` |
| Contrato de leitura (CQRS, 5.7.0+) | `Query` | `boundary.input` | `BuscarProdutoPorIdQuery extends Query<Long,Produto>` |
| Contrato de escrita (CQRS, 5.7.0+) | `Command` | `boundary.input` | `CadastrarClienteCommand extends Command<...>` |
| Service de domínio | *(sem sufixo)* / `Service` | `core.services` | `RoletaService`, `ParametroService` |
| Adapter | `Adapter` | `infra.persistence.adapter` | `CampanhaRepositoryAdapter` |
| Entidade JPA | `Model` | `infra.persistence.model` | `CampanhaModel` |
| Client HTTP | `Client` / `HttpClient` | `infra.external` | `SalesforceSCAPIClient` |
| Controller | `ControllerImpl` | `infra.entrypoint.web` | `ColecaoControllerImpl` |
| Wire de entrada/saída | `Request` / `Response` | gerado (`model.*`) | `ColecaoResponse` |
| Objeto de parâmetro de consulta (dado) | `Query` | domínio | `BuscaPorTermoQuery` |
| Comando de escrita (dado) | `Comando` | domínio | `TermoAceiteComando` |
| Config / properties | `Config` / `Properties` | `infra.config` | `SalesforceSCAPIConfigValues` |

> ⚠️ **Não confunda o marker CQRS com o objeto de parâmetro.** `Query<I,O>`/`Command<I,O>` (5.7.0+)
> são markers do **use case** (comportamento); `BuscaPorTermoQuery`/`TermoAceiteComando` são **dados**
> (o input `I`). O objeto de dado **não** implementa o marker. Detalhes e exemplos em
> [use-cases-cqrs](#2-casos-de-uso-e-cqrs-marker-interfaces-do-app-commons--570).

### O sufixo `DTO` não faz mais sentido — cada objeto já tem papel preciso

`DTO` é um **não-papel**: só diz "objeto que transporta dados", o que não distingue nada. No nosso
layering, todo objeto **já tem um papel claro**, e o mapper (MapStruct) é quem faz a ponte entre
eles — então não sobra lugar para um genérico "DTO":

| Camada | Papel | Nome |
|---|---|---|
| Borda (wire) | contrato HTTP de entrada/saída | `*Request` / `*Response` (gerados) |
| Domínio | POJO puro com significado de negócio | nome limpo (`Produto`, `Campanha`) |
| Persistência | entidade JPA | `*Model` (`ProdutoModel`) |
| Parâmetro de consulta / comando | value object de domínio | `*Query` / `*Comando` |

Como o **`*Model`** cobre persistência e o **POJO puro** cobre o domínio (com `*ModelMapper`
MapStruct traduzindo entre eles, e `*Request`/`*Response` no wire), qualquer classe `*DTO` é ou um
`Model`, ou um POJO de domínio, ou um `Request`/`Response` mal nomeado. Renomeie para o papel real.

Domínio **não carrega validação jakarta** (`@NotBlank`/`@Positive`/...): quem valida é a borda
gerada. Objeto de domínio é sobre significado, não sobre formato de entrada.

### Nome por capacidade, não por mecanismo

O nome descreve **o que a classe faz**, não a tecnologia por baixo. O exemplo canônico:
`ParametroPort`/`ParametroAdapter`, e **não** `ParametroRepositoryPort`, porque a leitura vem de uma
biblioteca (`app-commons-jpa-param`), não de um repositório próprio. Se amanhã a fonte mudar, o nome
continua verdadeiro. Da mesma forma: `SalesforceSearchPort` (busca), não `SalesforceHttpPort`.

### Versão só na borda

`V1`/`V2` representam contratos HTTP diferentes — isso é uma preocupação **da borda**. Então:
- **Controller:** pode ter `V1`/`V2` (`PbmControllerImpl`, `PbmControllerImplV1`).
- **Service, use case, port, adapter, repository, mapper:** **sem** sufixo de versão. O service é uma
  **capacidade** (ex.: `PbmCamposDinamicosService`), o repository é uma **projeção**. Se V1 e V2 da
  borda usam a mesma capacidade, elas convergem no mesmo service.

Quando um contrato realmente diverge (ex.: V1 retorna erro onde V2 retorna vazio), preserve a
divergência **na borda** (dois controllers, mapeamentos diferentes), não bifurcando o domínio. No
ms-produto o V1 do PBM foi inteiramente removido da camada de negócio; só o controller mantém V1.

### Mapper é tradução pura; se orquestra, é Assembler/Service

Um **mapper** converte um tipo em outro, campo a campo, sem efeito colateral e sem buscar dado. No
momento em que uma classe "mapper":
- injeta um Service/Port/Repository, **ou**
- compõe/deriva dados de mais de uma fonte, **ou**
- aplica regra de negócio,

ela deixou de ser mapper. Renomeie para o que ela é: **`*Assembler`** (monta um agregado de várias
partes) ou **`*Service`/`*Resolver`/`*Rule`** (tem lógica). Exemplos reais dessa correção no
ms-produto: um "TagMapper" que resolvia cor/texto virou `TagResolver`/`TagRule`; assemblers de
cadastro PBM viraram `CamposCadastroAssembler`.

**Assembler não é depósito, nem apelido de mapper.** A fronteira tem que ser real nos dois sentidos:
- Se a classe faz **só tradução pura**, ela é um **mapper MapStruct** — não a promova a "Assembler"
  só pelo nome. Um mapper puro rebatizado de Assembler é mentira ao contrário.
- Se a classe **compõe/deriva de várias fontes ou aplica regra**, ela é Assembler/Service — não a
  disfarce de mapper com `if`s escondidos.

⚠️ **Não replique o caso do `Produto` como modelo** — ali houve um "mapper disfarçado" mal resolvido
(tradução e composição misturadas sob o rótulo errado). É o exemplo do que **evitar**: separe a parte
de tradução pura (MapStruct) da parte de composição (Assembler/Service) em vez de amontoá-las numa
classe ambígua.

**Mapper é sempre MapStruct.** Um mapper de verdade (tradução pura) é uma **interface** anotada, não
uma classe escrita à mão — deixe o MapStruct gerar o `*MapperImpl`:

```java
@Mapper(componentModel = "spring", injectionStrategy = InjectionStrategy.CONSTRUCTOR)
public interface CampanhaModelMapper {
    Campanha toModel(CampanhaModel entity);
    CampanhaModel toEntity(Campanha domain);
}
```

- `componentModel = "spring"` → o `*MapperImpl` vira `@Component` injetável.
- `injectionStrategy = InjectionStrategy.CONSTRUCTOR` → dependência entre mappers (`uses = ...`) por
  construtor, **nunca** field injection.
- **Zero regra de negócio acoplada.** Proibido no mapper: `if/else` de negócio, chamada a
  Service/Port, acesso a banco, validação, cálculo com contexto externo. Regra de negócio grudada num
  mapper é a forma mais comum de lógica e I/O escaparem do lugar certo — somem do radar de teste e de
  performance, e ninguém pensa em olhar ali. Apareceu regra? Extraia para o domínio/use case e deixe
  o mapper **só traduzir**.
- `new XxxMapper()` à mão ou um "mapper" com lógica é P1: ou vira interface MapStruct pura, ou (se
  tem regra/composição) é Assembler/Service — nunca um híbrido.
- **`uses = OutroMapper.class` decide a projeção** de um agregado — um mapper que delega ao mapper
  errado (completo onde devia ser resumido) inverte o wire silenciosamente. Confira o `uses` contra
  o tipo do `items:` no spec (ver [interface-first-e-contrato](#4-borda-interface-first-e-contrato-http)).

Por que importa: um mapper puro é trivial de testar e reordenar; um "mapper" que busca dados esconde
I/O onde ninguém espera, e vira um ponto cego de performance e de teste.

### Menos código é melhor

Reduzir volume de código é uma meta explícita, não um efeito colateral.

- **Delete dead code sem cerimônia.** Query que ninguém chama, DTO só auto-referenciado, service
  órfão, entidade de tabela descontinuada, cache cujo `save` nunca é invocado, path B que o path A
  já cobre. Um `git grep`/análise de referências confirma; então remova. O ms-produto encolheu
  centenas de linhas assim (drenar path B, colapsar três tipos de Produto em um, matar
  `ProdutoServiceV1`).
- **Colapse duplicação em vez de parametrizar demais.** Três tipos que representam o mesmo conceito
  (`ProdutoDTO`/`ProdutoResumidoDTO`/`ProdutoModel`) viraram um domínio canônico `Produto`; a borda
  projeta o `Response` que cada endpoint precisa.
- **Cada tipo a menos é uma tradução a menos.** Toda classe de transporte extra é um mapper a mais
  para manter em sincronia. Prefira um domínio canônico + projeções na borda.
- O diff que **remove** com segurança costuma valer mais que o que adiciona. Só exige a mesma rede
  de testes (ver [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos)).

### Boas práticas gerais (travadas por `CodingStandardsRules` do app-commons-archunit)

- **Constructor injection**, nunca `@Autowired` em campo (use `@RequiredArgsConstructor` do Lombok +
  `final`). Torna a dependência explícita e a classe testável sem Spring.
- **`java.time.*`**, nunca `java.util.Date`/`Calendar`/Joda.
- **SLF4J** (`@Slf4j`/`private static final Logger`), nunca `System.out`/`System.err` nem
  `java.util.logging`.
- **Sem lançar exceção genérica** (`Exception`/`RuntimeException`/`Throwable`) — use as exceções do
  domínio/`BaseException` do `app-commons-core`.
- **Fields de instância não `public`** (exceto `static final`).
- **IDs numéricos são `Long`** (nunca `int`); **valores monetários são `BigDecimal`** com
  `RoundingMode.HALF_EVEN`.
- Substitua FQN inline por `import`; evite import não usado (ruído no diff).
- Remova logs de fluxo-feliz poluentes; log de erro no nível certo (`warn` para degradação
  esperada, `error` para falha real).

### Receita de rename com segurança (comprovada)

1. Token-replace com word-boundary preservando EOL/encoding (UTF-8 sem BOM).
2. `git mv` do arquivo — o nome público da classe deve casar com o do arquivo.
3. Se a classe é cacheada, atualize o `value-type` no `application.yml` (senão as entradas antigas
   viram miss).
4. `mvn test` compila tudo — **o build é o detector de colisão** cross-package (resolva com import
   ou FQN). Rode o ArchUnit para confirmar que nenhum sufixo/camada foi violado.


---


## 4. Borda interface-first e contrato HTTP

<sub>`references/interface-first-e-contrato.md`</sub>

A borda HTTP é **interface-first**: o contrato é o OpenAPI (spec), o wire é **gerado**, e cada
controller `implements` a interface gerada. Ninguém escreve `*Request`/`*Response` de borda à mão.

### Por que

- O spec vira a fonte da verdade. Divergência entre doc e código deixa de existir porque o código
  **é** o spec compilado.
- O `model.*` gerado carrega validação (jakarta) e serialização — a borda valida na entrada
  (retorna 400 sozinha), então o domínio não precisa carregar `@NotBlank`/`@Positive`.
- Refatorar o interior do serviço (colapsar tipos, trocar domínio) não muda o wire, porque o wire é
  o spec, não uma classe de domínio exposta por acidente.

### Como funciona (app-commons-codegen-openapi)

O `app-commons-codegen-openapi` (gerador `araujo-spring`) gera, a partir do OpenAPI:
- as **interfaces** de controller (com os `@GetMapping`/`@PostMapping`, params, `@ResponseStatus`);
- os tipos `model.*` (`*Request`/`*Response`, enums de wire).

O controller implementa a interface e só faz a ponte para o domínio:

```java
@RestController
@RequiredArgsConstructor
@AppSecurity.PermitAnyGroup
public class ColecaoControllerImpl implements ColeesController {   // interface gerada

    private final BuscarColecaoProdutosInputPort buscarColecaoProdutosInputPort;   // input port
    private final ColecaoWebMapper colecaoWebMapper;                                // domínio -> Response

    @Override
    public ColecaoResponse buscarProdutosPorColecao(TipoColecaoProdutos tipo, Long id,
                                                    OrdenacaoProduto ordenacao, Pageable pageable) {
        return colecaoWebMapper.toResponse(
            buscarColecaoProdutosInputPort.buscarColecaoProdutosPorTipo(
                TipoColecaoProdutosEnum.valueOf(tipo.name()), id, pageable, toOrdenacao(ordenacao)));
    }
}
```

Repare no padrão:
- **Enum de wire → enum de domínio por `name()`.** O `TipoColecaoProdutos` (gerado) chega já
  validado pela borda; converte-se para `TipoColecaoProdutosEnum` (domínio) via `valueOf(name())`.
  Valor inválido nem chega ao controller — a borda devolve 400.
- **A borda só fala `model.*` e domínio.** Não existe pacote `entrypoint/web/dto` à mão; o único
  transporte de wire é o gerado. O `*WebMapper` traduz domínio → `Response`.
- O use case devolve **domínio**; quem monta o `Response` é o mapper de borda, na infra.

### Envelope: prefira o `*Response` específico; `BaseResponse<T>` está deprecado

Cada endpoint deve retornar o seu **`*Response` específico** tipado no spec. O wrapper genérico
`BaseResponse<T>` do `app-commons-core` está **`@Deprecated(forRemoval)` desde 5.0.0** — não o
referencie no código à mão. O gerador ainda pode emitir um `import ...model.BaseResponse` não usado:
é inofensivo, mas não construa em cima dele.

Quando o endpoint precisa do **tipo cru** (sem qualquer envelope — ex.: um endpoint legado que o app
já consome assim), use no spec:

```yaml
x-return-type: raw
```

Gotchas comprovados: `Void` cru vem boxed; constraints de wire vão no **spec** (`minItems`,
`pattern`, `maximum`), não em anotação no domínio; enum-schema com nome diferente do domínio precisa
do mapeamento explícito por `name()`. O README do gerador pode enganar — o comportamento `raw` é do
template bundled; confirme no bytecode/na versão da lib se estiver em dúvida.

### Paginação e defaults

Paginação usa `x-spring-paginated` (gera `Pageable`). Defaults de página (ex.: "retorna tudo") vão
como `@PageableDefault(size = Integer.MAX_VALUE)` no parâmetro concreto do `@Override` — o Spring lê
a anotação do método concreto, como faz com `@ResponseStatus`. Limite explícito de página (proteção
contra `page` gigante, CWE-400) vai como constraint no `Query` de domínio ou na validação da borda.

### Preservação de contrato: status + corpo + **projeção**

O contrato com o app cliente vai além de status e shape — **a projeção importa**. Trocar o tipo de um
endpoint (ex.: produto **resumido → completo**, ou vice-versa) **muda o conjunto de campos** que o
app recebe: some `estoqueDisponivel`, aparece `estoqueGeral`, etc. Isso é quebra **silenciosa**
(compila, e "passa" se o snapshot for cúmplice) e derruba o Flutter. Regras:

- Ao migrar um endpoint, confirme o tipo do `items:`/response contra o **wire original de produção**
  (histórico git do DTO/response anterior), **não** contra o snapshot da migração. Um comentário
  tipo *"serializa completo apesar do fetch resumido"* é red flag de projeção invertida.
- **`boolean` no wire nunca serializa `null`.** Um campo `boolean` de `*Response` que sai `null`
  quebra o parser do app. Use os dois cintos: `default: false` no schema (gera
  `private Boolean campo = false;`) **e** derivação non-null no mapper
  (`estoqueGeral != null && estoqueGeral > 0`). Vale para todo flag derivado.
- Campo com serialização condicional (omitido quando `null`, formato de data) precisa de
  `x-field-extra-annotation`/config no schema **e** um snapshot capturado do wire **anterior**.

Mudar comportamento observável do contrato é **decisão de produto** — coordenada e documentada, nunca
de carona num refactor.

### Rede de contrato (snapshot) — o pré-requisito de toda migração swagger-first

Antes de migrar um endpoint para interface-first, **congele o wire atual com um teste de snapshot**.
Ele serializa a resposta real e compara byte-a-byte com um JSON fixture. Assim, ao trocar a
implementação por baixo, qualquer mudança acidental no formato quebra o teste — e você prova que a
migração é iso-comportamental para o app cliente.

Fixtures vivem em `src/test/resources/contract/*.json` (ex.: `colecao-produtos.json`,
`produto-completo.json`, `sugestoes-de-busca.json`). Fluxo:

1. **Escreva o snapshot do wire atual** (pré-req, commit próprio: `test(contrato): ...`).
2. **Prepare o spec** (schemas + paths no OpenAPI, sem tocar o endpoint: `feat(swagger): spec-prep`).
3. **Implemente a interface gerada** e remova o handler à mão (`feat(...): interface-first`).
4. O snapshot continua verde → o wire não mudou.

> ⚠️ **A ordem importa: o baseline só vale se capturado ANTES da mudança.** Um snapshot gerado
> *depois* do refactor congela o comportamento **novo** — vira cúmplice e mascara a regressão em vez
> de pegá-la. "Verde" prova que o wire não mudou *desde o snapshot*, não que o snapshot está certo.
> Ver [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação).

Ver a estratégia de fatiamento em [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos) e
os tipos de teste em [testes](#12-testes).

### Prefixo de rota / gateway

O prefixo externo (ex.: `/app/api/v1/produto`) vem do `gateway.forward-prefix` (app-commons) +
`server.forward-headers-strategy=framework` — o `X-Forwarded-Prefix` é injetado e o Spring monta
URLs externas corretas (swagger-ui, `Location`, redirect) **sem hardcode** do prefixo no código.
Ver [configuracao-centralizada](#5-configuração-centralizada-no-applicationyml).


---


## 5. Configuração centralizada no `application.yml`

<sub>`references/configuracao-centralizada.md`</sub>

**O `application.yml` é a fonte única de configuração do serviço.** URL de integração, credencial,
timeout, TTL de cache, header estático, pool, feature-flag de infra — tudo é declarado ali, com
`${ENV}` e perfis por ambiente. O código **lê** config; não a **define**. Config espalhada como
constante mágica, `@Value` solto no meio de um client ou string hardcoded é dívida: ninguém sabe
onde procurar, e mudar um timeout vira caça ao tesouro.

### Por que centralizar

- **Um lugar para olhar.** Operação, on-call e revisão sabem que toda configuração está no yml (+
  os `application-<profile>.yml`). Não há "config perdida" enterrada numa classe.
- **Muda sem recompilar.** `${ENV:default}` deixa o mesmo artefato rodar em qas/prd/local só
  trocando variável de ambiente.
- **Diferença por ambiente fica explícita.** Um valor que muda entre prd e local aparece lado a
  lado nos arquivos de perfil, não escondido num `if (profile.equals("prd"))`.

### Estrutura do arquivo (blocos nomeados)

Mantenha o yml organizado em blocos com cabeçalho, na ordem: Spring core (datasource, redis, jpa,
security) → server/management → **integrações** (restclient, aws) → **cache resiliente** → swagger.
Cada bloco de integração/infra deve ter um comentário curto explicando o **porquê** de um valor não
óbvio (ex.: por que o timeout do Redis difere por ambiente).

### Clients HTTP — `araujo.restclient` (app-commons-restclient)

Cada serviço externo é uma entrada em `services.<nome>`: url, timeouts, e **headers/credenciais
estáticos co-locados** (a lib os aplica como `defaultHeader` — nada de setar header na mão no
client). O client referencia o serviço por nome via `AraujoRestClientFactory.forService("<nome>")`.

```yaml
araujo:
  restclient:
    defaults:
      connectTimeout: 10s
      readTimeout: 20s
    services:
      clinicar:
        url: ${URL_CLINCARX}
        acceptJson: true
        headers:
          Authorization: "Basic ${CLINCARX_TOKEN}"     # credencial no yml, não no código
      pbm:
        url: ${BASE_URL_MULESOFT}/canaisvenda/v3/integradorpbm/
        headers:
          client_id: ${MULESOFT_APP_CLIENT_ID}
          access_token: ${MULESOFT_APP_ACCESS_TOKEN}
      salesforce-scapi:
        url: ${SALESFORCE_SCAPI_BASE_URL}
        scapi:                                          # sub-bloco próprio do serviço
          organization-id: ${SALESFORCE_SCAPI_ORGANIZATION_ID}
          client-id: ${SALESFORCE_SCAPI_CLIENT_ID}
          client-secret: ${SALESFORCE_SCAPI_CLIENT_SECRET}
```

Params específicos de um serviço (org/site/OAuth do Salesforce) podem ficar num sub-bloco
(`services.<nome>.scapi`) com bind próprio via uma `*ConfigValues`/`*Properties`
(`@ConfigurationProperties`) — a lib ignora sub-chaves desconhecidas. Detalhes de uso do client em
[integracoes-e-observabilidade](#7-integrações-externas-observabilidade-e-segurança).

### Cache — `araujo.resilient-cache`

Defaults globais + override por cache, tudo declarativo. TTL, `value-type`, L1, lock, bulkhead no
yml; o adapter só chama `get`/`getAll`. Detalhes em [caching-e-resiliencia](#6-caching-e-resiliência).

```yaml
araujo:
  resilient-cache:
    ttl: 2m
    ttl-jitter: 15s
    l1: { enabled: true, ttl: 5s, max-size: 500 }
    spring-cache-manager: { enabled: true }            # bridge @Cacheable -> AraujoCache (opt-in)
    caches:
      produtos:            { value-type: ...core.domain.dto.Produto, ttl: 3h }
      departamentos:       { value-type: "java.util.List<...DepartamentosSubdepartamentosDTO>", ttl: 2h }
      params:              { value-type: java.lang.String, ttl: 24h }
```

### Perfis por ambiente — `application-<profile>.yml`

O que muda por ambiente mora no arquivo do perfil (`application-prd.yml`, `application-local.yml`,
`application-qas.yml`, `application-dev.yml`), **sobrescrevendo** o base. Exemplo real: o timeout do
Lettuce (Redis) varia porque a latência do Redis difere por ambiente — 5s no base (qas), 2s no prd,
10s no local. Isso é uma decisão de config, então vive no yml de cada perfil, não num `if` no código.

### Regras práticas

- **Nada de segredo commitado.** Credenciais entram por `${ENV}`; o valor real vem do ambiente/secret
  manager. O yml só referencia a variável.
- **Prefixo de rota, feature-flags de infra, pool, threads** → yml (com default). Ex.:
  `gateway.forward-prefix`, `server.tomcat.max-threads: ${TOMCAT_MAX_THREADS:150}`.
- **Parâmetro de negócio dinâmico** (muda em runtime sem deploy) → **não** é config de yml; é
  `t_parametro` via `app-commons-jpa-param` (`ParametroService`/`ParamsService`). Config de yml é
  para o que se decide no deploy; parâmetro de negócio é para o que o time de negócio ajusta a
  quente. Ver [caching-e-resiliencia](#6-caching-e-resiliência) (bridge `params`).
- Ao renomear uma classe cacheada, **atualize o `value-type`** correspondente — senão as entradas
  antigas no Redis viram miss/erro de desserialização.
- Comentário explica o **porquê** de um valor não óbvio, não o óbvio. `readTimeout: 20s` não precisa
  de comentário; "timeout menor no prd porque o Redis do pod é mais rápido e queremos fast-fail"
  precisa.


---


## 6. Caching e resiliência

<sub>`references/caching-e-resiliencia.md`</sub>

Todo cache do serviço usa a lib **`app-commons-resilient-cache`** (bean `AraujoCache`): L1 (Caffeine
local) + L2 (Redis) + proteção anti-stampede (lock distribuído) + circuit breaker. Nada de
`RedisTemplate` cru espalhado nem `@Cacheable` solto pelo código. O padrão é **output port + cache
declarado no yml**.

### O padrão: port + `AraujoCache`, config no yml

O cache é um output port (`*CachePort`) cujo adapter usa o `AraujoCache`. TTL, `value-type`, L1,
lock e bulkhead são declarados em `araujo.resilient-cache` (ver
[configuracao-centralizada](#5-configuração-centralizada-no-applicationyml)); o adapter só chama a operação.

Operações principais:
- `get(key, loader)` — cache-aside get-or-load.
- `get(key, loader, Predicate cacheável)` — o `Predicate` evita cachear resultado vazio/inválido.
- `getAll(Set<K> keys, Function<Set<K>, Map<K,V>> batchLoader)` — **batch**, evita N+1: multi-get do
  que está em cache, chama o loader só para as chaves faltantes, e reidrata o cache. Prefira-o a um
  loop de `get` quando você tem uma lista de ids.
- `invalidateAll` / limpar por nome — o próprio adapter funcional implementa `CacheRepositoryPort`
  (nome + limpar); não há repo de clear-all separado. Prefixo de chave da lib: `araujo:cache:<nome>:`.

#### `value-type` e coleções genéricas

O `value-type` no yml diz à lib como desserializar a entrada do Redis. Desde a 5.6.2 ele aceita
**tipos genéricos canônicos**, então cache de coleção funciona sem virar `List<LinkedHashMap>`:

```yaml
caches:
  produtos:                { value-type: br.com...core.domain.dto.Produto, ttl: 3h }
  departamentos:           { value-type: "java.util.List<br.com...DepartamentosSubdepartamentosDTO>", ttl: 2h }
  departamentos-produtos:  { value-type: "java.util.List<java.lang.Long>", ttl: 2h }
```

Ao renomear/mover uma classe cacheada, **atualize o `value-type`** — senão as entradas antigas viram
miss/erro de desserialização até o TTL expirar.

#### Bridge Spring `@Cacheable` → `AraujoCache` (opt-in, caso especial)

`spring-cache-manager.enabled=true` torna um `AraujoCacheManager` `@Primary`, roteando
`@Cacheable(sync=true)`/`@CacheEvict` para o cache resiliente **sem o código acoplar no
`AraujoCache`**. Existe para um caso específico: rotear um `@Cacheable` de **lib** que você não
controla (ex.: o `@Cacheable("params")` do `app-commons-jpa-param`) para o cache resiliente. Só
caches **com `value-type`** viram resilientes; nomes sem `value-type` caem num `RedisCacheManager`
fallback. ⚠️ Ligar o bridge muda o `CacheManager` `@Primary` — confira o raio de impacto (todo
`@Cacheable(sync=true)` do seu código passa a ir pro bridge).

### Circuit breaker nas leituras Redis

Um `CircuitBreaker` (resilience4j) compartilhado envolve as **leituras** do Redis (`get`/`getList`/
`getAll` e `multiGet` de adapters de leitura). Redis lento/caído → circuito abre → **fast-fail** em
vez de esperar o command-timeout; o `catch` degrada (miss → DB / mantém SQL).

⚠️ **Tensão a calibrar:** um `spring.data.redis.timeout` (command-timeout do Lettuce) **alto** luta
com o fast-fail — cada chamada lenta bloqueia até o timeout **antes** de o CB abrir. Timeout por
ambiente é ok (pod de QA tem Redis mais lento; prd mais rápido; local o pior), mas um valor alto no
read-path quente é P1 de latência. Equilibre latência × falso-positivo do CB.

### Estoque/dado consolidado no Redis: chave ausente ≠ valor 0 (regra crítica)

Quando um read-path sobrescreve um dado (ex.: estoque) com um valor consolidado vindo do Redis
(produzido por outro sistema — no ms-produto, uma Lambda), a regra de ouro é distinguir **chave
ausente** de **valor 0**:

- **Chave ausente** no Redis → **falha de infra** → **mantenha a fonte SQL** (não zere). Se a regra
  fosse "ausente → 0" e o keyspace/cluster/formato estivesse errado, o catálogo inteiro apareceria
  zerado. **P0** se invertido.
- **Chave existe com `0`** → respeite o `0` (é a origem soberana daquele dado).

Implemente devolvendo um nullable (`Integer`) do parser: ausente/inválido → fora do mapa (mantém
SQL); valor válido, inclusive `0` → no mapa (sobrescreve). A mesma regra vale para todas as versões
do endpoint (mesmo port).

### Onde cachear (heurística)

- Cacheia o que é **caro e relativamente estável**: catálogo, departamentos, campos de cadastro,
  config de mensagem. TTL proporcional à volatilidade (produtos 3h; pré-autorização 15m).
- **Não** cacheie o que é volátil e soberano com fonte própria (estoque real, preço "ao vivo") como
  se fosse verdade estável — trate a origem como soberana e só sobrescreva com a regra acima.
- Use o `Predicate` de "cacheável" para não gravar resultado vazio (senão você cacheia o miss).
- Meça: um cache que ninguém lê (ou cujo `save` nunca é chamado) é dead code — remova (aconteceu no
  ms-produto: cache de coleção sem consumidor).

### Serialização é contrato interno

A serialização byte-a-byte do que vai pro Redis é travada por ITs de round-trip com Redis real
(testcontainers, `*CacheSerializacaoTest`, `@DisabledWithoutDocker` — pulados sem Docker, rodam na
pipe). Rode com Docker local antes de mexer em fiação de cache/boot. Ver
[testes](#12-testes).


---


## 7. Integrações externas, observabilidade e segurança

<sub>`references/integracoes-e-observabilidade.md`</sub>

Três pilares que andam juntos na borda de saída: como chamar um sistema externo, como enxergar isso
em produção, e como não vazar segredo/PII no caminho.

### Clients HTTP — `app-commons-restclient`

Todo client HTTP usa o `AraujoRestClientFactory.forService("<nome>")` (Spring `RestClient`). O
`<nome>` casa com a entrada em `araujo.restclient.services.<nome>` do yml (url, timeouts, headers/
credenciais estáticos) — ver [configuracao-centralizada](#5-configuração-centralizada-no-applicationyml). O OkHttp foi
100% removido do ecossistema; não reintroduza.

```java
@Service
@RequiredArgsConstructor
public class SalesforceSCAPIClient {
    private static final String SERVICE_NAME = "salesforce-scapi";
    private final AraujoRestClientFactory restClientFactory;
    private final SalesforceSCAPIConfigValues config;   // sub-bloco do yml, bind próprio

    @AraujoTrace("salesforce.scapi.buscar-produtos")
    public ShopperSearchResponse performProductSearch(String accessToken, ShopperSearchRequest params) {
        return restClientFactory.forService(SERVICE_NAME)
            .get()
            .uri(b -> b.pathSegment("search", "shopper-search", "v1", ...).build())
            .header(HttpHeaders.AUTHORIZATION, "Bearer " + accessToken)
            .accept(MediaType.APPLICATION_JSON)
            .exchange((request, response) -> {          // .exchange() p/ status sem lançar
                if (response.getStatusCode().isError()) {
                    throw new BaseException("Erro na busca SCAPI");
                }
                return response.bodyTo(String.class);
            });
    }
}
```

Padrões do client:
- **`.exchange((req, resp) -> ...)`** quando você precisa inspecionar o status sem que o RestClient
  lance sozinho — decide degradar vs propagar você mesmo.
- **Degrade defensivamente na borda de leitura.** Se o serviço externo pode devolver `null`/vazio,
  proteja: `Objects.requireNonNullElse(resp.getHits(), List.of())`. Um `"hits": null` do parceiro não
  deve derrubar a request.
- **Credenciais/headers estáticos no yml** (a lib os aplica como `defaultHeader`), nunca hardcoded no
  client. OAuth em runtime (obter token) é código; api-key/`client_id` fixos são config.
- O client fala tipos de `infra.external.dto`; a tradução para **domínio** é do adapter/mapper de
  integração. O client/`Shopper*`/DTO de parceiro **não vaza** para `core`/`application`.
- **Contrato externo é do parceiro, não seu.** Timeouts, retries e o mapeamento de erro moram aqui,
  isolados — `infra.external` não depende de `persistence` (travado por ArchUnit).

### Observabilidade — `@AraujoTrace`

`@AraujoTrace("nome.de.negocio")` (do `app-commons-observability`) cria um span real na transação
New Relic ativa (o aspect `AraujoAopAdvisor` existe na lib — **não é no-op**) e faz `noticeError` na
exceção. Pré-reqs no serviço: `newrelic-api` no classpath, `spring-boot-starter-aspectj`, e o
`-javaagent` do New Relic (sobe por padrão no EKS).

Onde instrumentar:
- **Entrypoints, clients externos e orquestradores/use cases de fan-out** — o que você quer ver na
  waterfall quando algo está lento.
- Nome de span é **negócio explícito**, ex.: `pbm.buscar-termo-aceite`, `salesforce.scapi.buscar-produtos`.
- **Nunca PII no nome do span** (jamais CPF, e-mail, token). O nome é dimensão indexada.

#### Atributos customizados — `NrContext` e `@AraujoAttribute`

`@AraujoTrace` cria o span; para **enriquecer** a transação/span com atributos de negócio que você
filtra e agrupa no New Relic (NRQL), use:

- **`@AraujoAttribute`** — declarativo, quando o atributo é um parâmetro do método.
- **`NrContext`** (`app-commons-observability.nr`, `@since 5.1.0`) — **API imperativa/fluente** para
  imputar atributos dinâmicos em qualquer ponto do código, sem anotação. É a forma recomendada quando
  o valor só existe no meio do fluxo:

```java
// one-shot
NrContext.put("pedido.id", pedidoId);
NrContext.putAll(Map.of("loja.id", lojaId, "canal", "APP"));

// condicional / lazy (só avalia o Supplier se a condição bater — bom p/ valor caro)
NrContext.putIf(isRetentativa, "pedido.tentativa", tentativa);
NrContext.putLazy(debug, "payload.tamanho", () -> calcularTamanho(payload));

// bloco com escopo (atributos prefixados, válidos só dentro do try)
try (var scope = NrContext.scoped("pagamento")) {
    scope.put("tipo", tipoPagamento);
    scope.put("bandeira", bandeira);
}

// span customizado com atributos + registro de erro
NrContext.span("consultar-estoque", span -> {
    span.put("sku", sku);
    return estoqueService.consultar(sku);
});
```

Também tem `builder()` fluente, `putWithMdc` (leva o atributo também para o MDC/log) e
`error(throwable, attrs)`. **Mesma regra do span: nunca PII** — atributo customizado é dimensão
indexada no NR; nada de CPF/e-mail/token. Prefira ids de negócio, status, canal, flags.

Logs: estruturados (JSON fora de local/dev, via o resource de logback do `app-commons-observability`),
SLF4J, no nível certo — `warn` para degradação esperada, `error` para falha real. `debug` de fluxo
feliz é ruído; remova.

### Segurança

Segurança é pilar de decisão, não checklist do fim:

- **Segredo sempre por `${ENV}`**, nunca hardcoded — e **sem default** para segredo obrigatório
  (`${JWT_KEY_ID:uuid-fixo}` é P1: dá a falsa sensação de que a app sobe sem o segredo). Chave
  **pública** pode ser versionada; privada/keystore, nunca.
- **PII mascarada no log** — use `@Obfuscate` (`app-commons-core`) / ofuscação nos campos sensíveis;
  identificador/segredo logado cru ou no campo errado é P1. Combine com "sem PII em span".
- **Auth pela borda** — resource server OAuth2/JWT via `app-commons-security`; a autorização de grupo
  fica em anotação no controller (`@AppSecurity.PermitAnyGroup`/equivalente), não espalhada.
- **Não baixe a guarda de validação** ao migrar: se você remove validação do domínio, confirme que a
  borda gerada (schema) passou a garanti-la — senão você abriu um buraco (ver
  [interface-first-e-contrato](#4-borda-interface-first-e-contrato-http)).
- Header fora do payload tipado (ex.: `Authorization`) é lido do `HttpServletRequest` no
  `*ControllerImpl` — não invente parâmetro no input port por causa disso.
- Grant OAuth2 legado (`password`) mantido por compat do app não bloqueia, mas **exige comentário +
  débito** (migrar para auth code + PKCE) registrado no `CHANGELOG`/`docs`.

### Serialização de payload de integração

Prefira o `JsonConverter` (bean injetável do `app-commons-core`, usa o `ObjectMapper` configurado
pelo Spring) ao `JsonUtils` estático (`@Deprecated`). O injetável respeita a config da app (datas,
inclusão de nulos) e é mockável em teste. Ao trocar a engine de JSON (ex.: Jackson 2 → 3), **valide
a serialização de saída dos payloads em QA** — divergência de módulo/config é silenciosa.


---


## 8. Desempenho e queries

<sub>`references/desempenho-e-queries.md`</sub>

Desempenho é pilar de decisão, não otimização prematura de fim de sprint. As regras aqui são as que
mais deram retorno no ms-produto (mega-queries de catálogo a frio) e valem para qualquer serviço.

### Resolva no banco, não em Java

- **Filtro/junção/agregação vai na query** (`@Query` JPQL/nativa ou método derivado), não em
  `stream().filter()` sobre o resultado. Trazer o mundo para a memória e filtrar é o anti-padrão de
  performance mais comum. O adapter monta a query; a seleção é do banco.
- **Projete só o que precisa.** Endpoint resumido busca colunas resumidas; não carregue o grafo
  inteiro para descartar. Menos coluna, menos join, menos rede.
- **Cuidado com `LEFT JOIN` que multiplica linhas.** Um join para imagem/atributo N:1 pode explodir
  o resultado e exigir `DISTINCT`/`GROUP BY` caro. Prefira subquery/lateral, ou traga a "primeira
  imagem" com um critério determinístico em vez de todas.

### N+1 é o inimigo silencioso

- Ao buscar por uma **lista de ids**, use **um** round-trip: query com `IN (:ids)` / **array param**
  (`= ANY(:ids)` no Postgres) e, no cache, `getAll(keys, batchLoader)` (ver
  [caching-e-resiliencia](#6-caching-e-resiliência)) — nunca um loop de `buscarPorId`.
- Passar um **array param** único (`= ANY(?)`) em vez de expandir `IN (?,?,...,?)` evita replanejar
  o statement a cada tamanho de lista e explode menos o plano.
- Lazy loading dentro de um loop = N+1 disfarçado. Busque o necessário de uma vez (join fetch
  consciente, ou uma segunda query em lote).

### Restrinja o escopo de cada sub-busca

Lições concretas do read-path de produto (aplicáveis por analogia):
- **Restrinja a query ao subconjunto certo.** Só produtos de PBM precisam do join de PBM; só vacinas
  precisam do ato vacinal. Guardar cada sub-busca por uma condição (produto é vacina? tem PBM?) evita
  varrer/juntar dado irrelevante para o catálogo inteiro.
- **Traga o dado consolidado da fonte certa.** Estoque vem do consolidado no Redis, com fallback para
  o SQL — **Redis-first + fallback**, não um join pesado no banco a cada request (e respeitando a
  regra chave-ausente-≠-0, ver [caching-e-resiliencia](#6-caching-e-resiliência)).
- **Uma coisa por vez, resolvida onde é barata.** Tag/selo/cor derivados são regra de domínio
  resolvida no use case sobre o dado já carregado — não um join extra por produto.

### Cachear o caro e estável

Ver [caching-e-resiliencia](#6-caching-e-resiliência). Em resumo: cacheia o caro e relativamente
estável (catálogo, departamentos, campos de cadastro) com TTL proporcional à volatilidade; não
transforme dado volátil/soberano (estoque, preço ao vivo) em verdade cacheada.

### Timeouts e pools são decisão de desempenho (no yml)

- **Command-timeout do Redis** alto luta com o fast-fail do circuit breaker — cada chamada lenta
  bloqueia até o timeout antes de degradar. Calibre por ambiente (ver
  [configuracao-centralizada](#5-configuração-centralizada-no-applicationyml)); valor alto no read-path quente é P1 de
  latência.
- **Pool do datasource (Hikari), threads do Tomcat, connect/read timeout dos clients** são config
  no yml, não constante no código. Dimensione ao perfil real de carga.

### Meça antes e depois

- Otimização sem número é palpite. Use o `@AraujoTrace` (spans no New Relic) para ver **onde** o
  tempo vai antes de mexer, e confirme a melhora depois.
- **Bytecode e plano de execução são a fonte da verdade**, não a intuição. `EXPLAIN (ANALYZE)` para
  a query; `javap` quando a dúvida é sobre o comportamento de uma lib.
- Toda otimização de fluxo/query **atualiza o `docs/`** do fluxo afetado e entra no `CHANGELOG`
  (ver [docs-e-changelog](#11-documentação-viva-e-changelog)) — o "porquê" da query estranha evita que
  alguém a "simplifique" de volta ao problema.

> Equilíbrio: não sacrifique legibilidade por microssegundos irrelevantes. Otimize o que o trace
> aponta como quente; deixe o resto simples. Um `DELETE` de código morto costuma acelerar mais (menos
> a compilar, carregar, cachear) do que um malabarismo de query num caminho frio.


---


## 9. Catálogo do app-commons (qual módulo usar para quê)

<sub>`references/catalogo-app-commons.md`</sub>

O `app-commons` é a lib compartilhada (multi-módulo Maven). Antes de escrever infra "na mão",
verifique se um módulo já resolve — o padrão do ecossistema é **consumir a lib**, não reimplementar.

> **Consuma via BOM.** Declare `app-commons-bom` no `dependencyManagement` e as dependências **sem
> `<version>`**. O BOM dita a versão de tudo (e a baseline de Java/Spring). **Não** use o agregador
> `app-commons` (deprecated desde 5.0, removido na 6.0). A versão vive numa property (ex.:
> `app.commons.version`) — subir = trocar uma linha, ler o guia de migração, rodar os testes.

### Módulos

| Módulo | Use quando | Traz |
|---|---|---|
| `app-commons-bom` | **sempre** (dependencyManagement) | versões alinhadas de todos os módulos |
| `app-commons-core` | sempre | `BaseException`, `BaseResponse` (deprecated), `JsonConverter`, `@Obfuscate`, `Versao`, **markers CQRS `UseCase`/`Command`/`Query`/`AggregateUseCase`** |
| `app-commons-request-context` | precisa de `RequestContext`/usuário/UserAgent | `AuthenticationHelper`, `UserIdResolver` |
| `app-commons-observability` | sempre que quiser trace/log estruturado | filtros de trace/MDC, New Relic, `@AraujoTrace`, `@AraujoAttribute` |
| `app-commons-security` | endpoint autenticado (OAuth2/JWT) | resource server, CORS, entrypoints |
| `app-commons-restclient` | qualquer client HTTP de saída | `AraujoRestClientFactory`, interceptors, propagação de header |
| `app-commons-codegen-openapi` | borda interface-first (gerar controller/model do spec) | gerador `araujo-spring`, `x-return-type`, Swagger UI |
| `app-commons-jpa-param` | ler parâmetro dinâmico de `t_parametro` | `ParamsService` (com cache) |
| `app-commons-resilient-cache` | qualquer cache (L1/L2, stampede, CB) | `AraujoCache`, bridge `@Cacheable`, batch `getAll` |
| `app-commons-recaptcha` | proteger endpoint com reCAPTCHA | `@Recaptcha` + Google reCAPTCHA Enterprise |
| `app-commons-archunit` (test) | sempre (guard de arquitetura) | `HexagonalArchitectureRules`, `CodingStandardsRules` |
| `app-commons-test` (test) | fixtures/asserts de teste | extensions JUnit 5, `DomainAssertions` |

### Grafo de dependência (para entender o que já vem junto)

`core` é a base. `observability → core, request-context`; `restclient → observability`;
`cache → observability`; `security/jpa-param/codegen → core`. Ou seja, ao depender de
`restclient` você já ganha `observability` (e trace) transitivamente. `bom`, `test` e `archunit`
são transversais.

### Regra de ouro sobre versões e memória

A lib **evolui rápido** e costuma estar à frente da implementação de referência (o ms-produto
consumia 5.6.2 enquanto a lib já era 5.7.0). Portanto:

- **Não afirme comportamento de lib por memória.** Confirme na versão vigente: `README` do módulo +
  `CHANGELOG.md` da lib; em dúvida sobre runtime, o **bytecode** (`javap -p -c/-v`) — o README já
  enganou (ex.: "params lê de outra tabela", "@AraujoTrace é no-op" — ambos falsos no bytecode).
- **Ao subir a versão**, leia o `MIGRATION-*.md` correspondente e rode a suíte completa.
- **Se algo do serviço deveria virar lib** (utilitário reimplementado em N serviços), proponha
  promover para o app-commons em vez de duplicar — mas só quando há mais de um consumidor real
  (ex.: o estoque consolidado ficou no ms-produto enquanto era consumidor único).
- **Se uma regra da lib divergir do padrão do time** (ex.: ArchUnit valida `*Entity` mas o time usa
  `*Model`), o **padrão do time prevalece** e a regra da lib é sinalizada para alinhamento — não
  distorça o código para caber numa regra defasada.


---


## 10. Fluxo de trabalho e commits pequenos

<sub>`references/fluxo-de-trabalho-e-commits.md`</sub>

Refatoração de arquitetura só é segura se for **fatiada**. O padrão do time (250+ commits atômicos no
ms-produto) é: cada commit **compila, passa a suíte e faz uma coisa**. Um PR gigante que mistura
refactor, feature e rename é irrevisável e arriscado.

### Commit atômico: uma intenção por commit

- **Um commit = uma mudança coesa** que deixa o repo verde. Se a mensagem precisa de "e" para
  descrever, provavelmente são dois commits.
- **Conventional commits** com escopo: `tipo(escopo): descrição`. Tipos usados no ecossistema:
  `feat`, `fix`, `refactor`, `perf`, `test`, `chore`, `build`, `docs`. Escopo é o subdomínio
  (`produto`, `pbm`, `roleta`, `cache`, `busca`, `colecao`, `departamento`).
- A descrição diz **o que muda e por quê em uma linha**, no imperativo/presente. Ex.:
  `refactor(produto): ProdutoService injeta ProdutoRepositoryPort (core sem infra)`.
- Commits são **assinados** (assinatura obrigatória); não use `--no-gpg-sign` sem pedido explícito.
  Se a assinatura falhar, resolva o gerenciador de credenciais/chave — não contorne desligando a
  assinatura.

### Rede de segurança ANTES da mudança

O princípio que torna refactor grande seguro: **primeiro trave o comportamento com um teste, depois
mude por baixo.** Para migração de contrato HTTP:

1. `test(contrato): snapshot do wire atual` — captura o wire **anterior** (baseline real, não a saída
   da mudança — ver [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação)).
2. `feat(swagger): spec-prep` — schemas + paths no OpenAPI, **sem tocar** o endpoint.
3. `feat(...): interface-first` — implementa a interface gerada, remove o handler à mão.
4. Snapshot continua verde → provou iso-comportamento.

Para pureza hexagonal, o guard ArchUnit é a rede: `test(arch): trava pureza core/application`
**antes** de começar a mover classes — assim qualquer regressão de import falha o build na hora.

### Fatiamento típico de uma migração grande

O ms-produto migrou domínio + borda inteiros assim, em fatias nomeadas e verificáveis:

- **Aditivo primeiro:** introduza o novo (port, domínio canônico, spec) **sem** remover o velho; os
  dois convivem numa ponte. Commit compila com ambos.
- **Migre consumidor por consumidor:** cada service/mapper passa a usar o novo caminho, um commit
  cada (`refactor(produto): X para Produto (drena o hub)`).
- **Drene o legado por último:** quando ninguém mais usa o caminho velho, delete-o
  (`refactor(...): remove ProdutoServiceV1 morto`). O `DELETE` é um commit próprio.
- **Renames são commits isolados** (`refactor(...): ProdutoRepositoryV2 -> ProdutoResumidoRepository`),
  fora da revisão de lógica — evitam ruído no diff que importa.

Assim cada passo é revisável, reversível e a suíte prova que nada quebrou entre eles.

### Ordem quando código depende de dado

Se a mudança de código só funciona **depois** de uma migração de dado (ex.: passar a ler uma coluna
como JSON), isso é um **gate de dado**: o script de dado precisa ir versionado e **junto/antes** do
código no deploy, com a ordem documentada. Código novo com dado velho = todo request cai no default,
silenciosamente ("verde" porque o teste não exercita o path). Documente no PR e no `CHANGELOG`.

### O PR

- **Escopo coeso e revisável.** Prefira vários PRs pequenos a um gigante. Se um PR grande é
  inevitável (migração em massa), organize os commits em fases legíveis e descreva o roteiro na
  descrição.
- **Suíte COMPLETA verde** antes de abrir/aprovar — não um `-Dtest=` direcionado (que não roda
  `@DataJpaTest`/`@SpringBootTest`). Ver [testes](#12-testes) e
  [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação).
- **Docs + CHANGELOG na mesma entrega** (ver [docs-e-changelog](#11-documentação-viva-e-changelog)). Um PR que
  muda um fluxo sem tocar os docs está incompleto.
- PRs no Azure DevOps (org `DrogariaAraujo`, projeto `APP - AraujoDigital`).

### Verificação honesta

- Reporte o resultado real: se um teste falhou, diga com a saída; se um passo foi pulado (ex.:
  `@DisabledWithoutDocker` sem Docker), diga que foi pulado. "Verde local" com subset **não** é
  "testado".
- O build é o detector: `mvn test` compila tudo e roda o ArchUnit + contrato. Rode-o (online contra
  o feed Azure — build offline pode falhar por deps não cacheadas).


---


## 11. Documentação viva e CHANGELOG

<sub>`references/docs-e-changelog.md`</sub>

Documentar é **parte do "pronto"**, não um extra para depois. Toda implementação que muda um fluxo,
um contrato ou uma decisão de infra atualiza a documentação **na mesma entrega**. Código que muda
comportamento sem tocar os docs está incompleto.

Por quê: o próximo dev (ou você em três meses) precisa entender o **porquê**, não só o o quê. Uma
query estranha sem doc vira alvo de "simplificação" que reintroduz o bug que ela evitava. Um fluxo
não documentado vira arqueologia de git a cada dúvida.

### Todo projeto tem `docs/`

Uma pasta `docs/` no raiz, em Markdown, versionada junto do código. A estrutura que funcionou no
ms-produto (adapte ao serviço):

| Arquivo | Cobre |
|---|---|
| `docs/README.md` | índice + visão de 1 parágrafo do serviço e como navegar os docs |
| `docs/arquitetura.md` | camadas, ports/adapters, convenções de design **deste** serviço |
| `docs/caching-e-resiliencia.md` | caches declarados, circuit breaker, timeouts, regras críticas |
| `docs/integracoes-externas.md` | cada sistema externo, o que usa, hardening, observabilidade |
| `docs/convencoes-e-build.md` | naming local, build/libs, testes, ambiente de dev |
| `docs/<fluxo>.md` | fluxos de negócio não óbvios (ex.: busca por cgid, estoque consolidado) |

Regras:
- **Os fluxos têm que ficar claros.** Um fluxo relevante (como um pedido/produto/busca atravessa as
  camadas e integrações) merece um doc com o caminho explícito — de preferência um diagrama simples
  (mermaid) ou uma sequência em texto. Se alguém não consegue seguir o fluxo pelos docs, falta doc.
- **Documente o "porquê" do não óbvio**, não o óbvio. Uma regra contra-intuitiva (chave ausente ≠ 0),
  um timeout por ambiente, um gate de dado — isso precisa de doc. Um getter não.
- **Link entre docs** e para o código (`file:linha`) quando ajudar. Docs que apodrecem enganam;
  atualize o doc no mesmo PR que muda o fluxo.

### Todo projeto tem `CHANGELOG.md`

O CHANGELOG **não era** padrão no ms-produto — e deveria ser. **Todo serviço tem um**, no raiz,
seguindo [Keep a Changelog](https://keepachangelog.com/pt-BR/) + [SemVer](https://semver.org/):

```markdown
# Changelog

Formato: Keep a Changelog. Versionamento: SemVer.

## [Não lançado]
### Added
- ...

## [2.4.0] — 2026-07-31
### Added
- feat(busca): busca de produtos por departamento via Salesforce (cgid). Substitui o CTE recursivo.
### Changed
- refactor(produto): domínio Produto canônico; borda interface-first (18/18 controllers).
### Fixed
- fix(estoque): chave ausente no Redis mantém o SQL; 0 explícito é respeitado.
### Removed
- refactor: remove ProdutoServiceV1 e o path B de resumidos (dead code).
```

Regras:
- Seções **Added / Changed / Fixed / Removed / Deprecated / Security**. Entrada por mudança
  relevante, na linguagem do **impacto**, não do commit cru.
- **Atualize junto com o código.** Uma entrada em `## [Não lançado]` a cada PR que muda
  comportamento; ao publicar, carimbe a versão + data (converta datas relativas para absolutas).
- **Deprecated tem plano.** Marcar algo como deprecated sem `@Deprecated` e sem caminho de saída é
  dívida silenciosa.
- Combine com o padrão de release do serviço (versão numa única property/BOM quando aplicável).

### A relação com esta skill

Esta skill é o padrão **transversal** (vive no app-commons). O `docs/` de cada serviço é a
**instância**: como o padrão se materializa e onde ele diverge (com o porquê). Quando um serviço
precisa fugir do padrão por uma peculiaridade, o `docs/` é onde o desvio é registrado — é isso que
mantém "guia, não dogma" honesto.


---


## 12. Testes

<sub>`references/testes.md`</sub>

A pirâmide do serviço tem quatro camadas, cada uma pegando uma classe de erro que as outras não
pegam. O objetivo não é cobertura por cobertura — é **rede de segurança** que deixa refatorar sem
medo.

### 1. Unit (Mockito) — lógica de negócio

`@ExtendWith(MockitoExtension.class)`, sem contexto Spring. Cobrem **use cases** e **services de
domínio**: caminho feliz **e cada `throw`**. Como o domínio é puro (POJO, sem framework), testa em
milissegundos.

- Mock de port de escrita (`salvar`) retorna `Optional` → teste também o caminho `orElseThrow`.
- Cobertura mede **lógica** (use cases, domain services, utils); infra/boilerplate são excluídos
  (`lombok.config` + `sonar.coverage.exclusions`). Meta de longo prazo ≥ ~85% na lógica de negócio;
  gate JaCoCo no `verify` como **ratchet** (sobe o piso conforme melhora).

### 2. Arquitetura (ArchUnit) — pureza e convenção

Trava as regras estruturais para que o **build** as garanta, não o revisor. Prefira as regras
reutilizáveis da lib (`app-commons-archunit`: `HexagonalArchitectureRules` + `CodingStandardsRules`)
a reescrever à mão — e aplique o **conjunto completo**, não um subset diluído.

```java
@AnalyzeClasses(packages = "br.com.araujo.apparaujo", importOptions = ImportOption.DoNotIncludeTests.class)
class ArchTest extends HexagonalArchitectureTest {}   // herança = zero config

class CodingTest extends CodingStandardsTest {}
```

Cobrem: `core`/`application` sem `infra`/Spring; sufixos de camada; controller não injeta repo; use
case não usa `EntityManager`/repo/`model.*`; entity não vaza; sem `Date` legado, field injection,
`System.out`, exceção genérica. Detalhe das regras em [arquitetura-hexagonal](#1-arquitetura-hexagonal-ports--adapters).

> ⚠️ **O que o ArchUnit NÃO pega hoje:** o naming `*Model` (a regra da lib valida `*Entity` em
> `persistence.entity`, pacote que o padrão `Model` não usa → regra órfã); e todos os smells
> semânticos (projeção de wire, boolean null, snapshot cúmplice, regra em mapper, erro de infra
> mascarado). Esses ficam para o review humano — ver [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação).

### 3. Contrato / snapshot — o wire HTTP

Serializam a resposta real e comparam com um JSON fixture em `src/test/resources/contract/*.json`.
Blindam migrações swagger-first: qualquer mudança acidental de formato/projeção quebra o teste. Ver
[interface-first-e-contrato](#4-borda-interface-first-e-contrato-http).

- **O baseline vale só se capturado ANTES da mudança** (do wire de produção/histórico git), nunca da
  saída da própria migração — senão o snapshot é cúmplice e mascara a regressão.
- Complete com `@WebMvcTest` do `ApiExceptionHandler` travando **status por exceção** + roteamento
  dos `*ControllerImpl`. Ausência disso é um buraco (P1).

### 4. Persistência / integração — schema e serialização reais

- **Adapters/persistência com Postgres embedded/real** (não H2). O ms-produto **migrou de H2 para
  Postgres embedded** de propósito: H2 mascara divergências (tipos, `@Builder.Default` virando null,
  SQL nativo específico do Postgres) que só quebram em produção. Valide o mapeamento contra o schema
  real.
- **Confira que os seeds batem com o schema gerado pelas entidades.** Deletar um `*Model` pode
  derrubar uma coluna que um `@Sql`/`INSERT` inline usa → quebra só aqui/na pipe. Antes de deletar um
  `Model` de tabela compartilhada: `grep` `FROM <tabela>`/`INSERT INTO <tabela>`.
- **Testcontainers é a boa prática para IT que toca infra real** — especialmente **Redis**. Um Redis
  de verdade num container pega o que um mock/fake nunca pega: serialização byte-a-byte, comportamento
  de TTL/eviction, quirks de Lettuce/cluster, o round-trip que o `value-type` do cache depende. Mockar
  o Redis testa o seu mock, não o Redis. O mesmo vale para o Postgres (container/embedded, não H2).
- **ITs de serialização de cache** (`*CacheSerializacaoTest`): `@SpringBootTest` + Testcontainers,
  round-trip com Redis real, travam a serialização byte-a-byte. São `@DisabledWithoutDocker` →
  **pulados sem Docker**, rodam na pipe. Rode com Docker local antes de mexer em fiação de
  cache/boot — e trate "pulado sem Docker" como **não verificado**, não como verde.

### O gate honesto: suíte COMPLETA verde

- **`mvn test` completo (ou o job de CI), não um `-Dtest=Foo` direcionado.** Uma suíte direcionada
  **compila** a árvore inteira (então "compila" não prova nada), mas **só roda os testes nomeados** —
  os `@DataJpaTest`/`@SpringBootTest` que você não nomeou (justamente os que pegam quebra de
  schema/contexto) **não rodam**. Aprovar em cima de subset = não testado.
- Rode **online** contra o feed Azure (`pkgs.dev.azure.com/DrogariaAraujo`) — build offline pode
  falhar por deps não cacheadas (testcontainers, archunit).
- **Reporte honesto:** teste pulado (sem Docker) é "pulado", não "verde"; falha é falha com a saída.
  Ver [revisao-e-armadilhas](#13-revisão-e-armadilhas-de-verificação).

### Novo teste acompanha a mudança

Toda correção de bug ganha um teste que **falha antes e passa depois** (senão a regressão volta).
Todo refactor de contrato ganha (ou reusa) o snapshot que prova iso-comportamento. Rede de segurança
**antes** da mudança grande — ver [fluxo-de-trabalho-e-commits](#10-fluxo-de-trabalho-e-commits-pequenos).


---


## 13. Revisão e armadilhas de verificação

<sub>`references/revisao-e-armadilhas.md`</sub>

Esta referência é o lado "verificar/revisar" do padrão — complementa o reviewer hexagonal do time
(rubrica canônica, mais detalhada). A ideia central: **o ArchUnit já falha o build para o
estrutural; o olho humano existe para o que o build NÃO enxerga.** Não gaste review no que o build
pega; gaste no que ele não pega.

### O que o build já garante (não revise à mão)

Camadas, sufixos, direção de dependência, `core` puro, sem `Date` legado / field injection /
`System.out` / exceção genérica — tudo isso o `app-commons-archunit` reprova sozinho. Só confirme que
o projeto aplica o **conjunto completo** das regras (não um subset diluído). Ver
[testes](#12-testes) e [arquitetura-hexagonal](#1-arquitetura-hexagonal-ports--adapters).

### O que só o humano pega (foque aqui)

- **Regressão de projeção de wire** — o campo que o app espera sumiu porque o response trocou de tipo
  (resumido↔completo). Compila, e "passa" se o snapshot for cúmplice. **P0.** Cruze sempre com o wire
  de produção/histórico git do response original, não com o snapshot da migração.
- **Boolean de `*Response` serializando `null`** — quebra o parser do app. **P1** (P0 se já quebrou).
  Exija `default: false` no schema **e** derivação non-null no mapper.
- **Erro de infra mascarado como "não encontrado"**, ou `BaseException` de negócio engolida — inverte
  o status HTTP. **P1.**
- **Regra de negócio em mapper**, ou lógica que devia estar numa `@Query` (filtro em Java). **P1.**
- **`@Transactional` no use case** (é da persistência). **P1.**
- **Validação no lugar errado**, PII em log/atributo/span, segredo com default hardcoded. **P0/P1.**
- **Naming por mecanismo obsoleto** (`*Repository*` para dado que não vem de repositório). **P2.**

### Armadilhas de "verde local" (o que fez a pipe quebrar depois)

1. **Suíte direcionada mascara falha.** `mvn -Dtest=Foo test` **compila** a árvore inteira (então
   "compila" não prova nada) mas **só roda os testes nomeados** — os `@DataJpaTest`/`@SpringBootTest`
   que você não nomeou (os que pegam quebra de schema/contexto) **não rodam**. Gate: **suíte
   COMPLETA** verde.
2. **`@DisabledWithoutDocker` é pulado local, roda na pipe.** ITs de cache/serialização (Redis via
   Testcontainers) não validam local sem Docker. "Pulado" ≠ "verde".
3. **Snapshot cúmplice.** Um baseline capturado *depois* da mudança congela o comportamento **novo** e
   passa a mascarar a regressão. Baseline de contrato só vale se veio do **wire anterior** (histórico
   git / o que o app de fato consome).
4. **Deletar `*Model` de tabela compartilhada é mudança de schema.** Com `ddl-auto=create` no teste,
   o schema pode passar a vir de outra entidade com menos colunas → seeds (`@Sql`/`INSERT`) e SQL
   nativo que citam a coluna sumida quebram **só** em `@DataJpaTest`/pipe. Antes de deletar: `grep`
   `FROM <tabela>` no SQL nativo e `INSERT INTO <tabela>` nos seeds.
5. **`@Builder.Default` + `@NoArgsConstructor`:** o default some quando o objeto é criado por `new`
   (campo vira `null` no insert). H2 mascara; Postgres quebra. Mais um motivo para
   [Testcontainers/Postgres, não H2](#12-testes).
6. **Gate de dado acoplado ao deploy:** código que só funciona **depois** de uma migração de dado
   (ex.: passou a ler uma coluna como JSON, mas as linhas ainda são String crua → tudo cai no
   default). Silencioso — "verde" porque o teste não exercita o path. Exija o script de dado
   versionado + a ordem de deploy documentada.

### Severidade (gate de merge)

- **P0 — bloqueia:** contrato quebrado (status/corpo/**projeção**); boolean nullable que já quebrou o
  app; violação de direção de camada; adapter/controller/model JPA fora de `infra`; `Request`/
  `Response` no domínio; Spring/`jakarta.validation` no `core`; exceção genérica; use case importando
  `model.*`; read-path que zera dado soberano por chave ausente.
- **P1 — bloqueia salvo acordo:** erro de negócio mascarado; regra em mapper; `@Transactional` no use
  case; `java.util.Date`; field injection; filtro em Java onde cabia `@Query`; PII em log/span;
  boolean nullable no wire; snapshot cúmplice; deletar `*Model` sem checar schema; gate de dado não
  documentado; command-timeout alto no read-path quente.
- **P2 — ajustar:** sufixo/pacote; naming por mecanismo; log ausente no catch; logger morto / dois
  padrões de logging; `@link`/comentário stale após rename; deprecated sem plano; código/cache morto.

### Toda observação vem com regra + severidade + correção

Não "acho que"; não elogio vazio. Cada apontamento cita a **regra**, a **severidade (P0/P1/P2)** e um
**patch ou caminho de correção**. E doa o que doer: bloqueie P0 mesmo sob prazo — contrato quebrado
com o app e vazamento de camada não são negociáveis.

> **Verde não é prova.** Um teste travado prova que o wire não *mudou desde o snapshot* — não que o
> snapshot está certo. Ceticismo é parte do método: confirme o baseline, rode a suíte completa,
> valide o que é `@DisabledWithoutDocker`.

Para o fluxo/rubrica completos de review de PR do time (heurísticas por stack Java/Flutter,
formato de saída), use o reviewer hexagonal dedicado — esta referência é o resumo operacional dele
dentro do padrão de construção.


---
