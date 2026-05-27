# PSC SafeWeb — guia para implementação futura (ContaÁgil app)

Este documento substitui código removido do repositório: aqui está **o que fazer depois**, em ordem sugerida, mais **SQL de referência** e notas de operação. Projeto-alvo: `dev.app.contaagil.com.br` (CodeIgniter).

**Apresentação para gestão (EAP completa, entregas, critérios de teste por atividade, du por folha):** [PSC_SAFEWEB_EAP_APRESENTACAO.html](./PSC_SAFEWEB_EAP_APRESENTACAO.html)

**Visão para diretoria (linguagem simples; sem detalhe de banco):** [PSC_SAFEWEB_EAP_VISAO_MACRO.md](./PSC_SAFEWEB_EAP_VISAO_MACRO.md)

---

## Estimativas e totais (espelho do HTML)

Valores abaixo são os mesmos da apresentação HTML; o detalhamento folha a folha (1.1.1, entregas, “como validar”) está apenas no HTML.

### Convenção

- **du** = dia útil (8 h), por **uma** pessoa (dev ou QA conforme a atividade).
- Cada pacote na EAP já inclui **testes do próprio pacote** (manual ou automatizado) e correção de bugs encontrados nesse ciclo. Nada considera “pronto” sem evidência de teste.
- A rodada **final** de qualidade (matriz completa em homolog + regressão) está na fase **1.10** da EAP (~**6 du**).
- São **ordens de grandeza** para planejamento — reestimar após o spike eNotas (EAP 1.9) e após o primeiro fluxo OAuth com token real em homologação.
- **Fora do total MVP:** o item opcional **1.6.5** (auditoria `PscApiAudit` no client HTTP) soma **2 du**; se ficar de fora do MVP, subtrair do planejamento.

### Totais por fase (du)

| Fase (EAP) | Descrição resumida | du |
|------------|-------------------|---:|
| 1.1 | Governança, escopo, riscos, DoD | 2,5 |
| 1.2 | Cadastro PSC, redirect URI, leitura contrato API | 2,5 |
| 1.3 | Segredos, env, chave de cifragem, rotação | 2 |
| 1.4 | Banco: inspeção, DDL, ALTER, doc rollback | 1,75 |
| 1.5 | Models, mapping, Finders + testes de persistência | 4 |
| 1.6 | Cliente HTTP, PKCE, cifra, service, auditoria opcional | 13 |
| 1.7 | Controllers, callback público, MY_Controller | 8,5 |
| 1.8 | Views, link Meus dados, mensagens de erro | 4,5 |
| 1.9 | Spike e decisão eNotas / A1 | 5,5 |
| 1.10 | QA: matriz homolog, PHPUnit, revisão de logs | 6 |
| 1.11 | Runbook, dry-run go-live (1.11.3 treino formal suporte **fora de escopo**) | 2 |
| **Total** | **Soma das folhas** | **52,25** |

### Resumo executivo de prazo

- **52,25 du** em sequência ≈ um perfil **full-stack sênior** equivalente (inclui testes embutidos por pacote + fase 1.10; **sem** a folha 1.11.3 de treino formal de suporte).
- Com **2 desenvolvedores** em paralelo e credenciais PSC estáveis, costuma cair para a faixa de **30–35 du de calendário** (~**7–9 semanas**), dependendo de filas externas (PSC, eNotas).

---

## 1. Objetivo do produto

Integrar **certificado na nuvem PSC SafeWeb** via **OAuth 2.0 com PKCE**, armazenar **access token cifrado**, expor fluxo na área do cliente (**Meus dados / certificado**) e permitir **convivência** com certificado **A1 (arquivo)** e fluxo **eNotas** atual, sem quebrar o que já usa PKCS#12.

### MVP sugerido

- Fluxo OAuth completo + persistência de sessão/token + status + revogação local.
- Hub, FAQ (mínimo), mensagens de erro principais.
- Convivência com A1 testada em homolog; decisão eNotas documentada (spike seção 6).

