# NovaBank Core

Base de données PostgreSQL d'une banque en ligne (module SQL avancé & administration PostgreSQL, Bachelor 2).
Projet individuel. Environnement : VM Ubuntu, PostgreSQL 18, SQL, Bash (sans Docker).

## Contenu du dépôt

| Fichier | Rôle |
|---|---|
| `01_install_server.sh` | Installe PostgreSQL, configure `ufw` et `pg_hba.conf` |
| `02_schema.sql` | Crée la base, les schémas `core` et `audit`, les 6 tables et leurs contraintes |
| `02b_data.sql` | Génère 1 000 clients, 3 000 comptes, 1 000 000 d'écritures |
| `03_roles.sql` | 4 profils avec le minimum de droits |
| `04_views.sql` | Vue qui cache des données, vue modifiable, vue matérialisée |
| `05_indexes.sql` | 4 index et mesures `EXPLAIN` avant/après |
| `06_triggers.sql` | Solde automatique, grand livre immuable, fraudes, audit |
| `07_procedures.sql` | Virement sûr, ouverture de compte, relevé |
| `08_tests.sql` | Tests : cas valides et interdits refusés |
| `test_concurrence.sh` | Deux virements simultanés sur le même compte |
| `run_all_tests.sh` | Reconstruit et lance tous les tests, affiche OK / ECHEC |
| `backup.sh` | Sauvegarde `pg_dump` |
| `make_env.sh` | Génère le fichier `.env` (mots de passe aléatoires) |
| `reset_all.sh` | **Reconstruit tout depuis zéro** |


## Comment rejouer

```bash
./make_env.sh             # crée .env avec 4 mots de passe aléatoires
./reset_all.sh --install  # installation + construction complète
sudo -u postgres psql -a < 08_tests.sql
./test_concurrence.sh
./backup.sh
# ou tout d'un coup : ./run_all_tests.sh
```

Les mots de passe sont dans `.env`, non versionné. Aucun mot de passe n'est dans les scripts : ils sont passés à `psql` par variables (`:'pwd_appli'`).

## Les 3 problèmes du prototype et leur solution

1. **Un employé a exporté tous les emails.** Chaque profil n'a que le minimum de droits (`REVOKE ... FROM PUBLIC` puis `GRANT` ciblés). Le support et l'analyste n'ont aucun droit sur les tables : ils passent par des vues (email masqué, vue matérialisée).
2. **Deux virements simultanés ont donné un solde négatif.** `sp_transfer` verrouille les deux comptes (`FOR UPDATE`, ordre fixe), lit le solde après le verrou, et tout se fait dans une seule transaction. `CHECK (balance >= 0)` reste en dernier recours.
3. **Historique trop lent.** Index `(account_id, created_at DESC)`.

## Profils

| Profil | Droits |
|---|---|
| `r_appli` | SELECT/INSERT/UPDATE sur clients et comptes, SELECT/INSERT sur virements, écritures, alertes, EXECUTE sur les procédures. Aucun UPDATE/DELETE sur les écritures |
| `r_support` | Uniquement la vue `v_customers_support` |
| `r_analyste` | Uniquement la vue matérialisée `mv_daily_volume` |
| `r_auditeur` | Lecture de `audit.audit_log` et `core.fraud_alerts` |



## Garanties de la base

- Email unique et valide, client majeur, IBAN unique, solde ≥ 0, montant > 0, comptes de virement différents (contraintes `CHECK`/`UNIQUE`).
- Grand livre immuable : triggers bloquant `UPDATE`, `DELETE`, `TRUNCATE` (même pour `postgres`).
- Un virement = exactement 2 écritures (1 débit + 1 crédit) : trigger de contrainte différé.
- Alertes : retrait ≥ 5 000 (`WITHDRAW_5000`), 5 virements sortants en 10 minutes (`TRANSFERS_10MIN`).
- Audit : changements sur `customers` et `accounts` journalisés (qui, quand, avant, après).

## Tests

`08_tests.sql` couvre toutes les contraintes, l'immuabilité, les fraudes, la procédure de virement (réussite et 4 refus sans écriture), l'audit, les droits de chaque profil et la cohérence des soldes. Les captures sont dans `screenshots/`.

## Limites connues

- `r_appli` garde un `INSERT` direct sur `transfers` et `transactions` : les triggers rattrapent les erreurs graves, mais une version plus stricte rendrait les procédures `SECURITY DEFINER` et retirerait ces droits.
- Les IBAN sont fictifs (pas de clé de contrôle).
- La détection de fraude se fait par trigger à l'insertion (pas de traitement différé).

## Déclaration d'usage de l'IA

J'ai utilisé Claude AI comme assistant pour comprendre mes scripts et les corrigés. Chaque scripts est testé relu et corrigé
