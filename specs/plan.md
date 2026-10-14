# Plano técnico

> Aqui entra o **como**. Toda decisão tem justificativa e aponta para o requisito (`spec.md`) ou regra (`constitution.md`) que a originou.
> Projeto lúdico de aprendizado. Não usar em produção.

## Contexto

Implementar o serviço descrito em `spec.md`: gerar senhas fortes, armazená-las e consultá-las por GUID. A stack, as regras de arquitetura e de qualidade já estão fixadas na `constitution.md`; este plano detalha como aplicá-las a este serviço.

Rastreabilidade: `RF-xx`, `RN-xx`, `B-xx`, `CA-xx` referem-se ao `spec.md`; `C-xx` à `constitution.md`; `D-xx` às decisões deste plano.

## Arquitetura

Fluxo de uma requisição (C-07, C-08, C-09):

```
HTTP -> Endpoint -> PasswordService -> IPasswordRepository -> SQLite (EF Core)
                         |
                         +-> IPasswordGenerator (regra de geração)
```

Estrutura de pastas (dentro de `src/`):

```
src/
  PasswordGenerator.Api/
    Endpoints/        # mapeamento HTTP; sem regra e sem DbContext
    Application/      # PasswordService, validação, resultados
    Domain/           # PasswordGenerator (regras RN-01..RN-10), entidade PasswordEntry
    Infrastructure/   # AppDbContext, PasswordRepository
    Program.cs
  PasswordGenerator.Api.Tests/
    Unit/             # geração, validação, serviço (com repositório falso)
    Integration/      # endpoints com WebApplicationFactory
```

Responsabilidades:

| Camada | Faz | Não faz |
|---|---|---|
| Endpoint | Recebe a requisição, chama o serviço, traduz o resultado em HTTP (201/200/400/404) | Regra de negócio, acesso a dados |
| `PasswordService` | Valida o pedido, usa o gerador, persiste via repositório, devolve resultado | Conhece HTTP |
| `PasswordGenerator` | Gera a senha conforme RN-01 a RN-10 | Persistência |
| `IPasswordRepository` | Salva e busca `PasswordEntry` | Regra de negócio |

## Decisões

