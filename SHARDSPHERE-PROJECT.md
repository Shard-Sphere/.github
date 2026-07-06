# ShardSphere — Documento de Projeto e Continuidade

> **Para que serve este arquivo:** registro vivo do projeto ShardSphere. Se você (humano ou agente) abriu isto sem contexto prévio, leia do começo — ele contém a tese, a arquitetura, todas as decisões tomadas, as regras firmes, as pendências em aberto e o próximo passo. Atualize-o conforme o projeto evolui.
>
> **➡️ COMO cada processo funciona hoje: ver [`SHARDSPHERE-FLOWS.md`](./SHARDSPHERE-FLOWS.md)** (fluxo a fluxo). Este arquivo = tese/decisões/histórico.
>
> **ATUALIZAÇÃO (sessão atual) — produto quase completo.** Implementado e validado desde o MVP: **rename FRIS→ShardSphere**; **quórum configurável**; **health score ativo** (write/read routing); **bridge autenticado** (Bearer); **versionamento vault/manifest**; **cripto de produção** (Argon2id no chunk + **2SKD** passphrase+recovery-key no vault, com migração v1→v2); **SSE do self-heal** (cross-device, validado ao vivo); **painel `/painel`** + notificações; **delete/format REAL** em todos os drivers (github=orphan/wipe-histórico; s3/b2=bucket; mega/webdav=conta; gdrive/dropbox=app); **rollover/containers** (pássaros, cap git 500MB/drive 2GB, índice no vault); **upload STREAMING** (pack por bloco de 16MB, RAM constante — **1GB testado**); **github auto-cria o repo** `ss-data`. **Decisão:** discos = **contas dedicadas** (format = wipe total, de propósito); zero-backend **parado** (segue backend atual). **Benchmark real:** 1GB em ~11min (~**5GB/h**), gargalo = rede dos free tiers (ver FLOWS §19). **Pendência menor:** zstd (descartado por ora), download streaming (>500MB), scrub proativo (a amadurecer), slow-disk tier (ideia futura).
>
> **Status:** Design + **MVP funcionando**. `frontend/` tem pipeline (compress→chunk→encrypt-por-chunk→GitHub) + **RAID (replica/merge)** + **self-heal/rebalance** + testes. **E2e GitHub validado** (replica 2 contas). **Backend `backend/` (Go) = bridge** (proxy burro pra CORS-bloqueados) validado. **7 providers prontos: GitHub (direto) + GitLab + Codeberg + Bitbucket + Google Drive + Backblaze B2 + Filebase/S3 (via bridge).** Cada um com live test (put/exists/get byte-exato/round-trip). **Rebalance cross-provider real validado** [GitHub+GitLab+Codeberg]. **Mais fáceis = credencial estática** (sem OAuth, sem cartão): **Backblaze B2** (keyId+appKey, 10GB, bucket via API) e **driver S3-compatível** (SigV4, serve Filebase/Wasabi/R2/iDrive — 1 driver, N provedores). Bridge tem **allowlist por sufixo** (`.backblazeb2.com`, `.storjshare.io`, `.filebase.com`, `.wasabisys.com`) pros hosts dinâmicos/S3. **Gotcha Filebase:** bucket S3 só funciona depois de criado via `@filebase/sdk` `BucketManager.create` (S3 PutBucket dá NoSuchBucket/TooManyBuckets; free tier = 1 bucket). Mais **Dropbox** (OAuth), **MEGA** (megajs, cripto client-side), **Koofr** + **OpenDrive** (driver **WebDAV genérico** `webdav.ts`, Basic auth, sem OAuth — reusa pra Nextcloud/pCloud). **9 DISTINTOS FORTES validados pro 3×3:** GitHub, GitLab, Google(15), Backblaze(10), Filebase(10 ×2), Dropbox(10 ×5), MEGA(20), Koofr(10), OpenDrive(5). **Critério de confiança:** comercial > doação (Codeberg 100MB-privado e Disroot/non-profit = só redundância/fora). **Critério de escala:** signup SEM telefone (Yandex/Microsoft fora). Total 11 drivers (Bitbucket desativado 1GB, Codeberg pequeno). **Topologia em árvore (RAID aninhado)** via `VITE_TOPOLOGY` (JSON) → `buildNode` monta RaidDriver aninhado. RAID-10 completo testado ponta a ponta + **stress 2 rounds** (789MB, 0 perda; fixes: RetryDriver 5s, replica resiliente `allSettled`, merge por hash). **Gotcha MEGA:** megajs CIFRA o buffer de input IN-PLACE no upload → driver CLONA antes (senão corrompe outros membros do mirror no put paralelo); login V2 às vezes crasha → retry; `fetch` injetável → bridge resolve CORS. **Gotcha Dropbox:** scope `files.content.write/read` + `files.metadata.read` no app, e token tem que ser gerado DEPOIS dos scopes. **Regra:** repo git nunca com nome de storage. **Desativados:** Bitbucket (1GB total free — driver mantido), Gitea.com (sem free claro/trial — removido). **Drivers escritos mas parados:** Box (OAuth redirect localhost), OneDrive (Azure exige directory/cartão). **Container model decidido** (500MB uniforme git+drive, nome de pássaro, cap 80%/conta). **Próximo arquitetura (decidido, a implementar):** quorum-ACK + hinted-handoff (Cassandra) no write; manifest com placement; ContainerManager; download/restore na UI; auto self-heal; zstd/Argon2id.
> **Última atualização:** sessão grande. **9 providers distintos** (+Dropbox +MEGA +Koofr +OpenDrive WebDAV). **LimitDriver** (cap por disco, `VITE_DISK_MAX_BYTES`). **names.ts** (50 pássaros, pro ContainerManager). **cleanup multi-provider**. **Chunk adaptativo** (comprimido/stripes, cap 16MB — pipeline pega `stripes` do RaidDriver merge). **Merge paralelo** (mirrors concorrentes; toggle `RaidDriver.sequentialMerge` pra A/B). **Métricas medidas:**
> - **Velocidade é BANDWIDTH-bound, não paralelo-bound.** A/B 15MB: seq 35.5s vs paralelo 46.4s (0.76× — paralelo NÃO acelerou; mesma banda no mesmo cano + overhead). Paralelo só ganha com banda sobrando.
> - **Per-disco (5MB, 1 amostra):** lentos = **b2 (0.14 up!) e opendrive (0.18 ambos)**; MEGA mid (0.6); rápidos = gdrive (1.56), gitlab (1.28), koofr. (MEGA NÃO era o gargalo — chute furado.)
> - **e2e:** 100MB up=88.5s (1.13 MB/s), down=25.6s. Adaptativo confirmado (1MB→3×0.33, cap 16MB). 0 perda.
> - **UX:** 300MB ≈ ~5min (sandbox) ou 1-12min (banda do user × 3 réplicas). OK pra backup background; ruim pra síncrono. Alavanca = menos réplicas / mais upstream, NÃO paralelo.
> - **Email** considerado (tier último-recurso, cc=N cópias 1 send) mas ADIADO: IMAP/SMTP precisa backend mail client (não a bridge); REST só Google/Microsoft; risco de ban por bulk self-storage.
> **Setup atual (enxuto, pra desenvolver):** 3 mirrors de 2 = **30GB úteis, 2 cópias cross-empresa**, cap 10GB/disco. `{"merge":[{"replica":["github1","gitlab"]},{"replica":["github2","gdrive"]},{"replica":["b2","koofr"]}]}`. Validado round-trip.
>
> **Stress test round 1 (95min, 789MB, 120 rounds): `finalLost=0` — ZERO perda.** Redundância aguentou 33 falhas de upload (todas codeberg 502) + read-errors aos montes. github1 passou de 1GB e seguiu. Achados → 3 fixes (TODOS APLICADOS):
> 1. ✅ **`RetryDriver`** (`core/retry.ts`): retry 5s × 3 em put/get contra rate limit/5xx/502. Envolve cada folha no `config.ts`.
> 2. ✅ **Replica resiliente** (`raid.ts`): `putChunks` replica usa `Promise.allSettled` — grava nos que dão certo, só falha se TODOS caírem (degradado → self-heal depois). Antes era tudo-ou-nada.
> 3. ✅ **Merge por hash** (`raid.ts`): distribui por `parseInt(id.slice(0,8),16) % n` (não `order`) → uniforme, arquivo de 1 chunk não cai sempre no filho 0.
> **Round 2 (com fixes, 90min, 395MB): `finalLost=0`, `filesFail=0` (era 33), filebase2 recebeu objetos (era 0).** Gitea deu 2 putErr mas a replica resiliente absorveu (arquivo ok). Trade-off: throughput ~metade (esperas de retry). **Fixes validados.** **Gap aberto:** put degradado deixa chunks faltando no disco que falhou; o `rebalance()` existe e funciona mas **não dispara automático** — falta auto-heal (pós-put ou agendado).

