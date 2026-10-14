# Tarefas

> Cada tarefa é pequena, ordenada e verificável sozinha, e aponta para o requisito que a originou.
> Origem: `RF/RN/B/CA` = `spec.md`; `C` = `constitution.md`; `D/R` = `plan.md`.
> Cada tarefa vive em sua própria branch `task/T-XX-descricao` e em seu próprio commit/PR (C-31, C-35, C-40).
> Em toda tarefa, "Concluída quando" inclui implicitamente `dotnet build` sem warnings e `dotnet test` passando (C-17, C-18, C-37).

## Fundação

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-01 | Criar a solução com os projetos `PasswordGenerator.Api` e `PasswordGenerator.Api.Tests` em `src/`, com `Nullable` e `TreatWarningsAsErrors=true`, e as pastas `Endpoints`, `Application`, `Domain`, `Infrastructure` | C-01, C-02, C-04, C-06, C-22, D-01, D-15 | | `dotnet build` e `dotnet test` concluem sem warnings (mesmo sem testes) |
| T-02 | Adicionar `*.db`, `*.db-shm` e `*.db-wal` ao `.gitignore` | D-20, R-10 | | `git check-ignore app.db` confirma que o arquivo é ignorado |

## Regras de negócio

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-03 | Criar as constantes de regra: tamanho mínimo/padrão 16, máximo 128 e os quatro conjuntos de caracteres (maiúsculas, minúsculas, dígitos, especiais `! @ # $ % ^ & * - _ + = ?`) | RN-01, RN-08, RN-10, RF-02, D-07, D-08 | T-01 | Teste unitário confirma os limites e que nenhum conjunto contém espaço em branco ou os caracteres proibidos |
| T-04 | Implementar a composição da senha com `RandomNumberGenerator`: um caractere sorteado de cada categoria e o restante sorteado do conjunto total até o tamanho pedido (ainda sem embaralhar) | RN-01 a RN-04, RN-09, B-08, B-10, D-05, D-06, R-11 | T-03 | Testes unitários: tamanho exato (16 e 128), as quatro categorias presentes, nenhum espaço em branco, especiais só do conjunto permitido |
| T-05 | Embaralhar a senha composta com Fisher-Yates usando `RandomNumberGenerator`, concluindo o gerador | RN-07, RN-09, D-05, D-06, R-06, R-11 | T-04 | Testes unitários: 10.000 senhas todas distintas, as quatro categorias continuam presentes, e a primeira posição não é sempre da mesma categoria |
| T-06 | Implementar a validação do tamanho solicitado, devolvendo erro por campo (`length`) | RF-02, RF-07, RN-01, RN-08, B-01, B-02, B-04, D-09 | T-03 | Testes unitários: 15 e 129 inválidos, 16 e 128 válidos, ausente usa 16; erro identifica o campo `length` |

## Persistência

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-07 | Criar a entidade `PasswordEntry` e o `AppDbContext`, com `EnsureCreated`, caminho do banco em `appsettings.json`, criação do diretório e `Default Timeout` de 5 s | RF-03, D-02, D-03, D-17, D-19, D-20, R-10 | T-01 | Teste com SQLite temporário cria o schema; com diretório inexistente a criação ocorre sem erro |
| T-08 | Implementar `IPasswordRepository` e `PasswordRepository` (salvar e buscar por id) | RF-03, RF-06, RN-06, D-17, C-09 | T-07 | Testes: salvar e buscar devolve a mesma senha; id inexistente devolve vazio; não há operação de atualizar nem de excluir |

## Serviço de aplicação

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-09 | Implementar `PasswordService.Generate`: valida, gera, persiste e devolve id e senha, com retentativa de GUID em chave duplicada (até 3 vezes) | RF-01, RF-04, RN-05, D-04, D-09, D-18, R-08 | T-05, T-06, T-08 | Testes com repositório falso: sucesso devolve id e senha; 1 duplicidade seguida de sucesso funciona; 3 duplicidades devolvem erro; tamanho inválido não grava |
| T-10 | Implementar `PasswordService.GetById`: converte o id com `Guid.TryParse` e distingue inválido, não encontrado e sucesso | RF-05, RF-06, RF-07, RF-08, B-05, B-06, B-07, B-09, D-11 | T-08 | Testes: `abc` e vazio dão inválido; GUID inexistente dá não encontrado; GUID existente devolve a senha, igual em consultas repetidas |

## API HTTP

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-11 | Configurar a aplicação: injeção de dependência (`DbContext`, repositório, serviço, gerador), `AddProblemDetails` e inicialização do banco | C-08, C-09, D-10, D-12, D-17, R-03 | T-07, T-08, T-09, T-10 | `dotnet run` inicia sem erros |
| T-12 | Criar o endpoint `POST /passwords` com corpo opcional e resposta 201 com `Location` | RF-01 a RF-04, RF-07, B-01 a B-04, CA-01 a CA-04, D-10, D-12, C-12, C-13 | T-11 | Testes de integração: sem corpo retorna 201 com senha de 16; `length` 20 retorna senha de 20; 15, 129, `"abc"` e `16.5` retornam 400 com `ProblemDetails` identificando `length` |
| T-13 | Criar o endpoint `GET /passwords/{id}` (id recebido como texto) | RF-05, RF-06, RF-08, B-05 a B-07, CA-05 a CA-07, D-11, C-12, C-14 | T-11 | Testes de integração: id existente retorna 200 com a mesma senha; `abc` retorna 400; GUID inexistente retorna 404 com `ProblemDetails`; a rota não usa `:guid` |
| T-14 | Configurar o Swagger (`Swashbuckle.AspNetCore`), com `/swagger` apenas em Development e códigos de resposta declarados nos dois endpoints | C-05, C-15, C-16, D-13, R-01, R-13 | T-12, T-13 | `/swagger` abre sem erro; teste lê o documento OpenAPI e confere os dois endpoints com os códigos 200/201/400/404 |

## Verificações transversais

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-15 | Testar persistência após reinício: gerar com uma instância da API, consultar com outra instância sobre o mesmo arquivo | RF-03, CA-05, D-14, R-05 | T-12, T-13 | Teste de integração passa e o arquivo temporário é removido ao final |
| T-16 | Testar geração concorrente: 50 `POST /passwords` em paralelo | R-09, D-19, CA-08 | T-12 | Teste de integração: todas as respostas são 201, com ids e senhas distintos |
| T-17 | Criar o teste de arquitetura por reflexão: tipos de `Endpoints` não dependem de `Infrastructure` nem de `DbContext` | C-07, D-21, R-12 | T-12, T-13 | Teste passa; `grep` por `DbContext` na pasta `Endpoints` não retorna nada |

## Fechamento

| ID | Tarefa | Origem | Depende de | Concluída quando |
|---|---|---|---|---|
| T-18 | Conferir a definição de pronto e preencher o `readme.md` (nome, o que foi implementado, aprendizados, problema) | Desafio (definição de pronto), C-17 a C-19, R-07 | T-14, T-15, T-16, T-17 | `dotnet build` sem warnings, `dotnet test` todos passando, `dotnet run` sem erros, e `readme.md` preenchido |