| ID | Decisão | Justificativa | Origem |
|---|---|---|---|
| D-01 | Dois projetos: `PasswordGenerator.Api` e `PasswordGenerator.Api.Tests` | Um projeto de API com pastas por camada basta para o tamanho do serviço; o de testes segue a convenção `<Projeto>.Tests` | C-06, C-22 |
| D-02 | Persistir com SQLite via EF Core (`Microsoft.EntityFrameworkCore.Sqlite`) em arquivo | A senha precisa sobreviver a reinício do serviço; arquivo (não memória) é o que garante isso | RF-03, CA-05, C-03 |
| D-03 | Criar o schema com `Database.EnsureCreated()` na inicialização, sem migrations | Há uma única tabela e nenhum plano de evolução; migrations exigiriam a ferramenta `dotnet-ef` (dependência extra) sem benefício aqui | C-10 |
| D-04 | Gerar o identificador com `Guid.NewGuid()` e usá-lo como chave primária | GUID é exigido; chave primária garante unicidade no banco além da probabilidade do GUID | RF-04, RN-05, C-24 |
| D-05 | Gerar caracteres com `System.Security.Cryptography.RandomNumberGenerator` (`GetInt32`), nunca `System.Random` | `Random` é previsível; o `RandomNumberGenerator` é criptograficamente seguro e já vem no .NET (sem dependência nova) | RN-09, CA-10 |
| D-06 | Algoritmo de geração: (1) sorteia 1 caractere de cada categoria (maiúscula, minúscula, dígito, especial); (2) completa até o tamanho com caracteres sorteados do conjunto total; (3) embaralha com Fisher-Yates usando `RandomNumberGenerator` | Garante as quatro categorias mesmo com 16 caracteres, sem precisar gerar e rejeitar; o embaralhamento evita que as categorias fiquem sempre nas primeiras posições | RN-02, RN-04, B-08 |
| D-07 | Conjuntos: maiúsculas `A-Z`; minúsculas `a-z`; dígitos `0-9`; especiais exatamente `! @ # $ % ^ & * - _ + = ?` (sem espaço) | Espelha o spec; nenhum conjunto contém espaço em branco, o que garante a RN-03 por construção | RN-03, RN-04, RN-10 |
| D-08 | Limites do tamanho como constantes: mínimo/padrão 16, máximo 128 | Centraliza RN-01, RN-08 e RF-02 em um só lugar, usado pelo validador e pelos testes | RN-01, RN-08, RF-02 |
| D-09 | Validação feita no `PasswordService`, que retorna um resultado (sucesso, inválido com erros por campo, ou não encontrado); o endpoint converte em HTTP | Mantém a regra fora do endpoint e permite testá-la sem HTTP | C-08, C-11 |
| D-10 | Erros de validação saem como `ValidationProblemDetails` (HTTP 400), com o nome do campo no dicionário de erros; não encontrado sai como `ProblemDetails` (HTTP 404) | Cumpre "400 com ProblemDetails" e "404 com ProblemDetails", e identifica o campo que falhou | RF-07, RF-08, C-13, C-14 |
| D-11 | O `id` da rota é recebido como `string` e convertido com `Guid.TryParse` no serviço; **não** usar a restrição de rota `{id:guid}` | Com `{id:guid}` um valor malformado não casa a rota e vira 404; o spec exige 400 para formato inválido e 404 só para GUID válido inexistente | B-05, B-06, CA-06, CA-07 |
| D-12 | O corpo do POST é opcional; `length` é `int?`. Corpo ausente ou `length` nulo usa o padrão 16. Valor não inteiro (texto, decimal) falha na desserialização e é convertido em 400 com `ProblemDetails` | `AddProblemDetails` deixa as falhas de binding no formato padrão exigido | RF-02, B-03, C-13 |
| D-13 | Documentar a API com `Swashbuckle.AspNetCore`; Swagger UI em `/swagger` apenas no ambiente Development; cada endpoint declara os códigos de resposta (`Produces`/`ProducesProblem`) | A constituição exige Swagger; o pacote embutido do .NET gera o documento OpenAPI mas não traz a UI, e o Swashbuckle entrega os dois | C-05, C-15, C-16 |
| D-14 | Testes: xUnit; integração com `WebApplicationFactory` (`Microsoft.AspNetCore.Mvc.Testing`), usando um arquivo SQLite temporário por teste | `WebApplicationFactory` sobe a API real em memória sem abrir porta; arquivo temporário permite testar persistência real (CA-05) sem tocar no banco de desenvolvimento | C-04, C-12, CA-05 |
| D-15 | Build estrito: `Nullable` habilitado e `TreatWarningsAsErrors=true` nos dois projetos | Torna "sem warnings" verificável: qualquer warning quebra o build | C-17 |
| D-16 | A senha é armazenada em texto puro | Criptografia está fora do escopo; sem isso a consulta devolve a senha original | Fora de escopo (spec), RF-06 |
| D-17 | `DbContext` e repositório com injeção de dependência (`AddDbContext`, `AddScoped`); string de conexão em `appsettings.json` | Atende à inversão de dependência e permite que os testes troquem o banco | C-09 |
| D-18 | Se a gravação falhar por chave primária duplicada, o serviço gera novo GUID e tenta de novo até 3 vezes; esgotadas as tentativas, a requisição termina em erro 500 (`ProblemDetails`) | A chave primária impede duplicidade, mas o plano precisa dizer o que acontece no caso improvável; tentar de novo é barato e evita falha desnecessária | RN-05, R-08 |
| D-19 | A string de conexão do SQLite define `Default Timeout` (espera por bloqueio) de 5 segundos | Gerações simultâneas disputam o mesmo arquivo; esperar um pouco evita o erro de banco bloqueado | R-09 |
| D-20 | O caminho do arquivo do banco vem de `appsettings.json`; a aplicação cria o diretório se ele não existir; `*.db`, `*.db-shm` e `*.db-wal` entram no `.gitignore` | Evita falha na primeira execução e impede que senhas armazenadas sejam versionadas por engano | RF-03, R-10 |
| D-21 | Um teste de arquitetura por reflexão (sem pacote novo) verifica que os tipos de `Endpoints` não dependem de `Infrastructure` nem de `DbContext` | Torna a C-07 verificável automaticamente, sem adicionar biblioteca de testes de arquitetura | C-07, C-10, R-12 |

### Dependências novas (C-10)

| Pacote | Projeto | Justificativa |
|---|---|---|
| `Microsoft.EntityFrameworkCore.Sqlite` | Api | D-02: persistência em SQLite via EF Core (C-03) |
| `Swashbuckle.AspNetCore` | Api | D-13: Swagger UI e documento OpenAPI (C-05) |
| `Microsoft.AspNetCore.Mvc.Testing` | Api.Tests | D-14: testes de integração dos endpoints (C-12) |
| `xunit`, `xunit.runner.visualstudio`, `Microsoft.NET.Test.Sdk` | Api.Tests | D-14: framework de testes (C-04) e executor do `dotnet test` |

Versões: usar as estáveis compatíveis com .NET 10, fixadas nos `.csproj`. Nenhum outro pacote entra sem nova justificativa neste plano.

## Modelo de dados

Tabela `PasswordEntries` (entidade `PasswordEntry`):

| Coluna | Tipo | Restrições | Observação |
|---|---|---|---|
| `Id` | GUID | Chave primária | Gerado pela aplicação (D-04) |
| `Password` | texto | Obrigatória, até 128 caracteres | Texto puro (D-16) |
| `CreatedAtUtc` | data/hora UTC | Obrigatória | Auditoria simples; não exposta na API |

Sem atualização nem exclusão de linhas (RN-06).

## Contratos

### POST /passwords

Gera e armazena uma senha. (RF-01 a RF-04)

Requisição (corpo opcional):

```json
{ "length": 20 }
```

