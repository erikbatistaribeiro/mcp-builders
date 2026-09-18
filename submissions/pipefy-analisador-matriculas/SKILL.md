# Analisador de Matrículas de Imóvel

Leia o documento de imóvel anexado a um card do Pipefy (PDF ou imagem), envie para OCR externo, interprete as informações da matrícula e preencha automaticamente os campos do card — incluindo a decisão de **Apto / Não apto / Necessário regularização documental** para alienação fiduciária.

---

## When to use

Use this skill when the user asks to analyze a real estate document attached to a Pipefy card:

- "Analise a matrícula do imóvel"
- "Leia o inteiro teor e preencha os campos"
- "Verifique se o imóvel tem restrições"
- "O imóvel está apto para financiamento?"
- "Extraia os dados da matrícula"

**Not for:** documents that are not real estate matrícula records (standalone contracts, invoices, topographic surveys without a matrícula number). For those, use a generic document extraction skill.

---

## Prerequisites

- A Pipefy card with an attachment field named **"Documento Imóvel"** containing a file URL (PDF up to 40 MB, or image: JPG, PNG, TIFF).
- The card must also contain a field **"id_card_pai"** (or equivalent label) holding the ID of a parent card in a second pipe that will receive a summary of the extracted data.
- The **Pipe Agente** (pipe where the document lives) must have the following output fields configured:

  | Field ID | Display name |
  |---|---|
  | `n_mero_matr_cula` | Número Matrícula |
  | `tipo_de_docuemento` | Tipo de Documento Imóvel |
  | `envolvidos` | Envolvidos |
  | `cart_rio_respons_vel` | Cartório Responsável |
  | `ano_registro` | Ano Registro |
  | `tipo_im_vel` | Tipo Imóvel |
  | `inalienabilidade` | Inalienabilidade? |
  | `motivo_inalienabilidade` | Motivo Inalienabilidade |
  | `aliena_o_fiduci_ria` | Alienação Fiduciária? |
  | `motivo_aliena_o_fiduci_ria` | Motivo Alienação Fiduciária |
  | `status_decis_o_empr_stimo` | Status decisão empréstimo |
  | `justificativa_decis_o_empr_stimo` | Justificativa decisão empréstimo |
  | `endere_o_im_vel` | Endereço Imóvel |
  | `valor_venal` | Valor Venal |
  | `pra_a` | Praça |
  | `iptu_existe` | IPTU Existe? |
  | `matricula_do_iptu` | Matrícula do IPTU |
  | `leitor_funcionou` | Leitor Funcionou? |

- The **Pipe CGI** (parent pipe) must have these 7 fields for the summary write-back:

  | Field ID | Display name |
  |---|---|
  | `matr_cula_cart_rios_de_im_veis_1` | Matrícula Cartórios de Imóveis |
  | `documento_do_im_vel` | Documento do Imóvel |
  | `tipo_do_im_vel_2` | Tipo do Imóvel |
  | `possui_inalienabilidade` | Possui Inalienabilidade |
  | `possui_aliena_o_fiduci_ria` | Possui Alienação Fiduciária |
  | `endere_o_do_im_vel` | Endereço do Imóvel |
  | `valor_venal_1` | Valor Venal |

- The agent running this skill needs **Pipe Member or Admin** access to read and update fields on both pipes.

---

## Tools

| Tool (MCP) | Purpose |
|---|---|
| `get_card` | Retrieve the card and all its fields, including the document URL and `id_card_pai` |
| `update_card_field` | Write extracted data to each output field on the Agente card |
| `move_card_to_phase` | Move the Agente card to the target phase after writing fields (phase ID `334111706`) |
| `update_card_field` (second pipe) | Write the 7-field summary to the parent CGI card using `id_card_pai` |

> The OCR and AI interpretation happen via an external HTTP endpoint (`/home-equity-automation/webhooks/documents`). The MCP agent sends the document URL to this endpoint and receives the structured JSON response. No local OCR tools are needed.

---

## Workflow

### Step 1 — Fetch the card and extract the document URL

Call `get_card` with the card ID received from the trigger (`cardFieldUpdated` on field `398567923`).

From the response, locate the field named **"Documento Imóvel"** (case-insensitive, accent-insensitive match). Extract the HTTP URL from its value using a URL regex pattern.

Also locate the **"id_card_pai"** field (try labels: `id card pai`, `id_card_pai`, `card pai`, `id card cgi`) and store its value for Step 4.

If the document field is empty or no URL is found, stop and report the issue on the card.

### Step 2 — Send document to OCR endpoint

POST the relative document URL to the OCR/AI endpoint:

