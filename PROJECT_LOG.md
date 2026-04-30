# Bio Frankenstein — Project Log

Multi-LLM hypothesis engine over open biomedical data. Multi-domain AI portfolio for NL startup visa application alongside Wetzoek.

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
