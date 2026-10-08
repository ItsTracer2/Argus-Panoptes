# argus-panoptes

Detection engineering et triage agentique d'un serveur exposé derrière un firewall : règles de détection portables (Sigma), SIEM d'entreprise, triage et réponse active par agents sans framework, le tout mesuré.

> **Statut : en cours de conception (v0).** Rien de ce qui suit n'est encore implémenté, ce document fixe la cible.

## Objectif

Surveiller en continu un serveur protégé par OPNsense, transformer les alertes de sécurité en décisions explicables, et appliquer des réponses actives (blocage d'IP) avec garde-fous. Le projet vise la **qualité de la détection et du triage**, pas le volume d'alertes : chaque alerte porte sa technique ATT&CK et son pourquoi, chaque action peut être justifiée de bout en bout, et l'apport des agents est **mesuré** contre une baseline sans agents.

### Choix structurants

- **Détection avant IA.** Les alertes viennent de règles versionnées dans le repo. Les agents interviennent après : triage, enrichissement, synthèse d'incident. Ils ne remplacent pas la détection.
- **Pas de framework d'agents** (ni CrewAI, ni LangGraph). Chaque agent est une fonction Python consommant un topic Kafka, appelant Ollama directement avec une sortie structurée (Pydantic), puis publiant sur le topic suivant. Pipeline déterministe, testable étape par étape, lisible ligne par ligne.
- **LLM local** via Ollama : aucune donnée de sécurité ne quitte l'infrastructure.
- **Corrélation vectorielle** (Qdrant) pour repérer campagnes et IP récurrentes par similarité de comportement.
- **Déploiement simple** : Docker Compose + Ansible, pas de Kubernetes pour ce périmètre.

## Détection (detection-as-code)

- **Sigma = source de vérité.** Les règles vivent dans `detections/sigma/`, une règle par comportement visé, avec ses tags MITRE ATT&CK.
- **Portabilité** : conversion via pySigma vers Elastic (cible principale), Splunk SPL (validation de portabilité, Splunk Free suffit) et Wazuh.
- **CI GitHub Actions** : lint, conversion et test de chaque règle contre des jeux de logs positifs et négatifs.
- **Chaque alerte est expliquée** : quelle règle, quel événement, pourquoi c'est suspect, quelle technique ATT&CK.

## Lab et télémétrie

- **Serveur défendu** derrière OPNsense (VM sur ESXi).
- **Télémétrie** : Sysmon (Windows), auditd (Linux), Suricata (Zeek optionnel).
- **Mini-lab Active Directory** isolé du LAN personnel : 1 contrôleur de domaine + 1 poste Windows avec Sysmon, cible des simulations Atomic Red Team. La détection en entreprise se joue surtout sur Windows/AD.

## Les agents

| Agent | Rôle | Topic Kafka (cible) |
|---|---|---|
| **Hermès** | Enrichissement : GeoIP, réputation (AbuseIPDB) | `argus.enrich` |
| **Hécate** | Classification de l'alerte (scan, brute-force, exploit…) | `argus.classify` |
| **Momos** | Challenger : cherche pourquoi la décision pourrait être fausse | `argus.challenge` |
| **Thémis** | Décision finale après pesée du pour et du contre | `argus.decide` |
| **Némésis** | Exécution de la réponse (blocage via OPNsense) | `argus.respond` |
| **Mnémosyne** | Corrélation et mémoire des campagnes (Qdrant) | `argus.correlate` |

La carte live (**Argus**) affiche les alertes et les décisions en temps réel.

## Garde-fous

Le serveur défendu est un vrai serveur : un faux positif bloque un utilisateur légitime. Principes retenus :

- **Allowlist en dur**, hors de portée des agents (mon IP, infra interne, services critiques).
- **Humain dans la boucle** : les décisions de blocage sont proposées et validées par un humain par défaut. L'auto-blocage s'active par seuil de confiance et par type d'action.
- **Momos est obligatoire** dans le chemin vers toute action destructive.
- **Traçabilité complète** : chaque décision conserve l'alerte d'origine, les données d'enrichissement, l'objection de Momos et le raisonnement de Thémis.
- **Réversibilité** : les blocages ont une durée de vie et sont annulables.

### Sécurité des agents (prompt injection)

Les agents lisent des données contrôlées par l'attaquant (noms d'utilisateur SSH, User-Agent, payloads HTTP). Un attaquant peut tenter d'y glisser des instructions pour faire bloquer une IP légitime ou faire classer son attaque comme bénigne.

