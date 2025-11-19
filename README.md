
# Projet Witness RO06

## Sujet

Modéliser une déchéterie municipale.

Les composants clés sont :
1. Une arrivée de déchets avec des déchets de différents types et de différentes taille (volume occupé dans le stockage)
2. Des posts de tri capables de traiter des déchets et en faire des palettes. Mobilisent des employés et peuvent tomber en panne.
3. Equipe de maintenance lente envoyée par l'agglomération en cas de panne.
4. Un incinérateur pour les déchets non traitables. Les déchets sont dirigés vers l'incinérateur si la décharge atteint un seuil critique et que les posts de traitement tombent en panne.
5. (pourquoi pas?) Part de déchets inflammables, rester en dessous d'un seuil. Ne pas les faire entrer dans le stockage si il y en a déjà trop. Suivre ce taux avec un variable et engendrer des décisions somehow.

## Objets

### Déchets

Article Witness.
Propriété type (Carton, Plastique, Organique)
Propriété taille (random gaussien 1-20kg pour bois, 1-5kg pour les autres pour l'instant) ; varie selon type.

Les déchets arrivent par vague à certaines heures de la journée,
en fonction de la rotation des camions poubelle.

### Employés

Ressource Witness.
Roles (Maintenance, Tri)

### Incinérateur

Génère de l'électricité pour la ville. Ne doit pas être surutilisée et doit fonctionner de préférence la nuit pour compenser le renouvelable.
Capacité de traitement flexible mais variable en fonction de l'heure de la journée.
Doit conserver une marge de déchets dans la décharge éligibles pour l'incinérateur, quitte à ne plus trier si l'incinérateur a besoin pour son fonctionnement.

### Poste de tri

Mobilise des employés pour trier des déchets avec une certaine cadence.
Récupère des déchets du dépôt, sinon de la pile, et génère des palettes ou redirige des déchets vers la décharge.

### Décharge

Stockage des déchets entrants.
**Dépôt**: Stocke les déchets pour la durée maximale d'une journée. Si la pile est vide, traiter la pile.
**Pile**: Déchets non triés et susceptibles d'être triés.
**Palettes**: Les déchets triés sont transformés en palette qui sont ensuite récupérées à intervalles réguliers.
**Décharge**: Déchets à incinérer dès que possible.
