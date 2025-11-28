
Résumé de ce qui a été implémenté :

# A priori

On représente par un article un groupe de déchets qui occupe un certain volume.
La propriété poids représente le poids cumulé de ce volume unitaire de déchets.
C'est pour ça qu'on utilise le nombre d'article fixe pour la décharge (qui est à même le sol)
et pour le shipping, alors qu'on a également une limite de poids (en plus du volume) pour le 
stockage de tri car dans notre facility il est placé dans des bennes spéciales qui ont une charge limite.

## Gestion des Arrivées & Ressources Humaines

Afin de rendre le modèle plus réaliste, des profils d'arrivée variables ont été configurés pour les camions et des plannings de travail ont été créés pour le personnel.

### 1. Profil d'arrivée des Camions Poubelle
L'arrivée des camions n'est plus constante. Elle suit désormais un **Profil d'arrivée (Arrival Profile)** défini sur une base de 24h, configuré via l'interface graphique (voir capture) pour simuler les tournées de collecte.

**Configuration du cycle (Heure de début | Durée | Inter-arrivée) :**
*   **00:00 - 05:00 (Nuit)** : Durée 300 min $\rightarrow$ Quantité **0** (Aucune arrivée).
*   **05:00 - 11:00 (Matin)** : Durée 360 min $\rightarrow$ 1 camion toutes les **20** min.
*   **11:00 - 13:00 (Midi)** : Durée 120 min $\rightarrow$ 1 camion toutes les **2** min (Pic d'activité).
*   **13:00 - 16:00 (Après-midi)** : Durée 180 min $\rightarrow$ 1 camion toutes les **12** min.
*   **16:00 - 00:00 (Soir)** : Durée 480 min $\rightarrow$ 1 camion toutes les **5** min.

### 2. Gestion du Personnel (Labor & Shifts)
Deux catégories d'employés ont été créées avec des horaires distincts pour couvrir les besoins de tri et de maintenance.

#### A. Opérateurs de Tri
*   **Ressource :** `OperateurTri`
*   **Rôle :** Assurent le fonctionnement du poste de tri et la maintenance mineure.
*   **Planning associé :** `PlanningOperateurTri`
*   **Séquence de travail (en minutes) :**
    1.  **300** Travail (5h)
    2.  **60** Pause
    3.  **300** Travail (5h)
    4.  **780** Repos (Fin de journée)

NOTE : Le déchargement ne nécessite pas de ressources car on considère que c'est l'équipe du camion poubelle
lui même qui se charge de décharger leur camion dans la décharge.
De même pour le poste de chargement, il s'agit du camioneur qui vient récupérer ses palettes.
Mais pour le chargement il y a quand même 20 minutes que passe un employé pour préparer les palettes (phase setup).

#### B. Techniciens de Maintenance
*   **Ressource :** `TechnicienMaintenance`
*   **Rôle :** Dédiés aux pannes majeures (ex: Incinérateur).
*   **Planning associé :** `PlanningTechnicien`
*   **Séquence de travail (en minutes) :**
    1.  **180** Travail (3h)
    2.  **60** Pause
    3.  **180** Travail (3h)
    4.  **1020** Repos (Fin de journée)

# Simulations

## Camion Poubelle

### Actions à la Création

LENGTH = 3
WIDTH = 1
HEIGHT = 1.25
PoidsCamion = Triangle (20,100,300) ! Triangle (400,5500,8000)

### Loi de Sortie

PUSH to ParkingPoubelle(1)

## Machine Déchargement

### Action en Entrée

! Variables locales pour le calcul temporaire
DIM vChargeRestante AS REAL
DIM vMasseCandidate AS REAL
DIM vTypeCandidat AS INTEGER
!
! Initialisation pour ce nouveau camion
VNbDechets = 0
vChargeRestante = PoidsCamion
!
! --- BOUCLE DE CALCUL ---
WHILE vChargeRestante > 0 AND VNbDechets < VMaxNbDechets
!
! A. Génération des candidats
	vTypeCandidat = DistTypeDechet ()
!
	IF vTypeCandidat = 1  ! Carton
		vMasseCandidate = Triangle (3.0,20.0,40.0)
	ELSEIF vTypeCandidat = 2  !Plastique
		vMasseCandidate = Triangle (0.5,10.0,50.0)
	ELSE
		vMasseCandidate = Triangle (2.0,10.0,20.0)
	ENDIF
!
! B. Vérification
	IF vChargeRestante - vMasseCandidate >= 0 
!
! C. On enregistre dans les tableaux globaux pour les attribuer aux objets créés plus tard
		VNbDechets = VNbDechets + 1
!
! On stocke les infos à l'index correspondant (ex: déchet 1, 2...)
		VTabTypes(VNbDechets) = vTypeCandidat
		VTabPoids(VNbDechets) = vMasseCandidate
!
! Décrémentation
		vChargeRestante = vChargeRestante - vMasseCandidate
!
	ELSE
! Trop lourd, on arrête
! On part du principe que la masse restante dans le camion correspond à des liquides de fuite ou d'autres matériaux non traitables
		vChargeRestante = 0
	ENDIF
!
ENDWHILE
!
! À la fin de ces actions, la machine sait maintenant que vNbDechets (ex: 15)
! est la Quantité de Sortie grâce au paramétrage de l'étape 2.

### Actions en Sortie

!
! M est une variable système WITNESS qui indique l'index de la pièce dans le lot de sortie
! On vérifie car M = 1 correspond au camion poubelle
IF M > 1
!
! 1. Récupération des attributs depuis les tableaux globaux
	TypeDechetE = VTabTypes(M - 1)
	PoidsDechet = VTabPoids(M - 1)
!
! 2. Gestion des autres attributs dérivés
	TypeDechetS = TypeDechetLabel(TypeDechetE)
!
! 3. Gestion de l'icône
	ICON = 130 + TypeDechetE
!
ENDIF

### Loi de Sortie

IF M = 1  ! Premier élément = camion vide
    ! Le camion vide part au rebut
    PUSH to SCRAP
ELSE
    ! --- Logique de restriction par Poids ---
    
    ! On vérifie si l'ajout de CE déchet fera déborder la pile
    IF (VPoidsPileActuel + PoidsDechet) <= VCapacitePileMax
        
        ! Il y a de la place en poids : On tente la Pile en priorité.
        ! Si la Pile est pleine en nombre de pièces,
        ! la règle standard "Pile, Décharge" enverra à la Décharge.
        PUSH to Pile, Décharge
        
    ELSE
        ! La Pile est trop lourde : Déroutement immédiat vers la Décharge
        ! (Ceci simule une zone de tri saturée)
        PUSH to Décharge
    ENDIF
ENDIF

### Note

Machine configurée en production

Problème Résolu : La machine output le camion poubelle lui même (car c'est une pièce en entrée) ce qui n'est pas correct. Dans la loi de sortie, on a défini un envoi au Scrap de la valeur M=1.

## Pile de Traitement

### Capacité

Possède une capacité en nombre d'éléments (200) qu'on peut faire correspondre à un volume
et une capacité en Masse (lié à des contraintes de stabilité) symbolisée par les variables :

VCapacitePileMax = 1000.0
et VPoidsPileActuel

## Poste de Tri

### Loi d'Entrée

PULL from Pile

### Loi de Sortie

IF TypeDechetE = 1
! Si le poids actuel + le poids de la pièce < Max, on stocke (si la capacité du stock n'est pas atteinte aussi)
	IF VStockPoidsActuel(1) + PoidsDechet <= VStockCapaciteMax(1)
		PUSH to StockCarton,Décharge
	ELSE 
! Sinon Décharge
		PUSH to Décharge
	ENDIF
ELSEIF TypeDechetE = 2
	IF VStockPoidsActuel(2) + PoidsDechet <= VStockCapaciteMax(2)
		PUSH to StockPlastique,Décharge
	ELSE 
		PUSH to Décharge
	ENDIF
ELSE 
	 ! REGLE IMPORTANTE : Maintien du feu de l'incinérateur
    ! Si le nombre de pièces dans la décharge est inférieur à 5% de sa capacité totale
    IF NPARTS(Décharge) < (0.05 * VCapaciteDecharge)
        
        ! On force l'envoi vers la décharge pour alimenter l'incinérateur
        PUSH to Décharge
        
    ELSE
        ! Sinon, fonctionnement normal (Stockage ou Décharge si stock plein)
        IF VStockPoidsActuel(3) + PoidsDechet <= VStockCapaciteMax(3)
            PUSH to StockOrganique, Décharge
        ELSE 
            PUSH to Décharge
        ENDIF
    ENDIF
ENDIF

## Incinérateur

### VCapaciteDecharge

VCapaciteDecharge = 200
! Permet de calculer les seuils de passage en fonctionnement rapide

### Loi d'Entrée

PULL from Décharge

### Actions en Entrée

! Variables locales
DIM vTauxRemplissage AS REAL
DIM vCoefPollutionMode AS REAL
DIM vEnergieLot AS REAL
DIM vPollutionLot AS REAL

! --- 1. GESTION DU MODE DE FONCTIONNEMENT ---

! Calcul du taux de remplissage actuel de la décharge
vTauxRemplissage = NPARTS(Décharge) / VCapaciteDecharge

IF VModeSurregime = 0
    ! Si on est en mode NORMAL
    ! On passe en surrégime si on dépasse 80%
    IF vTauxRemplissage >= 0.80
        VModeSurregime = 1
        VDebutSurregime = TIME
        PRINT "Démarrage Surrégime Incinérateur à : ", TIME
    ENDIF

ELSE
    ! Si on est en mode SURRÉGIME
    ! Condition d'arrêt 1 : On est repassé sous les 50%
    ! Condition d'arrêt 2 : Cela fait plus de 4h (240 min) qu'on est en surrégime
    
    IF vTauxRemplissage <= 0.50 OR (TIME - VDebutSurregime >= 240.0)
        VModeSurregime = 0
        PRINT "Arrêt Surrégime Incinérateur à : ", TIME
    ENDIF
ENDIF

! --- 2. CALCUL DES RÉSULTATS (ENERGIE & POLLUTION) ---

! Calcul de l'énergie théorique de la pièce (Masse * Pouvoir Calorifique du type)
vEnergieLot = PoidsDechet * VTabCalorifique(TypeDechetE)

! Calcul de la pollution de base
vPollutionLot = PoidsDechet * VTabPollution(TypeDechetE)

! Application des conséquences du mode
IF VModeSurregime = 1
    ! En surrégime :
    ! La pollution augmente "grandement" (ex: facteur 5 car combustion incomplète ou forcée)
    vPollutionLot = vPollutionLot * 5.0
    
    ! Optionnel : On pourrait dire que l'efficacité énergétique baisse un peu en surrégime,
    ! ou reste stable. Ici on la laisse stable.
ELSE
    ! Mode normal, pas de pénalité
    vPollutionLot = vPollutionLot * 1.0
ENDIF

! Mise à jour des variables globales de suivi
VEnergieProduite = VEnergieProduite + vEnergieLot
VPollutionTotale = VPollutionTotale + vPollutionLot

### Temps de Cycle

! On définit un temps de brûlage de base par Kg (ex: 2 min pour brûler 1kg)
! Si Mode = 1 (Surrégime), on divise ce temps par 3 (brûle 3x plus vite)
! Sinon temps normal.

PoidsDechet * 2.0 / (1 + (VModeSurregime * 2))

### Loi de Sortie

PUSH to SCRAP

### Initialisation de VCapaciteDechargeMax

VCapaciteDechargeMax = 200.0 ! 200 Tonnes pour la décharge

### Initialisation de VTabCalorifique

! --- Initialisation des Pouvoirs Calorifiques (PCI) en kWh/kg ---

! Type 1 : Carton / Papier
! Valeur moyenne : ~3.5 à 4.5 kWh/kg (bonne combustibilité)
VTabCalorifique(1) = 4.0

! Type 2 : Plastique
! Valeur moyenne : ~9.0 à 11.0 kWh/kg (très haute énergie, dérivé du pétrole)
VTabCalorifique(2) = 9.5

! Type 3 : Organique / Déchets alimentaires
! Valeur moyenne : ~0.8 à 1.5 kWh/kg (très faible à cause de la teneur en eau ~70-80%)
VTabCalorifique(3) = 1.2

### Initialisation de VTabPollution

! --- Initialisation des Facteurs de Pollution (Indice) ---

! Type 1 : Carton
! Base de référence (1.0). Carbone biogénique, fumées standards.
VTabPollution(1) = 1.0

! Type 2 : Plastique
! Indice élevé (3.5). Émissions de CO2 fossile + risques de composés toxiques (chlore/dioxines).
VTabPollution(2) = 3.5

! Type 3 : Organique
! Indice faible (0.3). Combustion "propre" mais incomplète si trop humide, génère surtout de la vapeur.
VTabPollution(3) = 0.3

## Stocks Triés

### Initialisation de VStockCapaciteMax

VStockCapaciteMax(1) = 500.0  ! 5 Tonnes max pour Carton
VStockCapaciteMax(2) = 300.0  ! 3 Tonnes max pour Plastique
VStockCapaciteMax(3) = 800.0  ! 8 Tonnes max pour Organique

### Action en Entrée pour chaque Stock

! En utilisant l'index approprié (1 pour Carton)
VStockPoidsActuel(1) = VStockPoidsActuel(1) + PoidsDechet

### Action en Sortie pour chaque Stock

VStockPoidsActuel(1) = VStockPoidsActuel(1) - PoidsDechet

## Chargement

### Temps de Cycle

! Pour simuler le temps d'arrivée d'un camion de livraison
Uniform(20,2000)

### Réglage

Un opérateur doit être présent initialement pour assembler la palette.
Il y a ensuite un temps de traitement qui correspond à l'appel d'un camion,
son arrivée et le chargement de la palette.

*   **Dissociation Tâche Humaine / Temps Machine (Via l'onglet Réglage) :**
    *   **Choix :** Nous avons déplacé les 20 minutes d'intervention humaine dans l'onglet **Réglage (Setup)** plutôt que dans le temps de cycle global.
    *   **Pourquoi ?** Si l'opérateur était assigné au *Cycle*, il resterait "bloqué" virtuellement pendant tout le trajet du camion (temps défini par la loi `Uniform`). En utilisant le *Réglage*, l'opérateur est mobilisé uniquement pour les 20 minutes de chargement, puis est libéré pour retourner au tri pendant que le camion effectue sa rotation autonome.

*   **Gestion de l'Urgence (Priorité = 2) :**
    *   **Choix :** La priorité a été rendue plus forte que celle du Poste de Tri.
    *   **Pourquoi ?** Les ressources `OperateurTri` sont partagées. Sans ce réglage, les opérateurs continueraient de trier indéfiniment tant qu'il y a des déchets, ignorant les camions en attente. Ce paramétrage force l'opérateur à quitter le poste de tri dès qu'une expédition est prête, garantissant la fluidité des sorties.

*   **Forçage Systématique (Opérations avant 1er réglage = 0) :**
    *   **Choix :** Le réglage est configuré pour se déclencher toutes les **1** opérations, avec un "Nbre d'opérations au 1er réglage" à **0**.
    *   **Pourquoi ?** Par défaut, WITNESS peut interpréter le réglage comme un changement de série. Ce paramétrage contraint le logiciel à considérer ce temps de chargement comme une étape obligatoire pour **chaque** palette, y compris la toute première de la simulation.

### Initialisation de VSeuilExpedition

! Seuil pour qu'un camion vienne chercher la marchandise (ex: groupe de 20 objets)
VSeuilExpedition = 20

### Initialisation de VTabPrixVente

! --- Initialisation des Prix de Vente (€/kg) ---
! Type 1 : Carton (env. 100€/tonne -> 0.10/kg)
VTabPrixVente(1) = 0.10

! Type 2 : Plastique (env. 300€/tonne -> 0.30/kg)
VTabPrixVente(2) = 0.30

! Type 3 : Organique (env. 20€/tonne -> 0.02/kg pour compost/méthanisation)
VTabPrixVente(3) = 0.02

### Loi d'Entrée

IF NPARTS(Chargement) = 0
    ! DÉCISION
    ! On ne lance un camion que si un stock atteint le Seuil d'Expédition
    ! Et on choisit le plus rempli parmi ceux qui dépassent ce seuil.

    IF NPARTS(StockCarton) >= VSeuilExpedition AND NPARTS(StockCarton) >= NPARTS(StockPlastique) AND NPARTS(StockCarton) >= NPARTS(StockOrganique)
        PULL from StockCarton
    ELSEIF NPARTS(StockPlastique) >= VSeuilExpedition AND NPARTS(StockPlastique) >= NPARTS(StockOrganique)
        PULL from StockPlastique
    ELSEIF NPARTS(StockOrganique) >= VSeuilExpedition
        PULL from StockOrganique
    ELSE
        WAIT
    ENDIF

ELSE
    ! REMPLISSAGE (VERROUILLAGE)
    ! Une fois la première pièce entrée, on ignore le seuil et les quantités.
    ! On complète le chargement uniquement avec le type mémorisé.
    
    IF VTypeLotEnCours = 1
        PULL from StockCarton
    ELSEIF VTypeLotEnCours = 2
        PULL from StockPlastique
    ELSE
        PULL from StockOrganique
    ENDIF
ENDIF
! Avant on utilisait : MOST PARTS StockCarton, StockPlastique, StockOrganique

### Action en Entrée

! Si c'est la première pièce du lot
IF NParts (Chargement) = 1 
	VTypeLotEnCours = TypeDechetE
ENDIF
!
! Calcul du revenu généré par ce déchet spécifique
! Formule : Poids du déchet (kg) * Prix au kg du type de déchet
VRevenuTotal = VRevenuTotal + PoidsDechet * VTabPrixVente(TypeDechetE)

### Action en Sortie

! Si c'est la dernière pièce du lot (correspondant au seuil)
IF M = VSeuilExpedition
   VTypeLotEnCours = 0
ENDIF

### Loi de Sortie

PUSH TO SHIP

### Note

Si on ajoute des postes à la machine de Chargement, il faut étendre la variable VTypeLotEnCours
pour qu'il en existe une instance pour chaque machine (/ un index de liste)

### Stratégie de Maintenance des Équipements

Afin de simuler les aléas techniques et la disponibilité des ressources humaines (les techniciens étant moins nombreux et ayant des plages horaires plus restreintes que les opérateurs), un système de pannes à deux niveaux a été configuré sur les machines critiques. Toutes les pannes sont définies sur la base du **Temps de Fonctionnement (Busy Time)**.

**1. Machine de Tri : Pannes Mineures**
*   **Fréquence :** Moyenne d'une panne toutes les 120 minutes (`NegExp(120)`).
*   **Durée :** Intervention courte de 5 à 15 minutes (`Uniform(5, 15)`).
*   **Gestion de la Ressource :** La réparation est assignée via la règle `TechnicienMaintenance OR OperateurTri`. Cette flexibilité est nécessaire pour éviter de bloquer la chaîne de tri : si l'équipe de maintenance est absente ou occupée, un opérateur de tri peut intervenir pour relancer la machine.

**2. Incinérateur : Pannes Hybrides**
L'incinérateur est soumis à deux types d'aléas distincts pour refléter sa complexité :
*   **Pannes Mineures (Réglages courants) :**
    *   **Fréquence :** Moyenne d'une panne toutes les 200 minutes (`NegExp(200)`).
    *   **Durée :** 10 à 20 minutes (`Uniform(10, 20)`).
    *   **Ressource :** Règle partagée `TechnicienMaintenance OR OperateurTri`. Les opérateurs peuvent effectuer ces réglages simples pour maintenir le flux si aucun technicien n'est disponible.
*   **Pannes Majeures (Casse critique) :**
    *   **Fréquence :** Événement rare, moyenne d'une fois toutes les 2000 minutes (`NegExp(2000)`).
    *   **Durée :** Immobilisation lourde de 2 à 4 heures (`Uniform(120, 240)`).
    *   **Ressource :** Exclusivement réservée au `TechnicienMaintenance`. Ces pannes nécessitent une compétence technique spécifique que les opérateurs ne possèdent pas.

### Exclusion des Postes de Chargement et Déchargement

Bien que le **Déchargement** (entrée des déchets) et le **Chargement** (expédition des matières premières) soient modélisés par des éléments "Machine" dans WITNESS, aucune panne ne leur a été attribuée.

Cette décision se justifie par la nature de ces opérations :
1.  **Déchargement :** Cet élément représente l'action du camion-poubelle déversant son contenu. Une "panne" ici signifierait une panne mécanique du camion lui-même. Or, les camions sont des entités externes ; leur indisponibilité est déjà simulée en amont par le profil d'arrivée variable (Arrival Profile).
2.  **Chargement :** De la même manière, cet élément simule l'arrivée d'un transporteur logistique externe venant récupérer les palettes. L'infrastructure du quai de chargement (simple espace physique) a une probabilité de panne négligeable par rapport aux équipements industriels comme le trieur ou l'incinérateur.
