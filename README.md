Laboratório de Entrega Contínua (CD) — .NET 10, Docker e GitHub Actions

Aluno(a): Rafael Machado Alves Vaz

RA: 22511559

Repositório: https://github.com/rafaelvaz-arch/ceub-cd-dotnet10

Stack Tecnológica: .NET 10 SDK (10.0.404), Docker (29.6.2), WSL 2 (Ubuntu 24.04) e GitHub Actions (Self-Hosted Runner com label ceub-cd).

1. Estrutura do Projeto e Instruções de Reprodução

Arquivos Versionados no Repositório

src/Ceub.Api/ — API Web minimalista em .NET 10 expondo os endpoints /, /health, /api/entregas e /api/novidade.

tests/Ceub.Api.Tests/ — Suíte de testes unitários xUnit (CatalogoTests.cs) atuando como Quality Gate no pipeline.

Dockerfile — Build multi-stage para empacotamento imutável da aplicação.

scripts/ — Scripts de automação de deploy, validação de healthcheck e rollback automático.

.github/workflows/ci-staging.yml — Pipeline de Integração Contínua e entrega automática em Staging (porta 8081).

.github/workflows/promover-producao.yml — Pipeline de promoção controlada para Production (porta 8082) via workflow_dispatch.

Como Reproduzir o Ambiente Localmente

Pré-requisitos: Ambiente Linux/WSL 2 com .NET 10 SDK, Docker em execução e um GitHub Actions Self-Hosted Runner registrado no repositório com a label ceub-cd ativo (./run.sh).

Clonar e validar localmente:

git clone https://github.com/[SEU-USUARIO]/ceub-cd-dotnet10.git
cd ceub-cd-dotnet10
dotnet test Ceub.Delivery.slnx -c Release


Disparar CI e Staging: Realizar um git push na branch main. O workflow ci-staging.yml compila, executa os testes, gera o artefato da imagem Docker e implanta o contêiner ceub-api-staging na porta 8081.

Promover para Produção: Na aba Actions do GitHub, selecionar Promover para Producao → Run workflow, informar o run_id de uma execução saudável do workflow anterior e executar.

2. Execução Saudável — CI e Staging

Link da Execução: https://github.com/[SEU-USUARIO]/ceub-cd-dotnet10/actions/runs/[RUN_ID_STAGING_SAUDAVEL]

Run ID: [RUN_ID_STAGING_SAUDAVEL]

Commit SHA: [HASH_CURTO_COMMIT_SAUDAVEL]

Artefato Gerado: Imagem Docker empacotada e anexada na execução do workflow (ceub-api-image).

Validação do Ambiente Staging (127.0.0.1:8081)

$ curl -i http://127.0.0.1:8081/
HTTP/1.1 200 OK
{"servico":"ceub-api","ambiente":"Staging","versao":"1.0.1",...}

$ curl -i http://127.0.0.1:8081/health
HTTP/1.1 200 OK


3. Promoção Saudável para Produção e Imutabilidade da Imagem

Link da Execução de Promoção: https://github.com/rafaelvaz-arch/ceub-cd-dotnet10/actions/runs/43736148627

Run ID de Origem Utilizado: 43736148627

Parâmetros: falha_controlada = false | novidade = false

Comprovação de Build Once, Deploy Many

Abaixo, a inspeção dos contêineres em execução comprova que Staging e Production executam exatamente a mesma imagem Docker (mesmo digest sha256), sem recompilação:

$ docker inspect --format '{{.Image}}' ceub-api-staging
sha256:5ef6456b8288c3c2284711b2ee2ccbdc8796aec90146a671da4e569eae25cbdf

$ docker inspect --format '{{.Image}}' ceub-api-production
sha256:5ef6456b8288c3c2284711b2ee2ccbdc8796aec90146a671da4e569eae25cbdf


$ curl -i http://127.0.0.1:8082/
HTTP/1.1 200 OK
{"servico":"ceub-api","ambiente":"Production","versao":"1.0.1",...}


4. Evidências por Experimento

Experimento A — Bloqueio de Deploy por Teste Unitário (Quality Gate) e Correção

Objetivo: Validar que falhas nos testes automatizados impedem a geração de artefato e a atualização dos ambientes.

Execução com Falha Proposital (Assert.Fail em tests/Ceub.Api.Tests/CatalogoTests.cs):

Link: https://github.com/rafaelvaz-arch/ceub-cd-dotnet10/actions/runs/43737574316

