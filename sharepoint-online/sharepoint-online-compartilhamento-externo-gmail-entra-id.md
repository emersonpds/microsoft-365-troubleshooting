# SharePoint Online — Compartilhamento externo falhando com usuários Gmail ou convidados externos

## Cenário

Em um cenário de suporte envolvendo SharePoint Online e OneDrive for Business, alguns usuários internos conseguiam compartilhar arquivos e pastas com usuários externos normalmente, enquanto outros enfrentavam falhas ao compartilhar com contas externas, como contas Gmail ou usuários convidados.

O comportamento observado era semelhante a este:

* O usuário externo recebia o convite de compartilhamento.
* Ao tentar acessar o link, era direcionado para autenticação ou validação por código.
* Em alguns casos, o acesso entrava em loop de solicitação de permissão.
* Em outros casos, o usuário externo não conseguia concluir o acesso mesmo após solicitar permissão novamente.
* O problema não ocorria de forma igual para todos os usuários ou todos os sites.

À primeira vista, o compartilhamento externo parecia estar habilitado no SharePoint Admin Center, mas o problema continuava acontecendo para determinados usuários externos.

> Todos os dados deste artigo foram anonimizados. Nomes de clientes, usuários, domínios, tenants e demais identificadores reais foram substituídos por exemplos genéricos.

---

## Ideia principal

Quando falamos de compartilhamento externo no SharePoint Online e OneDrive, não basta olhar apenas uma configuração.

O acesso externo pode ser impactado por diferentes camadas:

* Configuração de compartilhamento externo no nível da organização.
* Configuração de compartilhamento externo no nível do site.
* Configurações de colaboração externa no Microsoft Entra ID.
* Restrições de domínio permitidos ou bloqueados.
* Estado do usuário convidado no Entra ID.
* Tipo de link utilizado no compartilhamento.
* Integração do SharePoint e OneDrive com Microsoft Entra B2B.
* Autenticação por código de acesso único, também conhecido como OTP.

Um ponto importante é que, se a configuração do site for mais restritiva do que a configuração da organização, a configuração mais restritiva será aplicada.

Além disso, se as configurações de colaboração externa do Microsoft Entra estiverem bloqueando convites ou restringindo determinados domínios, o compartilhamento também pode falhar mesmo que o SharePoint pareça estar liberado.

---

## Aviso de atenção

Antes de alterar configurações globais de compartilhamento externo, valide o impacto de segurança.

Evite liberar compartilhamento externo de forma ampla apenas para resolver rapidamente o problema.

O ideal é identificar exatamente onde está o bloqueio:

* Organização.
* Site específico.
* Entra ID.
* Domínio externo.
* Usuário convidado antigo.
* Tipo de link.
* Política de segurança ou governança.

Alterações em compartilhamento externo podem afetar todos os sites do tenant, dependendo do local onde forem aplicadas.

---

## Referências Microsoft

