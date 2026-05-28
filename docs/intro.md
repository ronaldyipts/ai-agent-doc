---
sidebar_position: 1
---

# Chapter 1: Introduction

This specification defines the rationale of designing the LDS External AI agent.

The External Agent is a **REST API Server**. It is **not** presented as an end-user frontend application.

**LDS integration**: The main system (LDS) can send `courseInfo` and `form_state` via the **general_bot** and **ilo_bot** endpoints; the Agent uses them to provide contextual replies and ILO suggestions. See [Chapter 6: Main System Integration](./main_system_integration.md).

**Appendix note**: LDS options/patterns lookup API details are intentionally placed in [Chapter 7 (Appendix)](./lds_rest_api_for_chatbot.md), so the main reading flow starts from integration endpoints first.