### Fase 2 (fora do MVP imediato)

- UI dedicada a PAdES upload, se o produto exigir fluxo próprio.
- Auditoria HTTP automática em `PscApiAudit` (se não couber no MVP).
- Substituição total do P12 na eNotas **somente** após decisão formal Caminho A.

---

## 2. Pré-requisitos no portal PSC

1. Registrar a aplicação ContaÁgil.
2. Configurar **redirect URI** exatamente igual à URL real do ambiente, por exemplo:
   - `https://<host-do-app>/psc_oauth/callback`
3. Anotar `client_id`, `client_secret` e URL base da API OAuth conforme documentação PSC vigente.

---

## 3. Variáveis de configuração

Definir em `env-settings.php` (com `getenv`/fallback) ou só via ambiente:

| Constante / env | Uso |
|-----------------|-----|
| `PSC_SAFEWEB_BASE_URL` | Base da API OAuth PSC (ex.: microserviço OAuth v0). |
| `PSC_CLIENT_ID` | Cliente OAuth. |
| `PSC_CLIENT_SECRET` | Segredo (troca conforme runbook abaixo). |
| `PSC_TOKEN_ENCRYPTION_KEY` | Chave para cifrar token em repouso (recomendado ≥ 32 caracteres aleatórios). |
| `PSC_DEFAULT_OAUTH_SCOPE` | Ex.: `signature_session`, `single_signature`, `multi_signature`. |
| `PSC_AUTHORIZE_LIFETIME_SEC` | Lifetime do passo `/authorize` (ex.: 600). |

**Segurança:** não commitar segredos reais; preferir variáveis de ambiente no deploy.

---

## 4. Banco de dados (executar após inspecionar o ambiente)

Antes de aplicar: usar **mysql-dev** (ou equivalente) e validar nomes de tabelas/colunas com `DESCRIBE` / `SHOW CREATE TABLE` na base real. O script abaixo foi o planejado para o app (ajustar se o schema local divergir).

**Rollback conceitual:** dropar tabelas PSC e remover colunas em `Certificados` (somente se ainda não houver dependência de produção).

```sql
-- PSC SafeWeb — executar manualmente por ambiente se USE_DATABASE_VERSIONS estiver desligado.
-- Rollback: DROP TABLE PscApiAudit; DROP TABLE PscAccessTokens; DROP TABLE PscOAuthSessions;
--           ALTER TABLE Certificados DROP COLUMN PscCertificateAlias, ...

CREATE TABLE IF NOT EXISTS PscOAuthSessions (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  state VARCHAR(128) NOT NULL,
  code_challenge VARCHAR(255) NOT NULL,
  code_verifier TEXT NOT NULL,
  CodEmpresa INT NOT NULL,
  CodCliente INT NOT NULL,
  scope VARCHAR(64) NOT NULL DEFAULT 'signature_session',
  redirect_return_url VARCHAR(512) DEFAULT NULL,
  login_hint VARCHAR(20) DEFAULT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  consumed_at TIMESTAMP NULL DEFAULT NULL,
  ip_address VARCHAR(45) DEFAULT NULL,
  user_agent VARCHAR(512) DEFAULT NULL,
  UNIQUE KEY uk_psc_oauth_state (state),
  KEY idx_psc_oauth_created (created_at),
  KEY idx_psc_oauth_empresa (CodEmpresa)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS PscAccessTokens (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  CodEmpresa INT NOT NULL,
  CodCertificado INT DEFAULT NULL,
  scope VARCHAR(64) DEFAULT NULL,
  access_token_enc TEXT NOT NULL,
  expires_at DATETIME NOT NULL,
  authorized_identification_type VARCHAR(8) DEFAULT NULL,
  authorized_identification VARCHAR(20) DEFAULT NULL,
  slot_alias VARCHAR(255) DEFAULT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  revoked_at TIMESTAMP NULL DEFAULT NULL,
  KEY idx_psc_token_empresa (CodEmpresa),
  KEY idx_psc_token_expires (expires_at)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS PscApiAudit (
  id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT PRIMARY KEY,
  correlation_id CHAR(36) NOT NULL,
  CodEmpresa INT DEFAULT NULL,
  endpoint VARCHAR(255) NOT NULL,
  http_status SMALLINT DEFAULT NULL,
  duration_ms INT DEFAULT NULL,
  error_code VARCHAR(64) DEFAULT NULL,
  created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  KEY idx_psc_audit_corr (correlation_id),
  KEY idx_psc_audit_empresa (CodEmpresa)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

ALTER TABLE Certificados
  ADD COLUMN PscCertificateAlias VARCHAR(255) NULL DEFAULT NULL AFTER Api,
  ADD COLUMN PscAuthorizedDocument VARCHAR(20) NULL DEFAULT NULL AFTER PscCertificateAlias,
  ADD COLUMN PscTokenExpiresAt DATETIME NULL DEFAULT NULL AFTER PscAuthorizedDocument,
  ADD COLUMN PscMetadadosJson TEXT NULL DEFAULT NULL AFTER PscTokenExpiresAt;
```

