# Constituição do projeto

> Projeto lúdico de aprendizado de Spec-Driven Development. Não usar em produção.
> Toda regra abaixo é verificável: dá para olhar o código ou rodar um comando e dizer se foi cumprida.

## Stack

- **C-01** Linguagem e runtime: C# com .NET 10.
- **C-02** A API é uma Minimal API (sem Controllers).
- **C-03** Persistência em SQLite, acessada via EF Core.
- **C-04** Testes com xUnit.
- **C-05** Documentação interativa da API com Swagger (Swashbuckle.AspNetCore), justificada no `specs/plan.md` conforme C-10.
- **C-06** Toda a solução fica dentro de `src/`; a documentação de especificação fica em `specs/`.

## Arquitetura

- **C-07** Endpoint nunca acessa o banco diretamente: só chama um serviço de aplicação. Nenhum `DbContext` é referenciado em arquivos de endpoint.
- **C-08** Regra de negócio não vive no endpoint nem no handler HTTP. Endpoint apenas recebe, delega ao serviço e traduz o resultado em resposta HTTP.
- **C-09** O serviço de aplicação depende de abstrações (interfaces) injetadas por DI, não de classes concretas de infraestrutura.
- **C-10** Nenhuma dependência nova (pacote NuGet) entra sem justificativa escrita em `specs/plan.md`.

## Qualidade

- **C-11** Toda regra de negócio tem ao menos um teste unitário.
- **C-12** Todo endpoint tem ao menos um teste de integração de caminho feliz.
- **C-13** Erro de validação sempre retorna HTTP 400 com corpo `ProblemDetails`.
- **C-14** Recurso inexistente retorna HTTP 404 com corpo `ProblemDetails`.
- **C-15** Todo endpoint aparece no Swagger com seus códigos de resposta declarados (sucesso, 400 e 404 quando aplicáveis).
- **C-16** A Swagger UI fica acessível em `/swagger` quando a aplicação roda em ambiente Development.
- **C-17** `dotnet build` conclui sem erros e sem warnings.
- **C-18** `dotnet test` conclui com todos os testes passando.
- **C-19** A aplicação inicia (`dotnet run`) sem erros.

## Convenções

- **C-20** Nomes de código (classes, métodos, variáveis, arquivos) em inglês; documentação em `specs/` em português.
- **C-21** Uma classe pública por arquivo, com o nome do arquivo igual ao nome do tipo.
- **C-22** Projetos de teste espelham a estrutura dos projetos testados e seguem o nome `<Projeto>.Tests`.
- **C-23** Nomes de testes seguem o padrão `Metodo_Cenario_ResultadoEsperado`.
- **C-24** Respostas da API são JSON e identificadores de senha são GUIDs.

## Governança

- **C-25** Toda tarefa do `specs/tasks.md` aponta para o requisito que a originou (coluna Origem).
- **C-26** Cada tarefa é implementada isoladamente: uma tarefa por vez, sem alterar arquivos fora do escopo dela.
- **C-27** Toda decisão técnica relevante está registrada em `specs/plan.md` com justificativa.
- **C-28** Mudança nesta constituição só é feita de forma explícita, em commit próprio, nunca como efeito colateral de outra tarefa.
- **C-29** Ao concluir uma tarefa, a IA lista os arquivos alterados e o requisito atendido.
- **C-30** A IA nunca executa `git commit` (nem `push`) sem pedido explícito do autor naquela ocasião; por padrão, quem commita é o autor.
- **C-31** Cada tarefa é desenvolvida em uma branch própria, nomeada `task/T-XX-descricao-curta` (ex.: `task/T-05-endpoint-geracao`).
- **C-32** As specs (`constitution.md`, `spec.md`, `plan.md`, `tasks.md`) são commitadas antes de qualquer código, em uma única branch `docs/specs-iniciais`, com um commit `docs:` por arquivo, nesta ordem: constituição, spec, plano, tarefas.
- **C-33** Mensagens de commit seguem Conventional Commits (`feat`, `fix`, `docs`, `test`, `chore`, `refactor`), escritas em português.
- **C-34** Commits de implementação citam o ID da tarefa no escopo, ex.: `feat(T-05): cria endpoint de geração de senha`.
- **C-35** Um commit contém uma única tarefa; nunca mistura duas tarefas.
- **C-36** Mensagens de commit não contêm rodapé de coautoria (`Co-Authored-By`) nem marcas de ferramenta de IA.
- **C-37** Antes de commitar uma tarefa, `dotnet build` (sem warnings) e `dotnet test` (todos passando) são executados com sucesso.
- **C-38** Ao concluir uma tarefa, ela é marcada como concluída em `specs/tasks.md`.
- **C-39** Se a implementação revelar erro na spec ou no plano, a correção do documento vai em commit `docs:` separado e anterior ao código.
- **C-40** Cada branch de tarefa é integrada à `main` por Pull Request, com título igual à mensagem de commit da tarefa.
- **C-41** O Pull Request de tarefa usa squash merge; o Pull Request das specs iniciais (C-32) usa rebase merge, para preservar um commit por arquivo. Em ambos, a branch é apagada após o merge.
- **C-42** O Pull Request só é aberto com build e testes passando (C-37), e a descrição lista os arquivos alterados e o requisito atendido.
