# Arquitetura-Clean-Code-base
Um projeto base deve fornecer uma estrutura sólida e reutilizável que facilite o desenvolvimento de sistemas futuros.

# Estrutura do Projeto e Camada de Domínio

## 📂 Estrutura completa de pastas

```text
MinhaSolucao.sln
├── src/
│   ├── 01.Web.MVC/ (Projeto de Interface - Referencia 02 e 03)
│   │   ├── Controllers/
│   │   ├── ViewModels/
│   │   ├── Views/
│   │   └── wwwroot/ (JS, CSS, Imagens)
│   │
│   ├── 02.Domain/ (Projeto Core - Não referencia ninguém)
│   │   ├── Entities/       (Ex: BaseEntity.cs, Produto.cs)
│   │   ├── ValueObjects/   (Ex: Email.cs, Cpf.cs)
│   │   ├── Exceptions/     (Ex: DomainException.cs)
│   │   ├── Services/       (Domain Services - Lógica complexa)
│   │   └── Interfaces/     (Contratos)
│   │       ├── Repositories/ (Ex: IProdutoRepository.cs)
│   │       └── Services/     (Ex: IEstoqueService.cs)
│   │
│   └── 03.Infrastructure/ (Projeto de Suporte - Referencia 02)
│       ├── Context/      (EF Core DbContext)
│       ├── Mappings/     (Configurações Fluent API / Mapas do Postgres)
│       ├── Repositories/ (Implementação concreta dos repositórios)
│       └── Migrations/   (Arquivos gerados pelo EF Core para o banco)
│
└── tests/ (Opcional, mas recomendado para o futuro)
    └── 02.Domain.Tests/

```

---

## 📑 02_Entidades_Negocio.md

### 1. Definição da Camada Core (Domain)

* **Objetivo:** Conter o estado e o comportamento central da aplicação, independente de qualquer tecnologia externa (UI, Banco ou APIs).
* **Princípio da Dependência:** Esta camada não deve referenciar nenhuma outra camada do sistema.

### 2. Componentes de Domínio

* **Entities (Entidades):** Classes que possuem uma identidade única (geralmente um ID). Exemplo: BaseEntity contendo Guid Id, DateTime DataCadastro.
* **Value Objects (Objetos de Valor):** Tipos que não possuem identidade e são definidos por seus atributos (ex: Endereco, Cpf, Email). Devem ser imutáveis.
* **Domain Exceptions:** Exceções personalizadas para regras de negócio (ex: `DominioException`), evitando que erros técnicos de infraestrutura vazem para o usuário.

### 3. Abstrações e Inversão de Dependência

* **Interfaces de Repositórios:** Definem os contratos de persistência. Padrão: `IRepository`.
* **Interfaces de Serviços de Domínio:** Usadas quando uma regra de negócio envolve múltiplas entidades ou não pertence naturalmente a uma única entidade.

### 4. Regras de Ouro (Boas Práticas)

* **Always-Valid Domain:** Uma entidade nunca deve ser instanciada em um estado inválido. Use o construtor para exigir dados obrigatórios.
* **Encapsulamento Estrito:** Use protected set ou private set em propriedades para impedir que camadas externas alterem o estado do objeto sem passar pelas regras de negócio.
* **Sem Anemic Domain Model:** Evite entidades que sejam apenas "sacolas de getters e setters". A lógica de validação de dados da entidade deve morar nela mesma.

### 5. Estrutura de Pastas de 02.Domain

```text
02.Domain/ (Projeto Core - Não referencia ninguém)
│   ├── Entities/       (Ex: BaseEntity.cs, Produto.cs)
│   ├── ValueObjects/   (Ex: Email.cs, Cpf.cs)
│   ├── Exceptions/     (Ex: DomainException.cs)
│   ├── Services/       (Domain Services - Lógica complexa)
│   └── Interfaces/     (Contratos)
│       ├── Repositories/ (Ex: IProdutoRepository.cs)
│       └── Services/     (Ex: IEstoqueService.cs)

```

---

## 💻 Exemplos dos Códigos

### 1. Entities/ (Entidade Base e uma Entidade de Negócio)

**Local: `02.Domain/Entities/BaseEntity.cs**`

```csharp
public abstract class BaseEntity
{
    // Usamos Guid para facilitar a integração entre sistemas e segurança
    public Guid Id { get; protected set; } 
    public DateTime DataCriacao { get; private set; }
    public bool Ativo { get; private set; }

    protected BaseEntity()
    {
        Id = Guid.NewGuid();
        DataCriacao = DateTime.UtcNow;
        Ativo = true;
    }

    public void Desativar() => Ativo = false;
}

```

*Nota: Serve para não repetir propriedades globais; a entidade de negócio herda de BaseEntity.*

**Local: `02.Domain/Entities/Produto.cs**`

```csharp
public class Produto : BaseEntity // Herda tudo da base
{
    public string Nome { get; private set; }
    public decimal Preco { get; private set; }

    // O construtor recebe o que é específico do Produto
    public Produto(string nome, decimal preco) : base() // Colocando o base eu informo explicitamente que quero que BaseEntity seja executada antes de Produto, - 
    {                                                   // mas se nã informar, o C# faz isso por baixo dos panos.
        if (string.IsNullOrWhiteSpace(nome)) throw new DomainException("Nome é obrigatório.");
        if (preco <= 0) throw new DomainException("Preço deve ser maior que zero."); // Guards (Guardas) ou Validações de Domínio

        Nome = nome;
        Preco = preco;
    }

    public void AtualizarPreco(decimal novoPreco)
    {
        if (novoPreco <= 0) throw new DomainException("Novo preço inválido.");
        Preco = novoPreco;
    }
}

```

