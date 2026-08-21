# Pack Prompts Copilot 365 — Modèle multi-exercices (« bilan_optA » v2)

> **Modèle cible** : `MODEL ETAT FINANCIER - ANALYSTE` — feuilles **Saisie actif / Saisie passif /
> Saisie TCR** à 5 colonnes-exercices (N-4 → N) ; les autres onglets (Saisie, Etatsfin, Ratios, Hypo,
> PrSept, Dette, Graph) se remplissent automatiquement. **Unité du modèle : KDZD** (milliers de dinars).
>
> **Logique** : Copilot détecte les exercices dans le ou les PDF, produit un JSON multi-exercices ;
> le bouton « Saisie multi-exercices » (Office Script) trie (dernier exercice = N), remplit, contrôle,
> commente, génère la **Checklist** et **masque les colonnes des exercices non trouvés**.
>
> **Règle d'or** : ne jamais inventer de valeur — absent/illisible → `null`.

## Ne plus retaper les prompts — 3 options

| Option | Effort | Rendu |
|---|---|---|
| **A. Onglet « Prompts » dans le modele** | 2 min, immediat | Les 3 prompts (P0/P1/P2) colles chacun dans une cellule ; l'analyste double-clique, Ctrl+C, colle dans Copilot. Recommande pour le pilote. |
| **B. Galerie de prompts (Prompt Gallery)** | 10 min | Enregistrer P0/P1/P2 comme prompts d'equipe dans Copilot (icone « … » du volet Copilot > « Prompts enregistres ») : accessibles a tous les analystes en 2 clics. |
| **C. Skill Copilot « Import liasses »** | 15 min | Le texte de la skill (fin de ce document) encapsule tout le workflow : l'analyste tape juste « importer les liasses ». |

---

## Parcours analyste

1. Ouvrir le modèle sur SharePoint/OneDrive, ouvrir Copilot, attacher le(s) PDF (1 PDF = 1 exercice).
2. Lancer **P0** → vérifier l'inventaire des exercices et des pages.
3. Lancer **P1** (par lots de 2 PDF si plus de 2 fichiers, puis « fusionne les listes exercices »).
4. Demander : *« Écris le JSON final dans l'onglet Import_JSON, colonne A. »*
5. Cliquer sur le bouton **« Saisie multi-exercices »**.
6. Lancer **P2** (vérifications inter-exercices), revoir la **Checklist** et les commentaires.

---

## P0 — INVENTAIRE DES EXERCICES ET DES PAGES

```
Tu recois UN OU PLUSIEURS PDF de liasses fiscales algeriennes (Serie G).
ATTENTION : un exercice peut etre reparti sur PLUSIEURS PDF (ex. un PDF pour l'ACTIF,
un pour le PASSIF, un pour le TCR). Regroupe les PDF par annee d'exercice.

ETAPE 1 — Pour chaque PDF, releve :
- le nom du fichier,
- la date de cloture ("Exercice clos le JJ/MM/AAAA") et l'annee d'exercice,
- les numeros de page portant : BILAN (ACTIF), BILAN (PASSIF), COMPTE DE RESULTAT.
  Rappel : le COMPTE DE RESULTAT occupe souvent 2 pages (la 2e sans titre, de V-Resultat
  operationnel a IX-Resultat net) : liste les deux pages.

ETAPE 2 — Verifie la coherence : NIF (15 chiffres) et raison sociale identiques d'un PDF a l'autre.
Signale toute divergence.

Reponds en JSON strict (pas de markdown) :
{"exercices": [{"source_pdf": "<pdf principal>", "annee": AAAA, "exercice_clos": "JJ/MM/AAAA",
  "nif": "...", "raison_sociale": "...", "activite": "...",
  "pages_trouvees": {"ACTIF": {"pages": [n], "source_pdf": "..."},
                     "PASSIF": {"pages": [n], "source_pdf": "..."},
                     "TCR": {"pages": [n, n], "source_pdf": "..."}}}],
 "alertes": ["..."]}
```

---

## P1 — EXTRACTION MULTI-EXERCICES (ACTIF / PASSIF / TCR)

