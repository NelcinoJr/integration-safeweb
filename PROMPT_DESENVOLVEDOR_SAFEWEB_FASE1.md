# Prompt / briefing para desenvolvimento — PSC SafeWeb (Fase 1 enxuta)

**Para quem:** desenvolvedor que vai implementar no **dev.app** (CodeIgniter), com toque **mínimo** no **dev.admin** se combinado.

**Meta da Fase 1:** aplicar **migração de banco** necessária, manter fluxo atual de certificado **A1/eNotas** intacto (**smoke**), e liberar apenas o **novo fluxo PSC** suficiente para conectar autorização na nuvem (OAuth **QRCode + PKCE**), guardar token de forma segura e expor hub com estados — **sem** no primeiro corte obrigar CA, PAdES completo nem suíte inteira do `SAFEID_EAP_DEV_TEST.md`.

Leituras oficiais do repositório (ordem):

1. [PSC_SAFEWEB_GUIA_IMPLEMENTACAO_FUTURA.md](./PSC_SAFEWEB_GUIA_IMPLEMENTACAO_FUTURA.md) — SQL referência, ENV, artefatos por camada.
2. [PSC_SAFEWEB_EAP_VISAO_MACRO.md](./PSC_SAFEWEB_EAP_VISAO_MACRO.md) — fluxo OAuth, rotas referência `/certificado_psc` e `/psc_oauth/callback`, matriz mínima de testes §12.
3. [SAFEID_EAP_DEV_TEST.md](./SAFEID_EAP_DEV_TEST.md) — só como **inventário futuro**; **não** executar todos os PBs na Fase 1.

---

## 1. O que NÃO faz parte da Fase 1 (deferir explicitamente)

- Fluxo **Aplicação Confiável (CA)** completo (`authorize-ca`, `pwd_authorize`, telas PF/PJ lifetime).
- Pipeline **PAdES** (`start` / `apply` / `finish` / `info`) e UI dedicada PAdES.
- Implementar **de uma vez** `signature`, `signature-icp`, todas variações, `user-discovery` completo — só entram se Produto priorizar uma **única** operação MVP (ex.: um hash RAW) como entrega opcional pontual.
- Tabela opcional **`PscApiAudit`** / auditoria HTTP fina (**1.6.5** da macro — 2 du fora da pressa inicial).
- Collection Postman institucional + matriz SAFEID PB-08 completa (trocar por checklist curto manual em homolog + 2–3 testes PHPUnit focados quando couber tempo).

Registrar isso nas atas para **não** comparar esta fase ao pacote **37 du** SAFEID — aquele número é roadmap completo Dev+Teste.

---

## 2. Migração (manter igual hoje + preparar PSC)

Antes do SQL final: usar **mysql-dev** (conforme política da equipe): `DESCRIBE`/`SHOW CREATE` nas tabelas reais (`Certificados`, etc.). Não assumir nome de coluna.

**DDL planejado (referência)** — ver script em [PSC_SAFEWEB_GUIA_IMPLEMENTACAO_FUTURA.md](./PSC_SAFEWEB_GUIA_IMPLEMENTACAO_FUTURA.md) §4:

- Criar `PscOAuthSessions`, `PscAccessTokens` (obrigatórios ao fluxo).
- Opcional nesta fase: **omitir** `PscApiAudit` se combinado gestão/projeto — reduz trabalho sem bloquear OAuth.
- `ALTER TABLE Certificados`: colunas PSC (alias, doc autorizado, expiração, metadados JSON) todas **nullable** para não quebrar legado **A1/P12**.
- Rodar migração em homolog **limpa** e **upgrade** sobre base espelho; definir rollback em 1 página (drop colunas só se OPS aprovar política dados).

**Prova obrigatória de regressão:**

- Fluxo atual de cadastro/uso certificado arquivo continua igual (empresa só A1 smoke).
- Após PSC conectado: **NF/eNotas** com P12 deve continuar (convivência **Caminho B** até spike formal **1.9** virar obrigatório).

---

