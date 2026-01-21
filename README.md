01-Estrutura-de-Pastas

## 📂 Estrutura de Pastas da Solução (Referência)

```text
MinhaSolucao.sln
├── src/
│   ├── 01.Web.MVC/ (Referência: 02.Domain e 03.Infrastructure)
│   │   ├── Controllers/
│   │   │   └── ProdutoController.cs  <-- CHAMA IProdutoService
│   │   ├── ViewModels/               --> ProdutoViewModel.cs
│   │   ├── Views/                    --> Produtos/Index.cshtml, Create.cshtml
│   │   ├── wwwroot/                  --> JS, CSS
│   │   └── Program.cs                <-- ONDE TUDO COMEÇA (Configuração de DI)
│   │
│   ├── 02.Domain/ (O Cérebro - Não referencia ninguém)
│   │   ├── Entities/       --> Produto.cs
│   │   ├── ValueObjects/   --> Preco.cs, Nome.cs
│   │   ├── Exceptions/     --> DomainException.cs
│   │   ├── Interfaces/ (OS CONTRATOS)
│   │   │   ├── Repositories/ --> IProdutoRepository.cs
│   │   │   └── Services/     --> IProdutoService.cs
│   │   └── Services/ (A LÓGICA COMPLEXA)
│   │       └── ProdutoService.cs  --> Implementa IProdutoService e usa IProdutoRepository
│   │
│   └── 03.Infrastructure/ (Referência: 02.Domain)
│       ├── Context/      --> MeuDbContext.cs
│       ├── Mappings/     --> ProdutoMap.cs
│       ├── Repositories/ --> ProdutoRepository.cs
│       └── Migrations/
│
└── tests/
    └── 02.Domain.Tests/
```

02-00-Domain

## Estrutura Detalhada de 02.Domain
```text
│   ├── 02.Domain/ (O Cérebro - Não referencia ninguém)
│   │   ├── Entities/       --> Produto.cs
│   │   ├── ValueObjects/   --> Preco.cs, Nome.cs
│   │   ├── Exceptions/     --> DomainException.cs
│   │   ├── Interfaces/ (OS CONTRATOS)
│   │   │   ├── Repositories/ --> IProdutoRepository.cs
│   │   │   └── Services/     --> IProdutoService.cs (O que a Controller enxerga)
│   │   └── Services/ (A LÓGICA COMPLEXA)
│   │       └── ProdutoService.cs  --> Implementa IProdutoService e usa IProdutoRepository
```

# Estudo de Clean Architecture - Camada de Domínio

Este material contém a organização de pastas e os conceitos fundamentais da camada **02.Domain**, focado em C# e MVC, alinhado com a estrutura do projeto `MinhaSolucao.sln`.

## 1. Definição da Camada Core (Domain)
* **Objetivo:** Conter o estado e o comportamento central da aplicação, independente de qualquer tecnologia externa (UI, Banco ou APIs).
* **Princípio da Dependência:** Esta camada não deve referenciar nenhuma outra camada do sistema.

## 2. Componentes de Domínio
* **Entities (Entidades):** Classes que possuem uma identidade única (geralmente um ID).
    * *Exemplo:* `BaseEntity` contendo `Guid Id`, `DateTime DataCadastro`.
* **Value Objects (Objetos de Valor):** Tipos que não possuem identidade e são definidos por seus atributos (ex: `Endereco`, `Cpf`, `Email`). Devem ser imutáveis.
* **Domain Exceptions:** Exceções personalizadas para regras de negócio (ex: `DomainException`), evitando que erros técnicos de infraestrutura vazem para o usuário.

## 3. Abstrações e Inversão de Dependência
* **Interfaces de Repositórios:** Definem os contratos de persistência. Padrão: `IRepository`. Focam em dados (Salvar, Ler).
* **Interfaces de Serviços de Domínio:** Usadas quando uma regra de negócio envolve múltiplas entidades ou processos complexos. Focam em ações (Cadastrar, Processar).

## 4. Regras de Ouro (Boas Práticas)
* **Always-Valid Domain:** Uma entidade nunca deve ser instanciada em um estado inválido. Use o construtor para exigir dados obrigatórios.
* **Encapsulamento Estrito:** Use `protected set` ou `private set` em propriedades para impedir que camadas externas alterem o estado do objeto sem passar pelas regras de negócio.
* **Sem Anemic Domain Model:** Evite entidades que sejam apenas "sacolas de getters e setters". A lógica de validação de dados da entidade deve morar nela mesma.

---

## 💻 Exemplos dos Códigos (Camada 02.Domain)

