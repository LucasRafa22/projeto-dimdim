💻 AMBIENTE LOCAL (Docker Desktop)🔹 0 - Preparar ambiente
 
powershell
 
 
cd C:\
 
git clone https://github.com/Challenge-CLYVO/Dotnet-Challenge.git
 
git clone https://github.com/LucasRafa22/Database-Challenge.git
 
 
Use o código com cuidado.
 
 
🔹 1 - Build do Oracle
 
cd C:\Database-Challenge
 
docker build -f Dockerfile.oracle -t database-challenge .
 
 
🔹 2 - Volume Local
 
 
docker volume create oracle-data
 
 
🔹 3 - Rodar Oracle Local
 
docker run -d ^
  --name database-challenge ^
  -p 1521:1521 ^
  -p 5500:5500 ^
  -e ORACLE_PWD=oracle ^
  -v oracle-data:/opt/oracle/oradata ^
  database-challenge
 
 
🔹 4 - Verificar Oracle
 
docker ps
 
docker logs database-challenge
 
 
🔹 5 - Criar usuário (Evita usar o SYSTEM na API)
 
 
docker exec -it database-challenge bash
 
sqlplus system/oracle@FREEPDB1
 
 
(Dentro do SQLPlus, execute:)*
 
sql
 
 
CREATE USER app IDENTIFIED BY app;
 
GRANT CONNECT, RESOURCE TO app;
 
ALTER USER app QUOTA UNLIMITED ON USERS;
 
 
🔹 6 - DDL de Teste
 
CREATE TABLE TESTE ( ID NUMBER PRIMARY KEY, NOME VARCHAR2(100) );
 
 
🔹 7 - CRUD Completo (Validação)
 
INSERT INTO TESTE (ID, NOME) VALUES (1, 'Lucas');
 
COMMIT;
 
-- READ (Select)
 
SELECT * FROM TESTE;
 
-- UPDATE (Atualização)
 
UPDATE TESTE SET NOME = 'Lucas Atualizado' WHERE ID = 1;
 
COMMIT;
 
-- READ (Validar o Update)
 
SELECT * FROM TESTE;
 
-- DELETE (Remoção)
 
DELETE FROM TESTE WHERE ID = 1;
 
COMMIT;
 
-- READ (Validar o Delete - Deve retornar vazio)
 
SELECT * FROM TESTE;
 
 
🔹 8 - Build da API .NET
 
exit;
 
exit;
 
cd C:\Dotnet-Challenge
 
docker build -t dotnet-challenge .
 
 
🔹 9 - Rodar API Local
 
 
docker run -d --name dotnet-challenge -p 8080:8080 -e ConnectionStrings__RecommendaContextOracle="Data Source=host.docker.internal:1521/FREEPDB1;User ID=app;Password=app;" dotnet-challenge
 
 
🔹 10 - Testar API Local
 
 
curl http://localhost:8080/health
 
 
🔹 11 - Login na Azure
 
az login --use-device-code
 
 
🔹 12 - Criar Grupo de Recursos
 
az group create -l southafricanorth -n rg-challenge-hub
 
 
🔹 13 - Criar Registro de Containers (ACR)
 
az acr create --resource-group rg-challenge-hub --name challengehubrm565194 --sku Standard --admin-enabled true
 
 
🔹 14 - Taggear Imagens para a Nuvem
 
docker images
 
docker tag database-challenge:latest challengehubrm565194.azurecr.io/db:latest
 
docker tag dotnet-challenge:latest challengehubrm565194.azurecr.io/api:latest
 
docker images
 
 
🔹 15 - Push para o ACR
 
az acr login --name challengehubrm565194
 
docker push challengehubrm565194.azurecr.io/db:latest
 
docker push challengehubrm565194.azurecr.io/api:latest
 
 
🔹 16 - Criar Container da API na Azure (Injetando Banco FIAP com segurança e Forçando Swagger)

az container create ^
  --resource-group rg-challenge-hub ^
  --name challengehubrm565194-api ^
  --image challengehubrm565194.azurecr.io/api:latest ^
  --cpu 1 ^
  --memory 2 ^
  --ports 8080 ^
  --ip-address Public ^
  --os-type Linux ^
  --environment-variables ConnectionStrings__RecommendaContextOracle="Data Source=oracle.fiap.com.br:1521/orcl;User ID=<SEU_USUARIO>;Password=<SUA_SENHA>;" ASPNETCORE_ENVIRONMENT="Development"

🔹 17 - Criar Conta de Armazenamento (Persistência Cloud)
 
az storage account create --name challengehubrm565194 --resource-group rg-challenge-hub --sku Standard_LRS
 
 
🔹 18 - Criar File Share para o Oracle
 
az storage share create --name oracle-data --account-name challengehubrm565194
 
 
🔹 19 - Criar Container do Oracle na Azure (Com o Volume mapeado na pasta admin)
 
az storage account keys list --resource-group rg-challenge-hub --account-name challengehubrm565194 --query "[0].value" --output tsv
 
COPIE A KEY E COLE EM "SUA_KEY_DE_ACESSO"
 
az container create --resource-group rg-challenge-hub --name challengehubrm565194-db --image challengehubrm565194.azurecr.io/db:latest --cpu 2 --memory 4 --ports 1521 --ip-address Public --os-type Linux --environment-variables ORACLE_PWD=oracle --azure-file-volume-account-name challengehubrm565194 --azure-file-volume-account-key "SUA_KEY_DE_ACESSO" --azure-file-volume-share-name oracle-data --azure-file-volume-mount-path /opt/oracle/admin

 
 
🔹 20 - Obter IP Público do Oracle na Azure
 
az container show --resource-group rg-challenge-hub --name challengehubrm565194-db --query ipAddress.ip --output tsv
 
COPIE E GUARDE O IP PUBLICO E COLE EM SUA_IP_PUBLICA

 
🔹 21 - Criar Container da API .NET na Azure (Injetando a porta 1521 e liberando o Swagger)
az container create --resource-group rg-challenge-hub --name challengehubrm565194-api --image challengehubrm565194.azurecr.io/api:latest --cpu 1 --memory 2 --ports 8080 --ip-address Public --os-type Linux --environment-variables ConnectionStrings__RecommendaContextOracle="Data Source=SUA_IP_PUBLICA:1521/FREEPDB1;User ID=app;Password=app;" ASPNETCORE_ENVIRONMENT="Development"

🔹 22 - Testar API e Acessar o Swagger na Nuvem
az container show --resource-group rg-challenge-hub --name challengehubrm565194-api --query ipAddress.ip --output tsv
COPIE O IP DA API E SUBSTITUA EM "API_IP" NO SEU NAVEGADOR:
 

 
Interface do Swagger (Navegador):
Abra o seu navegador (Chrome, Edge, etc.) e digite:
http://API_IP:8080/swagger/index.html