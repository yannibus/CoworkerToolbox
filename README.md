# CoworkerToolbox — Agentforce Employee Agent

Démo/POC d'un **toolbox de délégation** pour l'Agentforce Coworker : un agent que le Coworker principal doit consulter pour aller chercher des capacités spécifiques — recherche web temps réel et analyse de fichiers/pièces jointes Salesforce — plutôt que de tenter de les gérer lui-même.

Développé et validé sur l'org **FY27** (`fy27demo@demo.com`). Ce projet est packagé pour être **auto-porteur** : déployable dans n'importe quel org client/prospect qui dispose des prérequis ci-dessous.

> **Historique :** ce package s'appelait `CoworkerSkills` jusqu'au 2026-09-04. Il a été refondu et renommé `CoworkerToolbox` pour mieux refléter son rôle (boîte à outils de délégation, pas un agent autonome). L'ancien Bot `CoworkerSkills` est désactivé sur FY27 et AGENTICDOC (pas supprimable via CLI — un agent publié ne peut être supprimé que depuis l'UI Setup) ; il n'est plus maintenu par ce repo.

## Contenu du package

| Metadata | Chemin | Rôle |
|---|---|---|
| `AiAuthoringBundle` | `force-app/main/default/aiAuthoringBundles/CoworkerToolbox/` | Agent Script — router + 4 subagents (`off_topic`, `ambiguous_question`, `GeneralWebSearch`, `Files_analysis`) |
| `GenAiPromptTemplate` | `force-app/main/default/genAiPromptTemplates/AAA_CoworkerToolbox_Files_Analysis.genAiPromptTemplate-meta.xml` | Prompt template `flex` (modèle `sfdc_ai__DefaultGPT5Mini`) — générique : résume tout type de document en langage simple et répond à toute question spécifique incluse dans la requête libre |

Le `<target>` du bundle a été retiré volontairement après retrieve : sans ça, le déploiement échoue sur tout org qui n'a pas déjà une version publiée de cet agent (cas de tout nouvel org client). Sans `<target>`, le premier déploiement crée un DRAFT v1 propre, prêt à publier.

## ⚠️ Prérequis NON packagés (fonctionnalités plateforme, pas des metadata)

Ces éléments ne se déploient pas — ils doivent être activés/vérifiés manuellement dans **chaque org cible** avant démo :

