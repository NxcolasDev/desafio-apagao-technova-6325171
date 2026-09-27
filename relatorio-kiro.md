# 🤖 Relatório de Uso de IA — Operação TechNova

## Identificação

- **Aluno:** Nicolas de Jesus Silva
- **RA:** 6325171
- **Ferramenta(s) de IA utilizada(s):** Gemini
- **Data de conclusão:** 26/09/2026

---

## Parte A — Estratégia Geral com a IA

### A.1 Como você dividiu o desafio para a IA não alucinar nem sobrecarregar?

Dividi o desafio rigorosamente **fase por fase**, aplicando o conceito de isolamento de contexto aprendido em aula. Em vez de pedir a correção do projeto inteiro de uma vez (o que geraria respostas genéricas ou código inventado), apresentava o `README.md` da fase atual, pedia para a IA interpretar a dor relatada e os erros propositais, e só então partir para o plano de ação e a execução.

### A.2 Qual foi seu "tamanho ideal de spec/prompt"?

O tamanho ideal foi o escopo de **uma fase por vez**. Eu enviava o `README.md` junto com os códigos-fonte da fase e pedia para a IA estruturar a resposta em três etapas: 
1. Identificar a causa raiz dos erros;
2. Explicar a solução técnica de cada item;
3. Fornecer os trechos de código corrigidos e o comando de verificação.

**Exemplo de prompt eficiente:**
> *"Estou na Fase 3 (Docker Compose). O README relata 5 erros na comunicação da API com o banco Postgres. Aqui está o docker-compose.yml [código]. Identifique cada um dos 5 erros, explique o motivo da quebra e me forneça o YAML corrigido junto com a explicação do healthcheck."*

### A.3 Como você usou o CI/CD como bússola junto com a IA?

Utilizei os scripts de validação (`scripts/verificar.sh X`) e as saídas do pipeline como bússola de progressão. A cada fase corrigida, eu executava a verificação local. Se o resultado fosse `✅ OK`, o feedback servia como confirmação para eu avançar de fase e abrir a próxima especificação para a IA.

---

## Parte B — Relato Fase por Fase

### Fase 1 — Git

- **Diagnóstico (o que estava quebrado):** Senha em texto puro exposta no `database.yml` e ausência de arquivo `.gitignore` para mascarar variáveis de ambiente.
- **Como usei a IA:** Pedi para isolar o segredo em variável de ambiente `APP_ENV` no YAML e configurar o `.gitignore` para arquivos `.env`.
- **A IA errou ou alucinou em algo? O quê?:** Não, seguiu a instrução direta de substituição da string pela variável de ambiente.
- **Como validei a correção:** Execução do `scripts/verificar.sh 1` e validação da ausência de segredos com `git log`.

### Fase 2 — Docker

- **Diagnóstico:** O container estava efetuando o build e executando o processo como usuário `root`, violando o princípio de menor privilégio.
- **Como usei a IA:** Pedi a reestruturação do `Dockerfile` garantindo a troca de permissões dos arquivos para o usuário sem privilégios `node`.
- **A IA errou/alucinou?:** Inicialmente esqueceu de passar a flag `--chown=node:node` na instrução `COPY`, o que mantinha os arquivos pertencendo ao root.
- **Como validei:** `scripts/verificar.sh 2` confirmando que a imagem buildou e rodou como non-root com a flag de resposta ativa no endpoint `/flag`.

### Fase 3 — Docker Compose

- **Diagnóstico:** Redes separadas entre API e banco, `DB_HOST` errado (`database`), falta do `DB_PASSWORD`, healthcheck apontando para comando inexistente (`pg_ready`) e `depends_on` sem aguardar condição de saúde.
- **Como usei a IA:** Passei os 5 problemas listados no `README.md` para a IA apontar as correções no arquivo `docker-compose.yml`.
- **A IA errou/alucinou?:** Não errou.
- **Como validei:** `scripts/verificar.sh 3` (rodou `docker compose up`, esperou o postgres ficar `healthy` e validou a resposta no endpoint `http://localhost:3000/flag`).

### Fase 4 — Terraform / HCL

- **Diagnóstico:** Bloco `required_providers` ausente, variável `ambiente` não declarada, argumento `conteudo` em português, sintaxe de interpolação incorreta `${var::ambiente}` e atributo de output inexistente.
- **Como usei a IA:** Pedi a correção da sintaxe HCL2 do `main.tf` e a declaração da variável no `variables.tf`.
- **A IA errou/alucinou?:** Não errou.
- **Como validei:** `terraform fmt`, `terraform validate` e `scripts/verificar.sh 4` gerando o arquivo `saida/flag.txt`.

### Fase 5 — VPC / Rede / Segurança

