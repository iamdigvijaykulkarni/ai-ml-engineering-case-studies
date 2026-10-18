# ai-ml-engineering-case-studies
Case studies of production ML &amp; GenAI systems: LLM-powered NL-to-SQL and multi-tenant entity linking

Design write-ups of production ML and GenAI systems I built. The source code is
proprietary; these documents cover architecture, design decisions and results.
All company names, data and infrastructure details are anonymized.

## [Natural-Language-to-SQL and Insights Service](./nl-to-sql-insights)
Self-hosted LLM service that lets business users query SQL Server in plain English.
**70 internal users · 76.5% accuracy on an 85-question verified test set ·
deterministic SQL safety validator**
`FastAPI` `LangChain` `Ollama` `FAISS` `SQL Server`

## [Multi-Tenant Entity Linking Platform](./entity-linking-platform)
Links noisy distributor records to brand master data for 30+ brands from one codebase.
**Item accuracy ~30% → ~91.7% on ~4.6M records · ~6,700 hours of manual effort saved**
`PyTorch` `BiLSTM` `Record linkage` `TF-IDF` `SQL Server`
