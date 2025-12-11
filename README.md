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
; --- INICIO DA OPERACAO SENTINEL (WEB GUI EDITION) ---
; Autor: Ivann
; Versao: Production-NoSSH
;
; O sistema espera receber a URL como argumento do Custom Destination.
; Formato do Target: gera-protocolo-sentinel,s,1(URL_DO_WEBHOOK)

; 1. Validacao de Seguranca
exten => s,1,NoOp(>>> INICIANDO PROTOCOLO SENTINEL <<<)
same => n,Set(URL_WEBHOOK=${ARG1})
same => n,GotoIf($["${URL_WEBHOOK}" = ""]?erro_url)

; 2. Geracao do Protocolo (Timestamp Puro)
same => n,Set(__PROTOCOLO=${STRFTIME(${EPOCH},,%Y%m%d%H%M%S)})

; 3. Coleta de Metadados
same => n,Set(D_ORIGEM=${CALLERID(num)})
same => n,Set(D_DESTINO=${CALLERID(dnid)})
same => n,Set(D_UNIQUEID=${UNIQUEID})
same => n,Set(D_CANAL=${CHANNEL})
same => n,Set(D_EPOCH=${EPOCH})

; 4. Auditoria Local (CDR)
; Grava o protocolo no campo 'Userfield' do relatorio de chamadas
same => n,Set(CDR(userfield)=${PROTOCOLO})

; 5. Disparo Externo (Webhook Assincrono)
; Utiliza o curl em background (&) para garantir zero latencia no audio
same => n,System(curl -s -H "Content-Type: application/json" -X POST -d '{"protocol":"${PROTOCOLO}", "caller_id":"${D_ORIGEM}", "did":"${D_DESTINO}", "unique_id":"${D_UNIQUEID}", "timestamp":"${D_EPOCH}", "channel":"${D_CANAL}"}' "${URL_WEBHOOK}" &)

; 6. Interacao com Cliente
same => n,Answer()
same => n,Wait(0.5)
; Certifique-se de ter criado a gravacao 'protocolo-intro' no System Recordings
same => n,Playback(custom/protocolo-intro)
same => n,SayDigits(${PROTOCOLO})
same => n,Wait(0.5)

; 7. Retorno para a GUI
same => n,Return()
```


; Bloco de Erro (Caso o Admin nao configure a URL)
same => n(erro_url),NoOp(!!! ERRO CRITICO: URL WEBHOOK AUSENTE NO CUSTOM DESTINATION !!!)
same => n,Return()

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

