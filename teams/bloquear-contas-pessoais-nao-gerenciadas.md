# Microsoft Teams: como bloquear ou limitar contato com contas pessoais e não gerenciadas

> Cenário real de suporte Microsoft 365, com dados anonimizados, focado em troubleshooting prático.

## Objetivo

Este artigo mostra como mitigar cenários em que usuários internos recebem contatos de contas externas, pessoais ou não gerenciadas no Microsoft Teams.

A configuração é feita principalmente pelo **Teams Admin Center**, com alternativa via PowerShell para validação e automação.

## Cenário

Uma organização identificou contatos suspeitos ou indesejados chegando pelo Microsoft Teams. A suspeita é que as mensagens estejam vindo de contas pessoais ou contas Teams não gerenciadas por uma organização.

Exemplo de cenário:

```text
Sintoma: usuários internos recebem mensagens de contas externas/pessoais no Teams
Risco percebido: tentativa de phishing, impersonation ou contato indevido
Necessidade: impedir ou reduzir contato iniciado por contas não gerenciadas
```

## Conceitos importantes

## Conceitos importantes

No Microsoft Teams, existem cenários diferentes de colaboração externa:

| Recurso | Descrição |
|---|---|
| External access | Permite comunicação com usuários externos ao tenant, incluindo outras organizações Microsoft 365 e, dependendo da configuração, contas não gerenciadas. |
| Guest access | Permite adicionar usuários externos como convidados dentro do tenant para colaboração em equipes, canais, arquivos e reuniões. |
| Unmanaged Teams accounts | Contas Teams não gerenciadas por uma organização, como contas pessoais/Microsoft Teams gratuito. |
| External domains | Domínios externos que podem ser permitidos ou bloqueados para comunicação federada. |

Para mitigar contato indevido, normalmente o foco está em **External access**, não em Guest access.

## Opção 1 — Bloquear comunicação com contas Teams não gerenciadas pelo portal

Essa é a abordagem mais direta quando a organização não precisa se comunicar com contas pessoais ou não gerenciadas.

### Passo a passo no Teams Admin Center

1. Acesse o **Teams Admin Center**.
2. Vá em **Users** > **External access**.
3. Localize a configuração relacionada a contas Teams não gerenciadas.
4. Desative a opção:

```text
People in my organization can communicate with unmanaged Teams accounts
```

5. Salve a alteração.

Com isso, os usuários da organização deixam de se comunicar com contas Teams não gerenciadas.

## Opção 2 — Permitir comunicação, mas impedir que contas não gerenciadas iniciem contato

Em alguns cenários, a empresa pode querer permitir que usuários internos iniciem contato com contas pessoais, mas não quer permitir que contas externas encontrem e iniciem conversas com os usuários internos.

### Passo a passo no Teams Admin Center

1. Acesse o **Teams Admin Center**.
2. Vá em **Users** > **External access**.
3. Mantenha habilitada a comunicação com contas Teams não gerenciadas, se essa for a necessidade do negócio.
4. Desmarque a opção:

```text
External users with Teams accounts not managed by an organization can contact users in my organization
```

5. Salve a alteração.

Resultado esperado:

- usuários internos ainda podem iniciar a comunicação, se permitido;
- contas não gerenciadas não conseguem pesquisar/localizar usuários internos por e-mail para iniciar contato.

## Opção 3 — Bloquear ou permitir domínios externos específicos

Quando o problema envolve organizações externas específicas, é possível restringir o External access por domínio.

### Permitir apenas domínios específicos

1. Acesse o **Teams Admin Center**.
2. Vá em **Users** > **External access**.
3. Em comunicação com organizações externas, selecione a opção para permitir apenas domínios específicos.
4. Adicione os domínios autorizados.
5. Salve a configuração.

### Bloquear domínios específicos

1. Acesse o **Teams Admin Center**.
2. Vá em **Users** > **External access**.
3. Selecione a opção para bloquear domínios específicos.
4. Adicione os domínios que devem ser bloqueados.
5. Salve a configuração.

> Atenção: bloquear domínios externos não é a mesma coisa que bloquear contas Teams pessoais/não gerenciadas. São controles diferentes dentro de External access.

## Opção 4 — Bloquear usuários externos específicos

Quando a ocorrência envolve um usuário externo específico, a organização pode usar a lista de bloqueio de usuários externos.

### Passo a passo no Teams Admin Center

