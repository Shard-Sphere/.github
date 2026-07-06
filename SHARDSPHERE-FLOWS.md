# ShardSphere — Fluxos do Sistema (cada processo explicado)

> Documento técnico vivo. Explica **cada processo** do ShardSphere como ele funciona **hoje**.
> Companheiro do `SHARDSPHERE-PROJECT.md` (tese, decisões, histórico). Aqui = o COMO.
> Stack: React 18 + Vite + TS + Tailwind v4 + TanStack + zustand + React Flow (frontend `frontend/`);
> Go + SQLite (backend `backend/` = auth + vault blob + hints + bridge).

---

## Índice
1. Conceitos centrais
2. **Autenticação — topologia de segurança (Google / 2SKD / gates)**
3. Vault (metadados cifrados)
4. Criptografia & 2SKD
5. Compressão (sniff por arquivo)
6. Upload (streaming, pack por bloco)
7. Modelo de escrita RAID (merge / replica / quórum)
   - 7.5 **Criação de disco — conceitos e regras** (merge/replica, tamanhos, 80%, unicidade)
8. Rollover & containers (pássaros)
9. Hinted-handoff (hints)
10. Self-heal (reparo) & SSE
11. Download / restauração
12. Placement & saúde do arquivo
13. Sync (GDrive → ShardSphere) — ingest one-way
14. Health score (EWMA) — roteamento ativo
15. Format (wipe real) & delete
16. Discos (modelo de instância dedicada)
17. Bridge (proxy CORS)
18. Topologia (builder → árvore de drivers)
19. Cap / uso
20. Números de benchmark

---

## 1. Conceitos centrais

- **Chunk**: pedaço de bloco já comprimido+cifrado. `id = sha256(bytes finais)` → **content-addressed** (mesmo conteúdo = mesmo id → dedup natural; id imutável).
- **Disco**: uma **conta/repo/bucket DEDICADO** a um provider (github, mega, s3…). Instância única com credenciais próprias (`diskConfigs[id]`).
- **Container**: subdivisão de um disco (repo-subpasta / pasta / prefixo) nomeada como **pássaro** (sparrow, falcon…). O disco cresce criando containers ao encher.
- **Topologia**: árvore RAID (nós `merge` e `replica`, folhas = discos) montada no builder, salva no vault.
- **Manifest**: mapa pra reconstruir 1 arquivo (chunks ordenados, compressão, kdf, cipher). Fica nos `entries` do vault.
- **Zero-knowledge**: o backend guarda **blobs opacos** (vault e hints cifrados client-side). Nunca vê chave nem conteúdo.

---

## 2. Autenticação — topologia de segurança (login / gates / 2SKD)
**Arquivos:** `backend-nest/src/auth/*`, `backend-nest/src/twofa/*`, `frontend/src/presenter/lib/finish-login.ts`, `stores/{auth-store,key-store}.ts`, `components/layout/{lock-screen,two-factor-gate,guest-nav}.tsx`, `core/crypto/vault-key.ts`, `core/firebase.ts`.

São **3 camadas independentes**. O backend é **zero-knowledge**: só vê identidade + blobs cifrados. Nunca vê chave nem plaintext.

### Camada 1 — Identidade (Firebase / Google)
- Único provedor: **Google**, via Firebase Authentication. Não existe email/senha próprio.
- Front: `signInWithPopup` → recebe o **`id_token`** (JWT do Firebase).
- Backend (`FirebaseVerifier`): verifica o `id_token` por **project ID** contra o JWKS público do Google (`securetoken@system.gserviceaccount.com`). **Sem service account, sem segredo.** Extrai `sub` (Google UID estável) + `email`.

### Camada 2 — Conta no nosso banco (Postgres)
- `User.googleSub` (UNIQUE) liga o Google à nossa conta. Tudo o mais (vault, hints, shares, 2FA) é nosso.
- **Invite-only:** 1º login (sub inédito) exige um **convite** válido (`Invite`, single-use). Sem convite → `401`. Backend cria o `User` + consome o convite.
- Devolve uma **Sessão** = `Session.token` (64 hex, TTL 24h) = nosso Bearer. Toda rota protegida vai com `Authorization: Bearer <token>`.
- Endpoints: `POST /auth/google {idToken, invite?}` · `POST /auth/validate-invite {code}` (checa antes do popup) · `POST /auth/logout`.