Comportamento observado: O job build falhou durante a etapa dotnet test (status Failed) e o job staging foi automaticamente ignorado (Skipped), preservando a integridade das instâncias nas portas 8081 e 8082.

Execução de Correção (git revert HEAD):

Link: https://github.com/rafaelvaz-arch/ceub-cd-dotnet10/actions/runs/43737900637

Comportamento observado: Após reverter o commit defeituoso, os jobs build e staging voltaram a executar com sucesso (Success), gerando um novo run_id saudável para promoção.

Experimento B — Falha Controlada em Produção e Rollback Automático

Objetivo: Simular uma regressão em tempo de execução que passa pelo build, mas falha no healthcheck de produção, acionando a reversão automática sem indisponibilidade.

Link da Execução (falha_controlada = true): https://github.com/rafaelvaz-arch/ceub-cd-dotnet10/actions/runs/43738297583

Trecho do Log de Rollback (Job production):

# Novo contêiner iniciado com simulação de falha ativada
# Verificando saúde em http://127.0.0.1:8082/health -> HTTP 503 Service Unavailable
# Falha detectada no healthcheck! Iniciando rollback automático para a versão anterior saudável...
# Contêiner anterior restaurado e validado com sucesso.


Validação HTTP após Recuperação:

$ curl -i http://127.0.0.1:8082/health
HTTP/1.1 200 OK

$ curl -i http://127.0.0.1:8082/
HTTP/1.1 200 OK


Experimento C — Ativação de Feature Flag sem Recompilação (404 vs 200)

Objetivo: Habilitar uma nova funcionalidade em Produção (novidade = true) apenas via configuração de ambiente, reutilizando exatamente a mesma imagem imutável já validada.

Link da Execução (novidade = true): https://github.com/rafaelvaz-arch/ceub-cd-dotnet10/actions/runs/43738656712

Comparação de Comportamento (8081 vs 8082) com a Mesma Imagem:

Staging (8081) — Feature Flag Desativada (404 Not Found):

$ curl -i http://127.0.0.1:8081/api/novidade
HTTP/1.1 404 Not Found
Content-Length: 0


Production (8082) — Feature Flag Ativada (200 OK):

$ curl -i http://127.0.0.1:8082/api/novidade
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8


Verificação de Identidade da Imagem Docker:

$ docker inspect --format 'Staging:    {{.Image}}' ceub-api-staging
Staging:    sha256:5ef6456b8288c3c2284711b2ee2ccbdc8796aec90146a671da4e569eae25cbdf

$ docker inspect --format 'Production: {{.Image}}' ceub-api-production
Production: sha256:5ef6456b8288c3c2284711b2ee2ccbdc8796aec90146a671da4e569eae25cbdf


5. Análise Técnica: Gate Manual, Estratégia de Deploy e Limites Locais

Papel do Gate Manual e Promoção de Artefato:
Enquanto o ambiente de Staging recebe atualizações continuamente a cada push aprovado nos testes (CI e Staging), a promoção para Production exige intervenção humana explícita via workflow_dispatch, informando o run_id de origem. O job validar-origem verifica se aquela execução realmente foi concluída com sucesso antes de baixar o artefato da imagem Docker. Isso assegura governança, rastreabilidade e o princípio build once, deploy many (evitando que diferenças de ambiente ou dependências alterem o binário entre homologação e produção).

Estratégia de Deploy e Recuperação:
Os ambientes são isolados em contêineres e portas distintas (8081 e 8082). Durante a implantação, o script substitui a instância e imediatamente submete o novo contêiner a uma verificação no endpoint /health. Caso a aplicação falhe ao subir ou retorne erro (como o 503 simulado no Experimento B), o mecanismo de rollback automático descarta o contêiner defeituoso e restaura a versão saudável anterior. Complementarmente, o uso de feature flags via variáveis de ambiente desacopla o deploy (entrega técnica do código) do release (liberação funcional ao usuário).

Limites do Ambiente Local:
Por ser executado em um único host local (WSL 2 + Docker Desktop) utilizando um self-hosted runner, o laboratório não contempla alta disponibilidade real, balanceamento de carga com zero-downtime (como estratégias Blue/Green ou Canary em orquestradores como Kubernetes), nem exposição externa com TLS/rede segregada. Além disso, em repositórios privados no plano gratuito do GitHub, proteções nativas de Environments (como aprovadores obrigatórios antes do início do job) possuem limitações, sendo supridas no fluxo pela validação programática de origem no workflow.