## 3. Implementação nova (SafeWeb) — cortes funcionais da Fase 1

### 3.1 Arquitetura (obrigatório ContaÁgil)

Respeitar: **View → Controller → Service → Finder → Model**.  
Sem lógica de negócio em Model/View gorda; Finders fazem só SQL parametrizado; segredos **apenas** `getenv()`/config ENV.

### 3.2 Configuração (ENV)

Lista mínima (detalhar em guia já existente): `PSC_SAFEWEB_BASE_URL`, `PSC_CLIENT_ID`, `PSC_CLIENT_SECRET`, `PSC_TOKEN_ENCRYPTION_KEY`, `PSC_DEFAULT_OAUTH_SCOPE`, `PSC_AUTHORIZE_LIFETIME_SEC`. Nada disso commitado em claro.

### 3.3 Segurança

- Token **somente** cifrado antes de gravar (**AES-256-GCM** especificado nos docs SAFEID/`1.6.3` macro — alinhar com utilitário escolhido).
- Logs: **proibido** `access_token`, `client_secret`, `code_verifier`, `code=` completo na URL nos logs (ver §1.6.1 em [PSC_SAFEWEB_EAP_VISAO_MACRO.md](./PSC_SAFEWEB_EAP_VISAO_MACRO.md)).

### 3.4 Componentes obrigatórios (nome ilustrativo, ajustar ao padrão do repo)

| Camada | Entrega |
|--------|---------|
| Cliente HTTP | Classe tipo `PscSafeWebClient`: GET query, POST form (`/oauth/token`), POST/GET JSON; timeouts; erros dominio. |
| PKCE | Geração `code_verifier` / `code_challenge` S256; persistir servidor-side com `state` + vínculo empresa/cliente na `PscOAuthSessions`. |
| Service | Orquestra `iniciarFluxoOAuth` e `processarCallback`; cifrar token; revogação **local**; convivência A1 (não apagar dados P12). |
| Finder | Por tabela PSC + extensões `Certificado` conforme mapping. |
| Controllers | Hub protegido; callback público (**exceção** em `MY_Controller`/guard de login igual doc §7). |
| Views | Estados: não conectado, sucesso pós-retorno, erros principais (`user_denied`, state inválido), status básico, revogar local. |
| UX | Link em **Certificado digital** / **Meus dados** conforme já descrito nos guias. |

**Rotas de referência:** hub `/certificado_psc`; callback público `/psc_oauth/callback`.

### 3.5 Testes aceitáveis na Fase 1 (leve mas real)

Homolog obrigatório (checklist tipo macro §12, sem planilha gigante):

- PKCE feliz.
- `user_denied`.
- state inválido.
- regressão rápida: login portal, caminho até certificado, smoke A1 onde existir empresa teste.

(Opcional) PHPUnit só em helpers PKCE e parsing JSON token se já houver infra.

---

## 4. dev.admin — se estiver combinado nesta mesma sprint

Somente cenário seguro rápido: **consulta read-only** (status PSC da empresa usando dados já gravados pelo app ou consulta Finder própria) — **sem** token PSC no admin — **sem** assinar documento pelo admin. Estimativa mínima: **0,5–1 du** só se já existir tela empresa/certificado reutilizável.

---

## 5. Critérios de aceite gerenciais (“pronto Fase 1”)

1. DDL aplicável e testada em espelho; documento rollback curto.
2. Fluxo PSC QRCode+PKEC operando em homolog com token cifrado e status na UI.
3. Evidências de regressão fluxo atual certificado arquivo e convivência básica.
4. Nenhuma credencial PSC em Git; revisão rápida de logs sem vazamento.

---

## 6. Linguagem para alinhar com liderança (du vs horas)

Na documentação do projeto **`du` = dia útil (≈ 8 h de uma pessoa)**, não “hora”. Orçamentos **grandes** (ex.: ~37 du no SAFEID) são **estrada completa Dev+Teste**; esta Fase 1 é **parcela menor** até primeiro valor entregável, com backlog explícito separado para CA/PAdES/suíte signatures.