```
POST https://<ocr-host>/home-equity-automation/webhooks/documents
Content-Type: application/json
Ocp-Apim-Subscription-Key: <api-key>

{
  "type": "imovel",
  "documentUrl": "<relative-url-from-step-1>"
}
```

The endpoint accepts PDF (up to 40 MB) and images. It returns a structured JSON object with the extracted matrícula fields and the lending decision.

### Step 3 — Normalize the response

Clean the JSON response from the OCR endpoint:

- Map `"não identificado"`, `"não se aplica"`, `"não há"` (and their unaccented variants) → `null`.
- Extract nested values safely: `inalienabilidade.existe`, `inalienabilidade.detalhes`, `alienacao_fiduciaria.existe`, `alienacao_fiduciaria.detalhes`, `inscricao_cadastral.possui`, `inscricao_cadastral.detalhe`, `cep.possui`, `cep.detalhe`.
- If neither `numero_matricula` nor `documento_analisado` is present in the response, the document was unreadable — stop and report.

The normalized output produces two field sets: one for the Agente pipe (18 fields) and one for the CGI parent pipe (7 fields).

### Step 4 — Write data to the Agente card and move to phase

Call `update_card_field` (or `move_card_to_phase` with `phaseFields`) to populate all 18 fields on the Agente card and move it to phase `334111706`.

Fields written:

| Field | Source |
|---|---|
| `n_mero_matr_cula` | `response.numero_matricula` |
| `tipo_de_docuemento` | `response.documento_analisado` |
| `envolvidos` | `response.envolvidos` |
| `cart_rio_respons_vel` | `response.cartorio_responsavel` |
| `ano_registro` | `response.ano_registro` |
| `tipo_im_vel` | `response.tipo_imovel` |
| `endere_o_im_vel` | `response.endereco_imovel` |
| `valor_venal` | `response.valor_venal_imovel` |
| `pra_a` | `response.praca.cidade` |
| `inalienabilidade` | `response.inalienabilidade.existe` |
| `motivo_inalienabilidade` | `response.inalienabilidade.detalhes` (fallback: `clausulas_impeditivas.detalhes`) |
| `aliena_o_fiduci_ria` | `response.alienacao_fiduciaria.existe` |
| `motivo_aliena_o_fiduci_ria` | `response.alienacao_fiduciaria.detalhes` |
| `status_decis_o_empr_stimo` | `response.decisao_emprestimo.status` |
| `justificativa_decis_o_empr_stimo` | `response.decisao_emprestimo.justificativa` |
| `iptu_existe` | `response.inscricao_cadastral.possui` |
| `matricula_do_iptu` | `response.inscricao_cadastral.detalhe` |

### Step 5 — Write summary to the parent CGI card

Using the `id_card_pai` captured in Step 1, call `update_card_field` on the CGI pipe (pipe ID `306806519`) to write the 7-field summary:

| Field | Source |
|---|---|
| `matr_cula_cart_rios_de_im_veis_1` | `response.numero_matricula` |
| `documento_do_im_vel` | `response.documento_analisado` |
| `tipo_do_im_vel_2` | `response.tipo_imovel` |
| `possui_inalienabilidade` | `response.inalienabilidade.existe` |
| `possui_aliena_o_fiduci_ria` | `response.alienacao_fiduciaria.existe` |
| `endere_o_do_im_vel` | `response.endereco_imovel` |
| `valor_venal_1` | `response.valor_venal_imovel` |

If `id_card_pai` was not found in Step 1, skip this step and log a warning — the Agente card write (Step 4) still proceeds normally.

---

## Decision rules

The OCR endpoint applies these rules internally when determining `decisao_emprestimo.status`. The agent must surface them verbatim to the card fields — do not reinterpret the decision:

| Condition | Status |
|---|---|
| `inalienabilidade.existe = Sim` | Não apto para alienação fiduciária |
| `clausulas_impeditivas.identificado = Sim` | Não apto para alienação fiduciária |
| `documento_analisado` is not a public deed (Escritura Pública) | Necessário regularização documental |
| `alienacao_fiduciaria.existe = Sim` with active encumbrance | Não apto para alienação fiduciária |
| None of the above conditions apply | Apto para alienação fiduciária |

---

## Error handling

If the OCR endpoint returns an error or the document is unreadable:

- Do not write partial data to card fields.
- Use `update_card_field` to set `leitor_funcionou` to the error status.
- Post the error explanation in plain Portuguese (non-technical, BizOps audience) as a card comment, including a suggested palliative action.

---

## Success criteria

- All applicable fields on the Agente card are populated (or explicitly `null` for missing data).
- The Agente card has been moved to phase `334111706`.
- The 7-field summary has been written to the parent CGI card (when `id_card_pai` is available).
- `leitor_funcionou` reflects the outcome of the operation.
- No raw JSON is exposed to the end user.