```
Tu es un moteur d'extraction de liasses fiscales algeriennes (Serie G, DGI).
Tu recois UN OU PLUSIEURS PDF. Chaque PDF = un exercice, avec pour chaque tableau les colonnes N et N-1.
Extrais les TROIS tableaux (Bilan Actif, Bilan Passif, Compte de Resultat) pour CHAQUE PDF.

REGLE D'OR : ne JAMAIS inventer de valeur. Case vide, absente ou illisible = null.

REGLES DE FORMAT :
- JSON strict uniquement : pas de markdown, pas de backticks, pas de commentaire.
- Montants EN DINARS TELS QU'IMPRIMES, nombres JSON sans separateurs : 1 553 799 devient 1553799.
  (NE convertis PAS en milliers : la conversion DZD -> KDZD est faite automatiquement apres import.)
- Montant entre parentheses = negatif : (1 553 799) devient -1553799.
- ACTIF : remplis montant_brut, amortissements_provisions_pertes, net_n (colonne Net N),
  net_n1 (colonne Net N-1). NE calcule rien : recopie les montants imprimes.
- PASSIF : remplis n (colonne N) et n1 (colonne N-1).
- TCR : remplis n_debit, n_credit, n1_debit, n1_credit tels qu'imprimes (ne calcule pas de solde).
  Le TCR tient souvent sur 2 pages : la 2e (sans titre ni en-tetes de colonnes, de V-Resultat
  operationnel a IX-Resultat net) fait partie du meme tableau. Extrais les deux pages ensemble.
- Inclus AUSSI les lignes de totaux imprimes (TOTAL ACTIF NON COURANT, TOTAL I, Chiffre d'affaires
  net, RESULTAT NET...) avec leur row_code : elles servent au controle, pas a la saisie.
- Ne retourne que les row_code avec au moins une valeur non nulle.
- Exercice : recopie la date "Exercice clos le". NIF : 15 chiffres colles.

STRUCTURE DE SORTIE :
{"exercices": [{
  "source_pdf": "<nom du fichier>", "annee": AAAA, "exercice_clos": "JJ/MM/AAAA",
  "nif": "<15 chiffres>", "raison_sociale": "<designation avec forme juridique>", "activite": "<activite principale>",
  "pages_trouvees": {"ACTIF": [n], "PASSIF": [n], "TCR": [n, n]},
  "tableaux": {
    "ACTIF":  {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"montant_brut": ..., "amortissements_provisions_pertes": ..., "net_n": ..., "net_n1": ...}}]},
    "PASSIF": {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"n": ..., "n1": ...}}]},
    "TCR":    {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"n_debit": ..., "n_credit": ..., "n1_debit": ..., "n1_credit": ...}}]}
  }}]}

SCHEMAS (row_code autorises) :
--- ACTIF : BILAN (ACTIF) ---
Colonnes autorisees : montant_brut, amortissements_provisions_pertes, net_n, net_n1
row_code autorises :
  ecarts_acquisition_goodwill = Ecart d acquisition goodwill
  immobilisations_incorporelles = Immobilisations incorporelles
  terrains = Terrains
  batiments = Batiments
  autres_immobilisations_corporelles = Autres Immobilisations corporelles
  immobilisations_corporelles = Immobilisations corporelles
  immobilisations_en_concession = Immobilisations en concession
  immobilisations_en_cours = Immobilisations en cours
  titres_mis_en_equivalence = Titres mis en equivalence
  autres_participations_creances = Autres participations et creances rattachees
  autres_titres_immobilises = Autres titres immobilises
  prets_actifs_financiers_non_courants = Prets et autres actifs financiers non courants
  impots_differes_actif = Impots Differes Actif
  immobilisations_financieres = Immobilisations financieres
  total_actif_non_courant = TOTAL ACTIF NON COURANT
  stocks_encours = Stocks et encours
  clients = Clients
  autres_debiteurs = Autres debiteurs
  impots_assimiles_actif = Impots et assimiles
  autres_creances_assimiles = Autres Creances et Emplois assimiles
  creances_emplois_assimiles = Creances et emplois assimiles
  disponibilites_assimiles = Disponibilites et assimiles
  placements_financiers_courants = Placements et autres actifs financiers courants
  tresorerie_actif = Tresorerie
  total_actif_courant = TOTAL ACTIF COURANT
  total_general_actif = TOTAL GENERAL ACTIF

--- PASSIF : BILAN (PASSIF) ---
Colonnes autorisees : n, n1
row_code autorises :
  capital_emis = Capital emis
  capital_non_appele = Capital non appele
  primes_reserves = Primes et reserves
  ecart_reevaluation = Ecart de reevaluation
  ecart_equivalence = Ecart d equivalence
  resultat_net_passif = Resultat net
  report_a_nouveau = Report a nouveau
  part_societe_consolidante = Part de la societe consolidante
  part_minoritaires = Part des minoritaires
  total_capitaux_propres = TOTAL I
  emprunts_dettes_financieres = Emprunts et dettes financieres
  impots_differes_provisionnes = Impots differes et provisionnes
  autres_dettes_non_courantes = Autres dettes non courantes
  provisions_produits_avance = Provisions et produits constatés d avance
  total_passifs_non_courants = TOTAL PASSIFS NON COURANTS II
  fournisseurs_rattaches = Fournisseurs et comptes rattaches
  impots_passif = Impots
  autres_dettes = Autres dettes
  tresorerie_passif = Tresorerie Passif
  total_passifs_courants = TOTAL PASSIFS COURANTS
  total_general_passif = TOTAL GENERAL PASSIF

--- TCR : COMPTE DE RESULTAT ---
Colonnes autorisees : n_debit, n_credit, n1_debit, n1_credit
row_code autorises :
  ventes_marchandises = Ventes de Marchandises
  produits_fabriques = Produits Fabriques
  prestations_services = Prestations de Services
  ventes_travaux = Ventes de Travaux
  produits_annexes = Produits Annexes
  rabais_remises_ristournes_accordes = Rabais remises ristournes accordes
  chiffre_affaires_net = Chiffre d affaires net
  production_stockee_destockee = Production Stockee ou destockee
  production_immobilisee = Production immobilisee
  subvention_exploitation = Subvention d exploitation
  production_exercice = I-Production de l exercice
  achats_marchandises_vendues = Achats de Marchandises vendues
  matieres_premieres = Matieres premieres
  autres_approvisionnements = Autres Approvisionnements
  variation_stocks = Variation des Stocks
  achats_etudes_prestations = Achats d Etudes et de Prestations de services
  rabais_remises_ristournes_obtenus_achats = Rabais remises ristournes obtenus sur achats
  autres_consommations = Autres consommations
  sous_traitance_generale = Sous-traitance generale
  locations = Locations
  entretien_reparations = Entretien reparations et maintenance
  primes_assurances = Primes d assurances
  personnel_exterieur = Personnel exterieur a l entreprise
  remuneration_intermediaires = Remuneration d intermediaires et honoraires
  publicite = Publicite
  deplacements_missions = Deplacements missions et receptions
  rabais_remises_ristournes_obtenus_services = Rabais remises ristournes obtenus sur services exterieurs
  autres_services = Autres services
  consommations_exercice = II-Consommations de l exercice
  valeur_ajoutee_exploitation = III-Valeur ajoutee d exploitation
  charges_personnel = Charges de personnel
  impots_taxes_assimiles = Impots et taxes et versements assimiles
  excedent_brut_exploitation = IV-Excedent brut d exploitation
  autres_produits_operationnels = Autres produits operationnels
  autres_charges_operationnelles = Autres charges operationnelles
  dotations_amortissements = Dotations aux amortissements
  provisions = Provisions
  pertes_valeur = Perte de Valeur
  reprises_pertes_valeur_provisions = Reprise sur pertes de valeur et provisions
  resultat_operationnel = V-Resultat operationnel
  produits_financiers = Produits financiers
  charges_financieres = Charges financieres
  resultat_financier = VI-Resultat Financier
  resultat_ordinaire = VII-Resultat ordinaire
  elements_extraordinaires_produits = Elements extraordinaires Produits
  elements_extraordinaires_charges = Elements extraordinaires Charges
  resultat_extraordinaire = VIII-Resultat extraordinaire
  impots_exigibles_resultats = Impots exigibles sur resultats
  impots_differes_resultats = Impots differes sur resultats
  resultat_net_exercice = RESULTAT NET DE L EXERCICE

Si le resultat est long, termine le JSON proprement ; si tout ne tient pas en une reponse,
dis "SUITE" et attends que je te demande la suite.
```

