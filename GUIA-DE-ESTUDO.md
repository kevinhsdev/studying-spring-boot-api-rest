# 📘 Guia de Estudo — API de Controle Financeiro com Spring Boot

Diário de aprendizado do projeto. Cada etapa explica **o que** foi criado, **para que serve** e **o que faz cada linha**.

## Índice

- [Visão geral do projeto](#visão-geral-do-projeto)
- [Etapa 1 — Esqueleto do projeto](#etapa-1--esqueleto-do-projeto) ✅
- [Etapa 2 — Guardar transações no banco (Entidade + Repository)](#etapa-2--guardar-transações-no-banco-entidade--repository) ⏳
- [Etapa 3 — Service + Controller (CRUD)](#etapa-3--service--controller-crud) ⬜
- Etapa 4 — DTOs, validação e tratamento de erros
- Etapa 5 — Resumo mensal (saldo e gastos por categoria)
- Etapa 6 — Swagger + testes automatizados
- Etapa 7 — README + GitHub
- [Glossário](#glossário)

---

## Visão geral do projeto

Uma **API REST** de finanças pessoais: cadastrar receitas e despesas, organizá-las por categoria e ver um resumo do mês.

**Stack:** Java 25 · Spring Boot 4.1.1 · Spring Data JPA (Hibernate) · PostgreSQL 17 (via Docker) · Maven

**Como rodar:** abrir o Docker Desktop → no IntelliJ, abrir `FinancasApplication.java` → clicar no ▶ verde.

---

## Etapa 1 — Esqueleto do projeto

Projeto gerado no [Spring Initializr](https://start.spring.io) (o "gerador oficial" de projetos Spring).

### Estrutura

```
Spring-boot/
├── pom.xml              ← "receita" do projeto: versão do Java, do Spring Boot e as dependências
├── mvnw / mvnw.cmd      ← Maven Wrapper: baixa o Maven sozinho, não precisa instalar nada
├── compose.yaml         ← descreve o container do PostgreSQL (banco/usuário/senha: financas)
└── src/
    ├── main/java/com/kevin/financas/FinancasApplication.java  ← ponto de entrada (@SpringBootApplication)
    ├── main/resources/application.properties                  ← configurações
    └── test/...                                                ← testes
```

### Dependências (no `pom.xml`)

| Dependência | Para que serve |
|---|---|
| **webmvc** | Criar endpoints REST (`@RestController`, `@GetMapping`...) |
| **data-jpa** | Transformar classes Java em tabelas e gerar o SQL por você |
| **postgresql** | Driver para o Java conseguir conectar no Postgres |
| **validation** | Validar a entrada (`@NotBlank`, `@Positive`...) |
| **docker-compose** | Ao rodar a aplicação, o Spring sobe o Postgres do `compose.yaml` **sozinho** e já configura a conexão |

### Docker

O **Docker** roda o PostgreSQL dentro de um "container" — uma mini-máquina isolada — para não precisar instalar o Postgres no Windows. O `compose.yaml` diz *qual* banco subir, e o Spring liga esse container automaticamente quando a aplicação inicia.

---

## Etapa 2 — Guardar transações no banco (Entidade + Repository)

### O problema

Nossa API vai guardar transações ("Mercado, R$ 250,90, despesa"). Elas precisam ficar num **banco de dados**, senão somem quando a aplicação desliga.

Só que Java e banco falam línguas diferentes:

| Java | Banco de dados |
|---|---|
| classes e objetos | tabelas e linhas |
| `transacao.getValor()` | `SELECT valor FROM transacao` |

Escrever SQL à mão para tudo é trabalhoso. Por isso existe o **JPA/Hibernate**, um **ORM** (*Object-Relational Mapping*). Você escreve uma classe Java comum com algumas anotações, e ele:

- **cria a tabela** a partir da classe;
- **converte** objeto ↔ linha da tabela;
- **gera o SQL** por você.

### O que criar

```
src/main/java/com/kevin/financas/
├── FinancasApplication.java      (já existe, não mexa)
└── transacao/                    ← nova pasta (pacote)
    ├── TipoTransacao.java        ← os tipos possíveis: RECEITA ou DESPESA
    ├── Transacao.java            ← a classe que vira tabela
    └── TransacaoRepository.java  ← quem salva/busca no banco
```

- **Criar o pacote no IntelliJ:** botão direito em `com.kevin.financas` → **New → Package** → `transacao`.
- **Criar cada arquivo:** botão direito no pacote `transacao` → **New → Java Class** → escolha o tipo (Class, Enum ou Interface) e digite o nome.

---

### 1️⃣ `TipoTransacao.java` (Enum)

**Para que serve:** uma transação só pode ser de dois tipos. Um **enum** é uma lista fechada de valores — ninguém consegue criar um tipo "DESPEZA" escrito errado, o compilador não deixa.

```java
package com.kevin.financas.transacao;

public enum TipoTransacao {
    RECEITA,
    DESPESA
}
```

| Linha | Explicação |
|---|---|
| `package com.kevin.financas.transacao;` | Diz em qual pacote (pasta) o arquivo mora. O IntelliJ escreve sozinho. |
| `public enum TipoTransacao {` | Declara um enum. `public` = outras classes podem usá-lo. |
| `RECEITA,` | Valor possível: dinheiro que **entra**. |
| `DESPESA` | Valor possível: dinheiro que **sai**. |
| `}` | Fecha o enum. |

Para usar em outro lugar: `TipoTransacao.DESPESA`.

---

### 2️⃣ `Transacao.java` (Class) — a entidade

**Para que serve:** é o "molde" de uma transação. Cada **campo** da classe vira uma **coluna** da tabela; cada **objeto** criado vira uma **linha**.

```java
package com.kevin.financas.transacao;

import jakarta.persistence.*;

import java.math.BigDecimal;
import java.time.LocalDate;

@Entity
public class Transacao {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String descricao;

    @Column(nullable = false, precision = 12, scale = 2)
    private BigDecimal valor;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private TipoTransacao tipo;

    @Column(nullable = false)
    private String categoria;

    @Column(nullable = false)
    private LocalDate data;

    protected Transacao() {
    }

    public Transacao(String descricao, BigDecimal valor, TipoTransacao tipo, String categoria, LocalDate data) {
        this.descricao = descricao;
        this.valor = valor;
        this.tipo = tipo;
        this.categoria = categoria;
        this.data = data;
    }

    public Long getId() { return id; }
    public String getDescricao() { return descricao; }
    public BigDecimal getValor() { return valor; }
    public TipoTransacao getTipo() { return tipo; }
    public String getCategoria() { return categoria; }
    public LocalDate getData() { return data; }
}
```

#### Imports — trazer ferramentas de outros lugares

| Linha | Explicação |
|---|---|
| `import jakarta.persistence.*;` | Traz **todas** as anotações do JPA (`@Entity`, `@Id`, `@Column`...). O `*` = "tudo deste pacote". |
| `import java.math.BigDecimal;` | Tipo numérico **exato**, para dinheiro. |
| `import java.time.LocalDate;` | Tipo para datas sem horário (`2026-10-10`). |

#### A classe

| Linha | Explicação |
|---|---|
| `@Entity` | **A anotação mais importante.** Diz ao Hibernate: "esta classe é uma tabela". O nome da tabela será `transacao`. |
| `public class Transacao {` | Declara a classe. |

> 💡 **O que é uma anotação (`@Algo`)?** Uma "etiqueta" colada no código. Sozinha não faz nada — mas o Spring/Hibernate leem essas etiquetas e agem de acordo.

#### Os campos (viram colunas)

| Linha | Explicação |
|---|---|
| `@Id` | Esta coluna é a **chave primária** — o "RG" único de cada linha. |
| `@GeneratedValue(strategy = GenerationType.IDENTITY)` | Você não escolhe o id: o **banco gera** 1, 2, 3... sozinho (auto-incremento). |
| `private Long id;` | O campo. `private` = só a própria classe acessa diretamente. `Long` (L maiúsculo) e não `long` porque pode ser `null` — e é, antes de salvar (ainda não tem id). |
| `@Column(nullable = false)` | `nullable = false` = a coluna **não pode ficar vazia** (`NOT NULL` no SQL). |
| `private String descricao;` | Texto, ex.: `"Mercado"`. |
| `@Column(nullable = false, precision = 12, scale = 2)` | `precision = 12` → até 12 dígitos no total; `scale = 2` → 2 depois da vírgula. Máximo: 9.999.999.999,99. |
| `private BigDecimal valor;` | **Por que não `double`?** Porque `double` erra contas com decimais: `0.1 + 0.2` dá `0.30000000000000004`. Com dinheiro isso é inaceitável. `BigDecimal` é exato. **(Pergunta clássica de entrevista!)** |
| `@Enumerated(EnumType.STRING)` | Grava o enum como **texto** (`"DESPESA"`). Sem isso, o Java grava a posição (0 ou 1) — e se um dia mudar a ordem do enum, todos os dados antigos ficam errados. |
| `private TipoTransacao tipo;` | Usa o enum criado antes. |
| `private String categoria;` | Texto, ex.: `"Alimentação"`, `"Transporte"`. |
| `private LocalDate data;` | Quando a transação aconteceu. |

#### Os construtores (como criar um objeto)

| Linha | Explicação |
|---|---|
| `protected Transacao() { }` | Construtor **vazio**. O Hibernate **exige**: ao ler uma linha do banco, ele cria um objeto vazio e depois preenche os campos. É `protected` (e não `public`) para **você** não usá-lo sem querer e criar uma transação sem dados. |
| `public Transacao(String descricao, ...)` | O construtor que **você** usa: recebe todos os dados de uma vez. Não tem `id` — quem gera é o banco. |
| `this.descricao = descricao;` | `this.descricao` = o **campo** da classe; `descricao` = o **parâmetro** que chegou. Copia o valor do parâmetro para o campo. O `this` diferencia os dois, que têm o mesmo nome. |

#### Os getters (como ler os dados)

| Linha | Explicação |
|---|---|
| `public Long getId() { return id; }` | Como os campos são `private`, outras classes leem por estes métodos `get...`. O Spring também usa os getters para transformar o objeto em JSON. |

**Não há setters** (`setValor`...) de propósito: ninguém altera a transação de qualquer jeito. Para editar, criamos um método específico na Etapa 3.

---

### 3️⃣ `TransacaoRepository.java` (Interface)

**Para que serve:** é quem **conversa com o banco** — salvar, buscar, listar, apagar.

```java
package com.kevin.financas.transacao;

import org.springframework.data.jpa.repository.JpaRepository;

public interface TransacaoRepository extends JpaRepository<Transacao, Long> {
}
```

| Linha | Explicação |
|---|---|
| `import ...JpaRepository;` | Traz a interface pronta do **Spring Data JPA**. |
| `public interface TransacaoRepository` | Uma **interface** é uma lista de métodos *sem implementação* — um "contrato". |
| `extends JpaRepository<Transacao, Long>` | "Herda" o contrato do `JpaRepository`. Entre `< >`: esta interface cuida da entidade **`Transacao`**, cujo id é do tipo **`Long`**. |
| `{ }` | Vazio mesmo! |

**A mágica do Spring Data:** você não escreve código nenhum. Quando a aplicação sobe, o Spring **cria uma classe por trás** que implementa tudo. Você ganha de graça:

| Método | SQL gerado |
|---|---|
| `save(transacao)` | `INSERT` (ou `UPDATE` se já existir) |
| `findAll()` | `SELECT * FROM transacao` |
| `findById(5L)` | `SELECT ... WHERE id = 5` |
| `deleteById(5L)` | `DELETE ... WHERE id = 5` |
| `count()` | `SELECT COUNT(*) ...` |

---

### 4️⃣ `application.properties` (em `src/main/resources`)

**Para que serve:** configurações da aplicação.

```properties
spring.application.name=financas

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true
```

| Linha | Explicação |
|---|---|
| `spring.application.name=financas` | Nome da aplicação (já existia). |
| `spring.jpa.hibernate.ddl-auto=update` | "Hibernate, olhe minhas `@Entity` e **crie/atualize as tabelas**." Sem isso a tabela não existe e dá erro. *(Em empresas usa-se Flyway no lugar.)* |
| `spring.jpa.show-sql=true` | Mostra no console **todo SQL** que o Hibernate gera. Ótimo para aprender. |
| `spring.jpa.properties.hibernate.format_sql=true` | Formata esse SQL em várias linhas, mais legível. |

Endereço, usuário e senha do banco não precisam ser configurados — o Spring lê do `compose.yaml` automaticamente.

---

### ✅ Como saber se deu certo

1. **Docker Desktop aberto.**
2. IntelliJ → `FinancasApplication.java` → **▶ verde**.
3. A primeira vez demora ~1 min (o Docker baixa o Postgres). No console, procure:

```sql
create table transacao (
    id bigint generated by default as identity,
    categoria varchar(255) not null,
    data date not null,
    descricao varchar(255) not null,
    tipo varchar(255) not null check ((tipo in ('RECEITA','DESPESA'))),
    valor numeric(12,2) not null,
    primary key (id)
)
```

Esse é o SQL que o Hibernate gerou **a partir da sua classe** — compare cada coluna com cada campo. O `check (tipo in ('RECEITA','DESPESA'))` veio do enum.

4. No final: `Started FinancasApplication`. Acessar `localhost:8080` ainda dá 404 — normal, ainda não há endpoints.

---

## Etapa 3 — Service + Controller (CRUD)

### A ideia geral

Uma requisição passa por três camadas, cada uma com uma função:

```
Requisição HTTP  →  Controller  →  Service  →  Repository  →  Postgres
   (JSON)           "porteiro"     "cérebro"     "braço"
```

- **Controller**: recebe o HTTP, chama o service e devolve a resposta. Não tem regra de negócio.
- **Service**: as regras ("se não existir, dá erro 404").
- **Repository**: só conversa com o banco (Etapa 2).

Separar assim é o que o recrutador quer ver: cada classe tem **uma** responsabilidade.

**CRUD** = **C**reate, **R**ead, **U**pdate, **D**elete — as 4 operações básicas.

Você vai criar **3 arquivos** e editar **1**, todos em `com.kevin.financas.transacao`.

---

### 1️⃣ `TransacaoRequest.java` — o "formulário" que chega

**Para que serve:** representa o JSON que o cliente envia ao criar/editar. Não recebemos a entidade `Transacao` direto porque o cliente não deve escolher o `id`, por exemplo.

```java
package com.kevin.financas.transacao;

import java.math.BigDecimal;
import java.time.LocalDate;

public record TransacaoRequest(
        String descricao,
        BigDecimal valor,
        TipoTransacao tipo,
        String categoria,
        LocalDate data
) {
}
```

| Linha | Explicação |
|---|---|
| `package ...transacao;` | Em qual pacote a classe mora. |
| `import java.math.BigDecimal;` | Tipo para dinheiro exato. |
| `import java.time.LocalDate;` | Tipo de data sem horário. |
| `public record TransacaoRequest(...)` | **Record** é uma classe só de dados: o Java gera sozinho construtor, getters (`descricao()`, `valor()`...), `equals` e `toString`. Os campos são imutáveis. Perfeito para "pacotes de dados" que entram/saem da API. |
| `String descricao, ...` | Cada item vira um campo. O nome deve ser **igual** à chave do JSON (`"descricao": "..."`). |

> 💡 O **Jackson** (biblioteca que o Spring usa para JSON ↔ Java) lê o JSON e chama o construtor do record com os valores.

---

### 2️⃣ Editar `Transacao.java` — permitir alteração

**Para que serve:** o PUT (editar) precisa mudar os dados de uma transação existente. Em vez de 5 setters, criamos **um** método com nome de negócio.

Cole dentro da classe, antes dos getters:

```java
    public void atualizar(String descricao, BigDecimal valor, TipoTransacao tipo, String categoria, LocalDate data) {
        this.descricao = descricao;
        this.valor = valor;
        this.tipo = tipo;
        this.categoria = categoria;
        this.data = data;
    }
```

| Linha | Explicação |
|---|---|
| `public void atualizar(...)` | Método público, não retorna nada (`void`), recebe os novos valores. |
| `this.descricao = descricao;` | Copia o parâmetro para o campo do objeto. |

> 💡 **Por que não setters?** Com setters, qualquer parte do código muda só o `valor` e deixa a transação incoerente. Um método `atualizar` deixa claro **como** a entidade pode mudar. Isso é *encapsulamento*.

---

### 3️⃣ `TransacaoService.java` — a regra de negócio

```java
package com.kevin.financas.transacao;

import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.server.ResponseStatusException;

import java.util.List;

@Service
public class TransacaoService {

    private final TransacaoRepository repository;

    public TransacaoService(TransacaoRepository repository) {
        this.repository = repository;
    }

    public List<Transacao> listar() {
        return repository.findAll();
    }

    public Transacao buscar(Long id) {
        return repository.findById(id)
                .orElseThrow(() -> new ResponseStatusException(HttpStatus.NOT_FOUND, "Transação não encontrada"));
    }

    public Transacao criar(TransacaoRequest request) {
        Transacao transacao = new Transacao(
                request.descricao(),
                request.valor(),
                request.tipo(),
                request.categoria(),
                request.data()
        );
        return repository.save(transacao);
    }

    @Transactional
    public Transacao atualizar(Long id, TransacaoRequest request) {
        Transacao transacao = buscar(id);
        transacao.atualizar(
                request.descricao(),
                request.valor(),
                request.tipo(),
                request.categoria(),
                request.data()
        );
        return transacao;
    }

    public void deletar(Long id) {
        Transacao transacao = buscar(id);
        repository.delete(transacao);
    }
}
```

#### Anotação e campos

| Linha | Explicação |
|---|---|
| `@Service` | Avisa ao Spring: "crie **um** objeto desta classe e gerencie ele" (um **bean**). Você nunca dá `new TransacaoService()` — o Spring faz isso. |
| `private final TransacaoRepository repository;` | O service **precisa** do repository para acessar o banco. `final` = depois de definido, não muda. |
| `public TransacaoService(TransacaoRepository repository)` | **Injeção de dependência pelo construtor.** O Spring vê que o construtor pede um `TransacaoRepository`, pega o bean que ele mesmo criou e passa aqui. É *o* conceito central do Spring — cai em entrevista. |
| `this.repository = repository;` | Guarda o repository recebido no campo. |

#### Métodos

| Linha | Explicação |
|---|---|
| `public List<Transacao> listar()` | Retorna uma lista de transações. |
| `return repository.findAll();` | Método pronto → `SELECT * FROM transacao`. |
| `public Transacao buscar(Long id)` | Busca uma pelo id. |
| `repository.findById(id)` | Retorna um **`Optional<Transacao>`** — uma "caixa" que pode estar vazia (id não existe). Força você a tratar o "não achei". |
| `.orElseThrow(() -> ...)` | "Se a caixa estiver vazia, lance esta exceção." O `() -> ...` é uma **lambda**: uma função curta que só roda se precisar. |
| `new ResponseStatusException(HttpStatus.NOT_FOUND, "...")` | Exceção especial do Spring: quando ela "escapa" até o controller, o Spring responde **HTTP 404**. |
| `public Transacao criar(TransacaoRequest request)` | Recebe o "formulário" do cliente. |
| `new Transacao(request.descricao(), ...)` | Monta a entidade. `request.descricao()` é o getter que o record gerou (sem "get"). |
| `return repository.save(transacao);` | `INSERT` no banco; devolve a transação **já com o `id`** gerado. |
| `@Transactional` | Tudo no método roda numa **transação de banco**: ou tudo dá certo, ou nada é salvo. |
| `Transacao transacao = buscar(id);` | Reaproveita o `buscar` — se não existir, já dá 404. |
| `transacao.atualizar(...)` | Muda os campos do objeto Java. |
| `return transacao;` | **Repare: não tem `save()`!** Dentro de um `@Transactional`, o Hibernate vigia os objetos que buscou; ao fim do método percebe que mudaram e faz o `UPDATE` sozinho. Isso se chama **dirty checking** — ótimo para entrevista. |
| `public void deletar(Long id)` | Busca (404 se não existir) e `repository.delete(...)` faz o `DELETE`. |

> ⚠️ Usar `ResponseStatusException` no service é um atalho (mistura HTTP com regra de negócio). Na Etapa 4 trocamos por uma exceção própria + tratador global de erros.

---

### 4️⃣ `TransacaoController.java` — a porta de entrada HTTP

```java
package com.kevin.financas.transacao;

import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/transacoes")
public class TransacaoController {

    private final TransacaoService service;

    public TransacaoController(TransacaoService service) {
        this.service = service;
    }

    @GetMapping
    public List<Transacao> listar() {
        return service.listar();
    }

    @GetMapping("/{id}")
    public Transacao buscar(@PathVariable Long id) {
        return service.buscar(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Transacao criar(@RequestBody TransacaoRequest request) {
        return service.criar(request);
    }

    @PutMapping("/{id}")
    public Transacao atualizar(@PathVariable Long id, @RequestBody TransacaoRequest request) {
        return service.atualizar(id, request);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deletar(@PathVariable Long id) {
        service.deletar(id);
    }
}
```

| Linha | Explicação |
|---|---|
| `import ...annotation.*;` | Importa todas as anotações web (`@GetMapping`, `@PostMapping`...). |
| `@RestController` | "Esta classe atende requisições HTTP, e o retorno dos métodos vira **JSON** automaticamente." |
| `@RequestMapping("/transacoes")` | Prefixo de URL: tudo aqui fica em `localhost:8080/transacoes...`. |
| `private final TransacaoService service;` + construtor | Mesma injeção de dependência: o controller pede o service. |
| `@GetMapping` | Responde a **GET /transacoes** → lista todas. |
| `@GetMapping("/{id}")` | Responde a **GET /transacoes/5**. `{id}` é um pedaço variável da URL. |
| `@PathVariable Long id` | Pega o `5` da URL e coloca no parâmetro `id` (convertendo para `Long`). |
| `@PostMapping` | Responde a **POST /transacoes** → cria. |
| `@ResponseStatus(HttpStatus.CREATED)` | Responde **201 Created** em vez do 200 padrão — o código certo para "criei algo". |
| `@RequestBody TransacaoRequest request` | Pega o JSON do **corpo** da requisição e converte em `TransacaoRequest`. |
| `@PutMapping("/{id}")` | Responde a **PUT /transacoes/5** → substitui os dados da 5. |
| `@DeleteMapping("/{id}")` | Responde a **DELETE /transacoes/5**. |
| `@ResponseStatus(HttpStatus.NO_CONTENT)` | Responde **204**: "deu certo, não tenho nada para devolver". |

O controller é "burro" de propósito: cada método só repassa para o service.

---

### 5️⃣ Testar

Rode a aplicação (▶) e use o **PowerShell** (o `curl` do PowerShell 5 não é o curl de verdade):

```powershell
# Criar (valor com ponto, data no formato ano-mês-dia)
Invoke-RestMethod -Method Post -Uri http://localhost:8080/transacoes -ContentType 'application/json' -Body '{"descricao":"Salario","valor":3000.00,"tipo":"RECEITA","categoria":"Trabalho","data":"2026-10-05"}'

Invoke-RestMethod -Method Post -Uri http://localhost:8080/transacoes -ContentType 'application/json' -Body '{"descricao":"Mercado","valor":250.90,"tipo":"DESPESA","categoria":"Alimentacao","data":"2026-10-08"}'

# Listar todas
Invoke-RestMethod http://localhost:8080/transacoes

# Buscar uma
Invoke-RestMethod http://localhost:8080/transacoes/1

# Editar
Invoke-RestMethod -Method Put -Uri http://localhost:8080/transacoes/2 -ContentType 'application/json' -Body '{"descricao":"Mercado do mes","valor":310.00,"tipo":"DESPESA","categoria":"Alimentacao","data":"2026-10-08"}'

# Apagar
Invoke-RestMethod -Method Delete -Uri http://localhost:8080/transacoes/2

# Id que não existe → deve dar erro 404
Invoke-RestMethod http://localhost:8080/transacoes/999
```

O `GET` também funciona direto no navegador: `http://localhost:8080/transacoes`.

👀 Observe o console do IntelliJ: cada `insert`, `select`, `update` e `delete` aparece. No PUT, confira que surge um `update` sem você ter chamado `save()`.

---

### 🧪 Desafio (opcional)

Fazer `GET /transacoes?tipo=DESPESA` filtrar por tipo:

1. No repository: `List<Transacao> findByTipo(TipoTransacao tipo);` — o Spring Data **entende o nome do método** e gera o SQL.
2. No controller: parâmetro `@RequestParam(required = false) TipoTransacao tipo`.
3. No service: se `tipo` for `null` → `findAll()`; senão → `findByTipo(tipo)`.

---

## Glossário

| Termo | Significado |
|---|---|
| **API REST** | Um "cardápio" de URLs que outros programas chamam via HTTP para ler/gravar dados, trocando JSON. |
| **JSON** | Formato de texto para dados: `{"descricao": "Mercado", "valor": 250.90}`. |
| **Spring Boot** | Framework que configura quase tudo sozinho para criar aplicações Java rapidamente. |
| **Maven** | Ferramenta que baixa dependências e compila o projeto (lê o `pom.xml`). |
| **Docker / container** | Roda programas (como o Postgres) isolados, sem instalar no Windows. |
| **ORM** | Ponte entre objetos Java e tabelas do banco. |
| **JPA / Hibernate** | JPA é a especificação (as regras); Hibernate é a implementação usada. |
| **Entidade (`@Entity`)** | Classe que vira tabela. |
| **Repository** | Camada que acessa o banco. |
| **Service** | Camada com as regras de negócio. |
| **Controller** | Camada que recebe as requisições HTTP. |
| **Bean** | Objeto criado e gerenciado pelo Spring. |
| **Injeção de dependência** | O Spring entrega os objetos que uma classe precisa (pelo construtor), em vez de ela dar `new`. |
| **Anotação (`@Algo`)** | Etiqueta no código que o Spring/Hibernate leem para agir. |
| **Enum** | Lista fechada de valores possíveis. |
| **Record** | Classe só de dados, imutável, com construtor/getters gerados. |
| **Optional** | "Caixa" que pode ter um valor ou estar vazia. |
| **Lambda (`() -> ...`)** | Função curta passada como valor. |
| **Transação (`@Transactional`)** | Bloco de operações no banco que acontece por completo ou não acontece. |
| **Dirty checking** | O Hibernate detecta objetos alterados e faz o `UPDATE` sozinho. |
| **CRUD** | Create, Read, Update, Delete. |
| **Códigos HTTP** | 200 OK · 201 Created · 204 No Content · 400 Bad Request · 404 Not Found · 500 erro no servidor. |
