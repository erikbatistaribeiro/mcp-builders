# Analisador de Matrículas de Imóvel

Recebe o documento de imóvel anexado a um card do Pipefy (PDF ou imagem), envia para OCR externo, interpreta as informações da matrícula e preenche automaticamente os campos do card — incluindo a decisão de **Apto / Não apto / Necessário regularização documental** para alienação fiduciária.

---

## Quando usar

Usar esta skill quando o documento de imóvel for anexado a um card do Pipefy e for necessário extrair as informações da matrícula:

- "Analise a matrícula do imóvel"
- "Leia o inteiro teor e preencha os campos"
- "Verifique se o imóvel tem restrições"
- "O imóvel está apto para financiamento?"
- "Extraia os dados da matrícula"

**Não usar para:** documentos que não sejam matrículas de imóvel (contratos avulsos, notas fiscais, laudos topográficos sem número de matrícula). Para esses casos, utilizar uma skill genérica de extração de documentos.

---

## Pré-requisitos

- Card do Pipefy com campo de anexo chamado **"Documento Imóvel"** contendo a URL do arquivo (PDF até 200 MB ou imagem: JPG, PNG, TIFF).
- O card deve conter também o campo **"id_card_pai"** (ou rótulo equivalente) com o ID do card pai em um segundo pipe, que receberá um resumo dos dados extraídos.
- O **Pipe Agente** (onde o documento fica) precisa ter os seguintes campos de saída configurados:

  | ID do campo | Nome de exibição |
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

- O **Pipe CGI** (pipe pai) precisa ter estes 7 campos para receber o resumo:

  | ID do campo | Nome de exibição |
  |---|---|
  | `matr_cula_cart_rios_de_im_veis_1` | Matrícula Cartórios de Imóveis |
  | `documento_do_im_vel` | Documento do Imóvel |
  | `tipo_do_im_vel_2` | Tipo do Imóvel |
  | `possui_inalienabilidade` | Possui Inalienabilidade |
  | `possui_aliena_o_fiduci_ria` | Possui Alienação Fiduciária |
  | `endere_o_do_im_vel` | Endereço do Imóvel |
  | `valor_venal_1` | Valor Venal |

- É necessário acesso de **Membro ou Admin** nos dois pipes para leitura e atualização dos campos.

---

## Ferramentas utilizadas

| Ferramenta (MCP) | Finalidade |
|---|---|
| `get_card` | Busca o card e todos os seus campos, incluindo a URL do documento e o `id_card_pai` |
| `update_card_field` | Grava os dados extraídos em cada campo do card do Pipe Agente |
| `move_card_to_phase` | Move o card do Pipe Agente para a fase correta após a gravação (fase `334111706`) |
| `update_card_field` (segundo pipe) | Grava o resumo de 7 campos no card pai do Pipe CGI usando o `id_card_pai` |

> O OCR e a interpretação do documento acontecem via endpoint HTTP externo (`/home-equity-automation/webhooks/documents`). A skill envia a URL do documento para esse endpoint e recebe de volta o JSON estruturado com os dados da matrícula. Nenhuma ferramenta local de OCR é necessária. No caso optamos por usar um OCR interno dentro da Azure, mas pode ser integrado com outros como o Google Cloud Vision.

---

## Fluxo de execução

### Passo 1 — Buscar o card e extrair a URL do documento

Busca o card pelo ID recebido do gatilho (`cardFieldUpdated` no campo `398567923`).

Na resposta, localiza o campo **"Documento Imóvel"** (busca sem distinção de maiúsculas/minúsculas e sem acentos). Extrai a URL HTTP do valor do campo.

Localiza também o campo **"id_card_pai"** (tentando os rótulos: `id card pai`, `id_card_pai`, `card pai`, `id card cgi`) e armazena o valor para o Passo 4.

Se o campo do documento estiver vazio ou nenhuma URL for encontrada, interrompe a execução e registra o problema no card.

### Passo 2 — Enviar documento para o endpoint de OCR