---

## 1. O que é o ShardSphere

ShardSphere é uma camada de software que transforma **várias contas de serviços gratuitos** (email, cloud drives, git) em **um único disco virtual distribuído, criptografado e controlado pelo usuário**.

O ShardSphere **não fornece armazenamento próprio**. O usuário conecta as próprias contas; o ShardSphere coordena: fragmenta, comprime, criptografa, distribui, replica e reconstrói os arquivos.

## 2. Tese / motivação (o "porquê")

O projeto **não** é sobre "storage grátis infinito". É sobre **soberania de dados**:

1. **Não ficar refém de fornecedor.** Se um provedor amanhã cobrar caro, mudar política ou bloquear a conta, os dados não podem ficar presos num único lugar. Distribuir + replicar elimina o ponto único de aprisionamento.
2. **Privacidade real.** A criptografia é client-side para que **nenhum provedor leia o conteúdo** — nem para treinar IA, nem para indexar, nem para vazar. O dado do usuário é dele.
3. **Custo não é o ponto central.** O usuário aceita que isso usa serviços gratuitos, mas a meta é controle e independência, não economia.

**Postura ética assumida:** uso pessoal e legítimo. Arquivos pequenos (chunks ≤16MB), respeitando os limites dos provedores (60–80% da capacidade), sem spam, sem sobrecarregar ninguém. Manter as contas ativas e **sem ban** é um objetivo explícito — nada de comportamento que queime conta.

## 3. Filosofia / princípios

- **Client-first:** toda a inteligência (compressão, criptografia, chunking, RAID, hashing) roda no cliente.
- **Zero-knowledge:** o backend nunca vê plaintext nem chave. No máximo, transita ciphertext.
- **Seus dados, suas contas, suas regras.**
- **Crescer por diversidade de drivers, não por quantidade de contas.**
- **Backend mínimo e barato:** coordenação + proxy, não processamento de dados.

## 4. Origem: a POC (`legacy/`)

Existe uma POC funcional em Go em `legacy/` que **provou o conceito** (rodou end-to-end ao menos uma vez — ver `legacy/results/CC.txt`). Pipeline da POC:

`zip → AES → split (5MB) → enviar como anexo de email (Gmail API + OAuth2) → buscar por subject → baixar anexos → remontar → decifrar → unzip`

**O `legacy/` é apenas referência.** Não é a base de código da V1.

**Aprendizados / bugs da POC a NÃO repetir:**
- Criptografia quebrada: AES-CFB, o IV é descartado no encrypt mas o decrypt lê IV dos 16 primeiros bytes → não casa. Sem auth tag. → Usar **AES-256-GCM** com IV/nonce prependido.
- IDs por **MD5(path)** → usar **SHA-256**.
- Ordenação lexical das partes quebra com ≥10 chunks (`part_10 < part_2`) → ordenação numérica.
- **Sem manifest persistido** — recuperação dependia só do subject do email → criar manifest/metadata explícito e cifrado.
- `zip` stdlib em vez de **zstd**. Chunk de 5MB (hoje a decisão é **16MB**).
- O Gmail provider usa **Gmail REST API + OAuth2** (não IMAP/SMTP) — isso é importante: confirma que email via REST+OAuth funciona, inclusive em browser.

## 5. Arquitetura geral

```
        Browser / Desktop / CLI  (ShardSphere Core: compress, encrypt, chunk, RAID, hash)
                     │
        ┌────────────┼─────────────────────────────┐
        │ (direto, onde CORS permite)               │ (onde CORS bloqueia)
        ▼                                            ▼
   Providers REST+OAuth                       Backend (coordenação + proxy)
   Gmail, Outlook, Google Drive,              - custódia de OAuth (a decidir)
   OneDrive, Dropbox, Box, GitHub             - proxy de ciphertext (no-log)
                                              - armazena manifest cifrado (blob opaco)
                                                     │
                                                     ▼
                                              GitLab, Bitbucket, Codeberg,
                                              drives tier-2, etc.
```

- **ShardSphere Core:** lógica única, reutilizável (browser/desktop/CLI). Stack do Core ainda não decidida (ver §12).
- **Backend = coordenação:** **em Go (decidido)**. Autenticação, identidades, mapa de chunks (manifest cifrado, opaco), health, e **proxy** para o que o browser não alcança por CORS. Relaia **só ciphertext**; nunca vê plaintext nem chave.
- **Dado vai client→provider direto** sempre que o CORS do provedor permitir.

### Organização de código / stack (DECIDIDO)

- **Backend:** **Go**. Pode processar, mas **só ciphertext e metadata** (roteamento, health, manifest opaco, proxy CORS, orquestração). Nunca plaintext, nunca token/chave usável. Sem Node.
- **Front:** **TypeScript + React**, arquitetura limpa em duas camadas:
  - **`core/`** — domínio TS puro, **framework-agnostic (zero React/UI)**. Pode usar libs de cripto/util (ex.: Argon2id-wasm, zstd-wasm). Aqui mora: cripto, compressão, chunking, hash, RAID placement, rebuild. Portável e testável fora do browser.
  - **`presenter/`** — React, só entrega de tela, pouca regra.
- A divisão preserva o client-first: o pipeline sensível vive no `core` (cliente), o backend é coordenação.

### Backend = bridge burra (DECIDIDO)

Os providers CORS-bloqueados (GitLab, Bitbucket, Codeberg) passam por um **proxy burro** em Go (`backend/main.go`):
- Endpoint `POST /bridge` recebe `{url, method, headers, body(b64)}`, **encaminha** ao provider e devolve `{status, contentType, body(b64)}`.
- **Não conhece provider nenhum** — toda a lógica de cada provider fica no **TS** (`core/`), montando a request nativa e passando pela ponte. 1 bridge serve todos.
- **Allowlist de hosts** (anti-SSRF). **Não loga** headers/body (token/ciphertext passam, não ficam). CORS liberado pro front.
- Zero-knowledge mantido: trafega só ciphertext + token transitório. Validado contra GitLab real.

### Estrutura de pastas / repositórios (DECIDIDO)

`backend/` e `frontend/` são **projetos separados, cada um com git próprio**. A raiz **não é repo git** (o `.git` da raiz foi removido — o `legacy/` era só POC, sem histórico a preservar). O doc mestre fica **solto na raiz**.

