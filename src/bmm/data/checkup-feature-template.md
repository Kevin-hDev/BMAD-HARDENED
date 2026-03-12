# Checkup Features — {project_name}

> Cree : {date} | Derniere verification : {date}
> Ce fichier est la source de verite pour le suivi des features a travers toutes les phases BMAD.
> Il est cree a la fin du brainstorming et verifie a la fin de CHAQUE phase.

---

## Features validees

| # | Feature | Source | Phase prevue | Brainstorm | PRD (FR) | Architecture | Epic/Story | Dev | Review |
|---|---------|--------|-------------|-----------|----------|-------------|-----------|-----|--------|
| 1 | (feature) | (source) | (phase) | ? | ? | ? | ? | ? | ? |

> Legende : ok = couvert | ? = pas encore verifie | X = manquant | N/A = pas applicable

---

## Features ajoutees en cours de route

| # | Feature | Ajoutee pendant phase | Raison | Toutes phases mises a jour ? |
|---|---------|----------------------|--------|------------------------------|

---

## Verifications par phase

### Fin brainstorming
- [ ] Toutes les features discutees sont listees ci-dessus
- [ ] Chaque feature a une source identifiee
- [ ] Chaque feature a une phase prevue

### Fin PRD
- [ ] Chaque feature a au moins 1 FR (Functional Requirement)
- [ ] Chaque FR est suffisamment detaille (acceptance criteria, edge cases)
- [ ] Aucune feature de la liste n'a ete oubliee dans le PRD
- [ ] Les FR de securite couvrent toutes les features securite

### Fin architecture
- [ ] Chaque feature a un composant technique identifie
- [ ] Les dependances entre features sont documentees
- [ ] Les choix techniques pour chaque feature sont justifies

### Fin epics/stories
- [ ] Chaque feature a au moins 1 story
- [ ] Chaque story a des acceptance criteria clairs
- [ ] Les stories securite-critiques ont des criteres de securite explicites
- [ ] La somme des stories couvre 100% des features

### Fin implementation (par sprint)
- [ ] Chaque story prevue dans le sprint est implementee
- [ ] Chaque story est passee en code review
- [ ] Les tests couvrent les acceptance criteria
- [ ] Aucune feature n'a ete silencieusement abandonee

---

## Rapport de gaps

> Rempli automatiquement par les agents a chaque verification de phase.
> Si des gaps sont detectes, ils sont listes ici avec la date et l'action corrective.

| Date | Phase | Feature manquante | Gap detecte | Action corrective | Statut |
|------|-------|-------------------|-------------|-------------------|--------|
