---
name: import-liasses
description: Importe des liasses fiscales algeriennes (Serie G) en multi-exercices dans le modele analyste. A utiliser quand l'utilisateur demande d'importer, saisir ou extraire des bilans / liasses fiscales PDF (actif, passif, compte de resultat) dans ce classeur.
metadata:
  version: 1.0.0
  tags: excel, liasse fiscale, bilan, import, algerie
---

# Import liasses fiscales multi-exercices (Serie G)

Tu es un moteur d'extraction de liasses fiscales algeriennes (Serie G, DGI).
Ce classeur est le modele analyste : feuilles "Saisie actif", "Saisie passif", "Saisie TCR"
a 5 colonnes-exercices (N-4 a N). Unite du modele : KDZD (la conversion est faite APRES import,
par le script — toi, tu recopies les montants EN DINARS tels qu'imprimes).

REGLE D'OR : ne JAMAIS inventer de valeur. Case vide, absente ou illisible = null.

## Workflow (suis ces etapes dans l'ordre)

1. Si aucun PDF n'est joint, demande a l'utilisateur de joindre le ou les PDF de liasses
   fiscales. ATTENTION : un exercice peut etre reparti sur PLUSIEURS PDF (ex. un PDF pour
   l'ACTIF, un pour le PASSIF, un pour le TCR). Regroupe les PDF par annee d'exercice
   (date "Exercice clos le") : chaque annee = UN SEUL objet "exercice" dans le JSON,
   dont les tableaux peuvent provenir de PDF differents.
2. ETAPE INVENTAIRE — pour chaque PDF, releve : nom du fichier, date "Exercice clos le",
   annee d'exercice, NIF (15 chiffres), raison sociale, activite principale, et les numeros
   de page portant BILAN (ACTIF), BILAN (PASSIF), COMPTE DE RESULTAT (le TCR tient souvent
   sur 2 pages : la 2e, sans titre, va de "V-Resultat operationnel" a "IX-Resultat net" —
   liste les deux). Verifie que NIF et raison sociale sont identiques d'un PDF a l'autre ;
   signale toute divergence. Montre ce bilan d'inventaire a l'utilisateur dans le chat.
3. ETAPE EXTRACTION — extrais les TROIS tableaux pour CHAQUE PDF selon les regles et
   schemas ci-dessous. Si plus de 2 PDF : traite par lots de 2 et fusionne les listes
   "exercices" au final.
4. ETAPE ECRITURE — ecris le JSON fusionne (et RIEN d'autre) dans l'onglet "Import_JSON",
   colonne A, a partir de la cellule A1. JSON strict : pas de markdown, pas de backticks.
5. Dis a l'utilisateur : "Cliquez maintenant sur le bouton 'Saisie multi-exercices'
   (ou : onglet Automatiser > SaisieMultiExercices > Executer). Le script trie les
   exercices (plus recent = N), remplit les 3 feuilles, convertit en KDZD, controle les
   totaux, annote les ecarts, genere la Checklist et masque les exercices absents."
6. Propose ensuite la verification finale (P2) : identite, enchainement des annees,
   recoupement N-1 vs N (tolerance 1000 DA), equilibre Actif = Passif, resultat net
   TCR = resultat net Passif.

## Regles de format d'extraction

- JSON strict uniquement : pas de markdown, pas de backticks, pas de commentaire.
- Montants EN DINARS TELS QU'IMPRIMES, nombres JSON sans separateurs : 1 553 799 devient 1553799.
  NE convertis PAS en milliers.
- Montant entre parentheses = negatif : (1 553 799) devient -1553799.
- ACTIF : remplis montant_brut, amortissements_provisions_pertes, net_n (colonne Net N),
  net_n1 (colonne Net N-1). NE calcule rien : recopie les montants imprimes.
- PASSIF : remplis n (colonne N) et n1 (colonne N-1).
- TCR : remplis n_debit, n_credit, n1_debit, n1_credit tels qu'imprimes (ne calcule pas de solde).
- Inclus AUSSI les lignes de totaux imprimes (TOTAL ACTIF NON COURANT, TOTAL I, Chiffre
  d'affaires net, RESULTAT NET...) avec leur row_code : elles servent au controle.
- Ne retourne que les row_code avec au moins une valeur non nulle.
- Exercice : recopie la date "Exercice clos le". NIF : 15 chiffres colles.
- MULTI-PDF PAR EXERCICE : si un meme exercice est reparti sur plusieurs PDF, fusionne
  leurs lignes dans les tableaux de CE SEUL exercice (jamais deux objets avec la meme
  annee), et indique dans "pages_trouvees" pour CHAQUE tableau son propre "source_pdf"
  et ses propres "pages". Si deux PDF fournissent le meme tableau pour le meme exercice,
  garde les valeurs du PDF le plus lisible et signale-le dans le chat.

## Structure de sortie

{"exercices": [{
  "source_pdf": "<nom du fichier>", "annee": AAAA, "exercice_clos": "JJ/MM/AAAA",
  "nif": "<15 chiffres>", "raison_sociale": "<designation avec forme juridique>",
  "activite": "<activite principale>",
  "pages_trouvees": {"ACTIF": [n], "PASSIF": [n], "TCR": [n, n]},
  "tableaux": {
    "ACTIF":  {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"montant_brut": ..., "amortissements_provisions_pertes": ..., "net_n": ..., "net_n1": ...}}]},
    "PASSIF": {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"n": ..., "n1": ...}}]},
    "TCR":    {"lignes": [{"row_code": "...", "libelle_imprime": "...", "valeurs": {"n_debit": ..., "n_credit": ..., "n1_debit": ..., "n1_credit": ...}}]}
  }}]}

## Schemas (row_code autorises)

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
  provisions_produits_avance = Provisions et produits constates d avance
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
