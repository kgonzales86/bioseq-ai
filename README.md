# 🧬 BioSeq AI

**An open-weight AI agent that turns plain-language questions into verified DNA/RNA sequence analyses.**

> "Analyze my sequence, calculate its GC content, translate it and find possible open reading frames."

BioSeq AI reads a question like that, decides which analyses are needed, runs them with tested Biopython code, and explains the results. The open-weight model runs on your own computer through [Ollama](https://ollama.com), so no sequence data leaves your machine.

Built collaboratively for **Hacktoberfest Toronto × HuskyHack 2026** as part of the **Best Open-Source AI Project** track.

## Project Team

BioSeq AI was developed as an **equal collaborative hackathon project** by:

- **Kristian Gonzales** — [GitHub](https://github.com/kgonzales86)
- **Stephanie Clark** — [GitHub](https://github.com/baldcypress-steph)
- **Kriesha Eyer**
- **Saniya Chadha**
- **Ali Al-Reda El-Zein**

The project was designed, developed, tested, and integrated collaboratively during HuskyHack 2026.

The original shared repository is hosted at:

`https://github.com/baldcypress-steph/bioseq-ai`

This repository is a fork maintained on Kristian Gonzales' GitHub profile to preserve the collaborative project, its development history, and future experimentation.

![tests](https://github.com/kgonzales86/bioseq-ai/actions/workflows/ci.yml/badge.svg)

---

## The problem

Routine sequence analysis still assumes you can write Python or use command-line tools, then interpret raw output. Chatbots look like a shortcut, but language models are bad at exactly the things sequence analysis needs: counting thousands of bases, copying long strings without errors, and translating codons.

Ask a chatbot for the GC content of a 400 bp sequence and you can easily get a confident, plausible, but incorrect number.

## How BioSeq AI solves it

**The model plans and explains. Tested tools compute. A checker verifies.**

```text
Your question ──► Open-weight LLM (Qwen3 via Ollama)
                    │  chooses tools, sets arguments
                    ▼
                  Tool registry (allowlist + argument validation)
                    │
                    ▼
                  Biopython tools ── analyze · transcribe · translate · find ORFs
                    │  exact results (JSON)
                    ▼
                  LLM writes the answer
                    │
                    ▼
                  Faithfulness checker ── every number and sequence in the answer
                    │                       must appear in a tool output
                    ▼
                  Answer + tool trace + replayable run manifest
```

What makes it more than a chatbot wrapper:

| Feature | Why it matters |
|---|---|
| **Sequence handles** | The model refers to sequences by ID (`seq1`) and never retypes DNA, so it cannot corrupt bases when calling a tool. |
| **Faithfulness checker** | Extracts every number and quoted sequence from the AI's answer and flags any that no tool produced. Hallucinated numbers are shown in the UI as warnings. |
| **Visible tool trace** | The app shows each tool the AI chose, with arguments and outputs, so users can see how the analysis was performed. |
| **Replayable runs** | Every run produces a manifest containing the question, input SHA-256, model, parameters, tool calls, and outputs. `bioseq replay` re-runs the analysis and confirms identical results. |
| **Robust harness for small models** | Validates tool names and arguments, returns errors to the model so it can retry, recovers tool calls written as text, strips `<think>` blocks, and enforces a step limit. |
| **Backend-agnostic** | Supports OpenAI-compatible endpoints including Ollama, llama.cpp, vLLM, and LM Studio. |
| **Agent Skill** | The same tools are packaged using the [Agent Skills](https://agentskills.io) standard for use with compatible agents. |
| **Benchmark** | Includes 15 tasks evaluating tool choice, factual accuracy, faithfulness, and performance across open-weight models and a no-LLM baseline. |

## Tools

| Tool | What it computes |
|---|---|
| `analyze_sequence` | Length, A/C/G/T/N counts, GC percentage (N excluded; `null` if every base is N) |
| `transcribe_sequence` | DNA coding strand → RNA |
| `translate_sequence` | Any of six frames (+1 to +3, −1 to −3), standard genetic code, optional stop at first stop codon |
| `find_orfs` | Every ATG→stop candidate in all six frames, 1-based coordinates on the input, minimum length |

All four tools live in [`bioseq_ai/bioinformatics_tools.py`](bioseq_ai/bioinformatics_tools.py), which can also run independently:

```bash
python bioseq_ai/bioinformatics_tools.py --fasta examples/demo_construct.fasta --action all
```

Input is validated so that only A, C, G, T, and N are accepted.

The application and agent accept FASTA files containing multiple records and analyze each one. The standalone script processes one record at a time.

---

## Quick start

### 1. Install

```bash
git clone https://github.com/kgonzales86/bioseq-ai.git
cd bioseq-ai

python3 -m venv .venv
source .venv/bin/activate

# Windows:
# .venv\Scripts\activate

pip install -r requirements.txt
pip install -e .
```

### 2. Get an open-weight model

Install [Ollama](https://ollama.com/download), then:

```bash
ollama pull qwen3:4b
```

Qwen3 is released under the Apache 2.0 license.

Other tool-calling models can also be used, such as:

```text
llama3.2:3b
qwen3:8b
```

Models can be configured using `--model` or the `BIOSEQ_MODEL` environment variable.

### 3. Run the app

```bash
streamlit run app.py
```

Paste or upload a FASTA file, choose an example question, and press **Run analysis**.

### Command line

```bash
bioseq ask examples/sample.fasta \
"What is the GC content of demo_construct, and does it contain an ORF?" \
--save run.json

bioseq replay run.json examples/sample.fasta

bioseq ask examples/sample.fasta "Find ORFs" --tools-only
```

### Tests

```bash
pytest -q
```

The agent loop is tested using a scripted fake model, allowing the test suite to run without requiring an LLM.

---

## Deploy on DigitalOcean

The model and the application run as separate services.

The app communicates through an OpenAI-compatible endpoint defined by `BIOSEQ_BASE_URL`, allowing the same code to run against Ollama locally, a hosted model service, or another compatible backend.

```bash
cp deploy/env.example .env
docker compose up -d --build
```

[`deploy/DEPLOY.md`](deploy/DEPLOY.md) documents deployment using a DigitalOcean CPU droplet, HTTPS configuration, model hosting, and API-key access.

---

## Example data

`examples/sample.fasta` contains two synthetic sequences generated with a fixed seed using `scripts/make_examples.py`.

| Sequence | Known values |
|---|---|
| `toy` (15 nt) | A=6, C=0, G=4, T=5 · GC 26.67% · protein `MKFG*` (frame −1: `LPKFH`) |
| `demo_construct` (406 nt) | GC 51.97% · designed 60 aa ORF on + strand at 46–228 · designed 40 aa ORF on − strand at 259–381 |

The ORF finder also identifies several shorter candidate ORFs on the reverse strand.

These can arise by chance on the opposite strand of a coding sequence, reinforcing that an ORF is only a **candidate coding region**, not a confirmed gene.

### Try a real gene

Download the FASTA sequence for human insulin mRNA:

[NCBI NM_000207](https://www.ncbi.nlm.nih.gov/nuccore/NM_000207)

Then ask:

> "Find the longest ORF and translate it."

The longest ORF should encode the 110-amino-acid preproinsulin protein:

[UniProt P01308](https://www.uniprot.org/uniprotkb/P01308)

---

## Benchmark

```bash
python benchmark/run_benchmark.py --models qwen3:4b llama3.2:3b
```

The benchmark writes results to:

```text
benchmark/results.md
```

Metrics include:

- **Tool recall** — whether required tools were selected
- **Extra tools** — unnecessary tool calls
- **Failed calls**
- **Facts in answer**
- **Faithful answers** — answers without unsupported numerical claims
- **Execution time per task**

The keyword-based baseline demonstrates what can be accomplished without an LLM and provides a comparison for evaluating the agent.

---

## Agent Skill

`skills/bioseq-analysis/` follows the Agent Skills open standard and contains:

- `SKILL.md`
- analysis scripts
- supporting references

Example:

```bash
python skills/bioseq-analysis/scripts/bioinformatics_tools.py \
--fasta examples/demo_construct.fasta \
--action orfs \
--min-orf-length 30
```

The skill's script mirrors `bioseq_ai/bioinformatics_tools.py`.

Running:

```bash
python scripts/sync_skill.py
```

synchronizes the copies, and automated testing detects if they drift apart.

---

## Project layout

```text
bioseq-ai/
├── app.py
│   └── Streamlit interface
│
├── bioseq_ai/
│   ├── bioinformatics_tools.py
│   │   └── Biopython analysis tools
│   ├── registry.py
│   │   └── Tool allowlist, schemas, validation and sequence store
│   ├── llm.py
│   │   └── OpenAI-compatible model client
│   ├── agent.py
│   │   └── Agent loop, tools-only baseline, run manifest and replay
│   ├── faithfulness.py
│   │   └── Validation of generated numbers and sequences
│   └── cli.py
│       └── bioseq ask / bioseq replay
│
├── skills/
│   └── bioseq-analysis/
│       └── Agent Skill package
│
├── benchmark/
│   └── Benchmark tasks and runner
│
├── examples/
│   └── sample.fasta
│
└── tests/
    └── Tests for tools, agent loop, model client,
        faithfulness checking and Agent Skill
```

---

## Limitations

- DNA input only; RNA and protein sequences are currently rejected.
- Uses the standard genetic code only.
- ORFs must begin with ATG and represent candidate coding regions rather than formal gene predictions.
- The faithfulness checker verifies numbers and quoted sequences but does not independently verify every statement in the model's biological interpretation.
- Small models may occasionally skip a required tool; the visible tool trace and benchmark make these failures observable.
- Intended for **research and educational use only** and not for clinical or diagnostic decision-making.

---

## Roadmap

Planned areas for future development include:

- BLAST, UniProt, and NCBI lookups with cached and cited accession identifiers
- Protein properties such as molecular weight and isoelectric point
- Motif-search functionality
- Additional biological sequence-analysis tools
- Continued evaluation of open-weight models for reliable tool-based bioinformatics analysis

---

## Contributors

BioSeq AI was created collaboratively during **Hacktoberfest Toronto × HuskyHack 2026** by:

- **Kristian Gonzales**
- **Stephanie Clark**
- **Kriesha Eyer**
- **Saniya Chadha**
- **Ali Al-Reda El-Zein**

This repository preserves the project's collaborative origins and development history.

## License

MIT — see [LICENSE](LICENSE).