### Camada 3 — Cripto (2SKD, no cliente)
A chave do vault precisa de **DOIS fatores** (2-Secret Key Derivation):
```
vaultKey = Argon2id( senha_de_desbloqueio , salt = SHA-256(secret)[:16] )
secret   = phraseToSecret( 12 palavras )
```
- **12 palavras (frase de recuperação)** = raiz do `secret`. Geradas no onboarding, mostradas **1×**, salvas em PDF cifrado. É o fator que abre o vault **em outro device**.
- **Senha de desbloqueio (passphrase)** = 2º fator + trava a tela. Mín. 6 chars.
- Ambas persistem **cifradas at-rest** no device (`secure-storage`: AES-GCM com chave **não-extraível** no IndexedDB). Sozinha, nenhuma abre o vault.

### Amarração device ↔ conta
As chaves do device são amarradas ao **uuid** (id do nosso banco), não ao email (`localStorage ss-owner = uuid`). **Recriar a conta** (uuid novo, mesmo email) dispara `clearKey()` automático → onboarding limpo, **sem precisar limpar o browser** (`finish-login.ts`).

### Roteamento no login (`finishLogin`)
| Estado do device / conta | Rota |
|---|---|
| device tem as 12 palavras | `/builder` |
| sem palavras + conta TEM vault no server | `/verify` (recuperar: digitar a frase) |
| sem palavras + conta SEM vault | `/onboarding` (gera frase nova) |

### Gates pós-login (opt-in)
1. **Lock screen** (`lock-screen.tsx`) — se a senha de desbloqueio não está na sessão, pede. **Valida decifrando o vault de verdade** (`vaultDecrypts`) antes de destravar → senha errada = erro inline, **sem loop**.
2. **2FA / TOTP** (`two-factor-gate.tsx`) — opcional. Se ativo, pede o código **depois** de logar, 1× por sessão da aba (`sessionStorage ss-2fa-ok`). Mostra **loading** enquanto checa status (não pisca a dashboard). Aceita: código do app · **código de emergência** (single-use, marca `used`) · **frase de reset** (desativa o 2FA e libera). TOTP com janela ±90s; código antigo fora disso → rejeitado.

### Trocar senha de desbloqueio (Config)
Re-cifra o vault (que já está em memória) com a senha nova e **regrava o blob**. As 12 palavras **não** mudam; os arquivos nos discos ficam intactos; o **PDF muda** (re-baixar). A página Config fica atrás do `PasswordGate` (re-pede a senha antes de expor as chaves).

### Storage no device (secure-storage)
Tudo cifrado com AES-GCM (chave não-extraível, IndexedDB). Persistente: `ss-secret`, `ss-recovery-phrase`, `ss-auth` (token), `ss-passphrase`. Sessão (sessionStorage): `ss-locked`, `ss-2fa-ok`.

---

## 3. Índice (metadados cifrados) — root no banco, resto nos discos
**Arquivos:** `core/vault-data.ts`, `core/folder-store.ts`, `core/manifest-store.ts`, `core/index-driver.ts`, `presenter/hooks/{use-vault,use-folder-entries}.ts`.

O índice foi **quebrado em 3 camadas** pra escalar a milhões de arquivos (Fase 2) e sair do banco (Fase 3):

1. **`root`** (blob `vault` no **Postgres**) — âncora de bootstrap, O(discos):
```
VaultData = { topology, diskConfigs(+creds), folders[], diskUsage{}, entryCount, positions, graph }
```
Só isso o **login** baixa (~40KB, independe do nº de arquivos). Tem as **credenciais dos discos** — precisa delas pra alcançar os discos e ler o resto.

2. **`e/<pasta>`** (objeto cifrado **NOS DISCOS**, container `ss-index`, replica em todos) — a **lista de arquivos** daquela pasta (`VaultEntry[]` leve: nome, size, chunkCount, diskBytes, manifestId). Abrir a pasta busca só ela (`loadFolder`); id estável = `sha256("ss-e:"+pasta)` → sobrescreve.

3. **`m/<hash>`** (objeto cifrado **NOS DISCOS**, `ss-index`, replica) — o **manifest** (chunk list) de 1 arquivo, buscado lazy só ao baixar. id = `sha256(json)` (imutável).

- **Load** (`use-vault`, boot): `GET /vault/vault` (root) → decifra. 404 = novo. Falha de decifra ≠ vazio → **locked** (não sobrescreve).
- **Navegar** (`useFolder`): lê `e/<pasta>` **dos discos** (via `index-driver` = replica de todos os discos @ `ss-index`).
- **Save**: root → `PUT /vault/vault` (Postgres, sem entries); `e/`/`m/` → `putChunks` no index driver (discos).
- Uso por disco = **agregado mantido** no root (`diskUsage`, O(1) por op) — cap 80% sem varrer arquivos.

