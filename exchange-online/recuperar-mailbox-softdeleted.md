# Exchange Online: recuperação de mailbox após exclusão do usuário

## Cenário

Em um cenário de incidente de segurança, o ambiente de um cliente foi comprometido e uma conta de usuário foi excluída. Como consequência, a mailbox associada também foi removida do Exchange Online.

O objetivo do atendimento era validar se ainda existia uma mailbox recuperável e, caso positivo, restaurar o conteúdo para uma nova conta/mailbox.

## Ideia principal

Antes de considerar os dados como perdidos, é importante validar o estado da mailbox no Exchange Online.

Dependendo do cenário, a mailbox pode estar em um dos seguintes estados:

- **SoftDeletedMailbox**;
- **SoftDeletedMailUser**;
- **InactiveMailbox**, quando havia retenção, hold ou políticas de compliance aplicadas.

Se a mailbox ainda estiver recuperável, uma alternativa é criar uma nova mailbox de destino e restaurar o conteúdo usando o cmdlet `New-MailboxRestoreRequest`.

## Referências Microsoft

- New-MailboxRestoreRequest:  
  https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/new-mailboxrestorerequest

- Delete or restore user mailboxes in Exchange Online:  
  https://learn.microsoft.com/en-us/exchange/recipients-in-exchange-online/delete-or-restore-mailboxes

- Restore inactive mailbox:  
  https://learn.microsoft.com/en-us/purview/restore-an-inactive-mailbox

- Set-MailboxRegionalConfiguration:  
  https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/set-mailboxregionalconfiguration

---

## Pré-requisitos

Instale e conecte ao Exchange Online PowerShell:

```powershell
Set-ExecutionPolicy RemoteSigned -Force

Install-Module -Name ExchangeOnlineManagement -Force

Import-Module ExchangeOnlineManagement

Connect-ExchangeOnline
```

## 1. Validar se a mailbox ainda está recuperável

Execute os comandos abaixo para verificar se existe mailbox em estado SoftDeleted, MailUser em estado SoftDeleted ou mailbox inativa:

```powershell


Get-Mailbox -SoftDeletedMailbox

Get-MailUser -SoftDeletedMailUser

Get-Mailbox -InactiveMailboxOnly
````

Se a mailbox aparecer em um desses comandos, existe possibilidade de análise para recuperação ou restore, dependendo do estado do objeto e das políticas aplicadas.

---

## 2. Coletar o ExchangeGuid da mailbox SoftDeleted

Caso a mailbox apareça como SoftDeleted, colete o `ExchangeGuid`:

```powershell
Get-Mailbox -SoftDeletedMailbox | Select-Object Name,PrimarySmtpAddress,ExchangeGuid
```

Guarde o valor do `ExchangeGuid`, pois ele será usado como origem no processo de restore.

Exemplo:

```text
Name                 PrimarySmtpAddress        ExchangeGuid
----                 ------------------        ------------
Usuario Exemplo      usuario@dominio.com       xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

---

## 3. Criar uma nova conta/mailbox de destino

Crie uma nova conta para o usuário pelo portal:

```text
Microsoft 365 admin center > Users > Active users > Add a user
```

Depois:

1. atribua a licença adequada;
2. aguarde o provisionamento da nova mailbox;
3. valide se a nova mailbox foi criada;
4. confirme idioma, região e fuso horário.

Também é possível criar a conta via PowerShell.

Exemplo:

```powershell
New-MsolUser `
  -UserPrincipalName usuario@dominio.com `
  -DisplayName "Usuario Exemplo" `
  -UsageLocation BR `
  -Password "SenhaTemporaria" `
  -ForceChangePassword $false `
  -PasswordNeverExpires $true
```

Aplicar licença:

```powershell
Set-MsolUserLicense `
  -UserPrincipalName usuario@dominio.com `
  -AddLicenses "tenant:SKU_DA_LICENCA"
```

> Ajuste o SKU conforme a licença disponível no tenant.
> Exemplo: Business Basic, Business Standard, Exchange Online Plan 1, entre outras.

---

## 4. Configurar idioma, data, hora e fuso horário da nova mailbox

Após o provisionamento da mailbox, configure a regionalização:

```powershell
Set-MailboxRegionalConfiguration `
  -Identity usuario@dominio.com `
  -Language pt-BR `
  -DateFormat dd/MM/yyyy `
  -TimeZone "E. South America Standard Time" `
  -TimeFormat HH:mm `
  -LocalizeDefaultFolderName
```

Essa etapa ajuda a evitar que as pastas padrão sejam provisionadas com nomes em outro idioma, como `Inbox`, `Sent Items` e `Deleted Items`.

---

## 5. Coletar informações da nova mailbox

