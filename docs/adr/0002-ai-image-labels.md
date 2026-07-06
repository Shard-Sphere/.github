# ADR 0002 — Classificação de imagens (labels por exemplo, CLIP on-device)

Status: aceito · Data: 2026-07-05

## Contexto

Documentar imagens de forma zero-knowledge: só no cliente, no **upload** (tem o plaintext), free.
O usuário quer labels **próprios** ("Lory") criados **por exemplo** (dá 3 fotos), com fallback pra
labels genéricos e "nada" se incerto. Reclassificar imagens já enviadas deve ser barato.

## Decisão

**CLIP on-device** (`@xenova/transformers`, ONNX, WebGPU→WASM; modelo ~90MB baixado 1× do HuggingFace —
só o modelo, nenhum dado do usuário sai). É o único free que faz **few-shot por exemplo** E **zero-shot genérico**.

Módulo isolado **`frontend/src/ai/`** (peer de `core`/`presenter`), lazy-load, sem depender de presenter;
`core` não depende de `ai`. Saída = só **dado** gravado na entry.

### Estrutura

```
src/ai/
  index.ts            API pública (analyzeImage, embedText)
  pipeline.ts         cascata: custom → genérico → nada
  types.ts            ImageAnalysis { embed?, labels? } · Label { name, prototype, exampleIds }
  clip/model.ts       lazy-load do CLIP (singleton)
  clip/embed-image.ts imagem/thumb → Float32Array
  clip/embed-text.ts  texto → Float32Array (cacheado)
  classify/vocab.ts   lista genérica ("cão, gato, praia, print, documento…")
  vector/quantize.ts  float32 ↔ int8 base64 (encolhe o vetor p/ ~512 bytes)
  vector/cosine.ts    similaridade
```

### Modelo de dados

- `VaultEntry.embed?: string` — CLIP vector int8 base64 (~512 bytes). Guardado em toda imagem → reclassificar depois é só recomparar (sem re-baixar).
- `VaultEntry.labels?: string[]` — labels aplicados.
- **`labels` blob** nos discos (index driver, `ss-index`, replica): `Label[] = { name, prototype:number[], exampleIds:string[], threshold? }`. Pequeno; carregado no upload (classificar) e na UI de treino.

### Fluxo (só no upload, imagem, IA ligada)

```
embed = embedImage(thumb)
1) custom:  p/ cada Label → cosine(embed, prototype) ≥ T_custom → aplica
2) genérico (se vazio): cosine(embed, embedText("a photo of a "+w)) p/ w in vocab → top ≥ T_generic → aplica
3) senão → labels = []
grava entry.embed (sempre) + entry.labels
```

### Área de treino / criar labels

Rota `/labels` (presenter): lista labels · criar (nome) · adicionar exemplos (escolhe imagens do vault) →
recomputa `prototype` = média normalizada dos embeds dos exemplos · **Reclassificar** → recomputa
`labels` das entries usando o `embed` salvo (barato; batch por pasta, background).

## Fases

1. `ai/` scaffold + CLIP wrappers + vector utils + `analyzeImage` (dep `@xenova/transformers` lazy).
2. Upload grava `entry.embed` (toda imagem) — sem classificar ainda.
3. `labels` blob nos discos + cascata de classificação no upload (`entry.labels`).
4. Rota `/labels` (criar/exemplos/prototype) + reclassificar.
5. (busca — decidida depois; reusa `embed` + cosine.)

## Notas / honestidade

- Limiares (`T_custom`, `T_generic`) precisam **calibração**; few-shot com 3 exemplos **erra às vezes** → UI deve deixar **corrigir** (melhora o protótipo).
- Só imagem; EXIF/pHash/OCR ficam fora deste ADR (podem entrar no mesmo `ai/` depois).
- Modelo baixa do HF CDN (público). Offline total = self-host do modelo (futuro).