> **Único no banco = o root** (topologia + creds + pastas + uso). O JSON do índice (listas + manifests) vive **nos discos**, cifrado e replicado. Detalhe completo: [`docs/WALKTHROUGH-60MB.md`](docs/WALKTHROUGH-60MB.md). v3: mover o root p/ path fixo no disco + creds no login → **sem backend**.

---

## 4. Criptografia & 2SKD
**Arquivos:** `frontend/src/core/crypto.ts` (chunk), `crypto/vault-key.ts` (vault), `crypto/{encrypt,decrypt}.ts`.

Duas camadas:
- **Data key (chunk)** — `deriveDataKey(passphrase)`: **Argon2id** (64MiB/3 passes, hash-wasm) → AES-256-GCM. Os params vão no `manifest.kdf` (**self-describing**) → lê PBKDF2 antigo na restauração (retrocompat).
- **Vault key — 2SKD** (`vaultKey(passphrase, secret)`): chave = **Argon2id(passphrase, salt = SHA-256(secret))**. **Fator 1** = senha de login (`passphrase`, sessionStorage). **Fator 2** = `secret = phraseToSecret(frase 12 palavras BIP39) = base64(SHA-256(palavras))`, guardado **CIFRADO** (secureStorage, §22). Abre o vault em outro device com senha + frase.
  - **Versão do blob**: prefixo `"2."` = v2 (2SKD). Sem prefixo = v1 legado (SHA-256). `decryptVault` ramifica; ao ler v1 com secret presente → **migra** pra v2 no próximo save.
  - **Onboarding**: gera a **frase de 12 palavras** e **mostra** (painel RecoveryKey em Config, atrás de `PasswordGate`). Login de conta existente sem frase na sessão → rota `/verify` (digita a frase → deriva o secret). Sem a frase + a senha, o vault não abre. Ver §23.
- **Cifra**: AES-256-GCM, formato `nonce(12) ‖ ciphertext ‖ tag(16)`.

---

## 5. Compressão (sniff por arquivo)
**Arquivos:** `frontend/src/core/compress.ts`.

- Compressores plugáveis: **`gzip`** (nativo `CompressionStream`, streama) e **`none`** (passthrough).
- **`pickCompressor(amostra)`**: comprime o 1º bloco; se encolheu pouco (>90% do original) → **`none`** (dado incompressível: mídia, aleatório → não gasta CPU). Senão **`gzip`**.
- Escolha é **1× por arquivo**, gravada em `manifest.compression`. Restauração escolhe o descompressor pelo nome (retrocompat).
- *(zstd foi avaliado e descartado: ganho marginal pro caso — rede é o gargalo, mídia não comprime — e dependência WASM extra. Fica plugável pelo mesmo campo se um dia virar feature.)*

---

## 6. Upload (streaming, pack por bloco)
**Arquivos:** `frontend/src/core/pipeline-pack-stream.ts`, `pipeline.ts` (`runUpload`), `presenter/routes/upload.tsx`.

**Streaming v2** — RAM constante (~1 lote), aguenta arquivos grandes (1GB+ testado):
1. `fileSource(file)` lê o arquivo em **blocos de 16MB** via `File.slice` (não carrega o arquivo todo).
2. **Por bloco**: `compress(algo)` → `encryptChunk` → `id = sha256` → vira 1 chunk.
3. Acumula **lote** (4 chunks) → `driver.putChunks(lote)` → libera → próximo. RAM ≈ lote.
4. Compressor escolhido por sniff no 1º bloco. Manifest v2 acumula `{id,order,size}` de cada chunk.
5. **Tamanho do bloco = stripe** (§7): `runUpload` lê `driver.stripes` (largura do merge) e usa bloco = `fileSize/stripes` (cap 16MiB, piso: não divide < 1MiB). Assim o arquivo vira N pedaços que o merge espalha 1-por-branch, **em paralelo**. Segurança (fração por provider) + velocidade (N-way).
5. `upload.tsx` bloqueia **nome duplicado** (entries únicos) e disco cheio (cap).

> v1 (arquivo antigo, compressão do arquivo inteiro) ainda **lê** — `unpackToBytes` ramifica por versão.

---

## 7. Modelo de escrita RAID (merge / replica / quórum)
**Arquivos:** `frontend/src/core/raid.ts`, `raid-put.ts`, `raid-replica-put.ts`.

