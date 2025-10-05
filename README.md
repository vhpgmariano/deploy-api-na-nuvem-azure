# deploy-api-na-nuvem-azure


-----

# README.md: Guia Prático de Deploy de uma API na Nuvem (Azure)

## Objetivo do Projeto

Este projeto tem como objetivo principal ser um guia prático e detalhado sobre como realizar o **deploy de uma API REST** na nuvem, utilizando a plataforma **Microsoft Azure**. O foco é na implementação do **Azure App Service** para hospedagem e na configuração de práticas recomendadas, como o uso de **Identidades Gerenciadas** e **Azure Key Vault** para segurança de credenciais.

A arquitetura final resultará em uma solução robusta, escalável e com baixo atrito na gestão de segredos.

## Tecnologias Utilizadas

| Categoria | Serviço / Tecnologia | Função |
| :--- | :--- | :--- |
| **Hospedagem da API** | **Azure App Service** | Serviço de *Platform as a Service* (PaaS) ideal para hospedar APIs Web de forma simples e escalável. |
| **Banco de Dados** | **Azure SQL Database** | Serviço de banco de dados relacional gerenciado (PaaS). |
| **Gerenciamento de Segredos** | **Azure Key Vault** | Armazenamento seguro de segredos, certificados e chaves criptográficas. |
| **Segurança e Acesso** | **Azure Entra ID (Identidades Gerenciadas)** | Permite que o App Service acesse o Key Vault e o SQL DB **sem usar senhas** no código. |

-----

## Arquitetura de Deploy

O processo de *deploy* segue uma arquitetura baseada em três pilares, focando em segurança e *serverless*:

1.  **API (Azure App Service):** Roda o código da API (ex: .NET, Node.js, Python).
2.  **Identidade (Azure Entra ID):** Atribuída ao App Service para que ele possa se autenticar nos outros recursos.
3.  **Segurança (Key Vault e SQL DB):** O App Service usa sua Identidade Gerenciada para obter a **string de conexão** do banco de dados a partir do Key Vault. O SQL Database usa a mesma Identidade Gerenciada para permitir o acesso.

## Roteiro de Deploy Detalhado

Este guia assume que você já possui o código de uma API funcional (ex: usando um *backend* de banco de dados).

### Fase 1: Preparação dos Recursos no Azure

#### 1\. Criar Recursos Base

  * Crie um **Azure SQL Database** (servidor e banco de dados).
  * Crie um **Azure Key Vault**.
  * Crie um **Azure App Service** (um Plano de Serviço e a instância do App Service).

#### 2\. Configuração de Segurança no Key Vault

  * No **Azure Key Vault**, armazene a *string de conexão* completa para o seu Azure SQL Database como um **Segredo** (Secret).
  * *Exemplo do Nome do Segredo:* `SqlConnectionString`

#### 3\. Configuração de Segurança no SQL Database

  * No **servidor SQL do Azure**, defina o **Administrador do Azure AD** (Azure Entra ID) para o seu próprio usuário ou um grupo de segurança.

### Fase 2: Configuração de Identidades Gerenciadas

Este é o passo crucial para eliminar senhas do seu código.

#### 1\. Ativar a Identidade no App Service

  * No **Azure App Service**, navegue até **"Identidade"** (Identity).
  * Ative a **Identidade Gerenciada Atribuída pelo Sistema** (*System-assigned managed identity*). Isso cria um *Service Principal* para o seu App Service no Azure Entra ID.

#### 2\. Conceder Acesso ao Key Vault

  * No **Azure Key Vault**, navegue até **"Políticas de Acesso"** (Access policies).
  * Adicione uma nova política:
      * **Permissões de Segredo (Secret permissions):** Marque **"Get"** (Obter) e **"List"** (Listar).
      * **Selecionar principal:** Procure e selecione o **nome do seu Azure App Service** (a Identidade Gerenciada que você acabou de criar).

#### 3\. Conceder Acesso ao SQL Database

  * No **servidor SQL do Azure**, use a ferramenta de gerenciamento (como SQL Server Management Studio ou Azure Data Studio) para conectar-se como o Administrador do Azure AD.
  * Crie um **usuário no banco de dados** a partir da Identidade Gerenciada do seu App Service (isso é feito com comandos T-SQL):
    ```sql
    CREATE USER [NomeDoAppService] FROM EXTERNAL PROVIDER;
    EXEC sp_addrolemember 'db_datareader', [NomeDoAppService];
    EXEC sp_addrolemember 'db_datawriter', [NomeDoAppService];
    ```

### Fase 3: Deploy do Código

#### 1\. Atualizar o Código da API

  * No seu projeto, altere a lógica de carregamento da *string de conexão* para que ela **leia do Azure Key Vault**, utilizando a biblioteca de SDK do Azure correspondente à sua linguagem (e.g., `Azure.Security.KeyVault.Secrets` no .NET).
  * A *string de conexão* no código será uma **referência** ao Key Vault, e não o valor em si.

#### 2\. Publicar

  * No Visual Studio, VS Code ou via pipeline de CI/CD (GitHub Actions / Azure DevOps), publique o código da API no **Azure App Service** configurado.

## Teste e Validação

Após o *deploy*, a API tentará:

1.  Usar sua **Identidade Gerenciada** para autenticar-se no **Azure Key Vault**.
2.  Obter o **Segredo** (`SqlConnectionString`).
3.  Usar a *string* para se conectar ao **Azure SQL Database**.
4.  O SQL DB valida a **Identidade Gerenciada** (que é o usuário no banco de dados) e concede acesso.
