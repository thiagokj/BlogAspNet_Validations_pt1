# BlogAspNet MVC

Projeto para revisão de conceito e aprendizado, 
sendo continuidade de outros projetos já publicados.

Alguns exemplos com MVC e MVVM.

## Nomeclatura de Endpoints

A melhor prática é adotar as convenções de mercado para definição de nomes.

**Endpoints:** usar texto minúsculo e no plural. Ex: [HttpGet("categorias")].
Para nomes compostos utilize um hífen. Ex: [HttpGet("v1/user-roles")].

**Versionamento:** para facilitar a manutenção, versione utilizando o prexifo v1, v2, v3, etc.
Ex: [HttpGet("v1/categories")]

```Csharp
using Blog.Data;
using Microsoft.AspNetCore.Mvc;

namespace BlogAspNet.Controllers
{
    [ApiController]
    public class CategoryController : ControllerBase
    {
        // A convenção para nomear Endpoints é usar texto minúsculo e no plural.
        // Caso seja um nome composto, utilize um hífen como separador. Ex: post-categories.
        [HttpGet("v1/categories")]
        //[HttpGet("categorias")] // Poderia ter outro endpoint pt-br apontando para mesma rota.
        public IActionResult Get(
        [FromServices] BlogDataContext context)
        {
            var categories = context.Categories.ToList();
            return Ok(categories);
        }
    }
}
```

**async/await:** Métodos assíncronos realizam tarefas paralelas, melhorando o fluxo
de execução da aplicação. Dessa forma, não é necessario ficar aguardando o retorno
de um processamento para iniciar outro, liberando recursos para demais tarefas.

```Csharp
[HttpGet("v1/categories")]
/*
    Métodos assíncronos são executados em uma única thread. Eles liberam a thread
    ativa para outras tarefas, enquanto uma operação de entrada/saída é executada.
*/
public async Task<IActionResult> GetAsync([FromServices] BlogDataContext context)
{
    // O "await" pausa a execução do método até que a operação seja concluída.
    var categories = await context.Categories.ToListAsync();
    return Ok(categories);
}
```

## CRUD básico

Exemplo de CRUD básico com tratamento de exceções:

```Csharp
 [ApiController]
    public class CategoryController : ControllerBase
    {
        // Retorna todas as categorias
        [HttpGet("v1/categories")]
        public async Task<IActionResult> GetAsync([FromServices] BlogDataContext context)
        {
            try
            {
                var categories = await context.Categories.ToListAsync();
                return Ok(categories);
            }
            catch (Exception ex)
            {
                return StatusCode(500, "05XXE2 - Falha interna no servidor.");
            }
        }

        // Retorna categoria conforme id
        [HttpGet("v1/categories/{id:int}")]
        public async Task<IActionResult> GetByIdAsync(
            [FromRoute] int id,
            [FromServices] BlogDataContext context)
        {
            try
            {
                var category = await context
                .Categories
                .FirstOrDefaultAsync(x => x.Id == id);

                return category == null ? NotFound() : Ok(category);
            }
            catch (Exception ex)
            {
                return StatusCode(500, "05XXE4 - Falha interna no servidor.");
            }
        }

        // Insere uma nova categoria
        [HttpPost("v1/categories")]
        public async Task<IActionResult> PostAsync(
            [FromBody] Category model,
            [FromServices] BlogDataContext context)
        {
            try
            {
                await context.Categories.AddAsync(model);
                await context.SaveChangesAsync();

                return Created($"v1/categories/{model.Id}", model);
            }
            catch (DbUpdateException ex)
            {
                return StatusCode(500, "5XXE5 - Não foi possível incluir a categoria.");
            }
            catch (Exception ex)
            {
                return StatusCode(500, "5XXE6 - Falha interna no servidor.");
            }

        }

        // Atualiza categoria conforme id
        [HttpPut("v1/categories/{id:int}")]
        public async Task<IActionResult> PutAsync(
            [FromRoute] int id,
            [FromBody] Category model,
            [FromServices] BlogDataContext context)
        {
            try
            {
                var category = await context
                    .Categories
                    .FirstOrDefaultAsync(x => x.Id == id);

                if (category == null) return NotFound();

                category.Name = model.Name;
                category.Slug = model.Slug;

                context.Categories.Update(category);
                await context.SaveChangesAsync();

                return Ok(model);
            }
            catch (DbUpdateException ex)
            {
                return StatusCode(500, "05XXE7 - Não foi possível alterar a categoria.");
            }
            catch (Exception ex)
            {
                return StatusCode(500, "05XXE8 - Falha interna no servidor.");
            }

        }
        
        // Exclui uma categoria
        [HttpDelete("v1/categories/{id:int}")]
        public async Task<IActionResult> DeleteAsync(
            [FromRoute] int id,
            [FromServices] BlogDataContext context)
        {
            try
            {
                var category = await context
                    .Categories
                    .FirstOrDefaultAsync(x => x.Id == id);

                if (category == null) return NotFound();

                context.Categories.Remove(category);
                await context.SaveChangesAsync();

                return Ok(category);
            }
            catch (DbUpdateException ex)
            {
                return StatusCode(500, "05XXE9 - Não foi possível excluir a categoria.");
            }
            catch (Exception ex)
            {
                return StatusCode(500, "05XXE10 - Falha interna no servidor.");
            }
        }
```

