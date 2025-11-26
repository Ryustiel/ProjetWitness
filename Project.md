
Résumé de ce qui a été implémenté :


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

IF M = 1
	! Le premier élément est la pièce d'entrée (camion poubelle)
	! On l'envoie au SCRAP pour symboliser qu'il n'a plus d'usage
	PUSH to SCRAP
ELSE 
	! Envoie à la décharge si la pile est pleine
	PUSH to Pile,Décharge
ENDIF

### Note

Machine configurée en production

Problème Résolu : La machine output le camion poubelle lui même (car c'est une pièce en entrée) ce qui n'est pas correct. Dans la loi de sortie, on a défini un envoi au Scrap de la valeur M=1.

## Poste de Tri

### Loi d'Entrée

PULL from Pile

### Loi de Sortie

IF TypeDechetE = 1
	PUSH to StockCarton,Décharge
ELSEIF TypeDechetE = 2
	PUSH to StockPlastique,Décharge
ELSE 
	PUSH to StockOrganique,Décharge
ENDIF

## Incinérateur

### Loi d'Entrée

PULL from Décharge

### Actions en Entrée

! Variables locales
DIM vTauxRemplissage AS REAL
DIM vCoefPollutionMode AS REAL
DIM vEnergieLot AS REAL
DIM vPollutionLot AS REAL

! --- 1. GESTION DU MODE DE FONCTIONNEMENT (HYSTÉRÉSIS) ---

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