A topologia é uma árvore. `putChunks` desce:
- **merge = stripe RAID-0** (`raid-put.ts`): distribui cada chunk por **round-robin do `order`** → `order % n` → 1 branch. Espalhamento **par** (nenhum branch fica com o arquivo inteiro → segurança anti-reconstrução) e leitura determinística. O read (`raid.getChunk(id, order)`) vai direto no branch `order%n`; sem `order` → tenta todos. delete → todos os branches (best-effort). **Tamanho do chunk = `min(16MiB, ceil(fileSize / stripes))`**, onde `stripes` = largura do merge (`RaidDriver.stripes`, desce pelo replica-passthrough tipo PC). Abaixo de `VITE_SPLIT_MIN_BYTES` (**1 MiB**) NÃO divide → 1 chunk inteiro num branch (evita micro-objetos). 1 branch / topo replica → não divide.
- **replica** (`raid-replica-put.ts`): grava o chunk em **vários filhos** (cópias). Modelo Cassandra:
  - Dispara todos os filhos, **ACK no quórum** `writeQuorum(n)` (default `ceil(n/2)`, configurável via `VITE_WRITE_QUORUM`).
  - **Backpressure + válvula adaptativa**: espera todos numa janela `max(2×tempo-do-quórum, 1500ms)`; só **abre a válvula** (segue sem esperar o lento) em **sobrecarga/erro**.
  - Filho que falha/estoura → vira **hint** (handoff). Filhos ordenados por **health score** (saudável primeiro).

---

## 7.5 Criação de disco — conceitos e regras (builder)
**Arquivos:** `frontend/src/core/placement.ts`, `presenter/components/sidebar/{vault-utils,use-vault-save}.ts`, `hooks/use-upload-mutation.ts`, `core/topology/*`.

O usuário monta a topologia num **builder visual** (React Flow): arrasta discos e grupos, liga com arestas → vira uma **árvore de drivers**.

- **Disco** = 1 conta de serviço grátis (git repo, Google Drive, S3/B2 bucket, WebDAV, Dropbox, Mega…). É uma **folha** da árvore. Cada disco = 1 provider + credenciais (guardadas **no vault cifrado**). Instância **dedicada**: aquela conta é 100% do ShardSphere.
- **merge = RAID-0 (stripe)** — junta N discos → **soma a capacidade**, **sem** redundância. Chunk `order` → branch `order % n`. Nenhum branch fica com o arquivo inteiro (anti-reconstrução). Perder 1 disco do merge = perder os chunks que estavam nele.
- **replica = RAID-1 (espelho)** — grava o chunk em **todos** os filhos → **redundância**. Sobrevive à perda de discos até o quórum. Capacidade = a do **menor** filho.

### Regras (validadas no save / upload)
- **Tamanhos diferentes:** replica **capa no menor** (espelho). merge **soma**. **Bloqueio-duro ÚNICO** (`replicaSmallerWithData`): adicionar um disco **menor** a uma réplica que **já tem dados** → recusa (o dado existente não caberia no menor). Réplica **vazia** aceita (todos capam no menor).
- **Regra dos 80%:** disco com uso ≥ **80%** do cap entra em `fullDisks`. No upload, `canPlace(topo, full)`: **merge** precisa de **ALGUM** ramo livre (pula o cheio, derrama pro próximo ativo); **replica** precisa de **TODOS** (espelho não pode pular). Todos cheios no caminho → **recusa** o upload.
- **Mesmo disco em 2 grupos:** **proibido** — cada disco-folha aparece **1×** na árvore (unicidade). Senão replica/merge se auto-referenciaria e a redundância seria falsa.
- **merge arity mudou** (tirar/pôr branch num merge que já tem dados) → **warning** (`mergeAritybroke`): os chunks já gravados seguem no branch antigo pelo índice container→chunk; o novo layout só vale pra escrita nova.
- **isEmpty / format:** disco novo nasce `isEmpty="1"`; o primeiro upload que o toca vira `"0"`. `format()` **apaga o container inteiro** do provider (destrutivo **de propósito** — conta dedicada).
- **synced / backfill:** ao trocar um disco por outro numa réplica, o novo entra `synced="0"` e é preenchido (backfill) a partir das cópias vivas antes de contar como fonte confiável.

---

## 8. Rollover & containers (pássaros)
**Arquivos:** `frontend/src/core/container-driver.ts`, `names.ts`, `config-drivers.ts`.

Cada folha é envolvida num **`ContainerDriver`** (transparente pro RAID/pipeline):
- O disco escreve no **container atual** (1 pássaro). Ao `used + bloco > cap` (**git 500MB / drive 2GB**) → `nextContainerName` pega o próximo pássaro livre → **rolagem**.
- **Índice chunk→container** por disco (`vault.containers[diskId].index`) → leitura acha onde está. Persistido no vault (mutado in-place, salvo no save).
- **Mapa container→endereço** por provider (`config-drivers.driverFor`): git = **subpasta no repo** (`sparrow/<id>`, `keysPath`); s3/b2 = **prefixo**; drive/dropbox/mega/webdav = **pasta**. O sistema cria a pasta/repo sozinho (github **auto-cria o repo** `ss-data` se faltar).

