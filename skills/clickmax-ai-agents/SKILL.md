---
name: clickmax-ai-agents
description: Use when the user wants to create, configure, publish, turn on/off or place an AI agent (AI SDR) on WhatsApp/Instagram/Telegram/Webchat connections in Clickmax, edit its persona, hand-off rules, knowledge or deployment settings.
---

## When this applies

Building or changing an AI agent of Meus agentes and where it works. Agent = who it is (persona, model, hand-off rules, knowledge). Deployment ("posto") = where and how it works (connections, pipeline, hours, follow-up, audience).

Not this skill:

- why the SDR did/did not answer, performance → `clickmax-ai-sdr-observability`
- the conversations themselves → messaging/inbox skills

## Key assumptions

- Agent runs on the customer's own provider key (OpenAI/Anthropic). No key in `ai_agents_credentials_list` → the user adds it in Meus agentes; never ask for the key in chat.
- Agent lifecycle: `draft` (created) → `ai_agents_publish` → `active` ⇄ `paused`. Edits (`ai_agents_update`) apply at once, also on an active agent.
- Deployment lifecycle: `draft` → `ai_agents_deployments_activate` → `active` ⇄ `paused`. Needs an active agent + ≥1 connection.
- It answers contacts only when agent `active` AND deployment `active`.
- There is no suggest/draft mode: an active agent on an active deployment sends to contacts by itself, nobody reviews the messages. `autonomy` in the output only mirrors the deployment status.
- Role available today: `sdr_inbound` only. Channels: WhatsApp, Instagram, Telegram, Webchat (`channel_instances_list`).
- `allow*` abilities (schedule meetings, create tasks, custom fields, tags) and audience ("Com quem o agente conversa": after meeting, upcoming meeting, won opportunity, owned opportunity, human elsewhere) default ON. Turning an audience switch off makes the agent stay silent for those contacts.
- Knowledge: URL and FAQ via tools; documents only in the app. Max 20 sources per agent.

## Thought process

1. New agent or existing one? Existing → `ai_agents_list` (`search`) → `ai_agents_get` before editing.
2. Missing essentials for create: key + model, name, persona (what it sells, tone, what to qualify, when to hand off). Ask for business facts; never invent prices, policies or guarantees.
3. Placement: which connections, which pipeline/stage receives the opportunities, hand-off team, business hours. Ask only what the user cares about; defaults are safe.
4. Going live is the user's decision: publish and activate each need explicit consent, and the user must know the agent will then answer contacts by itself.

## Execute guide

- Create end to end: `mcp__plugin_clickmax_clickmax__ai_agents_credentials_list` → `mcp__plugin_clickmax_clickmax__ai_agents_models_list({ credentialId })` (prefer `curated` + `available`) → `mcp__plugin_clickmax_clickmax__ai_agents_create` → optional `mcp__plugin_clickmax_clickmax__ai_agents_knowledge_add_faq` / `mcp__plugin_clickmax_clickmax__ai_agents_knowledge_add_url` → `mcp__plugin_clickmax_clickmax__ai_agents_deployments_create({ agentId, role: 'sdr_inbound', channelInstanceIds, ... })`. Stop there and summarize; publish/activate only after the user says go.
- Go live: `mcp__plugin_clickmax_clickmax__ai_agents_publish` → `mcp__plugin_clickmax_clickmax__ai_agents_deployments_activate`. Before asking, say plainly that from then on the agent answers contacts by itself on those connections.
- Edit persona: `mcp__plugin_clickmax_clickmax__ai_agents_get` → change the text the user asked → `mcp__plugin_clickmax_clickmax__ai_agents_update({ persona })`. Escalation is replaced whole: send every rule to keep (reuse existing `id`s).
- Audience/abilities/follow-up/hours: `mcp__plugin_clickmax_clickmax__ai_agents_deployments_update`; `null` clears pipeline, stage, calendar, team or follow-up.
- Stop now: `mcp__plugin_clickmax_clickmax__ai_agents_pause` (everywhere) or `mcp__plugin_clickmax_clickmax__ai_agents_deployments_pause` (one deployment). No confirmation needed.

## Report

- Say the state that matters to contacts: "rascunho — ainda não responde ninguém", "ativo, respondendo sozinho no WhatsApp X", "pausado".
- After create, list what is still missing to go live (publish, activate).

## Warnings

- Editing an active agent or deployment changes live conversations immediately — say it before applying (both updates ask for approval).
- FAQ answers are repeated to contacts as fact: only use answers the user gave or confirmed.
- A 409 on activate = agent not published/active or deployment without connection; fix that, do not retry blindly.

## Anti-patterns

- Activating "só para testar" or "para sugerir respostas": activating means it answers real contacts by itself — say so; test on a connection with no real traffic instead.
- Writing a long generic persona instead of the user's business facts.
- Creating a second agent when the user wanted to place the existing one on another connection (that is a deployment).

---

Clickmax skill revision: `862a8ca93146`
