# The North Face : regrouper et recommander des produits à partir de leurs descriptions

Projet réalisé dans le cadre de la certification Concepteur Développeur en Science des
Données (RNCP35288), bloc 3 : analyse prédictive de données structurées par l'IA.

## Le projet

Le service marketing veut améliorer les ventes du site en ligne. Il a deux idées : proposer
sur chaque page produit une section « vous aimerez aussi », et revoir les catégories du
catalogue à partir de ce que disent réellement les descriptions.

Nous avons construit trois choses, dans un seul notebook :

- **des groupes de produits** aux descriptions proches, obtenus par un clustering
  (regroupement automatique, sans catégories connues à l'avance) ;
- **un système de recommandation** qui, pour un produit donné, propose cinq produits
  proches ;
- **des thèmes** extraits automatiquement des descriptions, qui peuvent servir de base à de
  nouvelles catégories.

Le périmètre est un catalogue de 500 fiches produit. Il n'y a pas de données de ventes ni
de clients : tout repose sur le texte des descriptions.

## Les données

- **Catalogue produit** (`data/raw/sample-data.csv`), fourni par Jedha à partir du jeu
  Kaggle « product-item-data » : 500 lignes, deux colonnes, l'identifiant du produit et sa
  description. Les descriptions sont en anglais, au format HTML, longues de 400 à 3 500
  caractères. Il n'y a ni valeur manquante ni doublon.

Les fiches citent des tissus et programmes de la marque Patagonia : le jeu de données
Kaggle est étiqueté The North Face, mais le catalogue vient d'ailleurs. Cela ne change rien
au travail.

## Comment ça marche

1. **Exploration** : taille du jeu, valeurs manquantes, doublons, longueur des descriptions,
   lecture d'une fiche complète.
2. **Nettoyage** : retrait des balises HTML et de la ponctuation, passage en minuscules.
3. **Lemmatisation** avec spaCy : chaque mot est ramené à sa forme de dictionnaire, et les
   mots vides (articles, prépositions) sont retirés.
4. **Vectorisation TF-IDF** : chaque description devient un vecteur de nombres, un par mot
   du vocabulaire, qui mesure l'importance du mot dans la fiche pondérée par sa rareté dans
   le catalogue. Le vocabulaire est filtré pour écarter les mots vus dans une seule fiche et
   ceux présents dans plus de 90 % des fiches.
5. **Clustering DBSCAN** avec la distance cosinus : le rayon de voisinage est choisi après
   lecture de la distance de chaque produit à son plus proche voisin, puis ajusté par essais
   successifs. Un nuage de mots par cluster permet de nommer les groupes.
6. **Recommandation** : une fonction renvoie, pour un identifiant de produit, les cinq
   produits de son cluster dont la description est la plus proche. Une cellule de saisie
   permet de la tester.
7. **Extraction de thèmes** par TruncatedSVD (méthode LSA) : les descriptions sont
   projetées sur 14 axes, chaque produit reçoit son thème principal, et un nuage de mots par
   thème permet de les nommer.

Les choix qui ont compté :

- **Filtrer le vocabulaire plutôt que le garder entier** : 42 % des mots n'apparaissaient
  que dans une seule fiche et ne pouvaient rapprocher aucun produit d'un autre.
- **Régler le rayon de DBSCAN à partir d'une mesure** plutôt qu'à l'aveugle : la
  distribution des distances au plus proche voisin donne l'ordre de grandeur, les essais
  font le reste.
- **Construire les nuages de mots depuis les scores TF-IDF** plutôt qu'en comptant les
  mots : les phrases répétées dans toutes les fiches (programme de recyclage, section
  « Details ») masquaient sinon ce qui distingue chaque groupe.
- **Recommander aussi pour les produits sans cluster** : pour les produits que DBSCAN
  laisse isolés, la recherche s'étend au catalogue entier. L'énoncé ne le demande pas, mais
  un produit sans suggestion n'a pas de sens pour le site.

## Les résultats

| Étape | Résultat |
|---|---|
| Vocabulaire après filtrage | 2 063 mots, contre 3 560 avant |
| Clusters DBSCAN (eps = 0,65, min_samples = 5) | 17 clusters, 88 produits sur 500 non rattachés |
| Plus gros cluster | 155 produits (vêtements techniques d'extérieur) |
| Thèmes LSA | 14 axes, dont 13 thèmes lisibles |

Les clusters correspondent à des familles de catalogue : sous-vêtements techniques, sacs,
maillots de bain, t-shirts en coton bio, polaires, laine mérinos, chaussures, chapeaux,
brassières, chemises anti-UV. Un petit cluster isole des livres et posters photo, qui ne
sont pas des vêtements.

Les thèmes LSA retrouvent ces mêmes familles. À la différence des clusters, chaque produit
reçoit un thème, et un produit peut peser sur plusieurs thèmes à la fois, ce qui permet de
le classer dans plusieurs rayons du site.

Le système de recommandation propose, pour un pantalon de randonnée, les autres pantalons
de la même gamme ; pour un t-shirt en Capilene, les autres hauts dans ce tissu.

## Exécuter le projet

Arborescence :

- `02-The_North_Face_ecommerce.ipynb` : le notebook, seul livrable ;
- `data/raw/sample-data.csv` : le catalogue d'origine, jamais modifié ;
- `README.md`.

Prérequis : Python 3.13 avec pandas, scikit-learn, spaCy, plotly, matplotlib et wordcloud.
Le modèle de langue anglaise de spaCy s'installe séparément, par la commande
`python -m spacy download en_core_web_sm`.

Aucune clé ni fichier de configuration n'est nécessaire.

Lancement : ouvrir le notebook et exécuter les cellules dans l'ordre, depuis la racine du
projet. La cellule de saisie de la partie recommandation attend qu'on tape un identifiant
de produit, entre 1 et 500.

## Limites et suites possibles

- **Les fiches quasi identiques** d'un même modèle, par exemple les versions homme et femme,
  sont les premières recommandées. Dédoublonner ces variantes donnerait des suggestions
  plus variées.
- **Le plus gros cluster reste large** et mêle vestes et pantalons : DBSCAN peine quand la
  densité varie d'une famille à l'autre.
- **Certains thèmes LSA se recouvrent**, deux axes parlant de laine mérinos ou de protection
  solaire : un axe n'est pas une catégorie nette.
- **Aucune donnée de ventes** : les recommandations reposent sur la ressemblance des textes,
  pas sur ce que les clients achètent ensemble. Croiser les deux serait la suite naturelle.