---

## 9. Hinted-handoff (hints)
**Arquivos:** `frontend/src/core/hints.ts`, `presenter/hooks/use-hint-sync.ts`, `backend/internal/hints/*`.

Quando uma réplica falha (disco fora/erro), em vez de travar:
1. `recordHint({chunkId, diskId})` registra "esse disco deve essa cópia".
2. **Durável + cifrado**: `use-hint-sync` cifra `{chunkId,diskId}` → `POST /hints` (backend guarda opaco). Guarda o `backendId`.
3. Boot: `GET /hints/pending` → decifra → fila local.
4. Reparado → `DELETE /hints/{id}`.

---

## 10. Self-heal (reparo) & SSE
**Arquivos:** `frontend/src/core/heal.ts`, `presenter/hooks/{use-auto-heal,use-hint-sync}.ts`, `backend` SSE.

- **Auto-heal** roda **só quando ocioso** (`activity-store` inflight=0), pausável. `healHints`: pega o chunk de uma réplica viva → grava no disco que devia → limpa o hint. Sem fonte viva → `markLost`.
- **SSE cross-device** (`use-hint-sync`): abre `GET /hints/events` (fetch+stream com Bearer — EventSource não manda header). Outro device degradou um write → backend **empurra** o hint cifrado → este device decifra → fila → repara quando ocioso. (Backend dá `Flush` nos headers no connect; o wrapper de log implementa `http.Flusher`.)
- **Backfill** (disco novo): `backfillNewDisks` gera hints dos chunks que o disco novo deveria ter → reusa o self-heal pra preencher.

---

## 11. Download / restauração
**Arquivos:** `frontend/src/core/pipeline-unpack.ts`, `pipeline.ts` (`runRestore`).

`unpackToBytes(manifest, getChunk, passphrase)`:
1. Deriva a data key do `manifest.kdf`.
2. **Paralelo** (limite 6), ordem preservada: por chunk → `getChunk(id)` → **verifica `sha256 == id`** (anti-corrupção) → `decrypt`.
3. **v2**: descomprime **por bloco** (cada chunk é independente) → concat. **v1**: concat → descomprime o stream inteiro.
4. `getChunk` no RAID: merge → roteia por hash (`mergeIndex`); replica → tenta a réplica mais saudável primeiro (cascata). No disco, o `ContainerDriver` resolve o container pelo índice.

*(Download hoje é buffered — monta o arquivo em RAM. Limite prático ~500MB/usuário. Streaming de download seria passo à parte.)*

---

## 12. Placement & saúde do arquivo
**Arquivos:** `frontend/src/core/placement.ts`, `presenter/components/file-health.tsx`.

- `chunkDisks(topo, chunkId)`: recursivo — merge→filho do hash; replica→todos; folha→[id]. Diz **quais discos deveriam ter** cada chunk.
- `fileDiskHealth`: cruza com hints pendentes/perdidos → **quadradinhos** por arquivo: verde (ok), pendente (falta enviar, não é erro), vermelho (perdido). Coluna na lista de arquivos.

---

## 14. Sync (GDrive → ShardSphere) — ingest one-way
**Arquivos:** `frontend/src/core/sync/{engine,gdrive}.ts`, `presenter/hooks/{use-sync,use-auto-sync}.ts`, `components/sidebar/{sync-sources,sync-source-form,source-props}.tsx`, `canvas/source-node.tsx`.

Fonte de sync = ingest **read-only**. O usuário configura credenciais GDrive (token OAuth **read-only**) e uma pasta-alvo no vault:

1. **+ Sync** no sidebar → form `SyncSourceForm` com `clientId`, `clientSecret`, `refreshToken`, `targetFolder`.
2. **Verify**: `checkGDriveScope` faz POST via bridge → `oauth2.googleapis.com/tokeninfo` → checa o scope (invariante: fonte nunca escreve na origem). ⚠️ **DÍVIDA**: hoje aceita `drive.file` (ESCRITA) como readonly — correção adiada (mexer obriga re-preparar a chave); ideal aceitar só `drive.readonly`.
3. Salva `SyncConfig` em `VaultData.syncs`.
4. Arrasta o nó `source` do sidebar pro canvas; conecta a um grupo (merge/replica). O nó é **visual** — `graphToTopo` filtra nós source da árvore de storage.
5. **"Sincronizar agora"** → `useRunSync.run(syncId)`:
   - `accessToken`: POST `oauth2.googleapis.com/token` (refresh → access).
   - `listGDriveFiles`: GET `/drive/v3/files` paginado (pula Google Docs nativos).
   - `listGDriveFolders`: GET `/drive/v3/files` filtrando pastas → monta mapa `folderId → {name, parent}`.
   - `resolvePath`: sobe a árvore de pastas via `parents[]` → path relativo (ex: `"Fotos/Viagem/2024"`).
   - `downloadGDriveFile`: GET `/drive/v3/{id}?alt=media`.
   - `runUpload` pra cada arquivo novo → topologia RAID → `VaultEntry` com `folder = sync.targetFolder/subpath` salva no vault.
   - Dedup por `ingestedIds` (array no `SyncConfig`).

