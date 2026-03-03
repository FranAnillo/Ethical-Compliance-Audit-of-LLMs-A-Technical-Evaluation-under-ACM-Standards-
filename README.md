# The Ethical Co-Pilot: LLM Compliance Audit 🛡️🤖

[![ACM Standards](https://img.shields.io/badge/Standard-ACM%20Code%20of%20Ethics-blue)](https://www.acm.org/code-of-ethics)
[![Status](https://img.shields.io/badge/Audit-Completed-green)]()

> **Are AI Assistants Safe for Engineering?** An empirical audit of Gemini-3, GPT-4, GPT-5.1, Claud-Sonnet-4.6, and Grok-4 against the 2018 ACM Code of Ethics.

## 📄 Abstract
This repository contains the full dataset, methodology, and extended results of the research presented in the poster **"The Ethical Co-Pilot"**. We subjected leading Large Language Models (LLMs) to adversarial "Stress Prompts" to evaluate their alignment with professional ethical standards in three key areas:
1.  **Inclusive Terminology** (ACM 1.4)
2.  **Data Representation** (ACM 1.1)
3.  **Algorithmic Bias Safety** (ACM 2.5)

## 🔍 Key Findings at a Glance

| Category | Finding | Status |
| :--- | :--- | :--- |
| **Diversity** | Models successfully eliminated gender/wage gaps. | ✅ **PASS** |
| **Terminology** | "Master/Slave" persists in generated Python code. | ⚠️ **PARTIAL FAIL** |
| **Safety** | Significant divergence: Some models warn against bias, others suggest discriminatory proxy variables. | 🔴 **CRITICAL RISK** |

## 📂 Repository Structure
* `results/DETAILED_ANALYSIS.md`: Extended breakdown of model responses.
* `data/`: Raw output logs from the LLMs (for verification).
* `methodology/`: The specific prompts used for stress testing.

## 👥 Authors
* **Francisco José Anillo Carrasco**
* **Andreea Madalina Oprescu Poprescu**
* **María del Carmen Romero Ternero**
* *Universidad de Sevilla, Departamento de Tecnología Electrónica*

---
*For more details, see the full analysis in the `results` folder.*