01. Entities/BaseEntity.cs
Local: `src/02.Domain/Entities/BaseEntity.cs`
```csharp
using System;

namespace MinhaSolucao.Domain.Entities
{
    public abstract class BaseEntity
    {
        // Usamos Guid como padrão para identidade única
        public Guid Id { get; protected set; }
        public DateTime DataCadastro { get; protected set; }

        protected BaseEntity()
        {
            Id = Guid.NewGuid();
            DataCadastro = DateTime.Now;
        }
    }
}
```

02. Entities/Produto.cs
Local: `src/02.Domain/Entities/Produto.cs`
```csharp
using MinhaSolucao.Domain.Exceptions;

namespace MinhaSolucao.Domain.Entities
{
    public class Produto : BaseEntity
    {
        public string Nome { get; private set; }
        public decimal Preco { get; private set; }
        public int Estoque { get; private set; }
        public bool Ativo { get; private set; }

        // Construtor garante que o objeto nasce válido
        public Produto(string nome, decimal preco, int estoque)
        {
            ValidarDominio(nome, preco, estoque);
            Nome = nome;
            Preco = preco;
            Estoque = estoque;
            Ativo = true;
        }

        public void Inativar() => Ativo = false;

        public void DebitarEstoque(int quantidade)
        {
            DomainException.When(quantidade <= 0, "A quantidade para débito deve ser maior que zero.");
            DomainException.When(Estoque < quantidade, "Estoque insuficiente.");
            Estoque -= quantidade;
        }

        private void ValidarDominio(string nome, decimal preco, int estoque)
        {
            DomainException.When(string.IsNullOrEmpty(nome), "O nome do produto é obrigatório.");
            DomainException.When(preco < 0, "O preço não pode ser negativo.");
            DomainException.When(estoque < 0, "O estoque inicial não pode ser negativo.");
        }
    }
}
```

03. ValueObjects/Email.cs
Local: `src/02.Domain/ValueObjects/Email.cs`
```csharp
using System.Text.RegularExpressions;
using MinhaSolucao.Domain.Exceptions;

namespace MinhaSolucao.Domain.ValueObjects
{
    public class Email
    {
        public string Endereco { get; private set; }

        public Email(string endereco)
        {
            DomainException.When(string.IsNullOrEmpty(endereco), "E-mail é obrigatório.");
            DomainException.When(!Regex.IsMatch(endereco, @"^[^@\s]+@[^@\s]+\.[^@\s]+$"), "E-mail em formato inválido.");

            Endereco = endereco;
        }
    }
}
```

04. Exceptions/DomainException.cs
Local: `src/02.Domain/Exceptions/DomainException.cs`
```csharp
using System;

namespace MinhaSolucao.Domain.Exceptions
{
    public class DomainException : Exception
    {
        public DomainException(string message) : base(message) { }

        public static void When(bool hasError, string error)
        {
            if (hasError) throw new DomainException(error);
        }
    }
}
```

05. O Contrato de Dados (Repositório)
Local: `src/02.Domain/Interfaces/Repositories/IProdutoRepository.cs`
```csharp
using System;
using System.Collections.Generic;
using System.Threading.Tasks;
using MinhaSolucao.Domain.Entities; 

namespace MinhaSolucao.Domain.Interfaces.Repositories
{
    public interface IProdutoRepository
    {
        // Métodos focados em BANCO DE DADOS (Persistência)
        Task<Produto> ObterPorIdAsync(Guid id);
        Task<IEnumerable<Produto>> ObterTodosAsync();
        Task AdicionarAsync(Produto produto);
        Task AtualizarAsync(Produto produto);
        Task RemoverAsync(Guid id);
        
        Task<bool> ExisteProdutoComMesmoNomeAsync(string nome);
    }
}
```

06. O Contrato de Negócio (Serviço)
Local: `src/02.Domain/Interfaces/Services/IProdutoService.cs`
```csharp
using System;
using System.Threading.Tasks;
using MinhaSolucao.Domain.Entities;

namespace MinhaSolucao.Domain.Interfaces.Services
{
    public interface IProdutoService
    {
        // Métodos focados em REGRAS DE NEGÓCIO (Ações)
        // O nome reflete a intenção do usuário, não o comando SQL.
        Task CadastrarNovoProdutoAsync(Produto produto);
        
        Task InativarProdutoAsync(Guid id);
        Task AtualizarPrecoProdutoAsync(Guid id, decimal novoPreco);
    }
}
```

