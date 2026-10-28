# AI Security Writeups

This directory contains writeups for AI Security and LLM exploitation challenges from KPMG CTF 2026 Campus Edition.

## Challenge Index

| Challenge | Points | Vulnerability Class | Key Technique | Final Flag |
| :--- | :--- | :--- | :--- | :--- |
| [AI SecureBot Support](securebot-and-excessive-agency.md#challenge-1-ai-securebot-support) | 20 | Prompt Injection / Jailbreak | Internal QA Evaluator persona override & JSON schema enforcement | `KPMG_CTF{Ulcpla31x2GGPqhybX6d0wpjhecBs5GVIJKhcJzJ5EE}` |
| [AcmeBank Assistant](securebot-and-excessive-agency.md#challenge-2-acmebank-assistant-excessive-agency) | 30 | LLM Excessive Agency / Tool IDOR | Forced tool invocation of `lookup_user("admin")` with verbatim response | `KPMG_CTF{XbGEyIlxVPj65COEujnrzMAx4Yf5bUrjPeg6ywd11cE}` |
