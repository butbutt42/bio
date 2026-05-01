# Bio Frankenstein — Project Log

Multi-LLM hypothesis engine over open biomedical data. Multi-domain AI portfolio for NL startup visa application alongside Wetzoek.

## 2026-05-01 — Bio downloads + parsing + embed pipeline live

### Downloads complete (~57 GB on hh-storage `/storage/bio/`)

| Source | Size | Items | Status |
|---|---|---|---|
| ClinVar XML (`ClinVarVCVRelease_00-latest.xml.gz`) | 5.4 GB | ~2.5M variants | ✅ DONE |
| UniProt SwissProt (`uniprot_sprot.dat.gz`) | 661 MB | 570K curated proteins | ✅ DONE |
| PubMed baseline (1500 .xml.gz files) | 51 GB | ~36M abstracts (will filter to bio) | ✅ DONE |
| Open Targets parquet (diseases subdir) | 176 MB | 28K diseases | 🟡 partial — `bio-ot-download.service` continues remaining 8 subdirs |
| DisGeNET, OpenSNP | — | — | ❌ skipped (auth-gated / dead URLs) |

### Naming bug discovered & fixed
ClinVar URL renamed by NCBI: `ClinVarFullRelease_00-latest.xml.gz` → `ClinVarVCVRelease_00-latest.xml.gz` (VCV = Variation Centric View, current format since 2024). Daemon updated.

PubMed naming: was `pubmed25n*` (2025) → now `pubmed26n*` (2026 release year). Daemon hard-coded 25, replaced 335 zero-byte placeholder files, restarted with correct pattern.

### Parsing pipeline (CPU on hh-storage, streaming XML — low RAM)

`/storage/bio/parse_bio_all.py` — uses `ET.iterparse` (streaming, ~14 MB RAM regardless of file size).

| Source | Output | Items | Status |
|---|---|---|---|
| ClinVar | `parsed/clinvar.jsonl` 810 MB | 2.5M | ✅ DONE |
| UniProt | `parsed/uniprot.jsonl` 110 MB | 570K | ✅ DONE |
| Open Targets diseases | `parsed/open_targets.jsonl` 5.5 MB | 28K | ✅ DONE |
| PubMed FILTERED (MeSH bio + year≥2015 + abstract>200) | `parsed/pubmed_bio.jsonl` | ~5-8M expected | 🟢 running |

PubMed filter strategy explained at length to user:
- 36M total → filter to ~15-20% bio-relevant
- KEEP MeSH: Neoplasms, Genetics, Drug Repositioning, Mutation, Pharmacogenetics, Clinical Trials as Topic, Molecular Targeted Therapy, Precision Medicine, etc.
- SKIP MeSH: Pediatric Surgery, Public Health, Hospital Management, Diet Therapy
- Year ≥ 2015 (recent research only)
- Abstract length > 200 chars (skip editorials/letters)

### Embedding pipeline on GPU2 (RTX 4060 Ti, 16 GB VRAM)

User: "на гпу 2 пока нет юхеров и до понедельника их небудет типо можно использовать видеокарту" — GPU2 (refuge.help speech-gateway) has no users until Monday, OK to use GPU.