1. Acesse o **Teams Admin Center**.
2. Vá em **Users** > **External access**.
3. Habilite a opção para bloquear usuários externos específicos.
4. Adicione o endereço do usuário externo a ser bloqueado.
5. Salve a configuração.

Essa abordagem é útil quando o problema é pontual, mas não substitui uma política mais ampla quando o risco envolve várias contas externas.

## Validação via PowerShell

Além do portal, é possível validar as configurações com o módulo do Microsoft Teams PowerShell.

### 1. Conectar ao Microsoft Teams PowerShell

```powershell
Connect-MicrosoftTeams
```

### 2. Validar configuração atual do tenant

```powershell
Get-CsTenantFederationConfiguration | Select-Object `
  AllowFederatedUsers,
  AllowedDomains,
  BlockedDomains,
  AllowTeamsConsumer,
  AllowTeamsConsumerInbound,
  AllowPublicUsers
```

Campos importantes:

| Campo | Significado |
|---|---|
| `AllowFederatedUsers` | Controla comunicação federada com outras organizações. |
| `AllowedDomains` | Domínios explicitamente permitidos. |
| `BlockedDomains` | Domínios explicitamente bloqueados. |
| `AllowTeamsConsumer` | Controla comunicação com contas Teams não gerenciadas. |
| `AllowTeamsConsumerInbound` | Controla se contas não gerenciadas podem iniciar contato com usuários internos. |
| `AllowPublicUsers` | Configuração relacionada a usuários públicos/Skype, quando aplicável. |

### 3. Bloquear comunicação com contas Teams não gerenciadas

```powershell
Set-CsTenantFederationConfiguration `
  -AllowTeamsConsumer $false `
  -AllowTeamsConsumerInbound $false
```

### 4. Permitir que internos iniciem contato, mas bloquear entrada iniciada por contas não gerenciadas

```powershell
Set-CsTenantFederationConfiguration `
  -AllowTeamsConsumer $true `
  -AllowTeamsConsumerInbound $false
```

## Exemplo de política por usuário

Quando a organização não quer aplicar a mesma regra para todos, pode trabalhar com políticas de External access.

Exemplo criando uma política que bloqueia acesso a contas Teams não gerenciadas:

```powershell
New-CsExternalAccessPolicy `
  -Identity "BloquearTeamsPessoal" `
  -EnableTeamsConsumerAccess $false `
  -EnableTeamsConsumerInbound $false
```

Atribuir a política a um usuário:

```powershell
Grant-CsExternalAccessPolicy `
  -Identity usuario@empresa.com.br `
  -PolicyName "BloquearTeamsPessoal"
```

## Pontos de atenção

1. **External access e Guest access são recursos diferentes.**  
   Bloquear contas pessoais/não gerenciadas em External access não necessariamente bloqueia convidados já adicionados no tenant.

2. **Contas externas de organizações Microsoft 365 são diferentes de contas pessoais.**  
   Uma empresa pode bloquear contas pessoais e ainda permitir comunicação com domínios corporativos confiáveis.

3. **Reuniões podem ter regras separadas.**  
   Dependendo das políticas de reunião, participantes externos ou anônimos podem ter comportamentos diferentes em reuniões.

4. **A configuração deve seguir a necessidade do negócio.**  
   Se a organização não tem necessidade de conversar com contas pessoais, a recomendação mais segura é desabilitar esse tipo de comunicação.

## Recomendação prática

Para um cenário de mitigação de phishing ou contato indevido por contas pessoais/não gerenciadas, a configuração mais conservadora é:

```text
People in my organization can communicate with unmanaged Teams accounts = Off
External users with Teams accounts not managed by an organization can contact users in my organization = Off
```

Caso o negócio precise permitir algum nível de comunicação, uma alternativa intermediária é:

```text
People in my organization can communicate with unmanaged Teams accounts = On
External users with Teams accounts not managed by an organization can contact users in my organization = Off
```

Assim, o contato só ocorre quando iniciado pelos usuários internos.


## Referências oficiais

- Microsoft Learn — Manage external meetings and chat with people and organizations using Microsoft identities  
  https://learn.microsoft.com/en-us/microsoftteams/trusted-organizations-external-meetings-chat

- Microsoft Learn — Use guest access and external access to collaborate with people outside your organization  
  https://learn.microsoft.com/en-us/microsoftteams/communicate-with-users-from-other-organizations

- Microsoft Learn — New-CsExternalAccessPolicy  
  https://learn.microsoft.com/en-us/powershell/module/skypeforbusiness/new-csexternalaccesspolicy