- **Diagnóstico:** Security Group do RDS exposto para `0.0.0.0/0` na porta 5432 e ausência de rota de saída para a internet na tabela de roteamento da subnet pública.
- **Como usei a IA:** Pedi para ajustar o ingress do SG do RDS referenciando o ID do SG da API (`security_groups`) e adicionar a rota default apontando para o Internet Gateway.
- **A IA errou/alucinou?:** Não errou.
- **Como validei:** `scripts/verificar.sh 5` e conferência visual da regra de menor privilégio no `main.tf`.

### Fase 6 — RDS + Remote State

- **Diagnóstico:** Backend S3 sem encriptação e sem controle de concorrência (DynamoDB); RDS público (`publicly_accessible = true`), sem encriptação de armazenamento e fora das subnets privadas.
- **Como usei a IA:** Solicitei a inclusão das diretivas de segurança no bloco `backend "s3"` e nos parâmetros da instância `aws_db_instance`.
- **A IA errou/alucinou?:** Não errou.
- **Como validei:** `terraform validate` com `-backend=false` e `scripts/verificar.sh 6`.

### Fase 7 — Módulos

- **Diagnóstico:** Violação do princípio DRY com declarações duplicadas de `resource "local_file"` para ambientes de `dev` e `staging` na raiz.
- **Como usei a IA:** Solicitei a criação da estrutura de módulo em `modules/ambiente/` (com `main.tf` e `variables.tf`) e a refatoração do `main.tf` principal para chamar o módulo duas vezes.
- **A IA errou/alucinou?:** Não errou.
- **Como validei:** `scripts/verificar.sh 7` confirmando a criação dos arquivos `saida/dev.txt` e `saida/staging.txt`.

### Fase 8 — AWS Academy (execução real)

- **Diagnóstico / objetivo:** Provisionar a infraestrutura de forma real no AWS Academy Learner Lab respeitando as restrições do perfil `LabRole`, capturar evidências de execução e destruição da infraestrutura.
- **Como usei a IA:** Pedi a criação de uma declaração simples de VPC no `main.tf` compatível com o ambiente do Learner Lab e a estrutura do arquivo `evidencia.md`.
- **A IA errou/alucinou?:** Tentou inicialmente sugerir a criação de roles via Terraform, mas ajustei a instrução lembrando das restrições do Learner Lab para reutilizar a role pré-existente.
- **Como validei:** `aws sts get-caller-identity` no terminal WSL, preenchimento das credenciais reais no `evidencia.md` e verificação com `scripts/verificar.sh 8`.

---

## Parte C — Reflexão Crítica

### C.1 Qual foi a pior alucinação da IA no desafio e como você a percebeu?

A IA não teve alucinações graves porque mantive a estratégia de condução por etapas pequenas. A principal desatenção foi na Fase 2 (Docker), onde ela sugeriu alterar o usuário para `USER node`, mas esqueceu de ajustar a propriedade dos arquivos copiados com `--chown=node:node`. Percebi a falha ao analisar o fluxo do container e o comportamento dos arquivos no build.

### C.2 Em qual fase a IA MAIS ajudou? E em qual você teve que assumir o controle e resolver "no braço"?

- **Mais ajudou:** Fase 1 (Git) e Fase 7 (Módulos), por acelerar a escrita repetitiva de código estruturado de módulos no Terraform.
- **Assumi o controle:** Fase 2 (Docker) e Fase 8 (AWS Lab), onde tive que resolver na prática no terminal WSL os conflitos de containers antigos presos na porta 3000 e a formatação das credenciais temporárias no arquivo `~/.aws/credentials`.

### C.3 O que você faria diferente na próxima vez que usar IA para DevOps?

Manteria a mesma estratégia por ser muito eficiente, mas adicionaria uma etapa de revisão de sintaxe antes de aplicar os comandos no terminal para evitar conflitos de portas ou formatação.

### C.4 Você conseguiria ter validado as respostas da IA se NÃO tivesse feito as aulas 01 a 07?

De forma alguma. Sem o conhecimento técnico das aulas 01 a 07, eu não saberia identificar se o `docker-compose` estava falhando por causa de rede, healthcheck ou dependência, nem entenderia conceitos cruciais como *state locking* com DynamoDB ou regra de menor privilégio em Security Groups. O conhecimento das aulas é o que dá a capacidade de avaliar se a resposta da IA é válida ou errada.

---

## Checklist Final

- [x] Preenchi a estratégia geral (Parte A)
- [x] Relatei as 8 fases individualmente (Parte B)
- [x] Respondi a reflexão crítica (Parte C)
- [x] Meu CI está 100% verde (todas as fases + gate final)
- [x] Meu PR está aberto no repositório do desafio