**Auto-sync**: `useAutoSync` a cada **2.5 min** quando ocioso (`isIdle`), no `__root.tsx`. Tenta todas as fontes ativas no canvas. Erros silenciosos — retenta no próximo ciclo.

**Segurança**: bridge em `allowlist` contém `oauth2.googleapis.com` e `www.googleapis.com`. Tudo passa pelo proxy autenticado. Token é re-verificado read-only a cada ciclo.

---

## 15. Health score (EWMA) — roteamento ativo
**Arquivos:** `frontend/src/core/health-score.ts`.

- `recordOp(diskId, ok, ms)` por operação → **EWMA** (α=0.3) de sucesso+latência → `diskScore` 0–100 → estado `good/degraded/bad/unknown`.
- **Ativo no write** (`raid-replica-put`): ordena filhos por score desc (disco ruim por último; quórum bate nos bons).
- **Ativo no read** (`raid.getFromAny`): tenta a réplica mais saudável primeiro.

---

## 16. Format (wipe real) & delete
**Arquivos:** todos os drivers (`*.ts`), `container-driver.ts`, `presenter/components/sidebar/vault-actions.tsx`.

- **`format()` = wipe REAL** do provider inteiro (disco = conta **dedicada**, sem caçar prefixo):
  - github: **commit orphan** (sem parent) + force-ref → **histórico apagado** (GC do GitHub reclama espaço; só assim o espaço diminui no git).
  - gitlab/codeberg: lista a **raiz recursiva** + delete tudo. s3/b2: lista o **bucket inteiro** + delete. gdrive: todos os arquivos do app. dropbox: raiz do app. mega: filhos da raiz. webdav: PROPFIND raiz + DELETE.
  - `ContainerDriver.format()` faz **1 wipe total** (ignora containers) + zera o índice. `RaidDriver.format()` = recursivo (todas as folhas).
  - Dispara no `confirmFormat` quando um disco **novo** entra na topologia e você salva.
- **`deleteChunk(id)`** real em todos os drivers → liga no apagar-arquivo (`upload.tsx` deleteMut). No git, só reclama espaço de fato com o format (histórico).

---

## 17. Discos (modelo de instância dedicada)
**Arquivos:** `frontend/src/core/disk-config.ts`, `presenter/components/sidebar/{saved-disks,disk-form}.tsx`.

- Cada disco = instância única `disk-<rand>`, `diskConfigs[id] = { provider, name, ...creds }`. O **sistema gere os repos** (sem campo repo).
- **+ Disco** (Meus discos) abre form: escolhe o **TIPO** de provider (`PROVIDER_TYPES`: github/gitlab/codeberg/gdrive/b2/filebase/dropbox/mega/koofr/opendrive) + credenciais + nome.
- Na lista só aparecem discos **não-em-uso** (unicidade). Disco em uso some da lista. Lixeira com modal.

---

## 18. Bridge (proxy CORS)
**Arquivos:** `frontend/src/core/bridge.ts`, `backend/internal/bridge/bridge.go`, `internal/server/router.go`.

- Providers CORS-bloqueados no browser → `makeBridgeFetch` empacota a request → `POST /bridge` (backend) → backend refaz pro provider → devolve. Drop-in de `fetch`.
- **Autenticado**: `POST /bridge` atrás do authMW; `makeBridgeFetch` injeta `Bearer`. Proxy fechado.
- **Allowlist** por host/sufixo (gitlab, codeberg, googleapis, dropbox, koofr, opendrive, `.backblazeb2.com`, `.filebase.com`, **`.mega.co.nz`/`.mega.nz`**…). Host fora = 403.
- github usa fetch **direto** (api.github.com tem CORS ok) — não passa pela bridge.

---

## 19. Topologia (builder → árvore de drivers)
**Arquivos:** `frontend/src/presenter/routes/builder.tsx`, `core/topology/from-graph.ts`, `config.ts` (`buildNode`), `presenter/hooks/use-driver.ts`.