### 2. ValueObjects/ (Objetos de Valor)

**Local: `02.Domain/ValueObjects/Email.cs**`

* **Imutabilidade:** Uma vez criado, o e-mail não muda. Se precisar mudar o e-mail, você cria um novo objeto.
* **Comparação por Valor:** Se você tiver dois objetos Email diferentes na memória, mas ambos com o endereço "teste@gmail.com", o C# dirá que eles são iguais (email1 == email2 será true). Com classes comuns, isso seria false.
* **Sintaxe Enxuta:** Menos código para escrever e manter.

```csharp
public record Email
{
  public string Endereco { get; }

  public Email(string endereco)
  {
    if (!endereco.Contains("@")) 
      throw new DomainException("E-mail inválido.");
      
    Endereco = endereco;
  }
}

```

**Uso do Value Object na Entidade:**

```csharp
public class Usuario : BaseEntity
{
  public string Nome { get; private set; }
  public Email Email { get; private set; }

  public Usuario(string nome, Email email) : base()
  {
    Nome = nome;
    Email = email;
  }
}

```

### 3. Interfaces/Repositories/ (Contratos de Dados)

**Local: `02.Domain/Interfaces/Repositories/IProdutoRepository.cs**`

```csharp
public interface IProdutoRepository
{
  Task<Produto> ObterPorIdAsync(Guid id);
  Task AdicionarAsync(Produto produto);
  Task<IEnumerable<Produto>> ListarTodosAsync();
}

```

### 4. Exceptions/ (Erros de Negócio)

**Local: `02.Domain/Exceptions/DomainException.cs**`

```csharp
public class DomainException : Exception
{
  public DomainException(string message) : base(message) { }
}

```

### 5. Services/ (Domain Services)

**Local: `02.Domain/Services/EstoqueDomainService.cs**`

```csharp
public class EstoqueDomainService : IEstoqueDomainService
{
  private readonly IProdutoRepository _produtoRepository;
  
  public EstoqueDomainService(IProdutoRepository produtoRepository)
  {
    _produtoRepository = produtoRepository;
  }

  public async Task TransferirEstoque(Guid origemId, Guid destinoId, int quantidade)
  {
    // Lógica complexa que orquestra múltiplas entidades ou validações
  }
}

```

### 1. `Entities/` (Entidade Base e uma Entidade de Negócio)



A entidade deve ser protegida contra estados inválidos.



```csharp

// BaseEntity.cs

public abstract class BaseEntity

{

  public Guid Id { get; protected set; }

  public DateTime DataCadastro { get; private set; }



  protected BaseEntity()

  {

    Id = Guid.NewGuid();

    DataCadastro = DateTime.UtcNow;

  }

}



// Produto.cs

public class Produto : BaseEntity

{

  public string Nome { get; private set; }

  public decimal Preco { get; private set; }



  public Produto(string nome, decimal preco)

  {

    if (string.IsNullOrWhiteSpace(nome)) throw new DomainException("Nome é obrigatório.");

    if (preco <= 0) throw new DomainException("Preço deve ser maior que zero.");



    Nome = nome;

    Preco = preco;

  }



  public void AtualizarPreco(decimal novoPreco)

  {

    if (novoPreco <= 0) throw new DomainException("Novo preço inválido.");

    Preco = novoPreco;

  }

}



```



### 2. `ValueObjects/` (Objetos de Valor)



Perfeito para tipos que não precisam de um ID próprio, apenas do valor.



```csharp

// Email.cs

public record Email

{

  public string Endereco { get; }



  public Email(string endereco)

  {

    if (!endereco.Contains("@")) throw new DomainException("E-mail inválido.");

    Endereco = endereco;

  }

}



```



### 3. `Interfaces/Repositories/` (Contratos de Dados)



O domínio define **o que** precisa, a infraestrutura dirá **como** fazer.



```csharp

// IProdutoRepository.cs

public interface IProdutoRepository

{

  Task<Produto> ObterPorIdAsync(Guid id);

  Task AdicionarAsync(Produto produto);

  Task<IEnumerable<Produto>> ObterTodosAsync();

}



```



### 4. `Exceptions/` (Erros de Negócio)



Ajuda a separar erros de sistema de erros de regra de negócio.



```csharp

// DomainException.cs

public class DomainException : Exception

{

  public DomainException(string message) : base(message) { }

}



```



### 5. `Services/` (Domain Services)



Usado quando uma lógica envolve mais de uma entidade e não cabe em uma só.



```csharp

// EstoqueDomainService.cs

public class EstoqueDomainService : IEstoqueDomainService

{

  private readonly IProdutoRepository _produtoRepository;



  public EstoqueDomainService(IProdutoRepository produtoRepository)

  {

    _produtoRepository = produtoRepository;

  }



  public async Task TransferirEstoque(Guid origemId, Guid destinoId, int quantidade)

  {

    // Lógica complexa que orquestra múltiplas entidades ou validações

  }

}
