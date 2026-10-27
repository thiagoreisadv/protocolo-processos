# Agente de Peticionamento Eletronico

## Resumo

Agente local que controla o navegador do usuario, navega em plataformas judiciais (PJe, e-Proc, PROJUDI, etc.), le formularios, propoe preenchimento via IA, e protocola peticoes apos confirmacao humana.

## Contexto

O usuario e advogado e precisa protocolar ate 10 peticoes por dia em diferentes plataformas judiciais brasileiras. Os documentos ja chegam prontos, assinados e organizados em pastas no computador. O gargalo e o trabalho manual de navegar na plataforma, preencher formularios e fazer upload dos arquivos.

## Requisitos

### Funcionalidades

1. **Interface local**: Pagina web local onde o usuario informa numero do processo e seleciona a pasta com documentos
2. **Navegacao automatica**: O agente conecta no navegador ja logado, navega ate a pagina de peticionamento do processo informado
3. **Leitura inteligente de formularios**: Usa Claude API para interpretar os campos do formulario da plataforma, independente de qual seja
4. **Proposta de preenchimento**: Mostra ao usuario o que pretende preencher em cada campo e quais arquivos vai anexar, antes de executar
5. **Confirmacao humana**: O usuario revisa e confirma (ou ajusta) antes do agente preencher e protocolar
6. **Execucao e feedback**: Preenche os campos, faz upload dos documentos, protocola, e exibe confirmacao com numero do protocolo

### Restricoes

- Tudo roda localmente na maquina do usuario (sem servidor externo)
- O usuario ja estara logado na plataforma judicial no navegador
- Os documentos ja estao prontos (PDF, assinados) em uma pasta local
- Nao gera peticoes, nao assina documentos

## Arquitetura

### Componentes

```
+-------------------+       +-------------------+       +-------------------+
|                   |       |                   |       |                   |
|  Frontend (HTML)  | <---> |  Backend (Python) | <---> |  Browser (Chrome) |
|  localhost:8000   |  API  |  FastAPI server    |  CDP  |  ja logado        |
|                   |       |                   |       |                   |
+-------------------+       +--------+----------+       +-------------------+
                                     |
                                     | API
                                     v
                            +-------------------+
                            |                   |
                            |  Claude API       |
                            |  (interpretar     |
                            |   formularios)    |
                            |                   |
                            +-------------------+
```

### Frontend

Pagina HTML unica servida pelo backend. Elementos:

- Campo de texto: numero do processo
- Seletor de pasta: caminho da pasta com documentos (ou input de texto com caminho)
- Seletor de plataforma: dropdown com PJe, e-Proc, PROJUDI (para saber a URL base)
- Botao "Iniciar"
- Painel de status: mostra progresso em tempo real
- Painel de confirmacao: mostra a proposta de preenchimento para o usuario revisar
- Botoes "Confirmar" e "Cancelar" na etapa de revisao
- Historico simples: lista das ultimas peticoes protocoladas (numero do processo, data, status)

### Backend (Python + FastAPI)

Servidor local que orquestra o fluxo:

#### Endpoints

- `POST /api/protocolar` — Inicia o fluxo de peticionamento
  - Input: `{ numero_processo: str, pasta_documentos: str, plataforma: str }`
  - Inicia o fluxo em background e retorna ID da tarefa
- `GET /api/status/{task_id}` — Retorna status da tarefa (via SSE para updates em tempo real)
- `POST /api/confirmar/{task_id}` — Confirma a proposta de preenchimento
- `POST /api/cancelar/{task_id}` — Cancela o fluxo
- `GET /api/historico` — Lista protocolos feitos

#### Modulo: Browser Controller

Responsavel por interagir com o navegador via Playwright CDP:

- `conectar_browser()` — Conecta ao Chrome via CDP (o usuario precisa iniciar Chrome com `--remote-debugging-port=9222`)
- `navegar_para_processo(plataforma, numero_processo)` — Navega ate a pagina de peticionamento do processo
- `capturar_formulario()` — Extrai o HTML do formulario atual
- `preencher_campo(selector, valor)` — Preenche um campo especifico
- `fazer_upload(selector, arquivos)` — Faz upload de arquivos
- `clicar_botao(selector)` — Clica em um botao (protocolar, etc.)
- `capturar_confirmacao()` — Apos protocolar, captura o numero/comprovante

