# Passo a passo — arquivo de 60 MB numa topologia merge[replica(3), replica(2)]

Cobre **todo** o caminho de um upload/download: split → compressão → cifra → placement
(merge+replica) → onde cada byte e cada JSON é gravado. Reflete o código atual (Fase 3: índice nos discos).

## A topologia

```
                 merge  (stripe: divide os chunks entre os 2 ramos)
                /                        \
        replica(A,B,C)                 replica(D,E)      ← cada ramo ESPELHA em todos os seus discos
        /    |    \                      /     \
       A     B     C                    D       E
```

- **merge** no topo → `stripes = 2` (largura = nº de ramos).
- ramo esquerdo = **replica de 3 discos** (A, B, C) → 3 cópias de cada chunk que cair aqui.
- ramo direito = **replica de 2 discos** (D, E) → 2 cópias.

## Constantes (código)

- `CHUNK_SIZE = 16 MiB` (teto do bloco) · `SPLIT_MIN = 1 MiB` (abaixo disso não divide).
- `blockSize = min(16 MiB, ceil(fileSize / stripes))`.
- cifra: **AES-256-GCM**, chave por-arquivo (2SKD → `HKDF(masterKey, fileSalt)`, fileSalt aleatório por upload).
- id do chunk = `sha256(bytes_cifrados)` (content-addressed).

---

## UPLOAD — passo a passo

### 1. Decide o tamanho do bloco
`fileSize = 62.914.560` (60 MiB), `stripes = 2`.
`blockSize = min(16 MiB, ceil(60MiB/2)=30MiB) = 16 MiB`.
60 MiB / 16 MiB = 3,75 → **4 chunks**: `[16MiB, 16MiB, 16MiB, 12MiB]`, orders `0,1,2,3`.

### 2. Compressão (sniff no 1º bloco)
Vídeo/zip (incompressível) → `gzip` não ajuda (>0,9) → **`compression: "none"`**. (Texto/CSV → `gzip`.)

### 3. Cifra por chunk
Cada bloco → `AES-256-GCM(chaveDoArquivo, bloco)` → `id = sha256(cifrado)`.
```
chunk0: id=651c…  order=0  size=16.777.216
chunk1: id=3ac3…  order=1  size=16.777.216
chunk2: id=a45c…  order=2  size=16.777.216
chunk3: id=9f8e…  order=3  size=12.582.912
```

### 4. Placement — merge (stripe por order) → replica (espelho)
`chunkDisks(topo, order)`: merge manda pro ramo `order % 2`; replica espelha em todos os discos do ramo.

| chunk | order | ramo (order%2) | discos (espelho) |
|---|---|---|---|
| chunk0 | 0 | 0 → replica(A,B,C) | **A, B, C** |
| chunk1 | 1 | 1 → replica(D,E)   | **D, E** |
| chunk2 | 2 | 0 → replica(A,B,C) | **A, B, C** |
| chunk3 | 3 | 1 → replica(D,E)   | **D, E** |

Resultado por disco (só os **dados**, cifrados):
```
A: chunk0, chunk2      (32 MiB)
B: chunk0, chunk2      (32 MiB)
C: chunk0, chunk2      (32 MiB)
D: chunk1, chunk3      (28 MiB)
E: chunk1, chunk3      (28 MiB)
```
- **merge NÃO duplica** — espalha (chunk0/2 num ramo, chunk1/3 no outro).
- **replica duplica** — esquerda 3×, direita 2×.
- Total gravado nos discos = 32·3 + 28·2 = **152 MiB** p/ um arquivo de 60 MiB (custo da redundância).
- Onde exatamente: cada disco escreve em `keys/<chunkId>` dentro do seu **container atual** (git = repo `sparrow`…; drive/s3 = pasta/prefixo). Rollover ao encher (git 500MB / drive 2GB).

### 5. Grava o MANIFEST (nos discos, replica em TODOS)
Descreve como remontar. `id = sha256(json)`. Cifrado, gravado no container **`ss-index`** de A,B,C,D,E (replica total):
```json
{
  "version": 2,
  "fileName": "video.mp4",
  "originalSize": 62914560,
  "compression": "none",
  "cipher": "AES-256-GCM",
  "kdf": { "algo": "hkdf-argon2id", "fileSaltB64": "OkCtoUTFsWmBoz8owqtpKQ==" },
  "chunkSize": 16777216,
  "driver": "raid:merge[raid:replica[A,B,C],raid:replica[D,E]]",
  "chunks": [
    { "id": "651c…", "order": 0, "size": 16777216 },
    { "id": "3ac3…", "order": 1, "size": 16777216 },
    { "id": "a45c…", "order": 2, "size": 16777216 },
    { "id": "9f8e…", "order": 3, "size": 12582912 }
  ]
}
```
→ salvo em `ss-index/keys/<manifestId>` em **A, B, C, D, E** (replicado). `manifestId` = sha256 desse JSON.