## ViewModels - Modelo com base no input do usuário

São modelos baseados em visualizações. Uma ViewModel é uma convenção, uma adaptação do que
seria uma visualização do usuário (tela html, app desktop, mobile...) para interação com as
entradas de dados (inputs).

Com a ViewModel, o método Post fica com a responsabilidade de informar apenas o necessário.

```Csharp
public class CreateCategoryViewModel
{
    public string Name { get; set; }
    public string Slug { get; set; }
}

...try
{
    var category = new
    {
        Id = 0,
        Name = model.Name,
        Slug = model.Slug.ToLower()
    };

    await context.Categories.AddAsync(category);
    await context.SaveChangesAsync();

    return Created($"v1/categories/{category.Id}", category);
}...
```

Se as informações de Criação e Edição forem as mesmas, pode ser criada apenas
uma representação de um EditorCategoryViewModel.

```Csharp
// Alteração no tipo da model no Put
public async Task<IActionResult> PutAsync(
            [FromRoute] int id,
            [FromBody] EditorCategoryViewModel model,
            [FromServices] BlogDataContext context)
            {
            // O Asp.Net faz a verificação automatica do ModelState baseado nas validações
            // informadas no modelo. Essa validação pode ser desabilitada para criação de 
            // retorno padronizado via builder.Services
            if (!ModelState.IsValid) return BadRequest();
            ...
```

Uma grande vantagem de utilizar os ViewModels são as validações, que podem ser tratadas
de forma exclusiva para telas, já seguindo uma abordagem Data Driven (Orientado a dados).
```Csharp
 public class EditorCategoryViewModel
    {
        [Required(ErrorMessage = "O nome é obrigatório.")]
        [StringLength(40, MinimumLength = 3,
            ErrorMessage = "Esse campo deve conter entre 3 e 40 caracteres.")]
        public string Name { get; set; }

        [Required]
        public string Slug { get; set; }
        ...
```

## Padronização de retorno

Uma boa prática é padronizar os retornos da API para tratamento de dados e erros.
Para isso, pode ser criada uma classe genérica ResultViewModel.

```Csharp
public class ResultViewModel<T>
    {
        public T Data { get; private set; }
        public List<string> Errors { get; private set; } = new(); // Inicializa a lista

        public ResultViewModel(T data, List<string> errors)
        {
            Data = data;
            Errors = errors;
        }...
    }
```

No Builder, podemos desabilitar o retorno padrão da ModelState e usar o ResultViewModel.

```Csharp
builder.Services
    .AddControllers()
    .ConfigureApiBehaviorOptions(options =>
    {
        options.SuppressModelStateInvalidFilter = true;
    });
```

Adicionando o padrão de retorno temos:

```Csharp
[HttpGet("v1/categories")]
public async Task<IActionResult> GetAsync([FromServices] BlogDataContext context)
{
    try
    {
        var categories = await context.Categories.ToListAsync();

        // Retorna um objeto OK com uma instância da estrutura
        // ResultViewModel<List<Category>>,
        // que encapsula a lista de categorias do tipo Category.    
        return Ok(new ResultViewModel<List<Category>>(categories));
    }
    catch
    {
        return StatusCode(500,
            new ResultViewModel<List<Category>>("05XXE2 - Falha interna no servidor."));
    }
}

[HttpGet("v1/categories/{id:int}")]
public async Task<IActionResult> GetByIdAsync(
    [FromRoute] int id,
    [FromServices] BlogDataContext context)
{
    try
    {
        var category = await context
        .Categories
        .FirstOrDefaultAsync(x => x.Id == id);

        if (category == null)
            // Retorna erro na estrutura ResultViewModel
            return NotFound(new ResultViewModel<Category>("Contéudo não encontrado"));
        
        // Retorna uma categoria na estrutura do ResultViewModel.
        return Ok(new ResultViewModel<Category>(category));
    }
    catch
    {
        return StatusCode(500,
            new ResultViewModel<Category>("05XXE4 - Falha interna no servidor."));
    }
}
```

## Extension Methods

Continuando as validações, podemos aproveitar os recursos do Asp.Net para criar métodos de extensão.

O metódo de extensão consiste em criar uma classe e adicionar métodos que 
fazem processos mais específicos, incluindo a palavra reservada this antes de declarar
o tipo do objeto.

Por padrão no C#, uma classe de extensão deve ser estática.

```Csharp
public static class ModelStateExtension
{
    // Adicionado "this" no prefixo do retorno, representando um método de extensão.
    public static List<string> GetErros(this ModelStateDictionary modelState)
    {
        var result = new List<string>();
        foreach (var item in modelState.Values)
        {
            foreach (var error in item.Errors)
            {
                result.Add(error.ErrorMessage);
            }
        }

        return result;
    }
}
```

Agora é possivel chamar o método base com a extensão adicionada.

```Csharp
 [HttpPost("v1/categories")]
        public async Task<IActionResult> PostAsync(
            [FromBody] EditorCategoryViewModel model,
            [FromServices] BlogDataContext context)
        {

            if (!ModelState.IsValid)
            {
                // Agora o ModelState pode invocar o método extendido GetErrors().
                return BadRequest(new ResultViewModel<Category>(ModelState.GetErrors()));
            }...
```





