# DAI Vault

This vault is the strategic and architectural knowledge base for DAI.

DAI is a multi-tenant product factory designed to create multiple niche decision products from one shared backend.

## What lives here

This vault is for:

- platform doctrine
- architecture notes
- niche strategy
- product thinking
- decision logs
- research summaries
- execution planning
- reusable templates

This vault is not the main home for source code.
Source code lives in the sibling monorepo:

- ..\dai

## Folder guide

## 00 Inbox
Quick capture. Raw ideas, scraps, rough notes.

## 01 Operating System
Permanent guidance for how DAI should be built and operated.

Suggested files:
- vision.md
- principles.md
- product-factory-model.md
- decision-rules.md
- glossary.md

## 02 Platform
Shared platform architecture that all niches build on.

Use this for:
- tenant model
- agents
- orchestration
- permissions
- observability
- billing
- schemas
- platform decisions

## 03 Niches
One folder per niche.

Current shells:
- sports-analytics
- crypto-analytics
- stock-analytics
- kalshi-analytics

Each niche should contain:
- thesis
- customers
- signals
- workflows
- monetization
- data sources
- prompts
- experiments
- decisions

## 04 Products
Productized outputs and v1 packaging by niche.

## 05 Research
External findings, market notes, technical references, competitor review, pricing notes, and compliance notes.

## 06 Execution
Roadmaps, weekly plans, launch checklists, and backlog notes.

## 07 Templates
Reusable note templates for niches, research, decisions, and product briefs.

## 08 Attachments
Images, exports, and supporting files.

## 09 Archive
Retired or inactive material.

## Git and backup

This vault should be versioned in its own git repo.

Recommended model:
- dai = code repo
- dai-vault = vault repo

That keeps strategic docs separate from implementation history.

## Working model

Use Obsidian for:
- browsing and editing notes
- linking ideas
- daily capture
- navigating the knowledge base

Use Claude Code for:
- creating and editing files across the vault and monorepo
- comparing docs to implementation
- generating new niche shells and documents
- keeping docs and code aligned

Use GPT for:
- research
- strategy
- architecture pressure testing
- niche comparison
- synthesis

## Path layout

Workspace root:
- C:\Users\trolo\source\repos\dai-workspace

Sibling folders:
- C:\Users\trolo\source\repos\dai-workspace\dai
- C:\Users\trolo\source\repos\dai-workspace\dai-vault
