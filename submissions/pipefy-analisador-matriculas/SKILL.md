# Evidência — Analisador de Matrículas de Imóvel

## O problema de processo

As equipes de crédito imobiliário (home equity) recebem diariamente matrículas de imóveis em PDF e imagem para análise manual. O analista precisava abrir cada documento, localizar informações como número de matrícula, cláusulas de inalienabilidade, alienação fiduciária e valor venal, preencher manualmente dois cards no Pipefy — um no Pipe Agente e outro no Pipe CGI pai — e escrever um parecer de decisão. O processo levava entre 15 e 40 minutos por documento e estava sujeito a erros de leitura e esquecimento de campos.

## O que a automação resolve

A automação é acionada quando o campo **"Documento Imóvel"** é atualizado em um card do Pipe Agente. A partir daí, ela:

1. Lê a URL do documento do card via API GraphQL do Pipefy.
2. Envia o documento para um endpoint de OCR especializado em matrículas imobiliárias brasileiras.
3. Normaliza a resposta JSON, removendo valores inválidos como "não identificado".
4. Preenche **18 campos** no card do Pipe Agente e move o card para a fase de resultado.
5. Preenche **7 campos** de resumo no card pai do Pipe CGI, usando o `id_card_pai` capturado do próprio card.

O tempo de análise passou de 15 a 40 minutos para aproximadamente 30 segundos por documento.

## Ambiente de execução

- **Plataforma:** Pipefy iPaaS (Advanced Automations via ActivePieces) — rodando nativamente dentro do Pipefy, sem ferramenta externa de automação
- **Gatilho:** `cardFieldUpdated` no campo `398567923` (Documento Imóvel), Pipe Agente `305714516`, Org `300738585`
- **OCR + interpretação:** Endpoint HTTP externo especializado em matrículas imobiliárias (`/home-equity-automation/webhooks/documents`) — aceita PDF até 40 MB e imagens
- **Pipes envolvidos:**
  - Pipe Agente: `305714516` (onde o documento é anexado — 18 campos preenchidos)
  - Pipe CGI: `306806519` (card pai — 7 campos de resumo preenchidos)

## Fluxo executado

```
Gatilho: campo "Documento Imóvel" atualizado no card
  │
  ├─ step_9: Busca o card completo via GraphQL (todos os campos)
  │
  ├─ step_1 (código): Extrai a URL do documento e captura o id_card_pai
  │           Normalização do nome do campo (sem distinção de maiúsculas/acentos)
  │
  ├─ step_3: POST para o endpoint de OCR
  │           body: { type: "imovel", documentUrl: "<url relativa>" }
  │           Retorna JSON com 15+ campos estruturados da matrícula
  │
  ├─ step_4 (código): Normaliza a resposta
  │           Remove "não identificado", "não se aplica", "não há" → null
  │           Extrai valores aninhados com segurança
  │
  ├─ step_5: Atualiza o card no Pipe Agente
  │           18 campos preenchidos + card movido para a fase 334111706
  │
  └─ step_6: Atualiza o card no Pipe CGI (card pai)
              7 campos de resumo gravados usando o id_card_pai
```

## Campos preenchidos automaticamente

**Pipe Agente (18 campos):**

| Campo | Dado extraído |
|---|---|
| `n_mero_matr_cula` | Número da matrícula |
| `tipo_de_docuemento` | Tipo de documento |
| `envolvidos` | Nomes dos envolvidos |
| `cart_rio_respons_vel` | Cartório responsável |
| `ano_registro` | Ano do registro |
| `tipo_im_vel` | Tipo de imóvel |
| `endere_o_im_vel` | Endereço completo |
| `valor_venal` | Valor venal |
| `pra_a` | Cidade/praça |
| `inalienabilidade` | Sim/Não |
| `motivo_inalienabilidade` | Detalhes da cláusula |
| `aliena_o_fiduci_ria` | Sim/Não |
| `motivo_aliena_o_fiduci_ria` | Detalhes da alienação |
| `status_decis_o_empr_stimo` | Apto / Não apto / Regularização |
| `justificativa_decis_o_empr_stimo` | Justificativa em linguagem de negócio |
| `iptu_existe` | Inscrição cadastral: Sim/Não |
| `matricula_do_iptu` | Número da inscrição municipal |
| `leitor_funcionou` | Status da operação |

**Pipe CGI / card pai (7 campos):**

| Campo | Dado extraído |
|---|---|
| `matr_cula_cart_rios_de_im_veis_1` | Número da matrícula |
| `documento_do_im_vel` | Tipo de documento |
| `tipo_do_im_vel_2` | Tipo de imóvel |
| `possui_inalienabilidade` | Sim/Não |
| `possui_aliena_o_fiduci_ria` | Sim/Não |
| `endere_o_do_im_vel` | Endereço completo |
| `valor_venal_1` | Valor venal |

## Exemplo de saída do endpoint de OCR

```json
{
  "numero_matricula": "12.345",
  "documento_analisado": "Escritura Pública de Venda e Compra",
  "cartorio_responsavel": "2º Ofício de Registro de Imóveis de Manaus",
  "tipo_imovel": "Apartamento",
  "inalienabilidade": { "existe": "Não", "detalhes": "não se aplica" },
  "alienacao_fiduciaria": { "existe": "Não", "detalhes": "não se aplica" },
  "endereco_imovel": "Rua Exemplo, 123, Apto 45, Manaus - AM",
  "valor_venal_imovel": "R$ 320.000,00",
  "decisao_emprestimo": {
    "status": "Apto para alienação fiduciária",
    "justificativa": "Escritura pública válida, sem restrições de inalienabilidade ou alienação fiduciária ativa registradas na matrícula."
  }
}
```

## Diferenciais desta automação

- **Dois pipes atualizados em uma única execução:** o card do Pipe Agente (análise completa) e o card pai do Pipe CGI (resumo executivo) são preenchidos na mesma automação, sem intervenção manual.
- **Normalização robusta:** valores como "não identificado" e "não se aplica" são convertidos para `null` antes de gravar, evitando ruído nos campos do Pipefy.
- **Resiliência no fluxo:** a atualização do card pai tem `continueOnFailure: true`, garantindo que a análise principal não seja bloqueada caso o `id_card_pai` esteja ausente no card.
- **Em produção:** esta automação está publicada (`status: PUBLISHED`) no iPaaS nativo do Pipefy, validada com documentos reais do processo de home equity.
