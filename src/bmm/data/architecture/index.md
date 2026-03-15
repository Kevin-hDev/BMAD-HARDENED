# Architecture Data Index

> Lire CE fichier en premier, puis charger UNIQUEMENT les fichiers dont les tags correspondent au contexte.
> JAMAIS charger tous les fichiers — sélectionner 3-5 max selon la pertinence.

---

## Comment sélectionner les fichiers

1. Identifier ce que la story touche (microservices, events, auth, data layer, etc.)
2. Matcher contre la colonne **Tags** ci-dessous
3. Charger UNIQUEMENT les fichiers qui correspondent (3-5 max)

---

## Patterns d'architecture

| Fichier | Tags | Résumé |
|---------|------|--------|
| pat-microservices.md | microservices, api, scalability | Découpage services, contrats d'API, isolation des domaines |
| pat-monolith.md | monolith, modular, simplicity | Monolithe modulaire, boundaries internes, migration paths |
| pat-hexagonal.md | hexagonal, ports-adapters, testability | Ports & Adapters, isolation du domaine, testabilité maximale |
| pat-event-driven.md | event-driven, async, cqrs | Event sourcing, CQRS, sagas, eventual consistency |
| pat-layered.md | layered, mvc, separation | MVC/MVVM, couches présentation/métier/données |
| pat-cqrs.md | cqrs, event-driven, scalability | Command/Query separation, read/write models distincts |

## Guides de décision

| Fichier | Tags | Résumé |
|---------|------|--------|
| dec-monolith-vs-micro.md | microservices, monolith, decision | Critères de choix, anti-patterns, coûts cachés |
| dec-sync-vs-async.md | async, event-driven, api | Quand utiliser sync vs async, patterns de compensation |
| dec-data-ownership.md | database, microservices, cqrs | Ownership des données, partage, consistency trade-offs |

## Référence transversale

| Fichier | Tags | Résumé |
|---------|------|--------|
| ref-bounded-contexts.md | all, domain-driven | DDD, bounded contexts, ubiquitous language |
| ref-adr-template.md | all, decision | Template ADR (Architecture Decision Record) |

---

## Tag Quick Reference

| Si la story touche... | Charger les fichiers tagués... |
|-----------------------|-------------------------------|
| Découpage en services | microservices, api |
| Performance / scalabilité | scalability, async, event-driven |
| Testabilité | hexagonal, ports-adapters, testability |
| Système événementiel | event-driven, cqrs, async |
| Application simple | monolith, layered |
| Gestion des données | database, cqrs, data-ownership |
| Décision d'architecture | decision, all |