```
shardsphere/                  # NÃO é repo git
  ShardSphere-PROJECT.md             # doc mestre (solto, referência de topo)
  legacy/                     # POC antiga — só referência, read-only
  backend/                    # repo git próprio — Go (coordenação, ciphertext/metadata)
  frontend/                   # repo git próprio — React
    src/
      core/                   # TS puro, framework-agnostic (cripto, chunk, compress, hash, raid, Driver)
      presenter/              # React, só tela
```

## 6. Conectividade por backend (matriz CORS / browser)

Realidade de o que dá pra acessar **direto do browser** (REST + OAuth2 + CORS) vs. o que exige **backend-proxy**:

### Email (decisão: só REST+OAuth, SEM IMAP)
- **Gmail** — browser-direto. Gmail REST API + OAuth2 PKCE, CORS ok. Scopes `gmail.send` + `gmail.readonly`.
- **Outlook** — browser-direto. Microsoft Graph + OAuth2, CORS ok. Scopes `Mail.Send` + `Mail.Read`.
- **Decisão:** V1 usa **apenas Gmail + Outlook**. IMAP (Yahoo, GMX, Mail.com, Zoho, Proton, Tuta) **descartado** — complicado, senha, etc.

### Cloud Drives
- **Browser-direto (OAuth2 + CORS ok):** **Google Drive** (scope `drive.file` p/ fugir de verificação pesada), **OneDrive** (Graph, `Files.ReadWrite`), **Dropbox**, **Box**.
- **Tier-2 (testar CORS / login diferente → provável backend):** MEGA (sem OAuth, login senha + E2E, lib `megajs`), pCloud, Koofr, Yandex Disk.
- **Sem API utilizável:** Icedrive, iCloud, Proton Drive.
- **Decisão V1:** Drive + OneDrive + Dropbox + Box (browser-limpos). MEGA fica para V2.

### Git (via REST API — Contents/Blobs, não git clone/push)
- **GitHub** — chamadas de arquivo browser-direto (CORS ok). **Pegadinha:** o passo OAuth `code→token` não tem CORS → usar **PAT** (usuário cola token) **ou** backend só para a troca do token. Depois, dados browser-direto.
- **GitLab / Bitbucket / Codeberg** — CORS provavelmente bloqueado → **via backend-proxy**. (Codeberg/Forgejo self-hosted: você controla o CORS.)
- **Decisão V1:** começar **só GitHub**; os outros entram quando o backend-proxy existir.

## 7. Estratégia de armazenamento no Git

- **Batch:** vários chunks num único commit (Git Data API: N blobs → 1 tree → 1 commit). Não é 1-chunk-1-commit.
- **Anti-inflar:** tratar o repo como **key-value flat** (arquivo no path = hash do chunk); sobrescrever/deletar via API; **squash periódico para um commit orphan** (estado atual sem histórico). Ressalva: o **GC físico é controlado pelo GitHub** — o tamanho contado só cai quando o GC server deles roda; o reclaim não é instantâneo. Mesmo assim é viável e não é blocker.
- Limites práticos: GitHub recomenda repo <1–5GB, blob <100MB. → **vários repos por conta**.
- Drives **não têm** esse problema → drives são a **base**, git é **complemento**.

### Gotchas do GitHub driver (descobertos no e2e real, já corrigidos)
- **Repo vazio:** a Git Data API (`/git/blobs`) retorna **409 "repository is empty"** sem um commit inicial. Solução: **bootstrap** com 1 arquivo via **Contents API** (`PUT /contents/.gitkeep`), que cria o 1º commit/branch em repo vazio.
- **Consistência eventual:** logo após o bootstrap, o repo ainda aparece como "empty" por ~1–2s. Esperar o `GET /git/ref/heads/<branch>` retornar 200 antes de criar blobs.
- **Contents API corrompe binário:** o campo `content` (base64) de `GET /contents/...` **não** volta byte-exato para binário. Ler pelo **blob API**: pegar o `sha` via Contents, depois `GET /git/blobs/<sha>` (byte-exato).
- **Discrição (anti-óbvio):** os chunks vão na pasta **`keys/`** (não `chunks/`); marcador `.gitkeep`; mensagens de commit neutras (`init`/`update`). Nada no repo revela que é storage de dados. **Nome do repo git NUNCA pode revelar storage** — usar nome de projeto de código (ex.: `fris-front-end`, `front-end-test`). Só **buckets de drive** (B2/S3, privados/opacos) podem se chamar `fris-storage`.

### Modelo de container unificado (DECIDIDO)
Regra ÚNICA pra todo provider (git e drive), sem ifs/else:
- **Container = ~500MB** (repo no git, pasta no drive). Rollover em ~480-550MB → cria o próximo.
- **Nome temático** (ex.: pássaros — `sparrow`, `falcon`, `heron`), **nunca com "store"/"backup"/"fris"**.
- **Cap por conta = 80% do free total** do provider (1 número por adapter, não lógica).
- **Placement no manifest:** chunk → container (necessário pro hinted-handoff/repair).
- Adapter só faz "grava neste container, abre outro quando encher" — não sabe se é git ou drive.
- **Provider pequeno mas confiável → `merge(conta1..N)` = 1 disco maior, 1 failure domain.** Ex.: Dropbox = merge(×5) = 10GB, Filebase = merge(×2) = 10GB. Mesmo app OAuth serve N contas (N refresh_tokens). Esse disco entra como 1 membro num mirror (1 empresa); redundância vem do replica acima. **Distintos preservados** (1 empresa = 1 disco, mesmo sendo multi-conta por dentro).
- 500MB cabe sob o per-repo mais apertado (**Codeberg 750MiB**) e **alivia rate limit** (GitHub 6 push/min/repo → mais repos = mais throughput).
- **Caps free reais (pesquisados):** GitHub ~1GB/repo soft (sem total de conta claro, escala via N repos), GitLab 10GB/projeto, **Codeberg = 750 MiB TOTAL por CONTA** (não por repo!) — e **privado/"non-promoted" = só 100 MiB total** (nosso caso); disco minúsculo, NÃO shardeia, bom só pra redundância, Google 15GB, Backblaze 10GB, Filebase 5GB + máx 1000 arquivos. Bitbucket free = 1GB TOTAL workspace (DESATIVADO, driver mantido). Gitea.com = sem free claro/trial (REMOVIDO).
- **Lição capacidade:** mirror = min(membros) → não espelhar drive grande com **Codeberg** (estrangula). GitHub/GitLab acompanham os drives via multi-repo. Topologia boa: drives+gitlab/github nos mirrors, Codeberg como bônus.
- **Escrita em batch:** N blobs → 1 tree (com `base_tree`) → 1 commit → atualiza ref. Tudo num commit.