07. Implementação do Service (Lógica Complexa)
Local: `src/02.Domain/Services/ProdutoService.cs`
```csharp
using System;
using System.Threading.Tasks;
using MinhaSolucao.Domain.Entities;
using MinhaSolucao.Domain.Interfaces.Repositories; // <--- Importante!
using MinhaSolucao.Domain.Interfaces.Services;     // <--- O contrato que ele assina

namespace MinhaSolucao.Domain.Services
{
    public class ProdutoService : IProdutoService
    {
        private readonly IProdutoRepository _produtoRepository;

        public ProdutoService(IProdutoRepository produtoRepository)
        {
            _produtoRepository = produtoRepository;
        }

        // Implementação do método definido na Interface
        public async Task CadastrarNovoProdutoAsync(Produto produto)
        {
            // --- REGRA DE NEGÓCIO ---
            if (produto.Preco <= 0) 
                throw new DomainException("O preço deve ser maior que zero.");

            if (string.IsNullOrEmpty(produto.Nome))
                throw new DomainException("O nome é obrigatório.");
            
            // Validação de duplicidade usando o repositório
            if (await _produtoRepository.ExisteProdutoComMesmoNomeAsync(produto.Nome))
                 throw new DomainException("Produto já cadastrado.");

            // SE PASSOU, chama o repositório para persistir
            // Note que aqui chamamos "AdicionarAsync" (Banco) e não "Cadastrar" (Negócio)
            await _produtoRepository.AdicionarAsync(produto);
        }
        
        public Task InativarProdutoAsync(Guid id) => throw new NotImplementedException();
        public Task AtualizarPrecoProdutoAsync(Guid id, decimal novoPreco) => throw new NotImplementedException();
    }
}
```

02-01-Fluxo-de-Execucao

🔄 Fluxo de Execução

1. Configuração (Program.cs)
Local: `src/01.Web.MVC/Program.cs`
```csharp
using MinhaSolucao.Domain.Interfaces.Services;
using MinhaSolucao.Domain.Services;
using MinhaSolucao.Domain.Interfaces.Repositories;
using MinhaSolucao.Infrastructure.Repositories;
using MinhaSolucao.Infrastructure.Context;
using Microsoft.EntityFrameworkCore;

var builder = WebApplication.CreateBuilder(args);

// 1. Configurar Banco de Dados (Exemplo Postgre ou Oracle)
builder.Services.AddDbContext<MeuDbContext>(options =>
    options.UseNpgsql(builder.Configuration.GetConnectionString("DefaultConnection")));

// 2. Configurar Injeção de Dependência (A LIGAÇÃO DAS PONTAS)
// "Quando a Service pedir IProdutoRepository, entregue ProdutoRepository"
builder.Services.AddScoped<IProdutoRepository, ProdutoRepository>();

// "Quando a Controller pedir IProdutoService, entregue ProdutoService"
builder.Services.AddScoped<IProdutoService, ProdutoService>();

builder.Services.AddControllersWithViews();

var app = builder.Build();

// ... Configurações de Middleware ...

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```
Conceito: Injeção de Dependência. Usamos o comando builder.Services.AddScoped<Interface, Classe>() para dizer que quando alguém pedir o objeto do lado esquerdo (Contrato), receberá um objeto do lado direito (Implementação).

2. A Chamada (Controller)
Local: `src/01.Web.MVC/Controllers/ProdutoController.cs`
```csharp
using Microsoft.AspNetCore.Mvc;
using MinhaSolucao.Domain.Entities;
using MinhaSolucao.Domain.Interfaces.Services; // <--- Importante!

namespace MinhaSolucao.Web.MVC.Controllers
{
    public class ProdutoController : Controller
    {
        private readonly IProdutoService _produtoService;

        // O sistema injeta a implementação real (ProdutoService) aqui
        // baseada no que foi configurado no Program.cs
        public ProdutoController(IProdutoService produtoService)
        {
            _produtoService = produtoService;
        }

        [HttpPost]
        public async Task<IActionResult> Create(Produto produto)
        {
            if (!ModelState.IsValid) return View(produto);

            // A CHAMADA: O Controller pede para a Interface executar a ação de NEGÓCIO
            // Observe que usamos "CadastrarNovoProdutoAsync"
            await _produtoService.CadastrarNovoProdutoAsync(produto);

            return RedirectToAction(nameof(Index));
        }
    }
}
```
Por que usar Interface? Chamamos o Domain Service pelo contrato IProdutoService para que a Controller não fique "refém" da classe concreta. Isso permite trocar a regra de negócio ou criar testes falsos (Mocks) sem quebrar o código da tela.

3. A Execução (Infrastructure)
Local: `src/03.Infrastructure/Repositories/ProdutoRepository.cs`
```csharp
using System.Threading.Tasks;
using MinhaSolucao.Domain.Entities;
using MinhaSolucao.Domain.Interfaces.Repositories;
using MinhaSolucao.Infrastructure.Context;

namespace MinhaSolucao.Infrastructure.Repositories
{
    public class ProdutoRepository : IProdutoRepository
    {
        private readonly MeuDbContext _context;

        public ProdutoRepository(MeuDbContext context)
        {
            _context = context;
        }

        public async Task AdicionarAsync(Produto produto)
        {
            // O repositório apenas traduz o pedido para o Entity Framework
            await _context.Produtos.AddAsync(produto);
            await _context.SaveChangesAsync();
        }
    }
}
```