- **bge-m3 server** on GPU2 port 8005 (port 8003 occupied by SSH tunnel) — `bge-m3.service` systemd Restart=always
- **`bio_embed_parallel.py`** spawns N workers, each handling JSONL chunk
- **`bio_embed_chunk.py`** per-worker, batch=64, infinite retry on bge-m3 fail (NEVER zero-fills)
- **2 workers** optimal (4 was OOM with 16 GB VRAM at batch 128)
- **Per-chunk `embed_progress.txt`** for crash-resume
- **Periodic rsync */15 min** → `hh-storage:/storage/bio/embed/` (incremental, --append-verify)
- **systemd `bio-embed-clinvar.service`** Restart=on-failure (enabled, won't disrupt running)

Combined rate: 122/s (61/s × 2 workers). ClinVar 4.49M / 122 = ~10 hours ETA.

### Bug fixed: silent zero-fill corruption in HippoRAG embed

Discovered overnight that bge-m3 server died on GPU1 (after vast.ai bounce — no systemd auto-start), and `embed_entities_v2.py` silently filled failed batches with zero vectors, corrupting ~700K rows in `entity_emb.f32`. Patches applied across HippoRAG + bio pipeline:

```python
# OLD (silent corruption):
except: vecs = [[0.0] * DIM for _ in batch]

# NEW (infinite retry, no zeros):
while vecs is None:
    attempt += 1; sleep(min(60, 5*attempt))
    try: vecs = post_embed(batch)
    except: ...
    if attempt > 30: raise RuntimeError
```

Plus `/root/scan_and_repair_zeros.py` tool — for future, scan f32 for zero rows and re-embed only those rows (instead of full restart).

### Auto-recovery (5 layers per pipeline)

1. **bge-m3 systemd** Restart=always — survives vast.ai bounce
2. **Embed script infinite retry** — no silent corruption on bge-m3 fail
3. **Per-chunk progress.txt** — resume exact row on crash
4. **systemd embedder service** — on-failure restart
5. **Periodic rsync** — progress safely on hh-storage even if GPU2 dies

### Telegram alerts (correct bot 8249459459 — i online channel, NOT @prmdleadbot)

- 🧬 daemon started
- 📚 PubMed every 50 files
- 📊 Open Targets each subdir
- ✅ source done
- 🚨 bge-m3 down (>5 min cron healthcheck)
- 🎉 all complete with summary

### Final ETA (everything done)

```
ClinVar embed: ~10h → done ~02:00 UTC 2026-05-02
UniProt embed: ~2h
Open Targets:  ~5 min
PubMed bio embed: ~12-16h after parse done

ALL embeddings on hh-storage by Sunday morning
```

### Files / paths (canonical)

- Raw downloads: `hh-storage:/storage/bio/{clinvar,uniprot,pubmed,open_targets,disgenet,opensnp}/`
- Parsed JSONL: `hh-storage:/storage/bio/parsed/<source>.jsonl`
- Embeddings (during): `gpu2:/root/bio_embed/<source>/chunk_<N>/{emb.f32,ids.json,embed_progress.txt}`
- Embeddings (after rsync): `hh-storage:/storage/bio/embed/<source>/chunk_<N>/...`
- Daemon: systemd `bio-download.service` (single-file sources), `bio-ot-download.service` (Open Targets parquet), `bio-embed-<source>.service` (per source)
- bge-m3 (embedder backend): `gpu2:/root/bge_m3_server.py` :8005, systemd `bge-m3.service`
- Status check: `ssh root@hh-storage '/storage/bio/status.sh'`

### Open TODO

- [ ] After ClinVar embed done → start UniProt + Open Targets embed (2 sources, can run sequentially or parallel)
- [ ] After PubMed parse done → embed bio subset
- [ ] Build query layer / Frankenstein-bio (multi-LLM consensus over all sources) — can start now in parallel
- [ ] Decide inference deployment (most likely GPU3 with all source memmap files for cosine + cross-source bridges)
- [ ] Demo for NL universities (AMC, LUMC, Erasmus MC, UMCG, Radboud)


## 2026-04-30 — Project init

### Strategic positioning

User explicitly framed:
- **Wetzoek = startup visa primary** (legal-tech AI, B2C subscriptions in NL)
- **Promodo = €400 one-off shabashka** (handoff project)
- **Bio Frankenstein = additional traction stream + visa application strengthener**

Target visa pitch: "multi-domain AI builder shipping to NL market" instead of single-product founder. 3 distinct products under solo founder = strong IND case.

### 5 core use cases

1. **Lit-mining accelerator** — 30-sec structured answers across PubMed + ClinVar + UniProt (vs 2-4h manual)
2. **Underexplored genes finder** — high ClinVar variants + low paper count = research opportunities
3. **Drug repurposing hypothesis engine** — multi-LLM consensus over evidence chains
4. **Variant interpretation (VUS classifier)** — open-source VarSome alternative for ClinVar VUS
5. **OpenSNP correlations** — uncontested niche (OpenSNP itself shut down 2024, data still public)

### Data sources

Total raw on hh-storage `/storage/bio/`: ~28 GB. NOT 100 TB (user initial overestimate). NOT taking raw sequencing data — we're not a wet lab.

| Database | Size | Items | Use |
|---|---|---|---|
| ClinVar XML | 2 GB | 2.5M variants | Variant interpretation |
| UniProt SwissProt | 1 GB | 570K curated proteins | Function annotations |
| Open Targets | 15 GB | 1.7M gene-disease pairs | Curated foundation |
| DisGeNET | 3 GB | 30K associations | Backup curated |
| OpenSNP | 5 GB | 10K participants | Phenotype correlations |
| PubMed abstracts (cancer/genomics) | 2 GB | ~500K papers | Literature backing |

Embeddings (bge-m3 1024d × 4B): ~21 GB total for ~5M items.

### Compute schedule

GPU1 (RTX 3090 Ti, embed worker) currently busy with HippoRAG arxiv Phase 4 (~12h ETA).
GPU3 is sacred — Qdrant inference host (wetzoek, pravo, arxiv, ua_court, kamerstukken). DO NOT touch.

```
Now → ~11:00 UTC tomorrow:
  GPU1 busy with HippoRAG → bio embedding waits
  Background: download datasets to hh-storage

Tomorrow ~11:00 UTC onward:
  GPU1 free → start bio embedding (~17h on bge-m3)

Day after tomorrow morning:
  Embeddings ready → integrate query + multi-LLM consensus layer
```

### Realistic monetization (Year 1-2)

NOT €10k MRR in 30 days. IS strong startup visa angle + potential career path.

| Customer | Pricing | Y1 count | Y2 MRR |
|---|---|---|---|
| Grad students | $20-50/mo | 20-50 | $3-15k |
| Postdocs/PIs | $100-300/mo | 5-20 | $5-25k |
| Small pharma R&D | $1-5k/mo | 0-2 | $5-50k |
| Genetic counselors | $200-500/mo | 0-5 | $3-15k |
| NL university institutional | €200-500/mo | 1-3 | €1.5-5k |
| Big pharma licensing | $20-50k/year | 0 | $3-8k MRR equiv |

Combined Y2 realistic: **$20-100k MRR range**.

### NL institutions target list

5 academic medical centers + 5 universities with biomedical depts:
- AMC (Amsterdam UMC) — ~2,000 researchers
- LUMC (Leiden) — ~1,500
- Erasmus MC (Rotterdam) — ~1,800
- UMCG (Groningen) — ~1,200
- Radboud UMC (Nijmegen) — ~1,500

Plus universities with bio faculties: Wageningen, TU Delft (bioinformatics), Utrecht, VU Amsterdam, Maastricht.

### Architecture sketch (TBD when HippoRAG done)

User wants to first see HOW HippoRAG arxiv graphs work, then design bio version. Reasonable — same OpenIE + entity merging + PPR methodology, just bio entity types (gene, protein, variant, disease, drug, pathway).

```
Query → Embed → Multi-source semantic retrieval (PubMed + ClinVar + Open Targets + UniProt + OpenSNP)
  → HippoRAG-style multi-hop bridges
  → Multi-LLM consensus (5 LLMs vote independently)
  → Structured JSON output (hypothesis + cited evidence + confidence + dissenting views)
```

### TODO

- [x] Create butbutt42/bio repo (user 23:03 UTC)
- [x] Create /storage/bio/ folder structure (23:17 UTC)
- [x] Start raw downloads in background (PID 1683860)
- [x] Memory file project_bio_frankenstein.md
- [x] PROJECT_LOG.md (this file)
- [ ] Wait for HippoRAG Phase 4 (~11:00 UTC tomorrow)
- [ ] Observe graph behavior on arxiv 8M-node corpus
- [ ] Start bio embedding on GPU1 (~17h)
- [ ] Multi-LLM consensus layer (Frankenstein-bio)
- [ ] Demo MVP for NL institutions (week 2)
- [ ] Pitch deck for AMC/LUMC/Erasmus MC/UMCG/Radboud bioinformatics depts

### Key constraint reminders

🚫 NOT pivoting from Wetzoek. Bio = additive 2nd product, not replacement.
🚫 GPU3 is Qdrant inference host. NEVER run embedding workers on GPU3.
🚫 Skip raw sequencing data forever. Curated/processed only.
✅ Use bge-m3 first (proven), optimize to PubMedBERT later (v2).
✅ Match HippoRAG methodology when arxiv graph proves itself.

### Session-end downloads status (2026-04-30 23:18 UTC)

```
clinvar/   ClinVarFullRelease_00-latest.xml.gz — 0 bytes (queued, ETA ~30 min)
uniprot/   uniprot_sprot.dat.gz — 692 MB DOWNLOADED ✓
disgenet/  curated.tsv.gz — 10 KB DOWNLOADED ✓ (small file)
open_targets/ — TBD next pass (per-file fetch needed)
opensnp/   — TBD next pass (URL needs verification)
pubmed/    — TBD next pass (eutils API approach needed)
```

Background script: `/storage/bio/download_all.sh` (nohup PID 1683860)
Log: `/storage/bio/download.log`
