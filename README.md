# CoworkerSkills — Agentforce Employee Agent

Démo/POC pour illustrer les capacités de délégation d'un Agentforce Coworker : recherche web temps réel et analyse de fichiers (pièces jointes) via prompt template.

Développé et validé sur l'org **FY27** (`fy27demo@demo.com`), source de référence `sf agent publish` (bundle actuellement en v2 côté FY27). Ce projet est packagé pour être **auto-porteur** : déployable dans n'importe quel org client/prospect dispose des prérequis ci-dessous.

## Contenu du package

| Metadata | Chemin | Rôle |
|---|---|---|
| `AiAuthoringBundle` | `force-app/main/default/aiAuthoringBundles/CoworkerSkills/` | Agent Script — router + 4 subagents (`off_topic`, `ambiguous_question`, `GeneralWebSearch`, `Files_analysis`) |
| `GenAiPromptTemplate` | `force-app/main/default/genAiPromptTemplates/AAA_Files_Analysis.genAiPromptTemplate-meta.xml` | Prompt template `flex` (modèle `sfdc_ai__DefaultGPT5Mini`) qui analyse un fichier (`ContentDocument`) + une requête libre |

Le `<target>` du bundle a été retiré volontairement après retrieve : sans ça, le déploiement échoue sur tout org qui n'a pas déjà la version `v2` de cet agent (cas de tout nouvel org client). Sans `<target>`, le premier déploiement crée un DRAFT v1 propre, prêt à publier.

## ⚠️ Prérequis NON packagés (fonctionnalités plateforme, pas des metadata)

Ces éléments ne se déploient pas — ils doivent être activés/vérifiés manuellement dans **chaque org cible** avant démo :

1. **Agentforce Coworker (Employee Agent) activé** sur l'org, avec licence Einstein Agent/Agentforce disponible.
2. **Web Search pour Agentforce activé** (Setup → Agentforce / Einstein Setup → Web Search) + un search provider configuré. Sans ça, l'action `WebSearch` (`EmployeeCopilot__WebSearch`, action standard, non-metadata) échouera silencieusement ou sera indisponible.
3. **Accès modèle** : le router utilise `sfdc_ai__DefaultEinsteinHyperClassifier`, le prompt template utilise `sfdc_ai__DefaultGPT5Mini` — modèles standard Models API, normalement disponibles dès qu'Agentforce est activé, mais à vérifier si l'org a des restrictions régionales/Trust Layer.
4. **Permission sets utilisateur** : les utilisateurs de démo doivent avoir le permission set standard **"Agentforce Coworker User"** (`Access_Ai_Search` ou `AISearchUser` selon l'org — packages managés préexistants, non inclus ici) pour voir l'agent dans le panneau Coworker.

## Pipeline de déploiement (par org cible)

```bash
sf config set target-org <alias-client>

# 1. Déployer le prompt template (dépendance de l'agent)
sf project deploy start --json --metadata GenAiPromptTemplate:AAA_Files_Analysis

# 2. Déployer l'authoring bundle (crée le DRAFT v1)
sf project deploy start --json --metadata AiAuthoringBundle:CoworkerSkills

# 3. Publier (crée Bot + BotVersion + GenAiPlannerBundle)
sf agent publish authoring-bundle --json --api-name CoworkerSkills

# 4. Activer
sf agent activate --json --api-name CoworkerSkills

# 5. Vérifier
sf agent preview start --json --api-name CoworkerSkills
```

Puis assigner le permission set "Agentforce Coworker User" aux utilisateurs de démo (étape manuelle, cf. prérequis #4).

## Développement

- API version : 66.0
- Org de référence locale : `FY27` (`sf config get target-org`)
- Pour modifier l'agent : voir skill `sf-ai-agentscript` (Agent Script DSL `subagent`/`agent_router`, jamais `topic`/`topic_selector`)