Colete o `ExchangeGuid` da nova mailbox:

```powershell
Get-Mailbox -Identity usuario@dominio.com | 
Format-List Name,ExchangeGuid,LegacyExchangeDN
```

Valide também a quantidade atual de itens da nova caixa antes do restore:

```powershell
Get-MailboxStatistics -Identity usuario@dominio.com | 
Select-Object DisplayName,ItemCount,TotalItemSize
```

Essa validação ajuda a comparar o antes e depois da restauração.

---

## 6. Restaurar o conteúdo da mailbox antiga para a nova

Com o `ExchangeGuid` da mailbox SoftDeleted e a nova mailbox de destino, execute o restore:

```powershell
New-MailboxRestoreRequest `
  -SourceMailbox "ExchangeGuid-da-mailbox-softdeleted" `
  -TargetMailbox "usuario@dominio.com" `
  -AllowLegacyDNMismatch
```

Também é possível usar o `ExchangeGuid` da mailbox nova como destino:

```powershell
New-MailboxRestoreRequest `
  -SourceMailbox "ExchangeGuid-da-mailbox-softdeleted" `
  -TargetMailbox "ExchangeGuid-da-mailbox-nova" `
  -AllowLegacyDNMismatch
```

O parâmetro `AllowLegacyDNMismatch` é utilizado quando a mailbox de origem e a mailbox de destino não possuem o mesmo `LegacyExchangeDN`.

---

## 7. Restaurar o archive, se existir

Caso a mailbox antiga tenha Online Archive e a nova conta também tenha archive habilitado, utilize:

```powershell
New-MailboxRestoreRequest `
  -SourceMailbox "ExchangeGuid-da-mailbox-softdeleted" `
  -SourceIsArchive `
  -TargetMailbox "usuario@dominio.com" `
  -TargetIsArchive `
  -AllowLegacyDNMismatch
```

Ou usando o `ExchangeGuid` da mailbox nova como destino:

```powershell
New-MailboxRestoreRequest `
  -SourceMailbox "ExchangeGuid-da-mailbox-softdeleted" `
  -SourceIsArchive `
  -TargetMailbox "ExchangeGuid-da-mailbox-nova" `
  -TargetIsArchive `
  -AllowLegacyDNMismatch
```

---

## 8. Acompanhar o processo de restore

Para acompanhar as solicitações de restore:

```powershell
Get-MailboxRestoreRequest
```

Para verificar estatísticas do processo:

```powershell
Get-MailboxRestoreRequest | Get-MailboxRestoreRequestStatistics
```

Se houver falha:

```powershell
Get-MailboxRestoreRequest -Status Failed | 
Get-MailboxRestoreRequestStatistics
```

Para relatório detalhado:

```powershell
Get-MailboxRestoreRequestStatistics `
  -Identity "MailboxRestoreRequestIdentity" `
  -IncludeReport
```

---

## 9. Validar o resultado final

Após a conclusão do restore, valide novamente a quantidade de itens e o tamanho da mailbox:

```powershell
Get-MailboxStatistics -Identity usuario@dominio.com | 
Select-Object DisplayName,ItemCount,TotalItemSize
```

Também é recomendado validar via Outlook Web se os dados foram restaurados corretamente.

Acesse:

```text
https://outlook.office.com
```

Valide:

* pastas restauradas;
* quantidade de itens;
* mensagens antigas;
* calendário;
* contatos;
* itens enviados;
* archive, se aplicável.

---

## Pontos de atenção

* Não exclua definitivamente objetos antes de validar as opções de recuperação.
* Se a conta foi excluída recentemente, avalie primeiro a restauração do usuário pelo Microsoft 365 admin center ou pelo Microsoft Entra admin center.
* Se a mailbox estiver inativa por retenção, hold ou compliance, o procedimento pode ser diferente.
* O restore copia o conteúdo da mailbox antiga para a nova mailbox de destino.
* O processo não “reconecta” necessariamente a mailbox antiga ao usuário original.
* Em cenários de incidente de segurança, valide também:

  * regras de caixa de entrada;
  * permissões delegadas;
  * encaminhamentos externos;
  * aplicativos consentidos;
  * logs de auditoria;
  * alterações administrativas suspeitas;
  * MFA e métodos de autenticação do usuário.

---

## Resumo

Neste tipo de cenário, o ponto mais importante é não agir no escuro.

Antes de recriar usuários, remover objetos ou considerar a perda definitiva dos dados, valide o estado da mailbox no Exchange Online.

Se a mailbox ainda estiver em estado recuperável, o `New-MailboxRestoreRequest` pode ser usado para copiar o conteúdo da mailbox antiga para uma nova mailbox ativa.

```
```
