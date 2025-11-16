---
jupytext:
  encoding: '# -*- coding: utf-8 -*-'
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.17.3
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

```{code-cell} ipython3
import pandas as pd
import numpy as np
from matplotlib import pyplot as plt
import seaborn as sns
```

# Les vélos sur le pont de Fremont

````{admonition} lien vers la version originale

Voir la version originale de ce code - par Jake Vanderplas - sur Youtube

<https://www.youtube.com/watch?v=_ZEWDGpM-vM&list=PLYCpMb24GpOC704uO9svUrihl-HY1tTJJ>
````

+++

On part des données publiques qui décrivent le trafic des vélos [sur 
le pont de Fremont (à Portland - Oregon)](https://www.google.com/maps/place/Fremont+Bridge/@45.5166602,-122.7147124,12.31z/data=!4m5!3m4!1s0x0:0x9014fe26b76a82db!8m2!3d45.5379639!4d-122.6830729)

```{code-cell} ipython3
# le nom du fichier des données
local_file = "data/fremont.csv"
```

```{code-cell} ipython3
!head $local_file
```

une observation (une ligne) contient:
- une date par heure - qui s'apparente à une durée -
- le nombre de vélos ayant traversé le pont dans chaque sens (East-sidewalk ou West-sidewalk)  
  pendant cette durée
- le nombre total de vélos ayant traversé le pont  
pendant cette durée

+++

## chargement

```{code-cell} ipython3
# version naïve
data = pd.read_csv(local_file); data.shape
```

```{code-cell} ipython3
data.head()
```

## regardons la place que prennent les données

+++

la mémoire est mesurée en multiples de 1 byte (octet) : kilobytes (KB) ou megabytes (MB)  
*originellement* c'était des multiples de $1024$ ($2^{10}$) bytes mais désormais ce sont des multiples de $1000$  
see [https://en.wikipedia.org/wiki/1024_(number)]

```{code-cell} ipython3
data.info()
```

notez:

- les éléments sont encodés sur `64` bits  
  (les éléments de type `object` aussi: ce sont des adresses en mémoire)

- chaque colonne `pandas` occupe donc $data.shape[0] * 8$ octets  
(les éléments manquants sont référencés dans leur colonne - ils occupent les 8 octets)

```{code-cell} ipython3
# donc un calcul grossier de l'occupation mémoire en MB pour 4 colonnes:
4 * (data.shape[0] * 8) / 1000000
```

```{code-cell} ipython3
# qu'on retrouve calculé là
data.memory_usage(index=False)
```

Mais comme les dates sont des chaînes de caractères, elles occupent:
- toujours les $8$ octets (indiquant l'adresse de la chaîne de caractère)
- plus *de la mémoire* pour la chaîne de caractères représentant les dates en mémoire ("08/01/2022 12:00:00 AM")

```{code-cell} ipython3
# regardons plus précisemment la mémoire occupée par chaque colonne
data.memory_usage(index=False, deep=True)
```

*The memory_usage parameter allows deep introspection mode, specially useful for big DataFrames and fine-tune memory optimization*

```{code-cell} ipython3
data.info(memory_usage='deep')
```

## suppression des doublons

+++

c'est un peu le bazar ce dataset...  
les données sont présentes en deux exemplaires !

```{code-cell} ipython3
# par exemple
data.loc[data['Date'] == '01/01/2014 12:00:00 AM']
```

```{code-cell} ipython3
# on calcule le nombre de lignes dupliquées
data.duplicated().sum()
```

du coup on nettoie

```{code-cell} ipython3
data.drop_duplicates(inplace=True);  data.shape
```

c'est quand même mieux...

```{code-cell} ipython3
data.memory_usage(index=False, deep=True) / 1000000 # en MegaByte
```

```{code-cell} ipython3
data.info(memory_usage='deep')
```

## parser les dates

```{code-cell} ipython3
# intéressant aussi, pour voir notamment les points manquants
data.info()
```

```{code-cell} ipython3
# ou tout simplement
data.dtypes
```

bref le point ici c'est que les dates sont **des chaines et pas des dates**

```{code-cell} ipython3
# la version lente

# avec cette forme, on demanderait à read_csv:
# de mettre la date comme index,
# et de parser les dates
# mais on ne va pas le faire car
# 1. c'est peu sûr
# 2. c'est trop lent

# data = pd.read_csv(local_file, index_col='Date', parse_dates=True); data.head()
```

```{code-cell} ipython3
# une meilleure idée, pour améliorer ces deux points d'un coup
# est de fournir nous-même le format des dates (*)

data['Date'] = pd.to_datetime(data['Date'], format="%m/%d/%Y %I:%M:%S %p")
data.set_index('Date', inplace=True)
```

(*) voir 
https://docs.python.org/3/library/datetime.html#strftime-and-strptime-behavior

```{code-cell} ipython3
data.info(memory_usage='deep')
```

maintenant les dates sont codées sur 64 bits

+++

## renommons les colonnes

```{code-cell} ipython3
# les noms de colonne ne sont pas pratiques du tout
data.columns = ['Total', 'West', 'East']
```

## données manquantes et extension types

+++

de manière totalement optionnelle, mais on remarque que les nombres ont été convertis en flottants

et ça c'est parce qu'il y a eu quelques interruptions de service, apparemment, avec le système de récolte de l'information

```{code-cell} ipython3
data[data['Total'].isna()]
```

```{code-cell} ipython3
# ou encore si on préfère

data[data.isna().any(axis=1)].shape
```

on pourrait nettoyer, mais ici on va choisir d'ignorer ces données manquantes; à la place on va les remettre sous la forme d'entiers

```{code-cell} ipython3
# ceci va nous permettre d'avoir des colonnes d'entiers - avec des nan

data = data.convert_dtypes(convert_integer=True)
data.head()
```

```{code-cell} ipython3
# ça ressemble à ceci

data.dtypes
```

```{code-cell} ipython3
# on a toujours les n/a, mais ce n'est pas grave
data[data.isna().any(axis=1)].shape
```

## à quoi ça ressemble

```{code-cell} ipython3
# on change les valeurs des paramètres de la taille de la figure
sns.set(rc={'figure.figsize': (12, 4)})
plt.rcParams["figure.figsize"] = (12, 4)
```

on plotte toutes les données

```{code-cell} ipython3
# un premier jet, pas terrible du tout

data[['East', 'West']].plot();
```

## ajustement

+++

ce serait plus lisible avec seulement un point par semaine ...

+++

on va aggréger (rééchantillonner) nos données  
on va passer d'une valeur par heure à une valeur par semaine  
vous combinez comme vous voulez : sum, mean, min, max...

```{code-cell} ipython3
# c'est plus lisible avec seulement un point par semaine
# on pourrait faire la moyenne aussi bien sûr,
# ça donnerait le même dessin mais avec les Y divisés par 7

data.resample('W').sum().plot(); 
```

le point c'est qu'on a quelques années de plus que sur la vidéo...

pour être en phase (pouvoir vérifier nos résultats par rapport à ceux de la vidéo), on va s'arrêter à la fin de 2017

(un détail à noter aussi, les données de la vidéo ne contenaient pas la colonne 'total'...)

```{code-cell} ipython3
# c'est facile de couper, la date correpond à l'index de la df
# et grâce au type 'datetime' on peut simplement faire une comparaison

data = data[data.index.year <= 2017]
```

````{admonition} quiz

ici on s'en sort bien car on coupe au début d'une année  
mais comment ferait-on pour couper au 12 Février à 14h32:30 ?

````

```{code-cell} ipython3
# on se sert du même format
feb12 = pd.to_datetime("02/12/2017 2:32:30 PM", format="%m/%d/%Y %I:%M:%S %p")
data_ = data[data.index <= feb12]

data_.tail(3)
```

```{code-cell} ipython3
data.resample('W').sum().plot();
```

## `resample()` ?

+++

décortiquons un peu cette histoire de `resample()`

```{code-cell} ipython3
# la forme du resample() est de:
data.resample('1W').sum().shape
```

```{code-cell} ipython3
# on vérifie que la version resamplée a bien 
# 7 * 24 = 168 fois moins d'entrées que la version brute
# puisqu'on a une mesure par heure et qu'on ré-échatillonne sur une semaine

(full, _), (resampled, _) = data.shape, data.resample('1W').sum().shape

full / resampled , 7 * 24
```

faisons le avec `resample` et à-la-main pour le premier jour

```{code-cell} ipython3
data.resample('D').sum().head(1)
```

```{code-cell} ipython3
# un peu trop compliqué juste pour voir les dates
first_hour = pd.to_datetime("10/03/2012 12:00:00 AM", format="%m/%d/%Y %I:%M:%S %p") # minuit
last_hour = pd.to_datetime("10/03/2012 11:00:00 PM", format="%m/%d/%Y %I:%M:%S %p") # 23h

data[(first_hour <= data.index) & (data.index <= last_hour)].sum(axis=0)
```

```{code-cell} ipython3
day = pd.to_datetime("10/03/2012") # le jour
day_after = pd.to_datetime("10/04/2012") # le lendemain

data[(day <= data.index) & (data.index < day_after)].sum(axis=0)
```

Faison-le à-la-main pour une semaine

Il faut choisir les bornes des semaine   
`resampling` est calculé par défaut de `Monday` à `Sunday`

```{code-cell} ipython3
# on a ces deux semaines
data.resample('1W').sum().head(2)
```

```{code-cell} ipython3
# la première semaine est alors incomplète : ce sont les jours jusqu'au premier dimanche
day_first = pd.to_datetime("10/03/2012 12:00:00 AM") # le premier jour
day_last = pd.to_datetime("10/07/2012 11:00:00 PM") # le dernier jour

data[(day_first <= data.index) & (data.index <= day_last)].sum(axis=0)
```

```{code-cell} ipython3
# correspondant à cela:
data.iloc[0:5*24].sum()
```

```{code-cell} ipython3
# la deuxième semaine est complète du lundi au dimanche suivant
day_first = pd.to_datetime("10/08/2012 12:00:00 AM") # le premier jour
day_last = pd.to_datetime("10/14/2012 11:00:00 PM") # le dernier jour

data[(day_first <= data.index) & (data.index <= day_last)].sum(axis=0)
```

et c'est tout à fait normal, il faut simplement toujours bien comprendre ce que font les fonctions

+++

## les rolling windows

+++

reprenons

```{code-cell} ipython3
data.resample('1W').sum().plot();
```

on va faire la somme sur une fenêtre tournante d'un an

```{code-cell} ipython3
# mais : méfiez-vous de l'échelle des Y: elle ne part pas toujours à zéro !!

data.resample('1D').sum().rolling(365).sum().plot();
```

```{code-cell} ipython3
# on fait en sorte que le bas de l'échelle des Y soit bien 0
# pour eviter l'effet de loupe

ax = data.resample('1D').sum().rolling(365).sum().plot()
ax.set_ylim(0, None);
```

comment se fait le rolling ?

```{code-cell} ipython3
# nos données commencent un 3 octobre 2012
#   - la première valeur du rolling sera celle du 2 octobre 2013
#   - la seconde valeur du rolling sera celle du 3 octobre 2013
#   - ...
data.resample('1D').sum().rolling(365).sum().dropna().head()
```

## Tendance des profils journaliers

+++

dans nos objets `datetime`, avec :
- l'accesseur `date` on accède à la date du jour
- l'accesseur `time` on accède aux heurse de la journée de $0$ à $23$

+++

on peu utiliser ces informations pour calculer les tendances des profils journaliers

```{code-cell} ipython3
data.index.time[0:3]
```

on va grouper toute la dataframe par les heures de la journée et en faire la moyenne

```{code-cell} ipython3
# regardons la tendance des profils journaliers en moyenne

data.groupby(data.index.time).mean().plot();
```

on remarque 2 pics un vers 8h et vers 17h

```{code-cell} ipython3
# on prend la colonnes 'Total' et on affiche ses 2 premiers pics
# c'est bien 8h et 17h les aller-retours du travail

data.groupby(data.index.time).mean()['Total'].nlargest(2)
```

pour y voir un peu mieux on va afficher les jours individuellement les uns des autres

- on veut dessiner autant de courbes que de jours

- chaque courbe a en X l'heure de la journée et en Y le nombre (total) de passages

pour ça on calcule une pivot table:
- en index les heures
- en colonne les jours
- en valeur les totaux

```{code-cell} ipython3
pivoted = data.pivot_table(index=data.index.time,
                           columns=data.index.date,
                           values='Total')
pivoted.head(2)                    
```

```{code-cell} ipython3
# ça pourrait se faire avec groupby/unstack
data.groupby(by=[data.index.time, data.index.date])['Total'].sum().unstack(level=-1).head(2)
```

```{code-cell} ipython3
# du coup on n'a plus qu'à dessiner
# on va mettre une transarence pour mettre en valeurs les endroits ou on a beaucoup de données
# avec alpha=0.01

pivoted.plot(legend=False, alpha=0.01);
```

c'est vraiment très informatif: on a deux familles de jours:
- les jours qui ont deux pics marqués à 8h et à 17h
- les jour qui ont un plateau avec un tout petit pic en milieu de journée

```{code-cell} ipython3
# on sauve la nouvelle dataframe pivoted avec en lignes les jours et en colonnes leurs 24 heures

df_pivoted = pivoted.T.copy()
df_pivoted.reset_index(names=['days'], inplace=True)

df_pivoted.to_csv('data/fremont_days_versus_24_hours.csv', index=False)
```

## classification

+++

Peut-on retrouver ces jours ?

i.e.

Peut-on classifier les jours en ces deux familles, qu'on voit très distinctement sur la figure ?

+++

### analyse en composante principale

+++

On va choisir pour étudier cette classification d'utiliser une analyse en composante principale ACP, on va:
- prendre $n$ colonnes de notre dataframe, ici les 24 heures
- calculer la matrice de covariance de ces colonnes
- calculer les vecteurs propres et les valeurs propres de la matrice obtenue

Après ce calcul, on a:
- $n$ valeurs propres qui nous donnent l'importance des "nouveaux" axes  
   (plus grandes sont les valeurs propres, plus importants sont les nouveaux axes)
- les $n$ nouveaux axes (vecteurs propres) qui sont des combinaisons linéaires des $n$ colonnes initiales

Les vecteurs propres forment la matrice de *transformation*

Cette matrice nous permet de *passer* nos données de leur espace de départ, à l'espace (orthogonal) de ces vecteurs propres.

Ce nouvel espace nous donne un nouvelle manière de voir nos données de départ - certains informations ont été supprimées (par exemple une rotation)

+++

La covariance de deux features mesure leur dépendance i.e. leur tendance à *varier ensemble*:
- dans le même sens quand $cov(X_i, X_j) > 0$
- dans le sens inverse quand $cov(X_i, X_j) < 0$
- elles sont indépendantes quand $cov(X_i, X_j) = 0$

+++

*"étant donné un ensemble de données de $m$ observations (lignes), chacune avec $n$ caractéristiques (colonnes) - i.e. espace en n-dimensions  
on cherche un sous-espace de plus petite dimension (ou de la même dimension) sur lequel on peut projeter les données orthogonalement  
tout en préservant la variance totale de l'ensemble de données dans toutes les dimensions"*

+++

Exemple d'une PCA sur un poisson:
- on dispose de données avec 2 colonnes dont le plot 2D donne ce nuage de points
- on a calculé les vecteurs propres et les valeurs propres de la matrice de covariance de ces deux colonnes
```{image} media/fish.png
:width: 300px
```
et bien:
- les deux segments rouges sont les axes des vecteurs propres  
   et leur longueur représente leurs valeurs propres respectives   
(un axe est plus important que l'autre - attention il faut les voir comme orthogonaux !)
- on a obtenu une matrice de transformation (ici une matrice de rotation)
  
... tourné ou pas, le poisson reste un poisson  
- la rotation (ici) n'ajoute pas d'information mais du bruit dans les données  
- on peut donc enlever la rotation des données du poisson pour les décorreler
- on aura un poisson horizontal

+++

Plusieurs intérêts de cette méthode
- réduire la dimension de votre problème
- rechercher des "meilleurs" attributs dans vos données (ceux qui décrivent le "mieux" votre problème)  
- décorreler vos données - enlever, de vos données, des informations inutiles (rotation)
- débruiter notre ensemble de données  
 en en enlevant les colonnes qui n'apportent pas assez d'information (au sens de la co-variance)

```{code-cell} ipython3
# on lit la nouvelle dataframe avec en lignes les jours et en colonnes les nombre de passages par heures - 24h
df_hours = pd.read_csv('data/fremont_days_versus_24_hours.csv')
```

```{code-cell} ipython3
df_hours.head(2)
```

```{code-cell} ipython3
# on enlève la colonne des jours
days = df_hours.pop('days')
```

```{code-cell} ipython3
# ces calculs étant sensibles aux valeurs manquantes
# on décide ici dans cet exemple de les mettre à zéro (ce n'est pas toujours ce qu'il faut faire)

pca_input = df_hours.fillna(0)
pca_input.head()
```

```{code-cell} ipython3
#pca_input['labels'] = df['labels']
```

```{code-cell} ipython3
# on trace les colonnes en fonction les unes des autres
# - on voit des séparations en 2 groupes mais elles ne sont pas très évidentes
# sns.pairplot(pca_input);

#sns.pairplot(pca_input, hue='labels'); # essayer avec les labels calculés ci-dessous
```

```{code-cell} ipython3
# on veut avoir un espace ortho-normé donc on centre/reduit nos données

# pour centrer-réduire on calcule X_cr = (X - mean(X)) / std(X)

pca_input_norm = (pca_input - pca_input.mean()) / pca_input.std() # centré/réduire

# en fait dès que vous travaillez avec des "distances" il faut normaliser vos données
```

```{code-cell} ipython3
# clairement on n'a plus besoin de rester en pandas
# toutes les données ont le même type et numpy préfère !

pca_input_norm = pca_input_norm.to_numpy()
```

```{code-cell} ipython3
# on veut calculer la covariance des colonnes des heures et pas des colonnes des jours
# on transpose notre dataframe

pca_input_norm_T = pca_input_norm.T
```

```{code-cell} ipython3
# on calcule la matrice de covariance
pca_cov = np.cov(pca_input_norm_T)
```

```{code-cell} ipython3
# on utilise la librairie d'algèbre linéaire de numpy
#    pour faire la décomposition en valeur et vecteurs propres
lambdas, A = np.linalg.eig(pca_cov)

plt.scatter(range(24), lambdas)
plt.title(f'les valeurs propres');
```

on remarque que peu de valeurs propres (ici 2 ou 3) contiennent presque toute l'information

```{code-cell} ipython3
# on sélectionne les deux premiers eigenvectors
P = A[:,0:2]
P
```

```{code-cell} ipython3
# on projette les données initiales (celles qui ont servi à calculer l'ACP) en dimension 2
pca_values = pca_input_norm.dot(P)
pca_values
```

```{code-cell} ipython3
pca_values.shape
```

```{code-cell} ipython3
# plot2d (très moche) des 2 colonnes avec le jour en texte

plt.figure(figsize=(30, 20))

text = days

plt.scatter(pca_values[:, 0], pca_values[:, 1])

for label, xi, yi in zip(text, pca_values[:, 0], pca_values[:, 1]):
    plt.annotate(
        label,
        xy=(xi, yi), xytext=(-3, 3),
        textcoords='offset points', ha='right', va='bottom')

plt.xlabel('PC1 (49%)')
plt.ylabel('PC2 (32%)')
plt.title('PCA of fremont dataset')
plt.savefig('fremont_plot2D.png')
plt.show()
```

```{code-cell} ipython3
# on peut créer une dataframe
# on lui met en index les jours
# et en colonne les 2 vecteurs propres
df_pca_values = pd.DataFrame(
    pca_values, 
    index=days,
    columns=[f'column {i}' for i in range(2)],
)

df_pca_values.head()
```

```{code-cell} ipython3
# on plot2d les deux premières colonnes
df_pca_values.plot.scatter(x='column 0', y='column 1', marker='.', color='green');
```

```{code-cell} ipython3
# pour trouver ces deux clusters, comme Jake, on utilise une GaussianMixture
# sur les deux premières colonnes
from sklearn.mixture import GaussianMixture
gmm = GaussianMixture(2)

# c'est ici que tout se passe
labels = gmm.fit(df_pca_values).predict(df_pca_values)
```

```{code-cell} ipython3
# la sortie est une association jour -> type
labels.shape, labels
```

```{code-cell} ipython3
# cette prédiction est bien en phase avec les deux clusters de tout à l'heure

df_pca_values.plot.scatter(x='column 0', y='column 1', c = labels, marker='.', cmap='viridis');
```

## avec les librairies dédiée
c'est ce qu'il faut utiliser

```{code-cell} ipython3
# ! pip install scikit-learn
```

```{code-cell} ipython3
# le calcul des ACP
from sklearn.decomposition import PCA
from sklearn.preprocessing import StandardScaler
```

```{code-cell} ipython3
# on lit la nouvelle dataframe avec en lignes les jours et en colonnes le nombre de passages (moyen) par heures - 24h
df_hours_sk = pd.read_csv('data/fremont_days_versus_24_hours.csv')
days_sk = df_hours_sk.pop('days')

# on enlève les nan
df_hours_sk.fillna(0, inplace=True)
```

```{code-cell} ipython3
# on normalise
pca_input_norm_sk = StandardScaler().fit_transform(df_hours_sk)
```

```{code-cell} ipython3
# on utilise le PCA de scikit-learn
pca_output_sk = PCA(n_components=2).fit_transform(pca_input_norm_sk)
```

```{code-cell} ipython3
# on obtient un tableau numpy avec deux colonnes seulement
# car on a demandé les deux premières composantes principales

type(pca_output_sk), pca_output_sk.shape
```

```{code-cell} ipython3
# on voit effectivement que cet ACP semble bien séparer deux clusters

plt.scatter(pca_output_sk[:, 0], pca_output_sk[:, 1], marker='.', color='green');
```

```{code-cell} ipython3
# pour les trouver ces deux clusters, Jake utilise une GaussianMixture

from sklearn.mixture import GaussianMixture
gmm_sk = GaussianMixture(2)

# c'est ici que tout se passe
labels_sk = gmm_sk.fit(pca_output_sk).predict(pca_output_sk)


# la sortie est une association jour -> type
labels_sk.shape, labels_sk
```

```{code-cell} ipython3
# cette prédiction est bien en phase avec les deux clusters de tout à l'heure

plt.scatter(pca_output_sk[:, 0], pca_output_sk[:, 1], c=labels_sk, marker='.', cmap='rainbow')
plt.colorbar();
```

### première famille : `label==1`

```{code-cell} ipython3
pivoted = pd.read_csv('data/fremont_days_versus_24_hours.csv').set_index('days').T

# on enlève les nan
pivoted.fillna(0, inplace=True)
```

```{code-cell} ipython3
pivoted.head(2)
```

```{code-cell} ipython3
# pour vérifier notre classification on peut redessiner
# les jours classés label==1

# ça correspond donc aux jours de la semaine (à moins que ce soit l'inverse..)

pivoted.loc[:, labels_sk==1].plot(legend=False, alpha=0.01);
```

### deuxième famille `label==0`

```{code-cell} ipython3
# et les jours classés label==0

pivoted.loc[:, labels_sk==0].plot(legend=False, alpha=0.05);
```

### les deux clusters avec le jour de la semaine

+++

essayons de vérifier que les deux clusters correspondent bien à l'intuition de départ

pour ça on redessine les deux clusters avec une couleur qui indique le jour de la semaine

```{code-cell} ipython3
# notre index horizontal n'est pas de type DatetimeIndex
pivoted.columns 
```

```{code-cell} ipython3
# on crée un index qui contient toutes nos dates et de type DatetimeIndex
dates = pd.DatetimeIndex(pivoted.columns)

pivoted.columns = dates
```

```{code-cell} ipython3
# on calcule un index sur les jours
# mais avec comme valeur 0 pour le lundi, ... et 6 pour le dimanche

dayofweek = pd.DatetimeIndex(pivoted.columns).dayofweek

dayofweek.shape, dayofweek
```

```{code-cell} ipython3
# qu'on va utiliser pour mettre les jours en couleur
# les jours de weekend sont en orange et rouge

plt.scatter(pca_output_sk[:, 0], pca_output_sk[:, 1], c=dayofweek, cmap='rainbow')
# pour la légende
plt.colorbar();
```

### les moutons noirs

```{code-cell} ipython3
# on remarque dans le cluster rouge-orange
# des jours d'une couleur qui jure

# pour comprendre à quoi ils correspondent 

# attention au numéro du label attribué par l'algo

odd_index = (labels_sk == 0) & (dayofweek < 5)
odd_index.shape, odd_index
```

```{code-cell} ipython3
# afficher les 48 jours qui sont dans cette catégorie

# comme on peut s'y attendre
# on y retrouve les jours fériés (4 juillet, Noel, ...)

odd_dates = dates[odd_index]
odd_dates, len(odd_dates)
```

```{code-cell} ipython3
# pour rafficher seulement ces jours-là

pivoted[odd_dates].plot(legend=False, alpha=0.3);
```

## ajout des labels à la dataframe

```{code-cell} ipython3
df_dates = pd.DataFrame(dates)
df_dates['labels'] = labels_sk
df_dates
```

```{code-cell} ipython3
data['days'] = pd.to_datetime(data.index.date)
```

```{code-cell} ipython3
df = pd.merge(left=data, left_on='days', right=df_dates, right_on='days')
```

### pairplot avec labels

```{code-cell} ipython3
pca_input['labels'] = df['labels']
```

```{code-cell} ipython3
# on trace les colonnes en fonction les unes des autres
# - on voit des séparations en 2 groupes mais elles ne sont pas très évidentes
# sns.pairplot(pca_input);

sns.pairplot(pca_input, hue='labels'); # essayer avec les labels calculés ci-dessous
```

```{code-cell} ipython3

```

```{code-cell} ipython3

```