### Drivers dos outros providers (validados via bridge, 1 commit por batch)
- **GitLab** (REST v4, auth `PRIVATE-TOKEN`): write = `POST /projects/:id/repository/commits` com `actions:[{action:create,file_path,content(b64),encoding:base64}]`. Read byte-exato = `/repository/files/{enc(path)}/raw?ref=`. `projectId` = id numérico. Repo com README → sem bootstrap.
- **Codeberg** (Forgejo/Gitea v1, auth `token <t>`): write = `POST /repos/{o}/{r}/contents` (ChangeFiles, `files:[{operation:create,path,content(b64)}]`). Read byte-exato = `/raw/{path}?ref=`. Token precisa scope `read:repository`+`write:repository`.
- **Bitbucket** (REST 2.0, auth **Basic email:token**, API token com scope repository R/W — App Passwords foram removidos; endpoints de listagem `/workspaces` e `/repositories` deram **410 deprecated**): write = `POST /repositories/{ws}/{r}/src` **multipart** (field-name = path = 1 commit p/ N arquivos). Read byte-exato = `/src/{branch}/{path}`.
- **Padrão comum:** pasta `keys/`, leitura sempre por endpoint **raw/blob** (Contents/inline corrompe binário). Todos os 3 CORS-bloqueados → via bridge Go; lógica no TS, `fetchImpl` injetável (bridge no browser, fetch direto nos testes).
- **Google Drive** (1º não-git, OAuth2): sem PAT — precisa **client_id/secret** (OAuth client no Google Cloud, redirect `http://localhost:53682/`) + **refresh_token** (helper `backend/tools/google-oauth.mjs` faz num clique). Driver troca refresh→access (1h, cacheado). Chunk = **arquivo nomeado pelo id** na pasta `fris-keys` (sem path/commit); `getChunk`/`exists` = `files.list?q=name='id'` → `files/{fileId}?alt=media` (byte-exato). **Gotchas de setup:** (1) **ativar a Drive API** no projeto (`console.developers.google.com/apis/api/drive.googleapis.com`), senão 403 "API not enabled"; (2) consent screen 403 → **Publish app** (scope `drive.file` é não-sensível, sem verificação) ou add test user; (3) Google **esconde o client secret** após criar → usar o popup de criação ou "Add secret". Hosts no allowlist da bridge: `oauth2.googleapis.com`, `www.googleapis.com`.

## 7.1 RAID / Política de armazenamento (composição recursiva) — DECIDIDO

Modelo de árvore composável, estilo ZFS/mergerfs. Cobre RAID-1/0/10 por composição.

### Estrutura
- **Nó** = **Provider-Disc** (folha = 1 provider/conta) **ou** **Group**. Group é também um nó → compõe recursivamente.
- **Um Provider-Disc participa de UM só nó** (não compartilha entre grupos). Isso torna a estrutura uma **árvore real** (folha aparece 1x) e garante separação de domínios de falha de forma **estrutural** (sem checagem em runtime).
- **Disco (virtual)** = o nó-raiz nomeado que o usuário usa para guardar arquivos (qualquer composição).

### Dois operadores de Group
- **REPLICA** (= MIRROR): cada chunk → **cópia em CADA filho**. Dá redundância.
- **MERGE** (= SPAN): cada chunk → grava em **UM filho só** (o mais vazio). Dá capacidade, **sem** redundância.

### Granularidade
- Roteamento por **chunk** (não por arquivo). MERGE espalha os chunks de um arquivo; REPLICA duplica cada chunk.

### Papel do MERGE de baixo nível — casar capacidade
- REPLICA usa `min(filhos)` → parear desiguais desperdiça. Solução: usar MERGE para **agregar providers pequenos até casar o tamanho** do provider grande.
- Ex.: `REPLICA(GitHub 10GB, MERGE(WP1..WP10 de 1GB = 10GB))`. `min(10,10)` casa, zero desperdício. Perder 1 WP não perde dado (a cópia está na perna GitHub).

### Capacidade (recursiva)
- **Provider-Disc** = capacidade útil × cap (0.8).
- **REPLICA[filhos]** = `min(filhos)`; nº de cópias = nº de filhos.
- **MERGE[filhos]** = `soma(filhos)`; cópias = 1.

### Redundância e segurança
- **Redundância só vem de nós REPLICA.** MERGE puro (sem REPLICA acima) = **inseguro** (perder 1 provider perde chunks).
- **Default V1 = REPLICA(2)** (≥2 cópias). MERGE-de-REPLICA liberado para capacidade+segurança. MERGE puro = avançado/explícito.
- **Failure domain = CONTA:** as réplicas de um mesmo chunk **nunca** caem na mesma conta (pega o caso de ban: 2 repos da mesma conta GitHub não valem como 2 cópias). A regra "disco em um nó só" + esta garantem domínios distintos.

### Self-heal (rebuild)
- Estados do disco: `healthy → degraded → failed` (via Health Score, §14).
- Falha detectada → **alerta** → **rebuild**: lê os chunks órfãos de uma **perna REPLICA viva** (cópia de **ciphertext puro**, sem decifrar) e re-grava em disco substituto.
- **Rebuild roda no CLIENTE** (desktop/CLI), não no servidor — porque precisa de tokens usáveis, e o servidor não os tem (token custody, §8). Cura só ocorre com um cliente online.
- **O cap de 80% é o headroom do rebuild:** a folga de 20% absorve a perda de um disco pequeno (os chunks órfãos redistribuem na folga dos irmãos do MERGE) sem precisar de disco novo imediato.

## 8. Modelo zero-knowledge do backend/proxy

- Backend = **relay burro de ciphertext**. No upload ele transita o destino (provider, repo/path, token) e o chunk cifrado — **não persiste** essa relação chunk→arquivo.
- O que o backend **nunca** vê: **plaintext** e a **chave de criptografia**.
- Nuance honesta: no instante do upload o backend **vê transitoriamente** o destino + token para rotear. "Não sabe de onde pertence" só vale com **no-log** rigoroso, TLS, processo efêmero, token por-request.
- **Manifest** (mapa de chunks → arquivos): cifrado client-side, armazenado como **blob opaco** (no backend ou nos próprios providers — a decidir). O backend só guarda bytes que não consegue ler.

### Custódia de token OAuth (DECIDIDO)

- **Tokens cifrados no cliente** com chave derivada da senha do usuário (Argon2id), guardados como **blob opaco** (pode ficar no backend). Dump do banco = ciphertext inútil sem a senha. O backend **nunca** tem token usável at-rest.
- **A chave de criptografia dos dados nunca vai ao backend** — fora de discussão.
- **Providers browser-direto** (Gmail, Outlook, Drive, OneDrive, Dropbox, Box, GitHub-data): token **nunca** passa pelo backend, o cliente usa direto. Só a minoria CORS-bloqueada passa token pelo proxy, **transitório, no-log, efêmero**.
- **Escopo mínimo nas credenciais** para limitar estrago se vazar: GitHub = **fine-grained PAT** preso aos repos do ShardSphere; Drive = scope **`drive.file`**. Token vazado → só os chunks cifrados daquele provider, nada além.
- **Keep-alive (manter conta viva) roda no cliente desktop/CLI, NUNCA no servidor.** Não colocar refresh token usável no servidor só para keep-alive. Renovação só quando o usuário está online (ou pelo desktop).
- **Transporte:** TLS (+ cert pinning no desktop/CLI). **Não** reinventar criptografia de canal com pubkey caseira — TLS já é chave-pública-contra-MITM, testado. Opcional V2+: envelopar o token transitório com a pubkey do processo-proxy, só se não confiar na borda que termina o TLS (ex.: CDN).

### Onboarding e hierarquia de chaves — estilo 1Password (DECIDIDO)

Modelo **2SKD (Two-Secret Key Derivation)**, igual 1Password:

```
Senha do usuário (memoriza)  +  Secret Key (gerada, vai no PDF/Emergency Kit)
                     │  Argon2id
                     ▼
              Master Key (MUK)          ← nunca sai do cliente
                     │ cifra
                     ▼
        Key Bundle (cifrado, fica no backend como blob opaco):
          ├─ Data Key      → cifra os chunks
          ├─ Manifest Key  → cifra o manifest
          └─ Token Key     → cifra o blob de tokens OAuth
```