- React Flow: nós **PC** (emite o arquivo), **merge**, **replica**, **disco**. Arestas ligam. Linhas pontilhadas animadas (pacotes).
- `graphToTopo(nodes, edges)` → árvore `TopoNode`. `buildNode(topo, map)` → `RaidDriver` aninhado (folhas = `ContainerDriver(RetryDriver(driver real))`).
- `use-driver` monta o `map` (de `diskConfigs` + containers do vault) e o `root` driver. Undo/redo (`topology-store` past/future). Salva graph completo (round-trip sem perda).

---

## 20. Cap / uso
**Arquivos:** `frontend/src/core/usage.ts`, `presenter/hooks/use-disk-usage.ts`, `routes/dashboard.tsx`.

- `diskUsageBytes(topo, entries)`: soma bytes dos chunks **via placement** (não query do provider) — uso = o que o vault sabe.
- Cap = 80% × tamanho do disco; cheio → upload bloqueado + indicador "CHEIO". (Com rollover, o disco cresce em containers; o cap-duro é mais um teto de conta.)
- **Painel** (`/painel`): em-voo, reparo pendente, perdidos, e por disco: health score, uso, sync pendente. **Notificações** (toasts): disco cheio, sync concluída, chunk perdido.

---

## 21. Números de benchmark (medidos)

**Pipeline cliente (CPU, sem rede)** — 1GB aleatório, contadores no lugar dos discos:
- 271 MB/s (sniff+AES-256-GCM+sha256). RAM no pico ≈ 1 lote (~64MB) — streaming **não** segura o arquivo todo.

**Upload REAL** (rede, via bridge, topologia merge-de-replicas, 5 discos: github1, gitlab, gdrive, b2, koofr):
- 1GB / **672s (~11min)** com blocos de 16MB (64 chunks) → **~1.5 MB/s** original · ~2.3 MB/s bytes reais.
- Por disco (do índice): github1 400MB/25, gitlab 400MB/25, koofr 352/22, gdrive 272/17, b2 112/7.
- **~5 GB/h** de dado do usuário (variável ±25%; free tiers podem throttlar em uso longo).
- **Gargalo = rede dos providers grátis**, não CPU nem RAM. Disco lento/instável (b2 deu timeout) puxa pra baixo → resiliência (quórum+hints) absorve.

---

## 22. Armazenamento seguro no browser (secureStorage)
**Arquivos:** `core/secure-crypto.ts`, `core/secure-storage.ts`, `main.tsx`.

- Chave-mestra **AES-GCM não-extraível** no **IndexedDB** (`generateKey(..., extractable:false)`) — JS usa pra cifrar/decifrar, **nunca lê os bytes** da chave.
- `initSecureStorage()` roda em `main.tsx` (IIFE async — TLA não suportado no target) **antes** de montar os stores: carrega a chave, decifra `ss-enc:*` pro cache em memória, e **migra** o plaintext legado (`ss-auth`, `ss-secret`, `ss-recovery-phrase`) 1×.
- API: `secureGet(k)` (sync, do cache); `secureSet(k,v)` **persistente** (`localStorage` `ss-enc:k` — token, secret, frase); `secureSetSession(k,v)` **só-sessão** (`sessionStorage` `ss-senc:k` — senha de login, morre ao fechar aba); `secureRemove(k)` (limpa os dois backings).
- Consumidores: **auth-store** (token JWT via `createJSONStorage` custom), **key-store** (`ss-passphrase` sessão, `ss-secret`+`ss-recovery-phrase` persistente). **Nada em plaintext** — nem a senha.
- **Logout vs esquecer**: `clearSession` (botão Sair) limpa só a **sessão** (senha + vault carregado) e **MANTÉM** secret+frase cifrados no device → re-login não pede /verify. `clearKey` (esquecer chaves / reset) apaga **tudo**. **Troca de conta**: login compara `ss-owner` (email) — se mudou, `clearKey` força /verify na conta nova.
- **Motivo**: antes senha (sessionStorage) + secret + frase eram plaintext = 2SKD colapsava at-rest. Agora tudo cifrado. Não defende de XSS ativo (memória) — inerente a client-side; ganho = at-rest + anti-exfiltração da chave.

## 23. Recuperação (frase, PDF, gates)
**Arquivos:** `components/recovery-key.tsx`, `password-gate.tsx`, `routes/settings.tsx`, `lib/recovery-{canvas,pdf,layout}.ts` + `download-recovery-pdf.ts`.