Se o `ALTER TABLE` falhar por colunas já existentes, ajustar manualmente.

---

## 5. Arquitetura no ContaÁgil (obrigatório)

Respeitar: **View → Controller → Service → Finder → Model**.

### 5.1 Artefatos sugeridos (a recriar na implementação)

| Camada | Itens |
|--------|--------|
| **Libraries** | Cliente HTTP PSC (authorize, token form-urlencoded, chamadas JSON com Bearer); utilitário PKCE (verifier, challenge S256, state); cifra de token (ex.: AES-256-CBC + HMAC ou padrão interno aprovado); **Service** orquestrando OAuth, persistência e chamadas de assinatura/discovery. |
| **Models** | `PscOAuthSession`, `PscAccessToken`, `PscApiAudit`; estender `Certificado` com propriedades PSC + setters; `mapping.php` para todas as entidades novas e colunas PSC em `Certificado`. |
| **Finders** | Um finder por tabela PSC; queries apenas na camada Finder. |
| **Controllers** | `Psc_oauth` (callback **público**: `$public = true`, `$use_guard = false` ou equivalente); `Certificado_psc` (hub, preparar fluxo, status, FAQ, app confiável, revogação local). |
| **Views** | Páginas em `application/views/pages/certificado_psc/`. |
| **Core** | Em `MY_Controller`: tratar **callback sem sessão de empresa** (ex.: não redirecionar para login quando a URI for `/psc_oauth/callback`); excluir `psc_oauth` do `logFluxo` se necessário; cuidado com acessos a `currentUser` / empresa quando null. |
| **UX** | Link na aba **Certificado digital** em `meus_dados.php` apontando para o hub PSC (quando o produto aprovar). |
| **Testes** | Testes unitários mínimos em PKCE (opcional); smoke manual conforme matriz abaixo. |
| **Docs operacionais** | Runbook (redirect, rotação de secret, depuração). |

### 5.2 Fluxo OAuth (resumo)

1. Usuário autenticado inicia fluxo: criar linha em `PscOAuthSessions` com `state`, `code_verifier`, `code_challenge`, `CodEmpresa`, `CodCliente`.
2. Redirecionar para URL de authorize PSC com `response_type=code`, PKCE, `state`, escopo e redirect para `site_url('psc_oauth/callback')`.
3. Callback troca `code` por token; validar `state`; marcar sessão como consumida; gravar token cifrado em `PscAccessTokens`; opcionalmente atualizar `Certificados` com metadados PSC **sem sobrescrever** certificado A1 real (política de convivência).

---