- Backend guarda só: blobs cifrados (manifest, tokens) + o Key Bundle cifrado. **Nunca** vê senha, Secret Key, MUK ou as chaves internas.
- **Emergency Kit (PDF):** gerado no onboarding com a **Secret Key**. Usuário guarda em lugar seguro. **PDF vazado sozinho NÃO decifra** — falta a senha (só na cabeça do usuário). São **2 fatores** (Secret Key + Senha).
- **Zero recovery:** perdeu PDF **e** esqueceu a senha = dado perdido pra sempre. Sem reset, sem backdoor. O onboarding **deve** forçar o usuário a confirmar que entendeu isso.
- **Chaves separadas** (Data / Manifest / Token) permitem rotação independente. V1 pode usar 1 chave, mas a estrutura já fica pronta.

### Manifest (DECIDIDO)

- Manifest (mapa chunks → arquivos → providers) = **blob cifrado client-side** (Manifest Key), armazenado **no backend como blob opaco**. Só o cliente decifra.
- **Sync multi-device** (futuro): precisa de **versionamento + lock** (ou CRDT) para evitar conflito quando 2 aparelhos editarem. V1 = mono-device, resolve depois.

## 9. Regras firmes (decididas)

1. **Cap de uso:** nunca usar mais que **80%** (config. 60–80%) da capacidade física de cada provider (headroom para reconstrução/migração).
2. **Anti-concentração:** nenhum provider pode concentrar fração grande da capacidade total — perder um provider nunca pode ser catastrófico.
3. **Redundância:** **replicação simples na V1**; erasure coding (Reed-Solomon) só em V2/V3.
4. **Chunk:** 16MB (configurável).
5. **Cripto:** compressão **zstd**; KDF **Argon2id**; cifra **AES-256-GCM** (nonce prependido); hash/ID **SHA-256**.
6. **Download sempre autenticado e privado** — nunca tornar arquivo público (link público = exposição + flag de abuso; e o dado é ciphertext mesmo, então não há ganho em expor).
7. **Email:** só Gmail + Outlook (REST+OAuth). Sem IMAP.
8. **Proxy NÃO rotativo para rate limit:** cota de API é **por-conta/token, não por-IP** — trocar IP não aumenta cota. E **rotacionar IP na mesma conta dispara "login suspeito" → risco de lock/ban** (o oposto do objetivo). Proxy serve para (a) contornar CORS (proxy fixo) e (b) esconder o IP de origem. Para manter conta viva: **1 IP estável e consistente por conta** + atividade orgânica.
9. **Manter contas ativas e sem ban** é requisito, não detalhe.
10. **Ordem do pipeline:** **comprimir → chunkar → cifrar-por-chunk.** Nunca cifrar antes de comprimir (ciphertext não comprime). Cada chunk é cifrado independente (AES-256-GCM, nonce único, tag própria) → verificável/baixável sozinho (essencial pro self-heal e paralelismo). Chunk id = SHA-256(`nonce‖ciphertext‖tag`).

## 10. Capacidade (premissas de trabalho)

- Por identidade (premissa teórica): Git ~31GB + Cloud ~86GB + Hosting ~22GB ≈ **~139GB físicos**.
- Com cap 80% + 3 cópias ≈ **~37GB úteis por identidade**.
- **10 identidades ≈ ~370GB úteis**; ~27 identidades ≈ 1TB útil.
- Crescimento **linear**: cada novo driver/identidade soma capacidade sem mudar arquitetura.
- **Meta MVP:** ~370–500GB úteis com ~10 identidades (1 conta por provedor de email). Tratar 139GB/identidade como **teto teórico** — validar real (free tiers mudam, rate limits, bandwidth caps).

## 11. Identidades

- **Identity** = uma conta de email usada como autenticador para criar contas nos providers (não como storage em si — exceto o próprio email driver).
- **Estratégia:** começar com **1 identidade por provedor (10 no total)**; crescer por **diversidade**, adicionando 2ª/3ª conta só onde o provedor se mostrar estável. (Email só Gmail/Outlook por enquanto.)
- Perder o email não apaga a conta do git/drive (o email é autenticador, o ativo é a conta no provider).

## 12. Pendências em aberto (NÃO decididas)

- **MEGA:** quando entra (V2).
- **GitLab/Bitbucket/Codeberg:** entram quando o backend-proxy existir.

## 13. Roadmap (alto nível)

- **V1:** Gmail+Outlook, Drive+OneDrive+Dropbox+Box, GitHub. Mirror (replicação simples). Chunking, zstd, AES-GCM, manifest cifrado. CLI/desktop. Backend mínimo (proxy + manifest).
- **V2:** MEGA; GitLab/Bitbucket/Codeberg via proxy; Desktop/Virtual Drive; migração automática; health check.
- **V3:** Plugins/marketplace; Stripe + Parity / Erasure Coding; compartilhamento.
- **V4:** TOR / multi-hop / proxy avançado.
- **V5:** drivers da comunidade; tiers premium/enterprise.

## 14. Mecanismos exigidos desde a V1

- **Health Score** por provider (disponibilidade, erros, espaço restante). Alimenta os estados `healthy/degraded/failed`.
- **Self-heal / migração automática:** provider caiu → alerta → re-replica os chunks órfãos a partir de uma perna viva (cópia de ciphertext) para disco substituto. **Roda no cliente** (desktop/CLI), nunca no servidor — precisa de tokens usáveis (ver §7.1 e §8). O cap de 80% serve de headroom.

## 15. Glossário

- **Identity:** conta de email autenticadora (Gmail/Outlook) usada para criar contas nos providers.
- **Driver:** adaptador de um backend (Git, Drive, Email...) com interface única `Put/Get/Delete/Exists/Health/Capacity`.
- **Engine/Provider:** instância concreta de um driver (ex.: GitHub, Google Drive). O resto do sistema não sabe qual fornecedor é.
- **Provider-Disc:** folha da árvore RAID = 1 provider/conta. Participa de um só nó.
- **Group:** nó RAID = REPLICA (cópia em todos os filhos) ou MERGE (um filho, o mais vazio). Compõe recursivamente (ver §7.1).
- **Disco (virtual):** nó-raiz nomeado que o usuário usa para guardar arquivos; qualquer composição de Provider-Discs e Groups.
- **REPLICA / MERGE:** os dois operadores de Group — redundância vs. capacidade.
- **Failure domain:** unidade que falha junto. No ShardSphere = **a conta**; réplicas de um chunk nunca compartilham conta.
- **Chunk:** fragmento de 16MB, comprimido + cifrado, endereçado por hash SHA-256.
- **Manifest:** mapa cifrado (chunks → arquivos → providers). Vive no cliente; backend só guarda como blob opaco.
- **Core:** o motor client-side (compress/encrypt/chunk/RAID/hash), reutilizável.
- **Master Key (MUK):** chave derivada (Argon2id) de Senha + Secret Key; nunca sai do cliente; cifra o Key Bundle.
- **Secret Key:** chave de alta entropia gerada no onboarding, entregue no Emergency Kit (PDF). Um dos 2 fatores (com a Senha).
- **Emergency Kit:** PDF com a Secret Key que o usuário guarda. Sozinho não decifra nada (falta a Senha).
- **Key Bundle:** conjunto cifrado (Data Key, Manifest Key, Token Key) guardado no backend como blob opaco.

## 16. Como continuar (próximo passo concreto)

### Fase 0 — MVP front-only, SEM backend (DECIDIDO)