- Les champs issus des logs sont traités comme des **données**, jamais mêlés aux instructions.
- Sorties **strictement structurées et validées** (Pydantic) ; toute sortie hors schéma est rejetée.
- Les agents de classification n'ont **aucun accès aux outils**.
- L'action de blocage passe par une **fonction déterministe** avec allowlist, hors du LLM.
- Un jeu de **logs piégés** sert de tests de non-régression.

## Évaluation

Le but est de chiffrer l'apport des agents, pas de l'affirmer.

- **Vérité terrain** : techniques ATT&CK rejouées avec Atomic Red Team sur le lab, mélangées à du trafic bénin.
- **Baseline mesurée avant les agents** (règles seules), puis trois configurations comparées : règles seules, règles + agents, règles + agents sans Momos.
- **Métriques** : taux de détection par technique, faux positifs, temps de triage, latence et coût par décision, taux de réussite des tests d'injection.

## Stack

| Couche | Technologie |
|---|---|
| Langage | Python |
| SIEM / détection | Elastic Security, agents Wazuh |
| Règles | Sigma, pySigma, MITRE ATT&CK |
| Télémétrie | Sysmon, auditd, Suricata |
| Bus d'événements | Kafka (à confirmer) |
| LLM | Ollama (local) |
| Schémas et validation | Pydantic |
| Base vectorielle | Qdrant |
| Pare-feu / réponse active | OPNsense (API) |
| Simulation d'attaques | Atomic Red Team |
| CI / déploiement | GitHub Actions, Docker Compose, Ansible |
| Frontend | React + WebSocket |
| Plus tard | TheHive/Cortex (ou Shuffle), MISP/OpenCTI |

## Questions ouvertes

- **Kafka** : Elastic alerte déjà. Kafka ne se justifie que pour découpler les étapes et rejouer les alertes. À confirmer, sinon file plus légère.
- **Wazuh + Elastic** : deux couches de collecte. À garder seulement si Wazuh apporte une valeur propre (intégrité des fichiers, détection hôte) ; sinon Elastic Agent seul.
- **Ressources ESXi** : Elastic, Ollama, Qdrant et le lab AD sur la même machine, à dimensionner avant de s'engager.

## Roadmap

- [ ] **v0.1** : Elastic + agents Wazuh, collecte OPNsense / SSH / auditd, schémas Pydantic des alertes
- [ ] **v0.2** : dépôt de règles Sigma avec tags ATT&CK, conversion pySigma, CI GitHub Actions
- [ ] **v0.3** : mini-lab AD + Sysmon + Atomic Red Team, **première mesure de baseline (règles seules)**
- [ ] **v0.4** : Hermès et Hécate (enrichissement et classification)
- [ ] **v0.5** : Momos et Thémis, décision en mode recommandation uniquement
- [ ] **v0.6** : Némésis, blocage OPNsense avec allowlist et expiration
- [ ] **v0.7** : durcissement prompt injection et tests de logs piégés
- [ ] **v0.8** : Mnémosyne, corrélation Qdrant
- [ ] **v0.9** : carte live
- [ ] **v1.0** : évaluation chiffrée contre la baseline, documentation, démo
- [ ] **v2** : TheHive/Cortex (ou Shuffle) pour la gestion de cas, MISP/OpenCTI, honeypot comme source supplémentaire

## Installation

À venir.

## Inspirations

Architecture inspirée de deux projets de recherche personnels en plateforme SOC agentique (pipeline de décision multi-agents, réponse active via OPNsense, visualisation en temps réel). Ici, la logique agent est entièrement réécrite avec une approche différente : fonctions pures, Kafka, sans framework d'orchestration, et une détection explicable et portable en amont.

## Licence

À définir.
