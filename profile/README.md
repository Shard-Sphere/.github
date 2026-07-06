# ShardSphere

**Fragmented. Encrypted. Unified.**

Armazenamento distribuído **client-first** e **zero-knowledge**: transforma várias contas de serviços grátis (GitHub, GitLab, Google Drive, Backblaze B2, S3/Filebase, Dropbox, MEGA, Koofr, WebDAV…) em **um disco virtual único, cifrado e controlado por você** — via uma topologia RAID (merge + replica).

O servidor nunca vê seus arquivos nem suas chaves. Toda a inteligência roda no cliente: lê em blocos → comprime → cifra por chunk → distribui/replica → reconstrói.

---

## Repositórios

| Repo | O quê |
|---|---|
| [**shardsphere-front-end**](https://github.com/Shard-Sphere/shardsphere-front-end) | Cliente (React + Vite). Todo o RAID, cripto, índice e a UI. |
| [**shardsphere-server-end**](https://github.com/Shard-Sphere/shardsphere-server-end) | Backend mínimo (NestJS): âncora de bootstrap + auth. |
| [**.github**](https://github.com/Shard-Sphere/.github) | Este — docs de arquitetura da org. |

## Como funciona (em uma imagem)

```
arquivo → comprime → cifra por chunk (AES-256-GCM, chave 2SKD) → RAID
                                                                 ├─ merge   (estripa = capacidade)
                                                                 └─ replica (espelha = redundância)
   → chunks nos discos (contas grátis) · índice cifrado TAMBÉM nos discos · reconstrói on-demand
```

## Pilares

- **Zero-knowledge (2SKD):** a chave do vault = `Argon2id(senha_de_desbloqueio, secret)`, onde `secret` vem de uma **frase de 12 palavras**. Precisa dos dois; nada disso sai do device.
- **RAID por topologia:** você monta a árvore (merge/replica) num builder visual. Perder um disco de uma réplica? Self-heal reconstrói das cópias vivas.
- **Índice nos discos, não no banco:** listas de pasta e manifests são cifrados e gravados **nas próprias contas** (replicados). Escala pra **milhões de arquivos** — o login baixa só um root minúsculo; cada pasta carrega sob demanda.
- **Labels de imagem por IA (opcional, on-device):** CLIP roda no seu device (free, nada sai) — crie labels próprios por exemplo ("Lory") ou caia num genérico; se incerto, não rotula.

## Segurança

- Cripto no cliente, AES-256-GCM, chave por-arquivo (HKDF).
- 2º fator opt-in: senha de desbloqueio + TOTP.
- Link de compartilhamento zero-knowledge com TTL.
- Discos = contas **dedicadas**; formatar apaga o provedor inteiro, de propósito.

## Documentação (neste repo)

- [`SHARDSPHERE-FLOWS.md`](../SHARDSPHERE-FLOWS.md) — cada processo explicado (login, upload, RAID, sync, share, índice).
- [`docs/WALKTHROUGH-60MB.md`](../docs/WALKTHROUGH-60MB.md) — passo a passo de um arquivo de 60MB (com os JSONs).
- [`docs/adr/`](../docs/adr/) — decisões de arquitetura (escala do índice, labels de imagem).
- [`SHARDSPHERE-PROJECT.md`](../SHARDSPHERE-PROJECT.md) — tese e decisões.
