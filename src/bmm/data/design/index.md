# Design Data Index

> Lire CE fichier en premier, puis charger UNIQUEMENT les fichiers dont les tags correspondent au contexte.
> JAMAIS charger tous les fichiers — sélectionner 3-5 max selon la pertinence.

---

## Comment sélectionner les fichiers

1. Identifier ce que la story touche (UI, accessibilité, système de design, animations, etc.)
2. Matcher contre la colonne **Tags** ci-dessous
3. Charger UNIQUEMENT les fichiers qui correspondent (3-5 max)

---

## Patterns UI/UX

| Fichier | Tags | Résumé |
|---------|------|--------|
| pat-accessibility.md | accessibility, a11y, wcag | WCAG 2.1, rôles ARIA, navigation clavier, contraste |
| pat-responsive.md | responsive, mobile, layout | Breakpoints, fluid grids, mobile-first, conteneurs |
| pat-design-system.md | design-system, tokens, components | Tokens de design, composants atomiques, documentation |
| pat-dark-mode.md | dark-mode, theming, tokens | Système de thèmes, tokens sémantiques, transitions |
| pat-animation.md | animation, motion, performance | Principes de mouvement, performance GPU, reduced-motion |
| pat-forms.md | forms, ux, validation | UX de formulaires, validation temps réel, états d'erreur |
| pat-loading-states.md | loading, skeleton, feedback | Skeletons, spinners, optimistic UI, feedback utilisateur |

## Anti-patterns (ce qu'il NE FAUT PAS faire)

| Fichier | Tags | Résumé |
|---------|------|--------|
| anti-ai-slop.md | all, quality | Éviter le rendu générique AI, authenticité visuelle |
| anti-dark-patterns.md | ux, ethics | Patterns manipulateurs, faux urgences, dark UX |

## Référence

| Fichier | Tags | Résumé |
|---------|------|--------|
| ref-design-rules.md | all | Règles de design globales (pointe vers shared/meta/design-rules.md) |
| ref-token-naming.md | design-system, tokens | Convention de nommage des tokens de design |

---

## Tag Quick Reference

| Si la story touche... | Charger les fichiers tagués... |
|-----------------------|-------------------------------|
| Composants UI | design-system, components |
| Accessibilité | accessibility, a11y, wcag |
| Mobile / responsive | responsive, mobile |
| Thème sombre | dark-mode, theming, tokens |
| Animations | animation, motion, performance |
| Formulaires | forms, ux, validation |
| États de chargement | loading, skeleton, feedback |
| Qualité visuelle | all, quality, anti-ai-slop |