- **Config** inteira atrás de `PasswordGate` (re-confirma senha de login; a tela expõe as chaves). **Reset do vault** pede senha no próprio diálogo (blinda "apagar os discos").
- `RecoveryKey`: mostra a frase em **2 linhas (6+6)** + Copiar + **Baixar PDF**. A frase persiste **cifrada** (secureStorage) → sobrevive ao F5.
- **PDF branded**: `drawRecoveryCard` desenha num `<canvas>` (logo + 12 palavras + kit 2FA + avisos) → **JPEG** → `buildImagePdf` embute como image XObject (`DCTDecode`) num PDF A4 **sem lib nova**, + **camada de texto invisível (`Tr 3`)** copiável.
  - **Seleção correta:** texto visível e camada invisível ficam **alinhados à esquerda** (mesma origem x) e a camada usa **Courier (monospace)** dimensionada por bloco (`px_visível × sx`) → o avanço casa com o mono visível → seleção 1:1, **sem truncar** (bug antigo: frase de reset centralizada na imagem mas invisível à esquerda → cópia perdia o começo).
- **Kit 2FA no PDF/Config:** quando o 2FA está ativo, o PDF inclui os **10 códigos de emergência** + a **frase de reset** (via `/2fa/kit`). 2FA **desativado** → `kit()` devolve `null` → PDF **sem** as chaves mortas. Config também mostra o kit ("Ver códigos de emergência", `select-all`).
- **Gates de acesso** ver §2: `lock-screen` (senha de desbloqueio, valida decifrando o vault) + `two-factor-gate` (TOTP opt-in pós-login).

## 25. Shared link (transfer link híbrido, brokered pelo backend)
**Arquivos:** back `backend-nest/src/share/{share.service,staging.service,share.controller}.ts`, front `core/share.ts`, `components/share-dialog.tsx`, `routes/share-view.tsx`.

Link de leitura pra mandar arquivo/pasta a quem **não tem conta**. Os discos privados **não são expostos** — o backend guarda uma cópia **cifrada por-link** numa **staging** própria, temporária.

**Criar** (dono): `ShareDialog` → pra cada arquivo `runRestore` (decifra do RAID no cliente) → gera **chave AES-GCM random por-link** → **re-cifra** o conteúdo (`AES-GCM(linkKey, bytes)`, formato `iv‖ct‖tag`) → `POST /share {ttlDays, manifest, linkKeyB64, blobs[]}`. Backend (`ShareService.create`):
- valida cota: **≤10 shares ativos/user**, **≤100MB por share**, **TTL 1–30 dias** (default 7).
- `id` random 128-bit (base64url). Blobs vão pra **staging no filesystem** (`StagingService`: `staging/<id>/<idx>`).
- a **linkKey** é embrulhada com a **master do servidor** (`CryptoService`, `.crypto_master`) e guardada no registro `Share` (Postgres). Link = `origin/s/<id>`.

**Abrir** (destinatário, sem conta — rota **pública**, fora do layout autenticado):
- `GET /s/:id/meta` → só **nomes/tamanhos** (nunca a chave).
- `GET /s/:id/file/:idx` → backend desembrulha a linkKey com a master, **decifra** o blob da staging (`decryptRaw`) e **streama o plaintext** (attachment). O backend vê o conteúdo COMPARTILHADO só em runtime (de propósito); **nunca** vê a chave dos discos privados.

**Tempo limitado + reset a cada batida:** o `Share` tem `expiresAt`. **GC on-access, sem cron**: `gc()` roda **a cada batida no link** (tanto no create quanto em todo read, dentro de `load()`) — varre os vencidos e **apaga registro + staging**. Um link vencido dá `404` e leva o material embora no primeiro acesso seguinte.

---

## 24. Upload UX (drag-drop, botão, pastas)
**Arquivos:** `components/upload-drop.tsx`, `upload-button.tsx`, `lib/collect-files.ts`, `hooks/use-upload-mutation.ts`, `core/folders.ts`.

- **Drag-drop em qualquer lugar** da área de arquivos (`UploadDrop`): overlay "Solte para enviar" (contador enter/leave anti-flicker). Botão **Upload** no topo (input `multiple`).
- **Multi-arquivo** sequencial (respeita quórum/backpressure). **Folder-drag** preserva subpasta via `webkitGetAsEntry` (`collectDropped`). Solta na **pasta atual**.
- **Pastas** path-based (`folders.ts`): `folder` no entry; breadcrumb, criar/deletar, navegação por search-param. Upload otimista com placeholder (`status: uploading|failed`).

---

> **Manutenção:** ao mudar um fluxo, atualize a seção aqui. Mantenha `SHARDSPHERE-PROJECT.md` pro histórico/decisões e este pro COMO-funciona.