---

## P2 — VÉRIFICATION FINALE MULTI-EXERCICES

```
Tu es le verificateur final. Voici le JSON multi-exercices extrait des liasses (onglet Import_JSON).
Verifie et reponds en JSON strict {"verifications": [{"controle": "...", "statut": "OK|ECART|NON_VERIFIABLE", "detail": "..."}]} :

1. Identite : NIF (15 chiffres) et raison sociale identiques sur tous les PDF.
2. Enchainement des exercices : les annees se suivent sans trou (ex. 2021-2025). Signale tout exercice manquant.
3. Recoupement : pour chaque annee Y couverte par deux PDF, la colonne N-1 du PDF Y+1 doit etre
   EGALE a la colonne N du PDF Y (tolerance 1000 DA). Liste UNIQUEMENT les postes divergents
   au-dela de 1000 DA.
4. Equilibre par exercice : ACTIF total_general_actif net_n = PASSIF total_general_passif n.
5. Resultat net : TCR resultat_net_exercice (credit - debit) = PASSIF resultat_net_passif n.
6. Total ACTIF = somme N courant + non courant ; CA net = somme des lignes de ventes.

Tolerance : +/- 1 DA = arrondi acceptable (statut OK, detail "arrondi"). Aucune valeur inventee.
```

---

