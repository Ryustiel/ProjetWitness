
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

### Action en Sortie

!
! M est une variable système WITNESS qui indique l'index de la pièce dans le lot de sortie
! On vérifie car M = 1 correspond au camion poubelle
IF M > 1 
!
! 1. Récupération des attributs depuis les tableaux globaux
	Dechet:TypeDechetE = VTabTypes(M - 1)
	Dechet:PoidsDechet = VTabPoids(M - 1)
!
! 2. Gestion des autres attributs dérivés
	Dechet:TypeDechetS = TypeDechetLabel(Dechet:TypeDechetE)
!
! 3. Gestion de l'icône
	Dechet:ICON = 130 + Dechet:TypeDechetE
!
ENDIF

### Loi de Sortie

IF M = 1
! Le premier élément est la pièce d'entrée (camion poubelle)
! On l'envoie au SCRAP pour symboliser qu'il n'a plus d'usage
	PUSH to SCRAP
ELSE 
	PUSH to Pile
ENDIF

### Note

Machine configurée en production

Problème Rencontré : La machine output le camion poubelle lui même (M=1) ce qui n'est pas correct.