| Resposta | Quando | Corpo |
|---|---|---|
| `201 Created` | Senha gerada; cabeçalho `Location: /passwords/{id}` | `{ "id": "<guid>", "password": "<senha>" }` |
| `400 Bad Request` | `length` < 16, > 128 ou não inteiro (B-01 a B-03) | `ValidationProblemDetails`, com erro em `length` |

### GET /passwords/{id}

Consulta uma senha pelo identificador. (RF-05, RF-06, RF-08)

| Resposta | Quando | Corpo |
|---|---|---|
| `200 OK` | Id existe | `{ "id": "<guid>", "password": "<senha>" }` |
| `400 Bad Request` | `id` não é um GUID válido ou está vazio (B-05, B-07) | `ValidationProblemDetails`, com erro em `id` |
| `404 Not Found` | GUID válido, mas inexistente (B-06) | `ProblemDetails` |

Notas:

- Todas as respostas são JSON (C-24).
- A definição final dos contratos é espelhada no Swagger (C-15); em caso de divergência, vale este plano.

## Riscos

Probabilidade (P) e Impacto (I): B = baixa, M = média, A = alta. "Verificação" diz como saber, olhando código ou rodando um comando, que o risco está tratado.

| ID | Risco | P | I | Mitigação | Verificação |
|---|---|---|---|---|---|
| R-01 | Versão do Swashbuckle incompatível com .NET 10 | M | M | Escolher versão compatível ao criar o projeto e fixá-la no `.csproj` | `dotnet run` em Development abre `/swagger` sem erro (C-16) |
| R-02 | `{id:guid}` na rota devolveria 404 para GUID malformado | A | A | D-11: receber `id` como `string` e validar no serviço | Teste de integração: `GET /passwords/abc` retorna 400 (B-05); a rota não contém `:guid` |
| R-03 | Falha de binding do corpo (ex.: `"length": "abc"`) sair fora do formato `ProblemDetails` | M | M | D-12: habilitar `AddProblemDetails` | Teste de integração: `length` textual e decimal retornam 400 com `ProblemDetails` (B-03) |
| R-04 | Warnings escondidos quebrando o build estrito (D-15) | M | B | Corrigir o warning na origem; supressão só com justificativa registrada neste plano | `dotnet build` conclui sem warnings; `.csproj` contém `TreatWarningsAsErrors=true` e não há `#pragma warning disable` sem comentário |
| R-05 | Testes de integração deixando arquivos SQLite temporários | M | B | Cada teste cria o arquivo em pasta temporária e o remove no descarte da fábrica | Após `dotnet test`, a pasta temporária dos testes não contém arquivos `.db` |
| R-06 | Teste de aleatoriedade (CA-10) ser instável | M | M | Testar propriedades objetivas (milhares de senhas únicas, presença das categorias), sem estatística fina | Teste unitário gera 10.000 senhas, todas distintas e com as quatro categorias; roda 10 vezes seguidas sem falhar |
| R-07 | Senha em texto puro vazar | B | B | Aceito por escopo (D-16) | O aviso "não usar em produção" está no `spec.md`, no `plan.md` e no `readme.md` |
| R-08 | Colisão de GUID ou falha de gravação por chave duplicada | B | M | Em caso de violação de chave primária, gerar novo GUID e tentar de novo até 3 vezes; esgotadas as tentativas, a requisição falha com erro 500 (`ProblemDetails`) | Teste unitário do serviço com repositório falso que falha por duplicidade: 1 falha seguida de sucesso retorna 201; 3 falhas retornam erro |
| R-09 | Concorrência no SQLite (várias gerações simultâneas) causando erro de banco bloqueado | M | M | Operações curtas, uma por requisição, e `Default Timeout` na string de conexão (D-19) | Teste de integração dispara 50 `POST /passwords` em paralelo e todos retornam 201 com ids distintos |
| R-10 | Arquivo do banco em local inexistente, sem permissão, ou versionado por engano no Git | M | A | O caminho vem de `appsettings.json`; a aplicação cria o diretório na inicialização; `*.db`, `*.db-shm` e `*.db-wal` entram no `.gitignore` | `dotnet run` com diretório inexistente inicia sem erro; `git check-ignore app.db` confirma que o arquivo é ignorado |
| R-11 | Viés na escolha dos caracteres por uso de `% tamanho` sobre bytes aleatórios | B | M | D-05: usar `RandomNumberGenerator.GetInt32(limite)`, que já é livre de viés | Busca no código por `%` aplicado a bytes aleatórios retorna vazio; teste de distribuição simples sobre um conjunto grande de senhas |
| R-12 | A IA violar a arquitetura (ex.: endpoint referenciando o `DbContext`) | M | A | Teste de arquitetura que falha se os tipos de `Endpoints` dependerem de `Infrastructure` (C-07) | Teste unitário passa; `grep` por `DbContext` na pasta `Endpoints` retorna vazio |
| R-13 | Swagger divergir dos contratos deste plano | M | M | Declarar `Produces`/`ProducesProblem` em todo endpoint (C-15) | Teste de integração lê o documento OpenAPI e confere que os dois endpoints e seus códigos 200/201/400/404 constam |