Primeiro milestone executável: **só o `frontend/`**, sem backend.
- React: 1 tela (input de arquivo + submit + log).
- `core/`: arquivo → **comprime → chunka (16MB) → cifra-por-chunk** → **commita direto no GitHub** (Git Data API, batch) com PAT do `.env` → gera manifest local.
- Sem backend (GitHub é browser-direto, CORS ok). Backend entra só na fase seguinte (manifest store + providers CORS-bloqueados).
- **Config via `.env` do React** (PAT + repo). OK para esta fase: cada usuário **self-hosta** o próprio build com o próprio `.env`/próprias contas (o PAT no bundle é o token do próprio usuário, na própria máquina). Vale inclusive para os ~5 colegas que vão usar (cada um com seu `.env`). Vira `.env`→banco+login só quando for app hospedado multi-usuário.
- **Faseamento:** rodar primeiro com **1 repo / 1 conta** (prova o pipeline ponta a ponta); depois ligar o **RAID engine + as 4 contas** (topologia abaixo).
- **Simplificações temporárias do MVP** (atrás de interfaces, swap trivial depois): compressão **gzip nativo** (`CompressionStream`) no lugar de zstd; derivação de chave **PBKDF2 nativo** no lugar de Argon2id; só **passphrase → Data Key** (o 2SKD/Key Bundle/Emergency Kit completo vem depois). Tudo isso reverte para o que diz §9/§8 nas fases seguintes.

**Topologia de teste do MVP (DECIDIDO):** 4 contas GitHub, **RAID-10** =
```
Disco (MERGE)
 ├─ Mirror A = REPLICA(GH1, GH2)
 └─ Mirror B = REPLICA(GH3, GH4)
```
Capacidade ~16GB úteis (GH ~10GB × 0.8 = 8GB; mirror = min = 8GB; merge = soma). Exercita REPLICA, MERGE, self-heal (matar GH1 → rebuild do GH2) e failure-domain (4 contas).

**Passos:**
1. Formalizar a base da V1 (rewrite) e desenhar a interface `Driver` (`Put/Get/Delete/Exists/Health/Capacity`).
2. Implementar o **`core/`** (TS puro): pipeline **zstd → AES-256-GCM → chunk 16MB → manifest**, derivação de chave **Argon2id** (2SKD), hash **SHA-256**.
3. Implementar o **GitHub driver** (browser-direto; auth via fine-grained PAT na V1; troca OAuth via backend depois).
4. Implementar o **RAID engine** com REPLICA + MERGE + placement por chunk + failure-domain por conta.
5. Backend **Go** mínimo: guardar manifest/Key Bundle (blobs opacos) + health. (Proxy só quando entrar provider CORS-bloqueado.)
6. Rodar a topologia RAID-10 de 4 contas ponta a ponta: upload → 2 cópias por chunk → matar 1 conta → self-heal.

---

## 17. SNAPSHOT DE IMPLEMENTAÇÃO ATUAL (supera o §16 — lê isto)

> Estado real depois de várias sessões. O §16 acima é o plano antigo; **isto aqui é o que existe.**

### 17.1 O que existe e roda
- **`frontend/`** (React + Vite + TS): pipeline + RAID + 9 drivers + config por `.env`. Build verde, **15 testes offline + ~15 live** (gated `GH_LIVE=1`).
- **`backend/`** (Go, stdlib): **bridge burra** (`main.go`) — `POST /bridge {url,method,headers,body(b64)}` encaminha pro provider, allowlist de hosts/sufixos (anti-SSRF), no-log, CORS. **Não conhece provider** — lógica toda no TS. Helpers OAuth em `backend/tools/` (google/ms/box/dropbox-oauth.mjs).
- **Pipeline (`core/pipeline.ts`):** `packBytes` = comprime(gzip) → **chunk adaptativo** (comprimido/stripes, cap 16MB) → cifra-por-chunk (AES-256-GCM, nonce‖ct‖tag, id=sha256). `unpackToBytes` = inverso, verifica id + tag. `runUpload`/`runRestore`.
- **RAID (`core/raid.ts`):** `RaidDriver(children, policy)`, policy `replica` | `merge`, **aninhável**. replica = `allSettled` (resiliente, só falha se TODOS caírem; degradado → self-heal). merge = distribui por **hash do id** (`parseInt(id[0:8],16)%n`), **paralelo** (`Promise.all`; toggle `sequentialMerge` p/ A/B). `getChunk` = fallback em ordem (email/lento por último). `rebalance()` = self-heal (re-replica de cópia viva). `exists()`. `get stripes` (nº de filhos do merge no topo).
- **Wrappers:** `RetryDriver` (5s × 3, rate limit/5xx), `LimitDriver` (cap bytes/disco, `VITE_DISK_MAX_BYTES`). Config envolve cada folha: `Limit(Retry(driver))`.
- **Topologia (`core/config.ts`):** `buildDriverMap` lê `.env` → mapa de drivers nomeados; `VITE_TOPOLOGY` (JSON) → `buildNode` monta árvore aninhada. Sem topologia = RAID flat.
- **Nomes (`core/names.ts`):** 50 pássaros pro futuro ContainerManager.

### 17.2 Os 9 drivers (todos byte-exato, live-testados)
| id | arquivo | empresa | GB | auth | transporte |
|---|---|---|---|---|---|
| github1/2 | `github.ts` | GitHub | ~10 (multi-repo) | PAT Bearer | **direto** |
| gitlab | `gitlab.ts` | GitLab | 10/proj | PRIVATE-TOKEN | bridge |
| codeberg | `codeberg.ts` | Codeberg | **100MiB priv** | token | bridge |
| gdrive | `google.ts` | Google | 15 | OAuth2 refresh | bridge |
| b2 | `b2.ts` | Backblaze | 10 | keyId+appKey estático | bridge |
| filebase1/2 | `s3.ts` | Filebase | 5 ea | S3 SigV4 estático | bridge |
| dropbox | `dropbox.ts` | Dropbox | 2 (×5=10) | OAuth2 refresh | bridge |
| mega | `mega.ts` | MEGA | 20 | megajs (email+senha) | bridge |
| koofr | `webdav.ts` | Koofr | 10 | app-password WebDAV | bridge |
| opendrive | `webdav.ts` | OpenDrive | 5 (×2=10) | app-password WebDAV | bridge |

**Desativados (driver existe):** Bitbucket (`bitbucket.ts`, 1GB total free). **Não construídos:** OneDrive (`onedrive.ts` escrito, Azure exige directory/cartão), Box (`box.ts` escrito, OAuth redirect precisa HTTPS/domínio — adiado), Email (IMAP/SMTP precisa backend mail client; deferido).

### 17.3 Gotchas por driver (custaram caro, não reaprender)
- **GitHub:** repo vazio → 409 nos blobs; bootstrap via Contents API (`.gitkeep`) + esperar ref propagar; **Contents API corrompe binário** → ler por blob sha. Leitura por blob, escrita Git Data batch (blobs→tree→commit→ref). 6 push/min/repo.
- **GitLab:** commits API `actions:[{action:create,content b64}]`; read `/files/{enc}/raw`.
- **Codeberg/Gitea:** ChangeFiles `POST /contents files:[]`; read `/raw/`. **750MiB TOTAL conta** (privado=100MiB) — minúsculo.
- **Filebase (S3):** bucket NÃO cria via S3 PutBucket (NoSuchBucket/TooManyBuckets) → criar via `@filebase/sdk` `BucketManager.create`; depois S3 funciona. 5GB+1000 arquivos. Driver S3 serve Wasabi/R2/iDrive tb (mas esses = trial/cartão).
- **B2:** authorize(cacheado)→get_upload_url→upload; hosts dinâmicos → allowlist por sufixo `.backblazeb2.com`. **Lento no upload (medido 0.14 MB/s 1 amostra)** — talvez get_upload_url por chunk (dá pra reusar).
- **Google/Dropbox/OneDrive:** OAuth2 refresh→access. Google: ativar Drive API + publish app (drive.file não-sensível); secret só no popup de criação. Dropbox: scopes `files.content.write/read`+`files.metadata.read` ANTES de gerar token.
- **MEGA:** megajs **cifra o buffer IN-PLACE no upload → CLONAR antes** (`new Uint8Array(c.bytes)`) senão corrompe os outros membros do mirror no put paralelo; login V2 às vezes crasha → retry; `fetch` injetável (bridge). megajs +240kB no bundle → lazy-load depois.
- **WebDAV (Koofr/OpenDrive):** Basic app-password, MKCOL pasta, PUT/GET/HEAD. OpenDrive lento (0.18 MB/s).