Envia a URL relativa do documento para o endpoint de OCR via POST:

```
POST https://<ocr-host>/home-equity-automation/webhooks/documents
Content-Type: application/json
Ocp-Apim-Subscription-Key: <api-key>

{
  "type": "imovel",
  "documentUrl": "<url-relativa-do-passo-1>"
}
```

O endpoint aceita PDF (até 40 MB) e imagens. Retorna um objeto JSON estruturado com os campos da matrícula e a decisão de crédito.

### Passo 3 — Normalizar a resposta

Limpa o JSON retornado pelo endpoint de OCR:

- Converte `"não identificado"`, `"não se aplica"`, `"não há"` (e variantes sem acento) → `null`.
- Extrai valores aninhados com segurança: `inalienabilidade.existe`, `inalienabilidade.detalhes`, `alienacao_fiduciaria.existe`, `alienacao_fiduciaria.detalhes`, `inscricao_cadastral.possui`, `inscricao_cadastral.detalhe`, `cep.possui`, `cep.detalhe`.
- Se nem `numero_matricula` nem `documento_analisado` estiverem presentes na resposta, o documento é considerado ilegível — interrompe a execução e reporta.

A saída normalizada gera dois conjuntos de campos: um para o Pipe Agente (18 campos) e outro para o Pipe CGI pai (7 campos).

### Passo 4 — Gravar dados no card do Pipe Agente e mover de fase

Preenche os 18 campos do card do Pipe Agente e move para a fase `334111706`.

| Campo | Origem |
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

### Passo 5 — Gravar resumo no card pai do Pipe CGI

Com o `id_card_pai` capturado no Passo 1, grava o resumo de 7 campos no Pipe CGI (pipe ID `306806519`):

| Campo | Origem |
|---|---|
| `matr_cula_cart_rios_de_im_veis_1` | `response.numero_matricula` |
| `documento_do_im_vel` | `response.documento_analisado` |
| `tipo_do_im_vel_2` | `response.tipo_imovel` |
| `possui_inalienabilidade` | `response.inalienabilidade.existe` |
| `possui_aliena_o_fiduci_ria` | `response.alienacao_fiduciaria.existe` |
| `endere_o_do_im_vel` | `response.endereco_imovel` |
| `valor_venal_1` | `response.valor_venal_imovel` |

Se o `id_card_pai` não for encontrado no Passo 1, este passo é ignorado e um aviso é registrado — a gravação no card do Pipe Agente (Passo 4) segue normalmente.

---

## Regras de decisão

O endpoint de OCR aplica estas regras internamente para determinar o `decisao_emprestimo.status`. Os valores são gravados no card exatamente como retornados — sem reinterpretação:

| Condição | Status |
|---|---|
| `inalienabilidade.existe = Sim` | Não apto para alienação fiduciária |
| `clausulas_impeditivas.identificado = Sim` | Não apto para alienação fiduciária |
| `documento_analisado` não é Escritura Pública | Necessário regularização documental |
| `alienacao_fiduciaria.existe = Sim` com ônus ativo | Não apto para alienação fiduciária |
| Nenhuma das condições acima | Apto para alienação fiduciária |

---

## Tratamento de erros

Se o endpoint de OCR retornar erro ou o documento for ilegível:

- Nenhum dado parcial é gravado nos campos do card.
- O campo `leitor_funcionou` é atualizado com o status do erro.
- Uma explicação em português claro (sem termos técnicos, voltada para equipe de operações) é postada como comentário no card, junto com uma ação paliativa sugerida.

---

## Critérios de sucesso

- Todos os campos aplicáveis do card do Pipe Agente estão preenchidos (ou explicitamente `null` quando o dado não foi encontrado).
- O card do Pipe Agente foi movido para a fase `334111706`.
- O resumo de 7 campos foi gravado no card pai do Pipe CGI (quando o `id_card_pai` está disponível).
- O campo `leitor_funcionou` reflete o resultado da operação.
- Nenhum JSON bruto é exibido para o usuário final.