## 6. Convivência com eNotas / A1 (spike)

**Situação típica no código:** `CANotas::certificado()` e integrações usam **arquivo PKCS#12 + senha**.

- **Caminho A:** validar na documentação/SDK eNotas se existe alternativa (PEM, outro fluxo) compatível com PSC.
- **Caminho B (usual até decisão):** manter **upload A1** para eNotas; usar **PSC** para assinaturas expostas pela API PSC (`signature`, `signature-icp`, `pades-signature`, etc.) e processos que não dependam do P12 da eNotas.

Próximos passos do spike: contato suporte eNotas / versão do pacote no `vendor`; protótipo em homologação; decisão registrada e UX em **Meus dados** alinhada ao produto.

---

## 7. Runbook operacional (após implementação)

### Rotação de `client_secret`

1. Gerar novo segredo no portal PSC.
2. Atualizar `PSC_CLIENT_SECRET` no deploy.
3. Reiniciar PHP-FPM/workers se necessário.
4. Tokens já emitidos seguem válidos até expirar; revogar no PSC se a política exigir.

### Depuração

- Sessões OAuth: `PscOAuthSessions` (`state`, `consumed_at`).
- Tokens: `PscAccessTokens` (`expires_at`, `revoked_at`) — **não** logar token em claro.
- Erros HTTP: logar corpo seguro (sem `access_token`).

### Matriz de testes manuais (homolog)

Execução consolidada e evidências (prints/planilha) entram na fase EAP **1.10** (~4 du só da matriz + regressões).

| Caso | Passo esperado |
|------|----------------|
| Sucesso PKCE | Hub → autorizar no PSC → resultado OK → status mostra titular |
| user_denied | Negar no PSC → mensagem adequada; sem token persistido indevidamente |
| state inválido | Callback com `state` aleatório → erro controlado |
| Token expirado | Renovar/reautorizar conforme UX; sem crash |
| Aplicação confiável | Se no escopo: sucesso e falha de credencial; segredo não em URL |
| Revogar local | Limpar/revogar registro local e validar UI |
| Convivência A1 | Empresa com P12/eNotas: após PSC, fluxo legado de NF continua (smoke) |
| Regressão app | Login, Meus dados, rotas críticas não PSC sem regressão |

### URIs de referência

- Hub: `/certificado_psc`
- Callback público: `/psc_oauth/callback`

---

## 8. Itens opcionais / qualidade

- Persistir linhas em `PscApiAudit` a partir do cliente HTTP (correlation id, status, duração).
- Tela ou fluxo para **PAdES upload** se o produto exigir.
- Teste PHPUnit isolado para funções PKCE (base64url, tamanho do challenge).

---

## 9. Checklist rápido antes do go-live

- [ ] SQL aplicado e validado no MySQL do ambiente (`DESCRIBE` / `SHOW CREATE`).
- [ ] Redirect URI registrado no PSC = URL real do callback.
- [ ] Segredos só em ambiente seguro; `PSC_TOKEN_ENCRYPTION_KEY` forte e rotacionável por procedimento interno.
- [ ] Matriz de testes da seção 7 **100% obrigatória** com evidências em homolog.
- [ ] Fluxo testado com empresa que já tem A1 (convivência) e cenário só PSC, se aplicável.
- [ ] Decisão eNotas (A ou B) documentada para o time fiscal/produto.
- [ ] Runbook operacional (seção 7) revisado e contato PSC definido para N2/N3.

---

*Documento gerado para planejamento; não substitui a documentação oficial PSC nem contratos de API atualizados. EAP em HTML: [PSC_SAFEWEB_EAP_APRESENTACAO.html](./PSC_SAFEWEB_EAP_APRESENTACAO.html). Visão diretoria (linguagem simples): [PSC_SAFEWEB_EAP_VISAO_MACRO.md](./PSC_SAFEWEB_EAP_VISAO_MACRO.md).*
