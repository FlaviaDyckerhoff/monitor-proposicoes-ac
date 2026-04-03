# 🏛️ Monitor Proposições AC — ALAC

Monitora automaticamente o SAPL (Sistema de Apoio ao Processo Legislativo) da Assembleia Legislativa do Acre e envia email quando há proposições novas. Roda **4x por dia** via GitHub Actions (8h, 12h, 17h e 21h, horário de Brasília).

---

## Como funciona

1. O GitHub Actions roda o script nos horários configurados
2. O script chama a API pública do SAPL (`sapl.al.ac.leg.br/api`)
3. Compara as proposições recebidas com as já registradas no `estado.json`
4. Se há proposições novas → envia email com a lista organizada por tipo
5. Salva o estado atualizado no repositório

---

## API utilizada

```
URL Base:  https://sapl.al.ac.leg.br
Endpoint:  GET /api/materia/materialegislativa/
Params:    ?ano=2026&page=1&page_size=100&ordering=-id
Swagger:   https://sapl.al.ac.leg.br/api/schema/swagger-ui/
```

API pública (SAPL Interlegis), sem autenticação, sem reCAPTCHA.

---

## Estrutura do repositório

```
monitor-proposicoes-ac/
├── monitor.js                      # Script principal
├── package.json                    # Dependências (só nodemailer)
├── estado.json                     # Estado salvo automaticamente pelo workflow
├── README.md                       # Este arquivo
└── .github/
    └── workflows/
        └── monitor.yml             # Workflow do GitHub Actions
```

---

## Setup — Passo a Passo

### PARTE 1 — Preparar o Gmail

Se já tem App Password de outro monitor, pode reutilizar — pule para a Parte 2.

**1.1** Acesse [myaccount.google.com/security](https://myaccount.google.com/security)

**1.2** Certifique-se de que a **Verificação em duas etapas** está ativa.

**1.3** Busque por **"Senhas de app"** e clique.

**1.4** Digite um nome (ex: `monitor-alac`) e clique em **Criar**.

**1.5** Copie a senha de **16 letras** gerada — ela só aparece uma vez.

---

### PARTE 2 — Criar o repositório no GitHub

**2.1** Acesse [github.com](https://github.com) → **+ → New repository**

**2.2** Preencha:
- **Repository name:** `monitor-proposicoes-ac`
- **Visibility:** Private

**2.3** Clique em **Create repository**

---

### PARTE 3 — Fazer upload dos arquivos

**3.1** Na página do repositório, clique em **"uploading an existing file"**

**3.2** Faça upload de:
```
monitor.js
package.json
README.md
```
Clique em **Commit changes**.

**3.3** O `monitor.yml` precisa de pasta específica. Clique em **Add file → Create new file**, digite:
```
.github/workflows/monitor.yml
```
Cole o conteúdo do arquivo `monitor.yml`. Clique em **Commit changes**.

---

### PARTE 4 — Configurar os Secrets

**4.1** No repositório: **Settings → Secrets and variables → Actions**

**4.2** Clique em **New repository secret** e crie os 3 secrets:

| Name | Valor |
|------|-------|
| `EMAIL_REMETENTE` | seu Gmail (ex: seuemail@gmail.com) |
| `EMAIL_SENHA` | a senha de 16 letras do App Password (sem espaços) |
| `EMAIL_DESTINO` | email onde quer receber os alertas |

---

### PARTE 5 — Testar

**5.1** Vá em **Actions → Monitor Proposições AC → Run workflow → Run workflow**

**5.2** Aguarde ~15 segundos. Verde = funcionou.

**5.3** O **primeiro run** envia email com as proposições recentes do ano (até 500) e salva o estado. A partir do segundo run, só envia se houver proposições novas.

---

## Horários de execução

| Horário BRT | Cron UTC |
|-------------|----------|
| 08:00       | 0 11 * * * |
| 12:00       | 0 15 * * * |
| 17:00       | 0 20 * * * |
| 21:00       | 0 0 * * *  |

---

## Notas sobre o SAPL

- **Tipo:** o campo `tipo` na API é um ID numérico. O script extrai o nome legível do campo `__str__` (ex: `"Projeto de Lei nº 42 de 2026"` → `"PROJETO DE LEI"`).
- **Autores:** o SAPL geralmente retorna `autores: []` inline. O script exibe `-` quando não há autor disponível — isso é normal no SAPL.
- **Paginação:** o script busca até 5 páginas × 100 = 500 proposições por execução. Para aumentar o backlog do primeiro run, ajuste `MAX_PAGINAS` no `monitor.js`.

---

## Resetar o estado

Para forçar o reenvio de todas as proposições:

1. No repositório, clique em `estado.json` → lápis
2. Substitua o conteúdo por:
```json
{"proposicoes_vistas":[],"ultima_execucao":""}
```
3. Commit → rode o workflow manualmente

---

## Problemas comuns

**Não aparece "Senhas de app" no Google**
→ Ative a verificação em duas etapas primeiro.

**Erro "Authentication failed" no log**
→ Verifique se `EMAIL_SENHA` foi colado sem espaços.

**Workflow não aparece em Actions**
→ Confirme que o arquivo está em `.github/workflows/monitor.yml`.

**Log mostra "0 proposições encontradas"**
→ Teste a API no browser: `https://sapl.al.ac.leg.br/api/materia/materialegislativa/?ano=2026&page=1&page_size=1`  
Se retornar JSON com `count` e `results`, a API está ok — pode ser problema transitório.

**Tipo aparece como "TIPO 21" no email**
→ Significa que o campo `__str__` não veio na resposta. Abra um issue — pode ser necessário ajustar o endpoint.
