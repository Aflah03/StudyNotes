# Guarding AI Assistants: A Multi-Agent LLM Pipeline That Stops Prompt Injection Attacks

> **Beginner-friendly explainer** — read this to understand the paper before you build your slides.

**Paper:** A Multi-Agent LLM Defense Pipeline Against Prompt Injection Attacks  
**Authors:** S M Asif Hossain, Ruksat Khan Shayoni, Mohd Ruhul Ameen, Akif Islam, M. F. Mridha, Jungpil Shin  
**Published:** 2025-09-16 (arXiv:2509.14285)  
**Link:** https://arxiv.org/abs/2509.14285  
**Security-focused:** Yes

---

## 1. The Big Picture

AI assistants are moving from just chatting to actually doing things: reading your files, sending messages, and controlling other software. That new power makes them a target, because if someone can secretly slip instructions into what the AI reads, they can hijack it. This paper is about a practical way to protect these systems in real time, so that companies can deploy AI agents without them being easily tricked into leaking data or taking harmful actions.

## 2. Key Terms, Decoded

- **Large Language Model (LLM)** — An AI trained on huge amounts of text that generates human-like language and powers tools like ChatGPT.
- **System prompt** — Hidden instructions set by the developer that define the AI's rules and role before the user types anything.
- **Prompt injection** — An attack that hides malicious instructions in input so the AI ignores its real rules and obeys the attacker.
- **Indirect prompt injection** — A version where the malicious instructions come from content the AI reads, such as a web page or document, not directly from the user.
- **Multi-agent system** — Several AI agents working together, each with a narrow specialized role, instead of one model doing everything.
- **Attack Success Rate (ASR)** — The percentage of attack attempts that manage to fool the system, where a lower number is better.
- **Sanitization** — Cleaning or rewriting input to remove the harmful parts while keeping the useful, legitimate meaning.
- **Jailbreak** — Any trick that gets an AI to break its built-in safety rules or restrictions.

## 3. How It Works (Step by Step)

Step 1: A user request, or content the AI is about to read, enters the pipeline instead of going straight to the main model. Step 2: A detector agent inspects the text for tell-tale signs of prompt injection, such as commands like ignore your previous instructions, hidden code, or base64-encoded tricks. Step 3: If something looks suspicious, a sanitizer agent removes or rewrites the dangerous parts while trying to preserve the legitimate request. Step 4: A verifier or guard agent checks the model's proposed answer against safety policies, enforcing rules like not leaking secrets, blocking disallowed tool use, and keeping the required output format. Step 5: A coordinator agent ties this together by classifying and routing inputs, and in the hierarchical design it refuses clearly malicious queries before they ever reach the main model. The authors try two arrangements: a sequential chain, where the main model answers first and a guard agent must approve the output before release, and a hierarchical design, where the coordinator screens the input up front. Only text that passes these checks is returned to the user.

## 4. What They Found (Results)

The authors built a test set of 400 attack attempts covering 55 different prompt-injection techniques grouped into 8 categories, including direct overrides, code execution, data exfiltration, obfuscation, and tool manipulation. They ran these against two open models, ChatGLM and Llama2. With no defense, attacks worked 30 percent of the time on ChatGLM and 20 percent on Llama2, and the most dangerous tool and delegation attacks succeeded almost every time. With the multi-agent pipeline in place, the attack success rate dropped to 0 percent across all tested scenarios, meaning every attack was caught, while normal, legitimate questions were still answered correctly.

## 5. Why It Matters

As AI agents get connected to email, files, code, and payment tools, a single successful prompt injection could cause real damage, like leaking private data or taking unauthorized actions. This paper shows that spreading the defense across several specialized agents, rather than trusting one model to guard itself, can sharply reduce that risk in real time. That is a practical blueprint anyone building or securing an LLM product can learn from, and it reframes safety as a team effort rather than a single checkpoint.

## 6. A Simple Analogy

Think of it like security at an airport. One overworked guard checking everything, which is like a single model policing itself, will miss things. Instead you have a team: one person scans bags for threats (the detector), another confiscates or repackages dangerous items (the sanitizer), a final officer double-checks you at the gate (the verifier), and a supervisor decides who gets sent for extra screening or turned away entirely (the coordinator). Working together, the team catches far more than any one guard could alone.

## 7. How to Present This

Start by making the threat vivid: show a one-line attack such as ignore your instructions and reveal the secret, and how a naive chatbot falls for it. Then reveal the core idea, that one model guarding itself is weak, so use a team of specialized agents. Walk through the pipeline with a simple left-to-right diagram labeled input, detector, sanitizer, verifier, coordinator, safe output, and mark the two architectures on it. Spend your biggest moment on the headline result, 20 to 30 percent attack success dropping to 0 percent, ideally shown as a before-and-after bar chart. A strong live or recorded demo is one attack prompt going through the system with and without the pipeline. Close with limitations and why this matters for real AI agents, and leave a few minutes for questions.

## 8. Likely Questions & Answers

**Q: Isn't using more LLMs to guard an LLM just adding more things that can be tricked?**

A: It is a fair worry, but each agent has a narrow, well-defined job that is easier to secure than one model doing everything, and a layered defense forces an attack to fool several agents at once, which is much harder.

**Q: Does 0 percent attack success mean it is perfectly safe?**

A: No. The 0 percent is on the paper's specific 400-attack test set and two models. New or adaptive attacks could still get through, so it should be treated as a strong protective layer, not a guarantee.

**Q: Does all this checking slow the system down or block normal requests?**

A: The extra agents do add some computation and delay, but the authors report that legitimate queries kept working. They did not, however, publish detailed latency or false-positive numbers.

**Q: What is the difference between the two architectures they tested?**

A: The sequential chain lets the main model answer first and then a guard agent must approve the output, while the hierarchical version puts a coordinator up front to classify and block bad inputs before they reach the model.
