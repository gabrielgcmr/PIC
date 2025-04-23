# PIC
Prevention Identification Codes (PIC) — unified coding of preventive actions

# Especificação de Codificação de Prevenções

> **Versão 0.1 – 22 abr 2025**

---
fty
## 1. Objetivo

Padronizar a identificação de todas as ações de prevenção (primária, secundária e terciária) utilizadas no **RastreaMed**, garantindo:

* interoperabilidade entre módulos (frontend, backend, BI),
* facilidade de auditoria e faturamento,
* leitura humana imediata,
* compatibilidade futura com IA de autotagging.

---

## 2. Estrutura do Código

```
Pn.DOMÍNIO[.CID11].MÉTODO:ALVO[&EXTENSÃO1&EXTENSÃO2...]
```

| Segmento            | Descrição                                                                                    | Regras             |
|---------------------|------------------------------------------------------------------------------------------------|--------------------|
| **Pn**              | Nível de prevenção (`P1`, `P2`, `P3`)                                                          | Tamanho fixo 2     |
| **DOMÍNIO**         | Sistema/tema de duas letras (`ON`, `CV`, `RN`, `NE`, `VC`, `EN`, `MU`, `RE`)                 | Maiúsculas, 2 chars|
| **CID11** *(P3)*    | Stem code do CID‑11 da doença existente (ex.: `8B11`)                                         | Opcional; só P3    |
| **MÉTODO**          | Tipo de ato ou modalidade de exame (tabela § 3)                                               | 2–4 chars          |
| **ALVO**            | Órgão, fluido ou classe terapêutica (tabela § 4)                                              | 2–4 chars          |
| **EXTENSÕES**       | Metadados opcionais (intervalo, meta laboratorial, variante)                                  | Prefixo `&`, sem espaço |

---

## 3. Lista de Métodos (reservados)

| Método | Categoria                       | Exemplos comuns                     |
|--------|---------------------------------|-------------------------------------|
| `MED`  | Medicação / suplemento          | `MED:STAT`, `MED:DOAC`, `MED:BIS`   |
| `BLD`  | Exame laboratorial (sangue)     | `BLD:LDL`, `BLD:ALB`, `BLD:HBA1C`   |
| `UR`   | Exame de urina                 | `UR:ACR`                            |
| `STL`  | Exame de fezes                  | `STL:FIT`                           |
| `MEAS` | Medição de sinal vital          | `MEAS:BP`, `MEAS:HR`                |
| **Imagem**                              |                                     |
| `XR`   | Radiografia                     | `XR:LU`, `XR:SPN`                   |
| `MG`   | Mamografia                      | `MG:MA`                             |
| `CT`   | Tomografia (incl. LDCT)         | `CT:LU&LDCT`                        |
| `DXA`  | Densitometria óssea            | `DXA:HP`                            |
| `US`   | Ultrassom                       | `US:AAA`                            |
| `MRI`  | Ressonância magnética           | `MRI:BRN`                           |
| **Fisiologia**                          |                                     |
| `ECG`  | Eletrocardiograma               | `ECG:CV`                            |
| `EEG`  | Eletroencefalograma             | `EEG:BRN`                           |
| `PFT`  | Prova de função pulmonar        | `PFT:LU`                            |
| **Outros**                              |                                     |
| `EDU`  | Educação / aconselhamento       | `EDU:SMK`, `EDU:DIET`               |
| `REH`  | Reabilitação                    | `REH:CV`, `REH:STROKE`              |

(Adicionar novos métodos requer alteração deste documento e bump de versão.)

---

## 4. Alvos/Targets frequentemente usados

| Alvo | Significado                |
|------|----------------------------|
| `CX` | Colo do útero              |
| `MA` | Mama                       |
| `LU` | Pulmão                     |
| `CO` | Cólon / reto              |
| `LDL`,`HDL`,`TG` | Lipídios sanguíneos  |
| `BP` | Pressão arterial           |
| `ALB`| Albumina (urina)           |
| `HP` | Quadril (hip) – DXA        |
| `VAC`| Vacina                     |

---

## 5. Extensões (metadados)

* **Intervalo:** `&INT3Y`, `&INT12M`
* **Meta bioquímica:** `&LDL<55`, `&A1C<7`
* **Variante de protocolo:** `&LDCT` (low‑dose CT), `&SELFHPV`
* **Posologia resumida:** `&DOSE40MG`, `&IV_ANUAL`

Múltiplas extensões podem ser concatenadas: `CT:LU&LDCT&INT1Y`.

---

## 6. Exemplos completos

| Nível | Código | Descrição resumida |
|-------|--------|--------------------|
| P1 | `P1.VC.HPV:VAC` | Vacina HPV em adolescentes |
| P2 | `P2.ON.CITO:CX` | Citologia cervical (rastreio) |
| P2 | `P2.ON.CT:LU&LDCT` | TC baixa dose para CA pulmão |
| P3 | `P3.CV.BA50.MED:STAT&LDL<55` | Estatina alta intensidade pós‑IAM |
| P3 | `P3.NE.8B11.ECG:CV&INT1Y` | ECG anual após AVC isquêmico |

---

## 7. Banco de Dados (PostgreSQL)

```sql
CREATE TABLE prevention_codes (
  code         TEXT PRIMARY KEY,
  level        CHAR(2)  NOT NULL CHECK (level IN ('P1','P2','P3')),
  domain       CHAR(2)  NOT NULL,
  cid          VARCHAR(8),            -- apenas P3
  method       VARCHAR(10) NOT NULL,
  target       VARCHAR(10) NOT NULL,
  extensions   TEXT[],
  title_pt     TEXT NOT NULL,
  title_en     TEXT,
  description  TEXT,
  interval     TEXT,
  tags         TEXT[],
  created_at   TIMESTAMPTZ DEFAULT now(),
  updated_at   TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_preventions_lookup
  ON prevention_codes (level, domain, method, target);
```

_Migrações versionadas na pasta `db/migrations/` conforme golang‑migrate._

---

## 8. Governança & Versionamento

1. **Versões semânticas** (`codes‑v1.0.0`, `codes‑v1.1.0` …)
2. Mudanças **aditivas** (novos métodos/alvos) → _minor bump_.
3. Mudanças **breaking** (alterar sintaxe ou remover método) → _major bump_.
4. Pull Requests via Git (`docs/`) revisados por ≥ 2 médicos.

---

## 9. Roadmap de Implementação

1. Implementar tabela & seed inicial.
2. Expor endpoint `/api/preventions` read‑only.
3. Integrar autocomplete no frontend.
4. IA de “autocódigo” (classificação) – MVP com LLM.
5. Dashboards de cobertura preventiva no Metabase.

---

_Last updated: 22 abr 2025 – ChatGPT draft_

