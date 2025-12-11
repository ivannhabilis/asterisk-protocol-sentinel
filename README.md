# 🛡️ Asterisk Protocol Sentinel (GUI Only Edition)

> "Auditoria total. 100% Gerenciado via Interface Web."

Este projeto implementa um sistema de geração de protocolos de atendimento no **IncrediblePBX / FreePBX** projetado especificamente para ambientes onde o acesso ao terminal é restrito. Toda a configuração é feita através do **Configuration File Editor** e menus nativos.

## 🎯 Capacidades

1.  **Protocolo Auditável:** Gera ID único (`YYYYMMDDHHMMSS`) falado ao cliente.
2.  **Integração Webhook:** Envia dados da chamada para API externa sem travar a ligação.
3.  **Configuração Dinâmica:** A URL do Webhook é definida por cliente na interface gráfica.
4.  **Deploy via Browser:** Código inserido diretamente no navegador.

## 🚀 Instalação Passo a Passo

### Passo 1: Inserir o Código (Dialplan)

1.  No menu superior, vá em **Admin** > **Config Edit** (ou *Configuration File Editor*).
2.  Na lista de arquivos à esquerda, localize e clique em `extensions_custom.conf`.
3.  Role até o final do arquivo.
4.  Cole o código abaixo (veja a seção **O Código**) no final do editor.
5.  Clique no botão **Save**.

### Passo 2: Preparar os Áudios

1.  Vá em **Admin** > **System Recordings**.
2.  Faça upload e nomeie os seguintes áudios:
    * Nome: `protocolo-intro` (Conteúdo: *"Por favor, anote o seu número de protocolo"*)
    * *Opcional:* `protocolo-fim` (Conteúdo: *"Aguarde enquanto transferimos"*)
3.  Clique em **Submit** e depois no botão vermelho **Apply Config** no topo da tela.

### Passo 3: Criar o Destino e Definir a URL

#### ⚙️ Modos de Configuração (Com ou Sem Webhook)

A flexibilidade é total. Você decide no momento de criar o **Custom Destination**.

##### Cenário A: Cliente quer Webhook (Auditoria Completa)
No campo **Target**, insira a URL entre parênteses:
`gera-protocolo-sentinel,s,1(https://api.cliente.com/hook)`

##### Cenário B: Cliente quer APENAS o Protocolo (Sem Integração)
No campo **Target**, deixe sem argumentos (ou parênteses vazios):
`gera-protocolo-sentinel,s,1`

**Resultado:** O sistema gerará o protocolo, gravará no CDR local e falará para o cliente, mas ignorará silenciosamente a etapa de envio de dados.

Aqui é onde você define para onde os dados vão, específico para cada cliente.

1.  Vá em **Admin** > **Custom Destinations**.
2.  Clique em **Add Destination**.
3.  Preencha da seguinte forma:
    * **Target:** `gera-protocolo-sentinel,s,1(SUA_URL_DO_WEBHOOK_AQUI)`
        * *Exemplo:* `gera-protocolo-sentinel,s,1(https://webhook.site/seu-hash-unico)`
        * **Nota:** A URL vai dentro dos parênteses. É assim que passamos o argumento para o código.
    * **Description:** `Gerador Protocolo`
    * **Return:** Selecione **Yes**.
    * **Destination:** Escolha para onde a chamada vai *após* o protocolo (Ex: *Queues > Suporte* ou *IVR > Menu Principal*).
4.  Clique em **Submit**.

### Passo 4: Ativar na Rota de Entrada

1.  Vá em **Connectivity** > **Inbound Routes**.
2.  Edite a rota do cliente (DID).
3.  Em **Set Destination**, escolha **Custom Destinations** > **Gerador Protocolo**.
4.  Clique em **Submit** e **Apply Config**.

## 💻 O Código (extensions_custom.conf)

Copie este bloco exato para dentro do *Configuration File Editor*:

```ini
[gera-protocolo-sentinel]
; --- INICIO DA OPERACAO SENTINEL (HYBRID MODE) ---
; Versao: 2.0 - Robustez Total
; O Webhook agora é opcional. O protocolo será gerado de qualquer forma.

; 1. MISSAO CRITICA: Gerar Protocolo e Gravar (Independente de Webhook)
exten => s,1,NoOp(>>> INICIANDO PROTOCOLO SENTINEL <<<)
same => n,Set(__PROTOCOLO=${STRFTIME(${EPOCH},,%Y%m%d%H%M%S)})
same => n,Set(CDR(userfield)=${PROTOCOLO})

; 2. ANALISE DE RECURSO: O cliente quer Webhook?
; Captura o argumento vindo do Custom Destination
same => n,Set(URL_WEBHOOK=${ARG1})

; CONDICIONAL: Se a URL estiver vazia, PULA para o label 'falar_protocolo'
; Sintaxe: GotoIf(CONDICAO?LABEL_SE_VERDADEIRO:LABEL_SE_FALSO)
same => n,GotoIf($["${URL_WEBHOOK}" = ""]?falar_protocolo)

; 3. MISSAO AUXILIAR: Envio de Dados (So executa se nao pulou acima)
same => n,NoOp(--- URL detectada: Enviando Webhook ---)
same => n,Set(D_ORIGEM=${CALLERID(num)})
same => n,Set(D_DESTINO=${CALLERID(dnid)})
same => n,Set(D_UNIQUEID=${UNIQUEID})
same => n,Set(D_CANAL=${CHANNEL})
same => n,Set(D_EPOCH=${EPOCH})

; Disparo assincrono (&) para nao travar o audio caso a API esteja lenta
same => n,System(curl -s -H "Content-Type: application/json" -X POST -d '{"protocol":"${PROTOCOLO}", "caller_id":"${D_ORIGEM}", "did":"${D_DESTINO}", "unique_id":"${D_UNIQUEID}", "timestamp":"${D_EPOCH}", "channel":"${D_CANAL}"}' "${URL_WEBHOOK}" &)

; 4. INTERACAO COM O CLIENTE (Ponto de encontro do salto)
same => n(falar_protocolo),NoOp(--- Iniciando Audio do Protocolo ---)
same => n,Answer()
same => n,Wait(0.5)

; Audio de introducao (Ex: "Ola, é um prazer falar com você. Anote seu protocolo:")
same => n,Playback(/var/lib/asterisk/sounds/pt-br/custom/protocolo-intro)

; Fala os digitos
same => n,SayDigits(${PROTOCOLO})
same => n,Wait(0.5)

; 5. CONCLUSAO
same => n,Return()
```

## 📡 Exemplo de Payload Recebido

```json
{
  "protocol": "20251210213005",
  "caller_id": "11999998888",
  "did": "1130004000",
  "unique_id": "167890.123",
  "timestamp": "1733878205",
  "channel": "PJSIP/trunk-001"
}
```

Este sistema foi projetado para operar sem necessidade de intervenção via console.

