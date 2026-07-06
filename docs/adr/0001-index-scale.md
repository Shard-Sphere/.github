# ADR 0001 — Escala do índice (10TB+ de arquivos)

Status: aceito · Data: 2026-07-04

## Contexto

Vault hoje = 1 blob cifrado no Postgres com TUDO (`VaultData`: topologia, diskConfigs, `entries[]` com `manifest` inline, `containers[].index`). Login busca+decifra+parse+segura+renderiza o blob **inteiro**; cada save re-envia inteiro → **O(N total)**. Quebra cedo: `encrypt()` estoura no `String.fromCharCode(...spread)` (~dezenas de MB), PUT capa em **200mb** (`main.ts`), parse/render/memória morrem depois.

**Goal:** enviar arquivos até a capacidade dos discos (**10TB+**), com login rápido e navegação fluida.

## Decisão

Separar **root pequeno** (O(discos), sempre carregado) do **índice pesado** (O(arquivos), paginado e content-addressed). O índice pesado é guardado como **payload do próprio RAID** numa pasta `.shardsphere` nos discos (replicado/healado de graça). v2 mantém o root no Postgres; v3 move tudo pro disco (backend sai).

### Modelo de dados

- **Root** (blob pequeno; Postgres no v2 → `.shardsphere/root` no v3): topologia, diskConfigs, árvore de pastas, **write-state por disco** (`container.current/used` + **agregado de bytes por disco**), ponteiro pra raiz do índice. **O(discos).**
- **Entry** (índice de browse, paginado): `{ id, folder, fileName, mime, size, thumbRef?, uploadedAt, manifestId, status? }`. **Leve** — sem chunks.
- **Manifest** (objeto separado, content-addressed por `id = sha256(manifest)`): `{ version, fileName, originalSize, compression, cipher, kdf, chunkSize, driver, chunks:[{ id, order, size, containers:{ diskId: container } }] }`. Buscado **lazy** só ao baixar. **chunk→container dobrado aqui** → mata o `containers[].index` global.
- **Páginas do índice**: árvore de pastas (pasta = nó); pasta gigante vira B-tree interno.
- **Oplog** (`.shardsphere/oplog`, v3): ops append-only p/ concorrência multi-device.

### Resolvendo os scans do vault inteiro (o nó do problema)

Hoje `usage`/`refcount`/`backfill` varrem `chunks` de todos os entries. Com manifest fora do root, isso seria O(N) fetches. Resolução:

- **usage (bytes/disco p/ o cap de 80%)** → **agregado mantido** por disco no root (incrementa no upload, decrementa no delete). **Sem scan.** (`container.used` já existe; formaliza como fonte de verdade.)
- **refcount (dedup cross-file no delete)** → **descartar dedup cross-file** no modelo paginado: cada manifest **é dono** dos seus chunks; delete = apaga os chunks do próprio manifest. Sem refcount global, sem scan. (Ganho de dedup é marginal p/ mídia; reavaliar depois com um índice de refcount próprio se fizer falta.)
- **backfill (rebuild de disco)** → inerentemente O(chunks), mas roda em **background** e **streama os manifests em páginas** (não segura tudo em memória).

### Migração / retrocompat

Ler entries antigos (manifest inline, `containers[].index` global) continua funcionando; migra **on-access/on-save** pro formato novo. Nunca grava vazio por cima (anti-clobber já existe).

## Fases (PDCA, cada uma isolável + build/test verde)

1. **Seams + split do manifest** — `entry.size` (denorm), acessor `manifestOf(entry)`, manifest vira blob content-addressed (`m/<id>`), chunk→container pro manifest, `usage` vira agregado, delete sem refcount global. **← começa aqui.**
2. **Entries paginados** — saem do root p/ páginas (árvore de pastas). Root guarda só o ponteiro.
3. **`.shardsphere` on-disk** — páginas viram payload do RAID (âncora `root` em path fixo + content-addressed). Backend = coordenador mínimo (seq).
4. **IndexedDB LRU + delta-sync** — navegação instantânea/offline; puxa só `seq>last`.
5. **UI virtualizada** — renderiza só o visível.
6. **Oplog** (concorrência) + **busca/labels/embeddings** (opt-in).

## v2 → v3

Root anchor migra Postgres → `.shardsphere/root`; seq do backend → oplog no disco; **credenciais dos discos = raiz** (o que o usuário fornece no login destrava tudo). Mesma árvore nas duas fases.

## Progresso (2026-07-04)

Fase 1 ciclos **1–4d feitos** (7 commits, build+test verde, sem regressão):
- 1: `entry.size` denorm → browse/list/preview sem manifest.
- 2: manifest → blob content-addressed (`m/<id>`), download lazy (`manifest-store.ts`).
- 3: `chunkCount`+`diskBytes` denorm; `usage` agregado (1º scan O(N) eliminado).
- 4a: `fileDiskHealthLight` (sem chunk ids, usa EWMA do disco).
- 4b/c: delete de arquivo/pasta sem scan cross-file (chunk cifrado por-arquivo → sem colisão); `refcount.ts` removido.
- 4d: `manifest` inline removido do root (só `manifestId`); backfill async com resolver.

**Manifests 100% fora do root.** Falta só o `containers[].index` (o outro O(chunks)).

### Ciclo 5 (pendente) — abordagem refinada

Dobrar chunk→container no manifest **quebra heal/format** (operam disco-wide sem manifest à mão) → descartado. Em vez disso: mover `ContainerState.index` do root p/ **blob por-disco `ci/<diskId>`** (cifrado), carregado **lazy** pelo `ContainerDriver` (heal/format/read/delete seguem usando o index em memória). Root fica só com `{current, used}` por disco. **Risco:** torna a construção dos drivers assíncrona + migra o index existente + heal/format/rollover dependem do load correto → **exige verificação ao vivo** (upload→rollover→download→heal). Não commitar às cegas.

> Nota: mesmo após o ciclo 5, o root ainda tem `entries[]` (O(arquivos)). Login 100% rápido só na **Fase 2** (paginar entries).

## Stress test

Adiado p/ **depois** da Fase 1 — validar o modelo **novo** (paginado) aguentando muitos arquivos, não re-provar que o monolítico quebra.