* [Overview of external sharing in SharePoint and OneDrive in Microsoft 365](https://learn.microsoft.com/pt-br/sharepoint/external-sharing-overview)
* [Manage sharing settings for SharePoint and OneDrive in Microsoft 365](https://learn.microsoft.com/pt-br/sharepoint/turn-external-sharing-on-or-off)
* [Change the sharing settings for a site](https://learn.microsoft.com/en-us/sharepoint/change-external-sharing-site)
* [Microsoft Entra B2B integration for SharePoint and OneDrive](https://learn.microsoft.com/en-us/sharepoint/sharepoint-azureb2b-integration)
* [Configure external collaboration settings](https://learn.microsoft.com/en-us/entra/external-id/external-collaboration-settings-configure)
* [Email one-time passcode authentication](https://learn.microsoft.com/en-us/entra/external-id/one-time-passcode)
* [Sharing errors in SharePoint and OneDrive](https://learn.microsoft.com/en-us/sharepoint/sharepoint-onedrive-error-message)

---

## Pré-requisitos

Para seguir com a análise, é recomendado ter:

* Acesso ao SharePoint Admin Center.
* Acesso ao Microsoft Entra admin center.
* Permissão de administrador do SharePoint ou Global Admin.
* PowerShell do SharePoint Online instalado, se for validar por linha de comando.
* URL do site afetado.
* E-mail externo utilizado no teste.
* Evidência do erro apresentado ao usuário externo.
* Informação se o problema ocorre com todos os sites ou apenas com um site específico.

---

## Passo a passo técnico

### 1. Confirmar o comportamento com o usuário externo

Antes de alterar qualquer configuração, valide exatamente o que acontece.

Colete as seguintes informações:

* Qual usuário interno realizou o compartilhamento?
* Qual site, biblioteca, pasta ou arquivo foi compartilhado?
* Qual e-mail externo recebeu o convite?
* O usuário externo é Gmail, Outlook.com, conta corporativa ou outro provedor?
* O erro ocorre no primeiro acesso ou após solicitar permissão?
* O problema ocorre em aba anônima/InPrivate?
* O usuário externo já existia como convidado no tenant?

Exemplo de dados anonimizados:

```text
Usuário interno: usuario.interno@dominio.com
Usuário externo: usuario.externo@gmail.com
Site: https://contoso.sharepoint.com/sites/projeto
Tipo de compartilhamento: Pessoas específicas
Comportamento: usuário externo solicita acesso em loop
```

---

### 2. Validar compartilhamento externo no nível da organização

No SharePoint Admin Center, acesse:

```text
SharePoint admin center > Policies > Sharing
```

Valide se o compartilhamento externo está habilitado no nível da organização.

As opções podem variar conforme o tenant, mas normalmente seguem esta lógica:

* Anyone
* New and existing guests
* Existing guests only
* Only people in your organization

Se a organização estiver configurada como `Only people in your organization`, o compartilhamento externo será bloqueado.

Também é possível validar via PowerShell:

```powershell
Connect-SPOService -Url https://contoso-admin.sharepoint.com
```

Depois:

```powershell
Get-SPOTenant | Select-Object `
SharingCapability, `
SharingDomainRestrictionMode, `
SharingAllowedDomainList, `
SharingBlockedDomainList, `
DefaultSharingLinkType
```

Pontos para observar:

* `SharingCapability`
* `SharingDomainRestrictionMode`
* `SharingAllowedDomainList`
* `SharingBlockedDomainList`
* Tipo padrão de link de compartilhamento

Se houver allow list ou block list de domínios, valide se o domínio externo está permitido.

Exemplo:

```text
gmail.com
outlook.com
dominio-parceiro.com
```

---

### 3. Validar compartilhamento externo no nível do site

Mesmo que o compartilhamento externo esteja liberado na organização, o site pode estar mais restritivo.

Valide o site afetado:

```powershell
Get-SPOSite -Identity https://contoso.sharepoint.com/sites/projeto |
Select-Object Url, SharingCapability, SharingDomainRestrictionMode, SharingAllowedDomainList, SharingBlockedDomainList
```

Se o site estiver com compartilhamento desabilitado ou mais restritivo, o compartilhamento pode falhar apenas naquele site.

Exemplo para habilitar compartilhamento somente com convidados autenticados:

```powershell
Set-SPOSite `
-Identity https://contoso.sharepoint.com/sites/projeto `
-SharingCapability ExternalUserSharingOnly
```

> Use a opção adequada para a política da organização. Não libere links anônimos se o objetivo for permitir apenas convidados autenticados.

---

### 4. Validar restrições de domínio no SharePoint

Verifique se existe restrição de domínio configurada.

Exemplo:

```powershell
Get-SPOTenant | Select-Object `
SharingDomainRestrictionMode, `
SharingAllowedDomainList, `
SharingBlockedDomainList
```

Se o modo estiver como `AllowList`, apenas os domínios listados poderão receber compartilhamento externo.

Se o modo estiver como `BlockList`, os domínios listados estarão bloqueados.

Exemplo de configuração com allow list:

```powershell
Set-SPOTenant `
-SharingDomainRestrictionMode AllowList `
-SharingAllowedDomainList "dominio-parceiro.com"
```

Exemplo de configuração com block list:

```powershell
Set-SPOTenant `
-SharingDomainRestrictionMode BlockList `
-SharingBlockedDomainList "dominio-bloqueado.com"
```

Ponto de atenção: se o domínio externo for `gmail.com` e a organização usa allow list apenas com domínios corporativos, o compartilhamento com Gmail pode ser bloqueado por design.

---

### 5. Validar configurações de colaboração externa no Microsoft Entra ID

Acesse:

```text
Microsoft Entra admin center > External Identities > External collaboration settings
```

Valide principalmente:

* Quem pode convidar usuários convidados.
* Se usuários membros podem convidar convidados.
* Se convidados podem convidar outros convidados.
* Se há restrições de colaboração por domínio.
* Se o domínio externo está bloqueado.
* Se o tenant permite colaboração com o tipo de usuário externo necessário.

Esse ponto é importante porque o SharePoint e o OneDrive podem depender das configurações do Microsoft Entra para criação e autenticação de convidados externos.

Se o Entra ID estiver restringindo convites, o SharePoint pode até permitir o início do compartilhamento, mas o usuário externo pode não conseguir concluir o acesso.

---

### 6. Validar autenticação por código de acesso único

Para usuários externos que não possuem conta corporativa Microsoft ou conta Microsoft compatível, o acesso pode depender do recurso de código de acesso único por e-mail, também chamado de one-time passcode ou OTP.

Acesse:

```text
Microsoft Entra admin center > External Identities > All identity providers
```

Valide se o método de autenticação por e-mail one-time passcode está habilitado conforme a política da organização.

Esse recurso permite que determinados usuários convidados autentiquem usando um código enviado ao e-mail, quando não for possível autenticar por outros métodos.

---

### 7. Validar se o usuário convidado já existe no tenant

Em alguns casos, o usuário externo já existe como convidado no Microsoft Entra ID, mas está em estado inconsistente, antigo ou associado a um convite anterior.

Acesse:

```text
Microsoft Entra admin center > Users > All users
```

Pesquise pelo e-mail externo.

Exemplo:

```text
usuario.externo@gmail.com
```

Valide:

* Se o usuário aparece como Guest.
* Se o UPN possui formato `#EXT#`.
* Se o e-mail está correto.
* Se o usuário está bloqueado.
* Se há mais de um objeto para o mesmo convidado.
* Se o convidado pertence ao tenant correto.

Também é possível validar com Microsoft Graph PowerShell:

```powershell
Connect-MgGraph -Scopes "User.Read.All"
```

Depois:

```powershell
Get-MgUser -All -Property DisplayName,UserPrincipalName,UserType,Mail,AccountEnabled |
Where-Object {
    $_.Mail -eq "usuario.externo@gmail.com" -or
    $_.UserPrincipalName -like "*usuario*externo*#EXT#*"
} |
Select-Object DisplayName, UserPrincipalName, UserType, Mail, AccountEnabled
```

Se houver um convidado antigo ou inconsistente, pode ser necessário remover o acesso anterior e reenviar o convite, mas essa ação deve ser avaliada com cuidado para não impactar permissões válidas em outros sites.

---

### 8. Validar o tipo de link usado no compartilhamento

Para troubleshooting, prefira testar com o tipo de link:

```text
Specific people / Pessoas específicas
```

Esse tipo de link ajuda a garantir que o acesso está vinculado ao usuário externo informado no convite.

Evite iniciar o teste com links do tipo:

```text
Anyone / Qualquer pessoa
```

Links anônimos podem mascarar o problema real de autenticação do convidado.

Também valide se o usuário interno está compartilhando diretamente com o e-mail externo correto.

---

### 9. Remover e refazer o compartilhamento

Após ajustar as configurações necessárias, remova o compartilhamento anterior e compartilhe novamente.

No arquivo ou pasta afetada:

```text
Manage access > Stop sharing
```

Depois, compartilhe novamente usando:

```text
Specific people / Pessoas específicas
```

Solicite ao usuário externo que teste em:

* Aba anônima/InPrivate.
* Outro navegador.
* Sem sessão anterior conectada.
* Usando exatamente o e-mail que recebeu o convite.

---

## Validação do resultado

Após os ajustes, valide:

* O usuário externo recebe novo convite.
* O link abre corretamente.
* O usuário consegue autenticar.
* O código OTP, se utilizado, é recebido e aceito.
* O acesso não entra mais em loop de solicitação.
* O usuário externo consegue abrir o arquivo, pasta ou site.
* O usuário aparece corretamente como convidado no Entra ID, quando aplicável.
* O acesso respeita o nível de permissão esperado.

Também valide se o compartilhamento continua restrito ao escopo necessário.

---

## Pontos de atenção

* O compartilhamento externo pode estar liberado na organização, mas bloqueado no site.
* A configuração mais restritiva prevalece.
* Restrições de domínio podem bloquear Gmail, Outlook.com ou domínios parceiros.
* Configurações do Microsoft Entra ID podem impedir a criação ou autenticação de convidados.
* Usuários convidados antigos podem causar comportamento inconsistente.
* O tipo de link usado no compartilhamento muda o fluxo de autenticação.
* Não é recomendado liberar links anônimos apenas para contornar erro de convidado.
* Sempre valide o impacto de segurança antes de alterar políticas globais.

---

## Resumo

Neste cenário, o problema de compartilhamento externo no SharePoint Online não estava relacionado apenas ao site ou ao arquivo compartilhado.

A investigação mostrou que o acesso externo precisa ser validado em camadas:

1. Compartilhamento externo no tenant.
2. Compartilhamento externo no site.
3. Restrições de domínio.
4. Configurações de colaboração externa no Microsoft Entra ID.
5. Autenticação por OTP.
6. Estado do usuário convidado.
7. Tipo de link utilizado.

A principal lição é: quando o usuário externo não consegue acessar um compartilhamento do SharePoint ou OneDrive, mesmo com o compartilhamento aparentemente habilitado, é necessário validar também a camada de identidade no Microsoft Entra ID.

Nem sempre o problema está no SharePoint. Muitas vezes, o bloqueio está na política de convidados, no domínio externo ou no objeto Guest criado anteriormente.