## Feuille `.Rules` du classeur

```
# Feuille .Rules — Model Etat Financier Analyste (multi-exercices)
- 3 feuilles de saisie : "Saisie actif" (groupes de 3 colonnes par exercice : brut / amort. / net),
  "Saisie passif" (1 colonne par exercice), "Saisie TCR" (paires debit/credit par exercice).
- Exercices : N = le plus recent, puis N-1 ... N-4. Maximum 5 exercices.
- Unite du modele : KDZD (milliers de dinars). Ne JAMAIS ecrire dans ces feuilles depuis Copilot :
  l'import se fait via le JSON dans "Import_JSON" + le bouton "Saisie multi-exercices".
- Ne jamais modifier une cellule contenant une formule (Net, sous-totaux, totaux, soldes TCR).
- L'onglet "Checklist" est genere automatiquement (pages trouvees OK/KO, recoupements, equilibres).
- Les onglets Saisie, Etatsfin, Ratios, Hypo, PrSept, Dette, Graph se recalculent seuls : ne pas y toucher.
```

---

## Skill Copilot in Excel

```
# Skill Copilot in Excel — "Import liasses multi-exercices"
Nom : Import liasses multi-exercices
Declencheur : "importer les liasses"
Instructions :
1. Demande le ou les PDF de liasses fiscales (un PDF = un exercice).
2. Execute le prompt P0 (inventaire des exercices et pages) et montre le resultat a l'utilisateur.
3. Execute le prompt P1 (extraction ACTIF/PASSIF/TCR, montants en DZD tels qu'imprimes).
   Si plus de 2 PDF : traite par lots de 2 et fusionne les listes "exercices".
4. Ecris le JSON fusionne dans l'onglet Import_JSON, colonne A, a partir de A1.
5. Dis a l'utilisateur de cliquer sur le bouton "Saisie multi-exercices" :
   le script trie les exercices (plus recent = N), remplit les 3 feuilles, convertit en KDZD,
   controle les totaux, annote les ecarts, genere la Checklist et masque les exercices absents.
6. Propose ensuite le prompt P2 (verification finale).
Regle d'or : ne jamais inventer de valeur ; case vide ou illisible = null.
```
