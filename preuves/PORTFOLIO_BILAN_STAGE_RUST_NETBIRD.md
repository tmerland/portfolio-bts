# Portfolio Technique - Stage Client Rust RDP (NetBird + beRackConnect)

Date: 2026-04-28  
Projet: `Client-Rust-rdp`  
Auteur: Thibault (Damien Ract dans l'historique git)

## 1) Objectif du stage

Construire et fiabiliser une application client Rust pour acces RDP securise, avec:
- un plan de controle API (auth/session/targets),
- un plan de donnees tunnel RDP,
- une couche reseau overlay zero-trust (NetBird),
- une orchestration locale headless (beRackConnect),
- une UX exploitable par des utilisateurs non techniques.

Le projet est passe d'un prototype client tunnel a une stack hybride prete pour deploiement pilote.
En parallele, j'ai contribue a la preparation de machines et a la mise en place d'un environnement Proxmox de qualification pour anticiper une migration de l'entreprise hors ESXi.

---

## 2) Methode d'audit utilisee pour ce bilan

Ce document est base sur:
- l'historique git complet du repository (`28 commits`, du `2026-02-14` au `2026-04-08`),
- l'analyse des jalons V1 a V16.2,
- les modules Rust effectivement presents dans `src/` et `src/services/`,
- les scripts de build/deploiement/diagnostic,
- les documents techniques de migration et runbook (`docs/`).

Principales sources internes:
- `docs/COMPTE_RENDU_GLOBAL_PROJET_CLIENT_RUST_2026-04-09.md`
- `docs/ROADMAP_WORKFLOW_REFONTE_V31.md`
- `docs/CONFORMITE_TUTEUR_SUIVI.md`
- `docs/RUNBOOK_EXPLOITATION_CLIENT_V31.md`
- `docs/MIGRATION_TROUBLESHOOT_LOG.md`
- `RAPPORT_SERVEUR_192.168.20.104.md`

---

## 3) Chronologie technique depuis le debut (commits)

## Phase 0 - Bootstrap projet (2026-02-14 -> 2026-02-16)

Commits:
- `7daf2de` Initial commit
- `46721d2` initial commit environnement
- `eb286fc` ajout des consignes
- `f97e239` hello world slint

Travail realise:
- initialisation repo et environnement,
- premier squelette Slint,
- cadrage du sujet.

## Phase 1 - Fondations client RDP (V1 -> V6) (2026-02-16 -> 2026-02-19)

Commits:
- `657eae8` V1
- `20d6e28` V2 tunnel fonctionnel (sans verification CA root cote client)
- `a87bd24` V3 RDP fonctionnel
- `ff30ae7` V4 tray + selection target RDP depuis serveur
- `1c52c11` V5 flux applicatif complet (PFX + target + mot de passe + RDP)
- `5b878b4` V6 bascule moteur TLS vers pile autonome

Travail technique cle:
- structure app Rust: `app.rs`, `config.rs`, `services/api.rs`, `services/tunnel.rs`,
- tunnel local loopback pour MSTSC (`127.0.0.1:13389`),
- recuperation des cibles via API,
- integration tray Windows (`src/tray.rs`),
- gestion identite PFX / stockage securise (`secure_store.rs`, `pfx_identity.rs`),
- migration vers pile TLS Rust autonome (moins de dependance a la stack native Windows).

## Phase 2 - Stabilisation UI + migration transport (V7 -> V9) (2026-03-06 -> 2026-03-10)

Commits:
- `fc28c66` V7 UI optimisee + nouvelles fonctions
- `bf41766` V8 UI stable + preparation migration UDP
- `ff6d60d` V9 UDP fonctionnel + indicateur TCP/UDP

Travail technique cle:
- simplification et stabilisation UI Slint,
- preparation puis activation du mode UDP/QUIC,
- indicateur visuel de transport actif.

## Phase 3 - Supervision, dashboard, update (V10 -> V13.2) (2026-03-16 -> 2026-03-31)

Commits:
- `f9c8162` V10 supervision clients + prise de controle
- `dc4e67f` V11 fixs supervision
- `c9b36c5` V12 ajout update + design dashboard
- `ba2213b` V12.1 fix module update
- `aa6e345` V12.2 UI/supervision fixes
- `375e09c` V13 final
- `fc1303f` V13.2 corrections

Travail technique cle:
- enrichissement dashboard admin (sessions/machines/updates/support),
- debut du canal update signe (manifest/signature + binaire de signature),
- structuration de la migration QUIC/UDP en parallele,
- ajout modules supervision (capture/input/support) cote client.

## Phase 4 - Refonte NetBird/beRackConnect (V14 -> V16.2) (2026-04-01 -> 2026-04-08)

Commits:
- `cc1a38e` V14 preparation beRackConnect + NetBird
- `3656d0c` V15 checkpoint NetBird fonctionnel + alignement J1/J2
- `c88f2c9` V16 fixes divers + avancement UI + service installer
- `85d1b01` V16.1 rollback services + checklist E2E
- `ff5fe63` V16.2 simplification UX installation composants

Travail technique cle:
- introduction de l'orchestrateur local `beRackConnect`,
- integration NetBird headless comme couche reseau,
- ajout de prerequis runtime et auto-repair services,
- mise en place d'un vrai workflow d'installation/rollback,
- formalisation scripts de validation terrain.
- preparation de machines de test et de qualification pour valider les parcours d'acces distant dans un environnement pilote.

---

## 4) Architecture finale obtenue pendant le stage

## 4.1 Client Rust

Composants principaux:
- `src/main.rs`: entrypoint, wiring UI, modes process (`--berack-connect-service`, `--support-agent`, etc.),
- `src/app.rs`: etat applicatif et orchestration actions UI,
- `ui/main.slint`: interface utilisateur,
- `src/services/*`: logique metier.

Services majeurs implementes:
- `api.rs`: control-plane HTTP,
- `tunnel.rs`: tunnel data-plane mTLS/QUIC,
- `rdp.rs`: lancement MSTSC et policies credentials,
- `netbird.rs`: bootstrap/controle overlay NetBird,
- `berack_connect.rs`: broker local/service Windows,
- `service_installer.rs`: workflow update/install/rollback services,
- `startup_prereq.rs`: evaluation prerequis au demarrage.

Capacites poste ajoutees (cadre J1/J2):
- `device_command.rs`, `device_audit.rs`,
- `file_browse.rs`, `file_transfer.rs`,
- `clipboard.rs`,
- `session_ipc.rs` / `session_helper_manager.rs`.

## 4.2 Configuration runtime

Fichier: `config/client.toml`
- mode distribution: `hybrid_preferred`,
- fallback tunnel mTLS (transport force UDP cote logique projet recente),
- NetBird auto-bootstrap configurable (`netbird_*`),
- canal update (`stable`, manifest/signature),
- verification TLS active (`verify_server_cert = true`).

## 4.3 Cote serveur (contributions via patches et integration)

Perimetre touche:
- API RDP (sessions/targets/support/device endpoints),
- dashboard admin (machines/sessions/updates/users/targets),
- enforcement machine deny,
- publication update artifacts versionnes.

Fichiers de patch serveur visibles dans le repo:
- `_remote_patch/main.py`
- `_remote_patch/www_admin_main.py`
- templates dashboard associes.

---

## 5) Securite et hardening implementes

## 5.1 Transport et identite

- mTLS client avec identite PFX,
- verification serveur TLS active en release,
- root/intermediate CA geres dans le packaging.

## 5.2 Controle d'acces

- session API/JWT avec lifecycle ferme explicitement,
- policy machine approval (pending/approved/denied),
- deny admin qui coupe l'acces cible et sessions associees (enforcement serveur).

## 5.3 Securite operationnelle poste

- orchestration privilegiee via service local (beRackConnect),
- workflow UAC unique pour prerequis,
- rollback de service installer en cas d'echec update.

---

## 6) Packaging, deploiement, exploitation

## 6.1 Packaging

Scripts majeurs:
- `scripts/build-manual-netbird-berackconnect-package.ps1`
- `scripts/build-hybrid-package.ps1`
- `scripts/install-berackconnect-netbird-manual.ps1`
- `scripts/start-client-with-prereq.ps1`

Livrable final principal en fin de stage:
- package manuel type `V30.2.x` (incluant NetBird MSI + scripts + launcher 1-clic).

## 6.2 Exploitation

Runbook:
- `docs/RUNBOOK_EXPLOITATION_CLIENT_V31.md`

Diagnostic:
- `scripts/collect-berackconnect-netbird-diagnostics.ps1`
- `docs/TROUBLESHOOTING_BERACKCONNECT_NETBIRD.md`

## 6.3 Validation continue

Scripts qualite:
- `scripts/validate-lot-a-workflow.ps1`
- `scripts/check-tuteur-conformite.ps1`
- `scripts/run-phase-c-e2e.ps1`
- `scripts/run-phase-c-e2e-admin.ps1`

---

## 7) Preuves de deploiement et de livraison

Etat documente (reference `2026-04-09`):
- artefact publie serveur:
  - `/var/www/rdp-updates/stable/30.2.9/client-rust-rdp-win64-manual-netbird-berackconnect-V30.2.9.zip`
- hash de controle:
  - `064f793354ec1b775ec486c6d0471e12ac6712ae3a45a257343a20f4023c1960`
- strategie de mise en production prudente:
  - publication versionnee sans bascule immediate du manifest actif,
  - pas de restart de services critiques pendant cette etape.

---

## 8) Resultats concrets de la contribution

Resultats techniques atteints:
- passage d'un prototype tunnel vers une architecture hybride robuste,
- integration headless NetBird + beRackConnect,
- workflow de lancement utilisateur simplifie avec auto-repair prerequis,
- ajout d'un cadre de validation (checks + E2E + troubleshooting),
- documentation de reprise structurante pour la suite du projet.

Resultats operationnels:
- deploiement manuel parc possible sans recompiler a chaque poste,
- reduction du risque de blocage utilisateur (fallback + remediation),
- meilleure maintenabilite pour l'equipe apres stage.
- contribution a la modernisation de l'infrastructure via la qualification d'un socle Proxmox pour des VM de migration.

---

## 9) Limites connues en fin de stage

- recette E2E finale a consolider en admin sur poste vierge selon roadmap,
- coexistence possible de stacks NetBird legacy a normaliser definitivement,
- UX encore perfectible sur quelques etats techniques,
- finalisation parcours update auto pour changements majeurs a encadrer.
- heterogeneite postes Windows (UAC/policies/droits) pouvant impacter certains flux d'installation auto.

---

## 10) Reprise projet conseillee (handover)

Ordre recommande:
1. executer `run-phase-c-e2e-admin.ps1` sur poste vierge de reference,
2. valider migration definitive vers NetBird officiel sur toutes les machines,
3. figer wording UX final (non technique),
4. lancer vague pilote update/services,
5. cloturer par recette client + serveur signee.

## 11) Preuves associees

- Bilan technique principal: [PORTFOLIO_BILAN_STAGE_RUST_NETBIRD.md](PORTFOLIO_BILAN_STAGE_RUST_NETBIRD.md)
- Bilan infrastructure et machines: [RAPPORT_SERVEUR_192.168.20.104.md](RAPPORT_SERVEUR_192.168.20.104.md)

---

## 11) Plan de cloture recommande (pilotage)

1. Vague pilote controlee:
- activation sur lot restreint,
- observation 24-48h (installation, overlay, fallback, erreurs services).

2. Finalisation UX:
- dernier passage wording non technique,
- validation rapide par utilisateurs non experts.

3. Dossier exploitation:
- runbook unique support N1/N2,
- matrice incident -> action -> preuve.

4. Cloture:
- recette signee client + serveur,
- note de conformite finale.

---

## 12) Resume portfolio (court)

J'ai contribue a la transformation complete d'un client RDP Rust vers une stack hybride securisee et exploitable en entreprise, en couvrant a la fois le coeur applicatif (Rust/Slint/services), l'integration reseau zero-trust (NetBird), l'orchestration locale privilegiee (beRackConnect), les scripts de deploiement/rollback, et le cadre de validation/documentation pour reprise projet.