1. **Agentforce Coworker (Employee Agent) activé** sur l'org, avec licence Einstein Agent/Agentforce disponible.
2. **Web Search pour Agentforce activé** (Setup → Agentforce / Einstein Setup → Web Search) + un search provider configuré. Sans ça, l'action `WebSearch` (`EmployeeCopilot__WebSearch`, action standard, non-metadata) échouera silencieusement ou sera indisponible.
3. **Accès modèle** : le router utilise `sfdc_ai__DefaultEinsteinHyperClassifier`, le prompt template utilise `sfdc_ai__DefaultGPT5Mini` — modèles standard Models API, normalement disponibles dès qu'Agentforce est activé, mais à vérifier si l'org a des restrictions régionales/Trust Layer.
4. **Permission sets utilisateur** : les utilisateurs de démo doivent avoir le permission set standard **"Agentforce Coworker User"** (`Access_Ai_Search` ou `AISearchUser` selon l'org — packages managés préexistants, non inclus ici) pour voir l'agent dans le panneau Coworker.

## ⚠️ Piège plateforme — `activeVersionIdentifier` sur un nouveau Flex Prompt Template

Un déploiement Metadata API brut (`sf project deploy start`) d'un **nouveau** `GenAiPromptTemplate` de type `flex` avec `status: Published` ne marque pas automatiquement sa version comme active au niveau racine (`<activeVersionIdentifier>` absent), même si l'unique version existante est bien `Published`. Résultat : l'action `generatePromptResponse://` de l'agent échoue au preview avec `Validation failed for action(s) 'File_Analysis' ... must specify at least one input/output` — la plateforme ne trouve aucune version "active" dont extraire le schéma d'inputs/outputs.

**Fix (à faire une fois par org, après le premier déploiement d'un nouveau template) :**
```bash
# 1. Retrieve pour récupérer le vrai versionIdentifier généré par l'org
sf project retrieve start --json --metadata "GenAiPromptTemplate:AAA_CoworkerToolbox_Files_Analysis"

# 2. Ajouter <activeVersionIdentifier> au niveau racine du XML, pointant vers ce hash
# 3. Redéployer
sf project deploy start --json --metadata GenAiPromptTemplate:AAA_CoworkerToolbox_Files_Analysis
```
Ce hash est **spécifique à chaque org** — ne pas le committer dans le fichier source partagé (il casserait le déploiement sur un autre org). Le fichier source de ce repo reste volontairement sans `<activeVersionIdentifier>` pour rester portable ; appliquer ce fix localement/temporairement à chaque nouvel org cible si le symptôme apparaît.

## Pipeline de déploiement (par org cible)

```bash
sf config set target-org <alias-client>

# 1. Déployer le prompt template (dépendance de l'agent)
sf project deploy start --json --metadata GenAiPromptTemplate:AAA_CoworkerToolbox_Files_Analysis

# 2. Déployer l'authoring bundle (crée le DRAFT v1)
sf project deploy start --json --metadata AiAuthoringBundle:CoworkerToolbox

# 3. Publier (crée Bot + BotVersion + GenAiPlannerBundle)
sf agent publish authoring-bundle --json --api-name CoworkerToolbox

# 4. Activer
sf agent activate --json --api-name CoworkerToolbox

# 5. Vérifier
sf agent preview start --json --api-name CoworkerToolbox
```

Puis assigner le permission set "Agentforce Coworker User" aux utilisateurs de démo (étape manuelle, cf. prérequis #4). Si le preview de l'étape 5 échoue sur une erreur "must specify at least one input/output", voir le piège `activeVersionIdentifier` ci-dessus.

## ✅ Validations (2026-09-04)

**Autoporteur (package `CoworkerSkills` d'origine) :** pipeline complet (`deploy` prompt template → `deploy` bundle → `publish` → `activate`) testé de bout en bout sur un org totalement vierge (`AGENTICDOC`) : succès sans intervention manuelle, DRAFT v1 propre créé. Live preview en DRAFT (router, `GeneralWebSearch` avec vraie citation, garde-fou `off_topic`) : comportement correct, y compris en français. Preview de l'agent publié a échoué sur `AGENTICDOC` faute de permission set "Coworker" sur cet org — confirme le prérequis #1, pas un défaut du package.

**Fix Files_analysis :** le subagent demandait à l'utilisateur d'uploader/coller le fichier au lieu de reconnaître une référence de record Salesforce (`Input:Files`, type `lightning__recordInfoType`). Corrigé en explicitant dans les instructions et la description de l'input que ce paramètre attend une référence résolue (`sfdc record:<Id>` / `sfdc contentversion:<Id>`), jamais un upload. Vérifié via `sf agent preview send --use-live-actions` : l'agent demande maintenant la bonne référence au lieu d'un upload. La résolution effective du fichier depuis un simple message texte n'est testable qu'en conditions réelles (UI Coworker / Agent API) — le CLI preview n'injecte pas d'objet `lightning__recordInfoType` typé.

**Refonte CoworkerToolbox :** renommage complet + repositionnement "boîte à outils de délégation", nouveau prompt d'analyse de fichiers générique (résumé + réponse à question libre, grounding strict, jamais d'invention), descriptions et inputs de tous les subagents clarifiés. Déployé, publié et activé sur FY27 et AGENTICDOC ; ancien `CoworkerSkills` désactivé sur les deux orgs.

## Développement

- API version : 66.0
- Org de référence locale : `FY27` (`sf config get target-org`)
- Pour modifier l'agent : voir skill `sf-ai-agentscript` (Agent Script DSL `subagent`/`agent_router`, jamais `topic`/`topic_selector`)
