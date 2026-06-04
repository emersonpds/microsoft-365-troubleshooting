# Mensagens legítimas caindo no Lixo Eletrônico/Spam em destinatários Microsoft

## Cenário

Em alguns cenários, mensagens legítimas enviadas por um domínio corporativo acabam sendo entregues na pasta **Lixo Eletrônico/Spam** de destinatários que utilizam Microsoft 365, Outlook.com ou ambientes protegidos pelo Exchange Online Protection.

Um ponto importante nesse tipo de análise é entender que a mensagem pode estar tecnicamente autenticada, com **SPF**, **DKIM** e **DMARC** aprovados, e ainda assim ser classificada como spam, phishing, bulk mail ou mensagem suspeita.

Isso pode ocorrer por outros sinais analisados pela proteção da Microsoft, como reputação do remetente, histórico de envio, conteúdo da mensagem, URLs, anexos, volume de envio, padrão de comportamento, políticas do tenant destinatário ou configurações específicas do usuário.

## Ideia principal

Quando uma mensagem legítima cai no Lixo Eletrônico de um destinatário Microsoft, o troubleshooting deve separar três pontos principais:

1. O envio foi aceito e entregue pelo lado do remetente?
2. A autenticação SPF, DKIM e DMARC está correta?
3. A classificação como spam ocorreu no tenant destinatário, no Outlook do usuário ou por alguma política/override específico?

Nem sempre o remetente consegue identificar a causa completa sozinho. Quando o destinatário está em outro tenant Microsoft, a análise mais precisa depende do cabeçalho completo da mensagem recebida ou de uma validação feita pelo administrador do tenant destinatário.

## Aviso de atenção

Não é recomendado resolver esse tipo de cenário apenas adicionando domínios ou remetentes em whitelist ampla e permanente.

A liberação de remetentes, domínios ou URLs deve ser feita com cautela, preferencialmente após análise do cabeçalho, validação da origem do bloqueio e confirmação de que a mensagem é realmente legítima.

Em casos de falso positivo, o caminho mais adequado é submeter a mensagem para análise da Microsoft e, se necessário, criar uma permissão temporária enquanto a análise é realizada.

## Referências Microsoft