### 17.4 Decisões de arquitetura (firmes)
- **Container 500MB uniforme** (repo git / pasta drive), nome de pássaro, cap 80%/conta — regra única, sem ifs/else (a implementar via ContainerManager).
- **Provider pequeno-confiável → merge(conta×N)** = 1 disco maior (Dropbox ×5=10, Filebase ×2=10, OpenDrive ×2=10). Mesmo app OAuth serve N contas.
- **Quorum-ACK + hinted-handoff (Cassandra)** no write: não bloqueia esperando todas as cópias; ACK ao atingir quorum (W garante cobertura cross-domínio), resto vira **hint** numa fila → repair async. Server = fila opaca + pacer (cego). (A IMPLEMENTAR — hoje é Limit(Retry) síncrono.)
- **Server burro:** bridge proxy + (futuro) SSE/pacer de hints. Front **offline-first** (PWA/Service Worker agenda heal). Inteligência no cliente.
- **Confiança:** comercial > doação/non-profit (Codeberg, Disroot = só redundância). **Escala:** signup SEM telefone (Yandex/Microsoft fora).
- **Distintos:** cada empresa = 1 disco (mesmo multi-conta por dentro). Replica = empresas diferentes.

### 17.5 Métricas medidas (NÃO refazer suposição)
- **Paralelo ajuda EM ESCALA (revisado):** A/B 15MB (+MEGA) = 0.76× (pequeno demais, overhead). A/B **100MB (3×2, sem MEGA) = 1.66×** (seq 84s vs paralelo 50.6s). Arquivo grande tem carga pra overlap os mirrors; pequeno não. Ganho 1.66× não 3× (banda parcialmente compartilhada + disco mais lento gate). **Write atual ~2 MB/s** (100MB/50.6s).
- **Per-disco lentos:** b2 (0.14 up), opendrive (0.18) — **não MEGA** (0.6, mid). Rápidos: gdrive 1.56, gitlab 1.28, koofr.

#### Tabela por provedor (bench 5MB, 1 amostra — ruído alto; cap = teto teórico free tier)
| disco | cap | up MB/s | up s | down MB/s | down s |
|---|---|---|---|---|---|
| b2 | 10 GB | 0.14 | 36.5 | 2.27 | 2.2 |
| opendrive | 5 GB | 0.18 | 27.4 | 0.19 | 26.5 |
| mega | 20 GB | 0.6 | 8.3 | 1.25 | 4.0 |
| filebase1 | 5 GB | 0.69 | 7.2 | 3.13 | 1.6 |
| dropbox | 2 GB | 0.78 | 6.4 | 2.38 | 2.1 |
| github1 | 10 GB | 1.0 | 5.0 | 2.78 | 1.8 |
| koofr | 10 GB | 1.25 | 4.0 | 0.58 | 8.6 |
| gitlab | 10 GB | 1.28 | 3.9 | 6.25 | 0.8 |
| gdrive | 15 GB | 1.56 | 3.2 | 2.27 | 2.2 |

Fora do bench (só catálogo `DISK_PROVIDERS`): codeberg 0.1GB, filebase2/3/4 5GB, github2/3/4 10GB, gitlab2 10GB.
Notas: b2 re-medido = **instável** (0.11–1.55, spike 143s/16MB), não 0.14 fixo. WebDAV (koofr/opendrive) GET lento. Refazer com 3 amostras pra cravar.
- **Throughput topologia:** 100MB up=88.5s (1.13 MB/s file). **300MB ≈ ~5min** (sandbox) / 1-12min real (upstream do user × 3 réplicas). **OK pra backup background, ruim pra síncrono.** Alavanca = menos réplicas / mais upstream.
- **0 perda** em todos os stress/e2e.

### 17.6 Setup atual (`.env`, enxuto pra desenvolver)
`VITE_DISK_MAX_BYTES=10737418240` (10GB). Topologia = 3 mirrors de 2, **30GB úteis, 2 cópias cross-empresa**:
```json
{"merge":[{"replica":["github1","gitlab"]},{"replica":["github2","gdrive"]},{"replica":["b2","koofr"]}]}
```
Rodar: `cd backend && go run .` (bridge :8080) + `cd frontend && npm run dev`. Testes live: `GH_LIVE=1 npx vitest run src/core/liveN.test.ts`.

### 17.7 Próximos passos (ordem)
1. **Manifest com placement persistido** (chunk→container→disco, cifrado) — base de quorum/repair/restore.
2. **ContainerManager** (500MB, bird-names, cap, rollover) — usa `names.ts`.
3. **Quorum-ACK + fila de hints** (substitui Limit/Retry síncrono).
4. **Download/restore na UI** (fecha o ciclo pro usuário; `runRestore` já existe).
5. **Auto self-heal** (drena hints; `rebalance()` já existe) + **SSE/pacer** no backend.
6. **2SKD/Argon2id + zstd** (trocar MVP-temporários gzip/PBKDF2).
7. **Otimização:** lazy-load megajs; reusar b2 upload-url; 2× réplica como opção (UX).

---

## 18. Regras de sistema — status e design (sessão jun/2026)

Estado: **MVP de discos/save/restore fechado.** Implementadas as regras:
- **Quorum-ACK + backpressure + válvula adaptativa** (write): espera todos; ao bater quórum `ceil(n/2)` abre janela adaptativa (2× tempo do quórum, mín 1.5s); sobrecarga/erro → ACKa no quórum, retardatário/falho vira hint. (`raid-replica-put.ts`)
- **Hinted-handoff + self-heal ocioso** (pausa/continua): hints **cifrados duráveis** no backend (`POST/GET/DELETE /hints`); auto-heal drena quando `inflight===0`. (`hints.ts`, `heal.ts`, `use-hint-sync.ts`, `use-auto-heal.ts`)
- **Saúde do arquivo**: quadradinhos por disco (verde/vazio/vermelho) via `placement.ts`.
- **Cap 80% + indicador de cheio**: uso lógico por disco somando chunks (`usage.ts`); barra + badge CHEIO no disk-node; **upload bloqueado** se algum disco bateu 80%.
- **Health score** (observe-only): EWMA sucesso+latência por disco → bolinha de status. AINDA não muda roteamento.

### 18.1 Rollover (container) — DESIGN, não implementado
**Container** = abstração: repo (git), pasta (drive), bucket/prefixo (S3). Quando enche, cria o próximo automático (nomes de pássaro, `names.ts`), transparente.

**Limites:**
- container cheio = bytes_no_container ≥ cap_container. **Decisão: git 500MB, drive/S3 2GB** (ou 500MB uniforme).
- conta cheia = bytes_no_disco ≥ **80% × sizeGB do tipo** (regra firme; já implementado no indicador/bloqueio).
- medição: **conta os bytes que enviamos** (não a API do provider — git tem lag de GC).

**O que falta (estrutural, por isso adiado):**
1. **Interface do driver**: `putChunks(chunks, container)` / `getChunk(id, container)` — ou o driver carrega "container atual".
2. **Addressing**: chunk em `<container>/<id>`.
3. **Índice de container por disco** (cifrado no vault): `Record<diskId, Record<chunkId, container>>` — o read precisa saber em qual repo/pasta o chunk está.
4. **ContainerManager** por disco: container atual, bytes, quando rolar, criar próximo.
5. Cada driver implementa "criar container" (novo repo via API / nova pasta).

