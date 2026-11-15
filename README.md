# Laboratorio CEUB: CD com .NET 10

Projeto de referencia do tutorial HTML. Leia as etapas do HTML antes de executar.

1. Use Ubuntu 24.04 x64, .NET SDK 10, Git e Docker.
2. Na trilha rápida, extraia este ZIP, abra a pasta ceub-cd-dotnet10 e execute `git init -b main`. Execute `dotnet restore Ceub.Delivery.slnx` e `dotnet test Ceub.Delivery.slnx -c Release`.
3. Crie repositorio PRIVADO e runner Linux x64 com label `ceub-cd`, fora desta pasta.
4. Se environments nao estiverem disponiveis em seu plano, remova somente as linhas `environment:` dos workflows.
5. Confirme que 8081 e 8082 estão livres e que o runner está Idle. Envie para main: CI publica e implanta em staging (localhost:8081).
6. Espere o CI terminar com sucesso. Execute Promover para Producao na main informando o run ID desse CI.
7. Producao didatica: localhost:8082. Use falha_controlada somente apos existir versao saudavel.

Nenhum token deve ser versionado. Nao adicione gatilhos de pull request ao runner local.
Os ambientes compartilham host e usam recreate, com interrupcao e um predecessor para rollback.
O arquivo TAR do artefato tem retencao de 7 dias. O ZIP nao inclui binarios, runner nem credenciais.