### 6. Grava a ENTRY na lista da pasta (nos discos, replica em TODOS)
A entry é **leve** (sem chunks). Vai pro objeto da pasta `e/<pasta>` (ex.: pasta `videos`):
```json
{
  "id": "2078ce2f-…",
  "folder": "videos",
  "fileName": "video.mp4",
  "mime": "video/mp4",
  "size": 62914560,
  "chunkCount": 4,
  "diskBytes": { "A": 33554432, "B": 33554432, "C": 33554432, "D": 29360128, "E": 29360128 },
  "manifestId": "<manifestId>",
  "uploadedAt": "2026-07-05T…"
}
```
`diskBytes` = bytes que caem em cada disco (chunk0+chunk2 nos A/B/C = 32 MiB; chunk1+chunk3 nos D/E = 28 MiB).
→ o array de entries da pasta `videos` (`e/videos`) é cifrado e salvo em `ss-index/keys/<sha256("ss-e:videos")>` em **A,B,C,D,E** (replica). id estável → sobrescreve a cada mudança.

### 7. Atualiza o ROOT (Postgres — âncora)
Só agregados, O(discos). **Não** guarda a lista de arquivos:
```json
{
  "topology": { "merge": [ {"replica":["A","B","C"]}, {"replica":["D","E"]} ] },
  "diskConfigs": { "A": {"provider":"github","token":"…"}, … },   // credenciais dos discos
  "folders": ["videos"],
  "diskUsage": { "A": 33554432, "B": 33554432, "C": 33554432, "D": 29360128, "E": 29360128 },
  "entryCount": 1,
  "graph": { … }, "positions": { … }
}
```
→ 1 blob `vault` no Postgres. **Isto é a única coisa no banco.**

---

## Onde tudo mora (resumo)

| O quê | Onde | Cópias |
|---|---|---|
| **dados** (4 chunks cifrados) | discos, `keys/<id>` no container | esquerda 3×, direita 2× |
| **manifest** (`m/`) | discos, `ss-index/keys/<id>` | replica em todos (5×) |
| **lista da pasta** (`e/videos`) | discos, `ss-index/keys/<id>` | replica em todos (5×) |
| **root** (topologia + creds + uso) | Postgres (âncora) | 1 |

O JSON do índice (manifest + lista) está **nos discos**, cifrado, replicado. O banco só segura o root (40KB) com as credenciais — o mínimo pra bootstrap.

---

## DOWNLOAD — passo a passo

1. **Login**: baixa o `vault` root (Postgres) → tem topologia + creds dos discos.
2. **Abre a pasta `videos`**: lê `ss-index/keys/<sha256("ss-e:videos")>` **dos discos** → decifra → lista de entries. Acha a entry do vídeo (com `manifestId`).
3. **Clica em baixar**: lê `ss-index/keys/<manifestId>` **dos discos** → decifra → o manifest (a lista de 4 chunks).
4. **Remonta** (`unpackToBytes`): p/ cada chunk na ordem 0→3:
   - `getChunk(id, order)` → RAID: `order%2` decide o ramo → replica → **`getFromAny`** (tenta os discos do ramo por saúde, o mais saudável primeiro).
   - chunk0/2 → lê de A **ou** B **ou** C (qualquer cópia viva). chunk1/3 → de D **ou** E.
5. Concatena na ordem → decifra (AES-GCM) → descomprime (`none` = nada) → **os 60 MiB originais**.

### Tolerância a falha (self-heal)
- Ramo esquerdo (chunk0/2): sobrevive perder **até 2** de {A,B,C}.
- Ramo direito (chunk1/3): sobrevive perder **1** de {D,E}.
- Disco fora na escrita → vira **hint** (fila cifrada) → self-heal copia da cópia viva quando o disco volta.
- Perder um ramo INTEIRO (ex.: A,B,C juntos) → perde chunk0/2 → arquivo irrecuperável (merge não replica entre ramos). Por isso réplica dentro do ramo.