**Proteção atual (sem rollover):** upload recusa se há disco cheio (evita tempestade de hints num disco que não cura). Suficiente pro caso comum (disco pequeno), enquanto o rollover completo não é feito.

### 18.2 Próximas regras
- **Health score → ativo**: usar o score no write (deprioriza ruim) e no read (fonte mais saudável).
- **Auto-migração**: disco 🔴 prolongado → drena pra outro vivo, tira de rotação.
- **Rollover completo** (§18.1) quando for priorizado.

---

## 19. Cliente / UX / Segurança — SNAPSHOT (sessão 2026-07-03)

> Camada de cliente/produto que o §17 (engine de storage) não cobre. **Isto é o que existe.** Build verde, **47 testes offline** + ~28 live (gated). Limite **85 linhas/arquivo** imposto no build (`scripts/check-line-length.sh`).

### 19.1 Chaves / 2SKD (final — Opção B)
- **Fator 1** = senha de login (`passphrase`). **Fator 2** = `secret = phraseToSecret(frase 12 palavras BIP39)` = `base64(SHA-256(palavras.join(" ")))`.
- **Chave do vault** = `Argon2id(passphrase, salt = SHA-256(secret)[:16])` (`core/crypto/vault-key.ts`). Lê v1 legado (SHA-256(pp)) e migra na próxima gravação.
- Login seta `passphrase`; onboarding gera a frase → `secret`. Login de conta existente sem frase na sessão → rota `/verify` (digita a frase → deriva o secret).

### 19.2 secureStorage (at-rest) — `core/secure-crypto.ts` + `core/secure-storage.ts`
- Chave-mestra **AES-GCM não-extraível** no **IndexedDB** (`generateKey(..., extractable:false)`); JS usa pra cifrar/decifrar mas **não lê os bytes** da chave.
- Persistente (`localStorage` `ss-enc:`): **`ss-auth`** (token JWT), **`ss-secret`**, **`ss-recovery-phrase`**. Só-sessão (`sessionStorage` `ss-senc:` via `secureSetSession`): **`ss-passphrase`** (senha de login — morre ao fechar aba). **Nada plaintext.** `MIGRATE` + migração da senha plaintext legada 1× no boot.
- `initSecureStorage()` roda em `main.tsx` (IIFE async, TLA não suportado no target) **antes** de montar os stores.
- **Logout (`clearSession`)** limpa só a sessão (senha + vault carregado) e **mantém** secret+frase cifrados → re-login sem /verify. **`clearKey`** (esquecer/reset) apaga tudo. **Troca de conta**: `ss-owner` (email) diferente no login → `clearKey`.
- **Por quê**: antes senha + secret + frase eram plaintext = 2SKD colapsava at-rest. Agora tudo cifrado.
- ⚠️ Não protege de XSS ativo (memória) — inerente a app client-side. Ganho = at-rest + anti-exfiltração da chave.

### 19.3 Recuperação — Config / PDF
- **Config atrás de `PasswordGate`** (re-confirma a senha de login; expõe as chaves). **Reset do vault** exige senha no diálogo (blinda "apagar os discos").
- `RecoveryKey`: frase em **2 linhas (6+6)**, Copiar + **Baixar PDF**.
- **PDF branded** (`lib/recovery-canvas.ts` + `recovery-pdf.ts` + `recovery-layout.ts`): canvas com logo + grid → **JPEG** embutido como image XObject (`DCTDecode`) num PDF A4 **sem lib nova** + **camada de texto invisível (Tr 3)** de 2 linhas → **copiável**.

### 19.4 Fonte de sync (ingest read-only — só gDrive por ora)
- **Nó `source`** no builder (como o PC): origem read-only que alimenta um **grupo** (merge/replica), **nunca um disco direto** (bloqueado no `onConnect`). `graphToTopo` **exclui** nós source + arestas da árvore de storage → read-only nunca recebe chunk.
- Sidebar: seção "Fontes de sync" (cards verdes tracejados, nome+tamanho via `providerType`); inspector `SourceProps` (nome, pasta-alvo, credenciais, reverificar read-only, "Sincronizar agora").
- **Engine** (`core/sync/engine.ts` + `gdrive-api.ts`): `accessToken` → `listGDriveFiles` (paginado) + `listGDriveFolders` → baixa o novo (pula ingeridos + docs Google nativos) → `runUpload` na `targetFolder`, **preservando subpastas** (`resolvePath`), concorrência 3. `useRunSync` reverifica read-only ANTES.
- ⚠️ **DÍVIDA**: `checkGDriveScope` hoje aceita `drive.file` (ESCRITA) como readonly — **adiado corrigir** (mexer obriga re-preparar a chave). Ideal: só `drive.readonly`.

### 19.5 Arquivos / UX
- **Pastas** path-based (`core/folders.ts`): `folder` no entry; breadcrumb, criar/deletar, navegação por search-param.
- **Preview**: `PreviewSheet` lateral com slide + X; thumbs precomputados no upload (`lib/thumbnail.ts`); viewers img/video/audio/pdf/txt/placeholder.
- **Upload drag-drop em QUALQUER lugar** (`components/upload-drop.tsx`): overlay "Solte para enviar"; botão **Upload** no topo (`upload-button.tsx`, input multiple); **multi-arquivo** sequencial; **folder-drag** preserva subpasta (`lib/collect-files.ts` via `webkitGetAsEntry`). Mutation em `hooks/use-upload-mutation.ts` (placeholder otimista, `status: uploading|failed`).

### 19.6 Design / marca
- **Tema** dark/light (`theme-store`, vars CSS runtime; tokens `--color-ss-*`). Design do `shardsphere-design-system.html` (teal `#14B8A6`).
- **`<Logo/>`** (`components/brand/logo.tsx` + `logo-svg.ts`): disco fragmentado + cadeado, wordmark + tagline "Fragmented. Encrypted. Unified.", theme-aware. TopBar (top-left, `size 50`) + login. Geometria compartilhada com o canvas do PDF.

### 19.7 Perf
- `useDriver`: **cache de módulo** — os ~6 consumidores compartilham 1 bundle enquanto os refs de store não mudam (era 6 driver-maps).
- Selectors zustand fatiados (TopBar); cuidado com selectors `?? []`/`?? {}` (retorno de ref nova = loop de re-render no zustand v5; usar default fora do selector).

### 19.8b Striping do merge (RAID-0 por largura) — 2026-07-03
- **Antes**: pack streaming fatiava em blocos fixos de 16MiB → arquivo pequeno = 1 chunk → merge mandava por **hash** pra 1 branch → um provider tinha o arquivo TODO, sem paralelismo.
- **Agora**: chunk = `min(16MiB, ceil(fileSize/stripes))` (stripes = largura do merge, desce pelo replica-PC); **round-robin por `order`** (`order%n`) → 1 stripe por branch, paralelo. **Piso 1MiB** (`VITE_SPLIT_MIN_BYTES`): abaixo não divide. Read `getChunk(id, order)` roteia por `order%n`; delete = todos os branches. Perfil deles é **replicar** (small files); split = **segurança anti-reconstrução** (nenhum provider tem o arquivo inteiro). Erasure segue descartado ([[erasure-ruled-out]]). Testes: `raid-stripe.test.ts`.

### 19.8 Testes novos
`folders`, `refcount`, `resolvePath` (`gdrive-api.test`), `runSync` (`engine.test`, fetch+pipeline mockados), `from-graph` (exclusão de source). Total **47 offline**.
