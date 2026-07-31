# AI-Agent Seminar Materials

Generated materials for 18 recent (post-2024) papers on AI agents and agent security.
For every paper you get three files:

- `abstracts/<slug>_abstract.tex` — one-page seminar abstract (compile with pdflatex)
- `slides/<slug>_slides.tex` — Beamer slide deck (compile with pdflatex, run twice for the table of contents)
- `explanations/<slug>_explanation.md` — plain-English explainer to help you understand the paper

## Before you compile

Open any `.tex` file and edit the block marked `EDIT YOUR DETAILS` at the top:
your name, roll number, class roll, guide, department/college, and the seminar date.
These fields are pre-filled with placeholders.

Compile a deck with:

```bash
pdflatex slides/defense-pipeline_slides.tex
pdflatex slides/defense-pipeline_slides.tex
```

## Recommended starting points (most beginner-friendly)

1. **defense-pipeline** — AIs guarding AIs against prompt injection (easiest, clean 30%->0% story)
2. **data-leak-injection** — one hidden sentence leaks your data (most demo-able attack)
3. **art-competition** — 22 agents attacked 1.8M times (most engaging)
4. **camel** — Google DeepMind: defeating prompt injection by design

## Security-focused papers

| Slug | Topic |
| --- | --- |
| `defense-pipeline` | Guarding AI Assistants: A Multi-Agent LLM Pipeline That Stops Prompt Injection Attacks |
| `data-leak-injection` | When Helpful Agents Overshare: How Simple Hidden Prompts Leak Personal Data |
| `art-competition` | When Agents Break: What 1.8 Million Attacks Reveal About AI Agent Security |
| `camel` | Defeating Prompt Injections by Design: How CaMeL Protects AI Agents |
| `progent` | Progent: Least-Privilege Security for AI Agents |
| `ipiguard` | IPIGuard: Planning Tool Use Up Front to Stop Prompt Injection in AI Agents |
| `chatinject` | ChatInject: How Forged Chat Formatting Tricks AI Agents into Obeying Attackers |
| `cua-threats` | JARVIS or Ultron? A Beginner's Guide to the Safety and Security Threats of Computer-Using AI Agents |
| `trustworthy-agents` | Trustworthy LLM Agents: Mapping the Threats and Defenses of AI Agents |
| `security-principles` | Old Rules for New Agents: Applying Classic Security Principles to LLM Agents |

## General AI-agent papers

| Slug | Topic |
| --- | --- |
| `multiagent-collab` | How AI Agents Team Up: Understanding Multi-Agent Collaboration in Large Language Models |
| `mars` | MARS: Peer Review for AI - Making Multi-Agent Reasoning Twice as Efficient |
| `reasoning-frameworks` | How AI Agents Think: A Beginner's Map of LLM Agentic Reasoning Frameworks |
| `llm-planning` | Large Language Models for Planning: How AI Agents Turn a Goal into Step-by-Step Actions |
| `self-evolving` | Self-Evolving AI Agents: How Agents Learn to Improve Themselves (What, When, How, and Where to Evolve) |
| `orchestration-mcp-a2a` | Orchestrating Teams of AI Agents: Architectures, the MCP and A2A Protocols, and Enterprise Adoption |
| `human-agent-collab` | Humans in the Loop: How People and LLM Agents Work Better Together |
| `eval-survey` | How Do We Know an AI Agent Is Any Good? A Survey on Evaluating LLM-Based Agents |

## Full paper list

| # | Slug | Paper | arXiv | Date |
| --- | --- | --- | --- | --- |
| 1 | `defense-pipeline` | A Multi-Agent LLM Defense Pipeline Against Prompt Injection Attacks | arXiv:2509.14285 | 2025-09-16 |
| 2 | `data-leak-injection` | Simple Prompt Injection Attacks Can Leak Personal Data Observed by LLM Agents During Task Execution | arXiv:2506.01055 | 2025-06-01 |
| 3 | `art-competition` | Security Challenges in AI Agent Deployment: Insights from a Large Scale Public Competition | arXiv:2507.20526 | 2025-07-28 |
| 4 | `camel` | Defeating Prompt Injections by Design | arXiv:2503.18813 | 2025-03-24 |
| 5 | `progent` | Progent: Securing AI Agents with Privilege Control | arXiv:2504.11703 | 2025-04-16 |
| 6 | `ipiguard` | IPIGuard: A Novel Tool Dependency Graph-Based Defense Against Indirect Prompt Injection in LLM Agents | arXiv:2508.15310 | 2025-08-21 |
| 7 | `chatinject` | ChatInject: Abusing Chat Templates for Prompt Injection in LLM Agents | arXiv:2509.22830 | 2025-09-26 |
| 8 | `cua-threats` | A Survey on the Safety and Security Threats of Computer-Using Agents: JARVIS or Ultron? | arXiv:2505.10924 | 2025-05-16 |
| 9 | `trustworthy-agents` | A Survey on Trustworthy LLM Agents: Threats and Countermeasures | arXiv:2503.09648 | 2025-03-12 |
| 10 | `security-principles` | LLM Agents Should Employ Security Principles | arXiv:2505.24019 | 2025-05-29 |
| 11 | `multiagent-collab` | Multi-Agent Collaboration Mechanisms: A Survey of LLMs | arXiv:2501.06322 | 2025-01-10 |
| 12 | `mars` | MARS: toward more efficient multi-agent collaboration for LLM reasoning | arXiv:2509.20502 | 2025-09-24 |
| 13 | `reasoning-frameworks` | LLM-based Agentic Reasoning Frameworks: A Survey from Methods to Scenarios | arXiv:2508.17692 | 2025-08-25 |
| 14 | `llm-planning` | Large Language Models for Planning: A Comprehensive and Systematic Survey | arXiv:2505.19683 | 2025-05-26 |
| 15 | `self-evolving` | A Survey of Self-Evolving Agents: What, When, How, and Where to Evolve on the Path to Artificial Super Intelligence | arXiv:2507.21046 | 2025-07-28 |
| 16 | `orchestration-mcp-a2a` | The Orchestration of Multi-Agent Systems: Architectures, Protocols, and Enterprise Adoption | arXiv:2601.13671 | 2026-01-20 |
| 17 | `human-agent-collab` | LLM-Based Human-Agent Collaboration and Interaction Systems: A Survey | arXiv:2505.00753 | 2025-05-01 |
| 18 | `eval-survey` | Survey on Evaluation of LLM-based Agents | arXiv:2503.16416 | 2025-03-20 |