#### Modulo: AI Interpreter

Responsavel por usar Claude API para interpretar formularios:

- `analisar_formulario(html_formulario, documentos_pasta)` — Envia o HTML do formulario e a lista de documentos para Claude, que retorna:
  ```json
  {
    "campos": [
      {
        "selector": "#tipoPeticao",
        "tipo": "select",
        "label": "Tipo de Peticao",
        "valor_sugerido": "Peticao Simples",
        "confianca": "alta"
      }
    ],
    "uploads": [
      {
        "selector": "#fileUpload",
        "arquivos": ["peticao.pdf", "procuracao.pdf"],
        "ordem": ["1-peticao.pdf", "2-procuracao.pdf"]
      }
    ],
    "observacoes": "Campo 'urgencia' encontrado mas nao preenchido — requer decisao do usuario"
  }
  ```
- Prompt do Claude inclui: HTML do formulario, lista de arquivos da pasta (nomes), e instrucoes para mapear campos

#### Modulo: Flow Orchestrator

Orquestra o fluxo completo:

1. Recebe requisicao (processo + pasta + plataforma)
2. Chama Browser Controller para navegar ate o processo
3. Chama Browser Controller para capturar o formulario
4. Chama AI Interpreter para analisar e propor preenchimento
5. Envia proposta ao frontend via SSE — **pausa aqui**
6. Aguarda confirmacao do usuario
7. Se confirmado: executa preenchimento, upload e protocolo
8. Captura confirmacao e registra no historico
9. Se cancelado: nao faz nada, registra cancelamento

### Plataformas Suportadas

Cada plataforma tem uma URL base e padrao de navegacao diferente. O sistema mantem um registro minimo por plataforma:

```python
PLATAFORMAS = {
    "eproc_trf4": {
        "nome": "e-Proc TRF4",
        "url_base": "https://eproc.trf4.jus.br",
        "url_processo": "https://eproc.trf4.jus.br/eproc2trf4/controlador.php?acao=processo_selecionar&num_processo={numero}"
    },
    "pje_tjsp": {
        "nome": "PJe TJSP",
        "url_base": "https://pje.tjsp.jus.br",
        "url_processo": "..."
    }
    # Expandir conforme necessidade
}
```

A IA interpreta os formularios independente da plataforma, mas a navegacao inicial (como chegar ao processo) precisa de URL por plataforma. Novas plataformas sao adicionadas com uma entrada no dicionario.

### Fluxo de Dados

```
Usuario informa processo + pasta + plataforma
        |
        v
Backend navega no browser ate a pagina de peticionamento
        |
        v
Backend extrai HTML do formulario
        |
        v
Claude API analisa HTML + lista de arquivos → proposta de preenchimento
        |
        v
Frontend exibe proposta ao usuario
        |
        v
Usuario confirma ou ajusta
        |
        v
Backend preenche formulario + faz upload + protocola
        |
        v
Backend captura confirmacao e registra no historico
```

## Stack Tecnica

- **Python 3.11+**
- **FastAPI** — servidor web local
- **Playwright** — controle do browser via CDP
- **Anthropic SDK** — chamadas a Claude API
- **HTML/CSS/JS** — frontend simples (sem framework)

## Seguranca

- Tudo roda localmente (localhost)
- Nenhum dado sai da maquina exceto as chamadas a Claude API
- O HTML do formulario enviado a Claude nao contem dados sensiveis do processo (apenas estrutura de campos)
- A chave da API Anthropic fica em variavel de ambiente local
- O usuario confirma toda acao antes da execucao

## Prerequisitos do Usuario

1. Python 3.11+ instalado
2. Google Chrome instalado
3. Chave da API Anthropic (variavel de ambiente `ANTHROPIC_API_KEY`)
4. Iniciar Chrome com flag de debug: `chrome --remote-debugging-port=9222`

## Fora do Escopo

- Geracao de peticoes (documentos ja vem prontos)
- Assinatura digital (documentos ja vem assinados)
- Processamento em lote (uma peticao por vez, pode expandir depois)
- Login automatico (usuario ja esta logado)
- Deploy em servidor (roda apenas local)