- [Anti-spam message headers in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/message-headers-eop-mdo)
- [Email authentication in Microsoft 365](https://learn.microsoft.com/en-us/defender-office-365/email-authentication-about)
- [Resolve false positives for legitimate blocked emails](https://learn.microsoft.com/en-us/defender-office-365/step-by-step-guides/how-to-handle-false-positives-in-microsoft-defender-for-office-365)
- [Manage allows and blocks in the Tenant Allow/Block List](https://learn.microsoft.com/en-us/defender-office-365/tenant-allow-block-list-about)
- [Message trace in the modern Exchange admin center](https://learn.microsoft.com/en-us/exchange/monitoring/trace-an-email-message/message-trace-modern-eac)
- [Get-MessageTraceV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracev2)
- [Get-MessageTraceDetailV2](https://learn.microsoft.com/en-us/powershell/module/exchangepowershell/get-messagetracedetailv2)

## Pré-requisitos

Antes de iniciar a análise, é importante coletar as seguintes informações:

- Remetente afetado: `usuario@dominio.com`
- Destinatário afetado: `destinatario@dominioexterno.com`
- Data e horário aproximado do envio
- Assunto da mensagem
- Cabeçalho completo da mensagem recebida pelo destinatário
- Evidência mostrando a mensagem na pasta Lixo Eletrônico/Spam
- Confirmação se o problema ocorre com:
  - apenas um destinatário;
  - vários destinatários do mesmo domínio;
  - vários domínios Microsoft diferentes;
  - mensagens específicas;
  - qualquer mensagem enviada pelo domínio.

## Passo a passo técnico

## 1. Confirmar se a mensagem saiu corretamente do tenant remetente

No Exchange Admin Center:

1. Acesse o **Exchange admin center**.
2. Vá em **Mail flow**.
3. Acesse **Message trace**.
4. Filtre por:
   - remetente;
   - destinatário;
   - período do envio;
   - assunto, se necessário.
5. Verifique o status da mensagem.

Se a mensagem aparece como **Delivered**, isso indica que ela foi processada pelo lado do remetente.

Porém, isso não confirma necessariamente que ela chegou na Caixa de Entrada do destinatário, principalmente quando o destinatário está em outro tenant. A mensagem pode ter sido entregue no ambiente de destino e, depois disso, classificada ou movida para Lixo Eletrônico/Spam por política, detecção ou configuração local.

## 2. Validar o envio pelo Exchange Online PowerShell

Conecte no Exchange Online:

```powershell
Connect-ExchangeOnline
```

Pesquise a mensagem pelo remetente e destinatário:

```powershell
$Sender = "usuario@dominio.com"
$Recipient = "destinatario@dominioexterno.com"
$StartDate = "06/04/2026 00:00:00"
$EndDate = "06/04/2026 23:59:59"

Get-MessageTraceV2 `
  -SenderAddress $Sender `
  -RecipientAddress $Recipient `
  -StartDate $StartDate `
  -EndDate $EndDate |
Format-List
```

Para obter os detalhes do evento:

```powershell
Get-MessageTraceV2 `
  -SenderAddress $Sender `
  -RecipientAddress $Recipient `
  -StartDate $StartDate `
  -EndDate $EndDate |
Get-MessageTraceDetailV2 |
Format-List
```

Também é possível consultar usando o `MessageTraceId`, quando disponível:

```powershell
$MessageTraceId = "00000000-0000-0000-0000-000000000000"

Get-MessageTraceV2 `
  -MessageTraceId $MessageTraceId `
  -StartDate $StartDate `
  -EndDate $EndDate |
Format-List
```

E consultar os detalhes da mensagem com o destinatário:

```powershell
Get-MessageTraceDetailV2 `
  -MessageTraceId $MessageTraceId `
  -RecipientAddress $Recipient |
Format-List
```

## 3. Coletar o cabeçalho completo da mensagem

O cabeçalho completo é uma das evidências mais importantes nesse tipo de cenário.

No **Outlook Web**:

1. Abra a mensagem.
2. Clique em **Mais ações**.
3. Selecione **Exibir origem da mensagem** ou opção equivalente.
4. Copie todo o cabeçalho.

No **Outlook Desktop**:

1. Abra a mensagem em uma nova janela.
2. Vá em **Arquivo**.
3. Acesse **Propriedades**.
4. Copie o conteúdo de **Cabeçalhos de Internet**.

## 4. Analisar os principais campos do cabeçalho

No cabeçalho, valide principalmente os seguintes campos:

```text
Authentication-Results
X-Forefront-Antispam-Report
X-Microsoft-Antispam
X-MS-Exchange-Organization-SCL
X-MS-Exchange-Organization-Network-Message-Id
X-MS-Office365-Filtering-Correlation-Id
```

Esses campos ajudam a entender como a mensagem foi autenticada, processada e classificada pela proteção da Microsoft.

## 5. Validar SPF, DKIM e DMARC

Procure no cabeçalho pelo campo `Authentication-Results`.

Exemplo esperado:

```text
spf=pass
dkim=pass
dmarc=pass
```

Quando os três resultados aparecem como `pass`, isso indica que a autenticação básica do domínio está correta.

Porém, isso não significa que a mensagem obrigatoriamente será entregue na Caixa de Entrada. A Microsoft também considera outros sinais, como reputação, comportamento de envio, conteúdo da mensagem, URLs, anexos, inteligência contra phishing e políticas configuradas no tenant destinatário.

## 6. Analisar o SCL

O `SCL`, ou **Spam Confidence Level**, indica a probabilidade de a mensagem ser spam.

Exemplos:

```text
SCL:-1
SCL:1
SCL:5
SCL:8
SCL:9
```

Interpretação prática:

- `SCL:-1`: a mensagem ignorou a filtragem de spam por regra, allow ou bypass.
- `SCL:0` ou `SCL:1`: baixa probabilidade de spam.
- `SCL:5` ou `SCL:6`: probabilidade média de spam, podendo ser entregue no Lixo Eletrônico.
- `SCL:7`, `SCL:8` ou `SCL:9`: alta probabilidade de spam, podendo ir para Lixo Eletrônico ou Quarentena, dependendo da política aplicada.

## 7. Analisar o BCL

O `BCL`, ou **Bulk Complaint Level**, é usado para classificar mensagens em massa ou mensagens com características de bulk mail.

Exemplo:

```text
BCL:7
```

Quanto maior o BCL, maior a chance de a mensagem ser tratada como bulk/spam, dependendo da configuração da política antispam do destinatário.

Esse ponto é comum em mensagens legítimas enviadas por plataformas de automação, sistemas de disparo, ERPs, CRMs, ferramentas de assinatura eletrônica ou campanhas comerciais.

## 8. Analisar a categoria aplicada à mensagem

No campo `X-Forefront-Antispam-Report`, procure por `CAT`.

Exemplos comuns:

```text
CAT:SPM
CAT:HSPM
CAT:PHISH
CAT:HPHISH
CAT:BULK
CAT:SPOOF
```

Interpretação prática:

- `SPM`: spam.
- `HSPM`: high confidence spam.
- `PHISH`: phishing.
- `HPHISH`: high confidence phishing.
- `BULK`: mensagem classificada como bulk.
- `SPOOF`: possível spoofing.

Quando a mensagem é classificada como `HPHISH`, a análise deve ser mais cuidadosa, pois a detecção pode estar relacionada a sinais avançados de phishing, reputação de URL, impersonation, spoofing ou outros indicadores de ameaça.

## 9. Validar se existe política ou override no tenant destinatário

Quando o destinatário também utiliza Microsoft 365, o administrador do tenant destinatário deve validar:

- Anti-spam policies;
- Anti-phishing policies;
- Safe Links;
- Safe Attachments;
- Tenant Allow/Block List;
- regras de transporte;
- lista de remetentes bloqueados do usuário;
- regras de caixa de correio;
- configuração de Junk Email no Outlook;
- Quarantine;
- Explorer ou Real-time detections, se houver Microsoft Defender for Office 365.

Esse ponto é importante porque o remetente normalmente não possui visibilidade das políticas aplicadas no ambiente destinatário.

## 10. Verificar se a mensagem foi movida após a entrega

Nem toda mensagem que aparece no Lixo Eletrônico foi necessariamente entregue diretamente nessa pasta.

Ela pode ter sido movida após a entrega por:

- regra de caixa de correio do usuário;
- bloqueio manual do remetente no Outlook;
- política aplicada após entrega;
- ação de segurança posterior;
- cliente de e-mail;
- add-in de segurança de terceiros;
- regra de transporte;
- configuração local do mailbox.

Por isso, é importante validar se o cabeçalho indica uma classificação direta como spam ou se houve algum outro mecanismo alterando a localização da mensagem depois da entrega.

## 11. Validar reputação e padrão de envio do remetente

Mesmo com SPF, DKIM e DMARC aprovados, alguns fatores podem impactar a entrega:

- envio em massa para muitos destinatários;
- mensagens muito parecidas enviadas em curto período;
- links encurtados;
- URLs com baixa reputação;
- anexos suspeitos;
- conteúdo com características de campanha comercial;
- domínio novo ou com pouco histórico;
- reclamações anteriores de usuários;
- automações enviando e-mails em grande volume;
- conta comprometida enviando mensagens fora do padrão;
- assinatura de e-mail com imagens, links ou redirecionamentos.

Uma validação útil é enviar mensagens de teste em etapas:

1. Mensagem simples, sem assinatura, imagem, link ou anexo.
2. Mensagem com assinatura corporativa.
3. Mensagem com link.
4. Mensagem com anexo.
5. Mensagem com o mesmo conteúdo da mensagem original.

Essa comparação ajuda a identificar se a classificação está relacionada ao domínio, ao conteúdo, à assinatura, ao link, ao anexo ou ao padrão de envio.

## 12. Submeter falso positivo para a Microsoft

Quando a mensagem é legítima, mas foi classificada como spam, phishing ou high confidence phishing, o caminho recomendado é submeter a mensagem para análise.

No Microsoft Defender portal:

1. Acesse **Submissions**.
2. Selecione **Email**.
3. Envie a mensagem como falso positivo.
4. Confirme que a mensagem foi validada como legítima.
5. Aguarde o resultado da análise.
6. Se necessário, aplique uma permissão temporária enquanto a análise é feita.

Essa abordagem é melhor do que criar uma liberação ampla e permanente sem identificar a causa da classificação.

## 13. Quando usar Tenant Allow/Block List

A **Tenant Allow/Block List** pode ser usada para permitir temporariamente um remetente, domínio, URL ou spoofed sender, dependendo do tipo de detecção.

Porém, é importante validar o tipo de bloqueio antes de criar uma entrada de allow.

Exemplos:

- Se a mensagem foi classificada como spam ou bulk, pode ser possível criar uma entrada de allow.
- Se foi detectada como spoofing, pode ser necessário tratar via submissão ou spoof intelligence.
- Se foi detectada como impersonation, o ajuste pode estar relacionado à política antiphishing.
- Se foi classificada como high confidence phishing, podem existir limitações para liberação direta, dependendo da detecção aplicada.

Também é importante lembrar que entradas de block podem ter precedência sobre entradas de allow. Portanto, antes de criar uma liberação, valide se já existe algum bloqueio configurado para o remetente, domínio, URL ou IP.

## Comandos PowerShell úteis

## Consultar mensagens por remetente e destinatário

```powershell
$Sender = "usuario@dominio.com"
$Recipient = "destinatario@dominioexterno.com"
$StartDate = "06/04/2026 00:00:00"
$EndDate = "06/04/2026 23:59:59"

Get-MessageTraceV2 `
  -SenderAddress $Sender `
  -RecipientAddress $Recipient `
  -StartDate $StartDate `
  -EndDate $EndDate |
Format-Table Received,SenderAddress,RecipientAddress,Subject,Status,MessageTraceId -AutoSize
```

## Consultar detalhes da mensagem

```powershell
Get-MessageTraceV2 `
  -SenderAddress $Sender `
  -RecipientAddress $Recipient `
  -StartDate $StartDate `
  -EndDate $EndDate |
Get-MessageTraceDetailV2 |
Format-Table Date,Event,Action,Detail -AutoSize
```

## Consultar mensagem por MessageTraceId

```powershell
$MessageTraceId = "00000000-0000-0000-0000-000000000000"

Get-MessageTraceV2 `
  -MessageTraceId $MessageTraceId `
  -StartDate $StartDate `
  -EndDate $EndDate |
Format-List
```

## Consultar detalhes por MessageTraceId e destinatário

```powershell
Get-MessageTraceDetailV2 `
  -MessageTraceId $MessageTraceId `
  -RecipientAddress $Recipient |
Format-List
```

## Validar políticas de outbound spam

```powershell
Get-HostedOutboundSpamFilterPolicy |
Format-List Name,RecipientLimitExternalPerHour,RecipientLimitInternalPerHour,RecipientLimitPerDay,ActionWhenThresholdReached
```

## Validar conectores de envio

```powershell
Get-OutboundConnector |
Format-List Name,Enabled,ConnectorType,RecipientDomains,SmartHosts,TlsSettings
```

## Validar regras de transporte que podem alterar SCL

```powershell
Get-TransportRule |
Where-Object {
    $_.SetSCL -ne $null -or
    $_.SetHeaderName -ne $null -or
    $_.PrependSubject -ne $null
} |
Format-List Name,State,Mode,Priority,SetSCL,SetHeaderName,SetHeaderValue,PrependSubject
```

## Validação do resultado

Após aplicar correções ou submeter o falso positivo, valide com novos testes controlados.

Recomendações:

1. Enviar uma nova mensagem simples, sem assinatura, imagem ou link.
2. Enviar uma mensagem com a assinatura corporativa.
3. Enviar uma mensagem com link do domínio.
4. Enviar uma mensagem com anexo comum, como PDF.
5. Comparar os cabeçalhos das mensagens recebidas.
6. Verificar se o `SCL`, `BCL`, `CAT` ou outro indicador mudou.
7. Confirmar se a mensagem chegou na Caixa de Entrada ou ainda foi para Lixo Eletrônico.
8. Validar se a mensagem foi submetida à Microsoft e se houve retorno da análise.

Essa comparação ajuda a identificar se o problema estava relacionado ao conteúdo, à reputação, ao remetente, ao domínio, à política do destinatário ou à detecção da Microsoft.

## Pontos de atenção

- SPF, DKIM e DMARC aprovados não garantem entrega na Caixa de Entrada.
- Se o destinatário está em outro tenant, o remetente não tem visibilidade completa das políticas aplicadas no destino.
- O cabeçalho completo da mensagem é essencial para análise.
- Message Trace do remetente mostra o fluxo de saída, mas não substitui a análise do tenant destinatário.
- Whitelist ampla deve ser evitada.
- A submissão de falso positivo é o caminho mais adequado quando a classificação da Microsoft aparenta estar incorreta.
- Mensagens classificadas como `HPHISH` exigem atenção maior.
- Links, assinaturas com imagens, redirecionadores e anexos podem alterar a classificação da mensagem.
- Se o problema ocorre apenas com um destinatário, pode ser configuração local do usuário ou política específica daquele tenant.
- Se o problema ocorre com vários destinatários Microsoft, pode indicar reputação, autenticação, conteúdo, URL ou padrão de envio.
- Entradas na Tenant Allow/Block List devem ser temporárias e bem justificadas.
- Sempre que possível, a correção deve tratar a causa raiz, não apenas forçar a entrega por allow list.

## Resumo

Quando mensagens legítimas caem no Lixo Eletrônico/Spam em destinatários Microsoft, a investigação não deve parar apenas na validação de SPF, DKIM e DMARC.

Essas validações são importantes, mas a classificação final pode considerar outros sinais, como reputação, conteúdo, URLs, anexos, volume de envio, BCL, SCL, políticas do destinatário, regras locais, configurações do Outlook ou overrides configurados no tenant.

O melhor caminho é coletar o cabeçalho completo da mensagem, revisar o Message Trace, analisar os campos antispam e validar se há políticas ou ações específicas no tenant destinatário.

Em casos de falso positivo, a recomendação é submeter a mensagem para análise da Microsoft e evitar liberações amplas e permanentes sem validação técnica.
