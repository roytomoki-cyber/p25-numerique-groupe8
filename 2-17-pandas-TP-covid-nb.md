---
jupytext:
  text_representation:
    extension: .md
    format_name: myst
    format_version: 0.13
    jupytext_version: 1.17.3
  encoding: '# -*- coding: utf-8 -*-'
kernelspec:
  display_name: Python 3 (ipykernel)
  language: python
  name: python3
---

# les données covid

+++

avant de commencer ce TP:
- mettez le fichier `2-17-pandas-TP-covid-nb.md` dans votre répertoire local `p25-numerique-groupe8`  
- mettez le fichier 'covid-frozen.csv` dans le sous-répertoire `data`
- mettez le fichier `covid-example.svg` dans le sous-répertoire `media`

```{code-cell} ipython3
import matplotlib.pyplot as plt
import pandas as pd
import numpy as np
```

ce sujet vise à acquérir et mettre en forme les données du COVID pour pouvoir produire facilement des diagrammes comme celui-ci  
comme vous le voyez on a choisi:

- une liste de pays,
- une liste de mesures - ici: *deaths* & *confirmed*,
- et une plage de temps spécifique

+++

![](media/covid-example.svg)

+++ {"tags": ["framed_cell"]}

## les données de Johns Hopkins

````{admonition} →
les données sur le corona virus étaient publiées par le département *Center for Systems Science and Engineering* (CSSE), de l'Université Johns Hopkins

* sur le dépôt github <https://github.com/CSSEGISandData/COVID-19> désormais archivé (figé)
* le format était brut, détaillé et touffu - un peu trop compliqué pour l'utiliser ici
````

````{admonition} →
un dépôt de *seconde main* <https://github.com/pomber/covid19>

* consolidait les données du CSSE en une unique source
* mis à jour quotidiennement
* le fichier `timeseries.json` est en format JSON (JavaScript Object Notation)
````

```{code-cell} ipython3
# voici le repo qui va vraiment vous servir, avec le fichier json

json_url = "https://pomber.github.io/covid19/timeseries.json"
```

```{code-cell} ipython3
# le repo qui va vraiment vous servir, avec le fichier json

json_url = "https://pomber.github.io/covid19/timeseries.json"
```

+++ {"tags": ["framed_cell"]}

## le format `json` ?

``````{admonition} →
vous connaissez le format `csv`, `json` est un format de données **plus structuré** qui exprime différents types
* des types de base: nombre, `str` (uniquement avec `""`), `false`,  `true`, `null`..
* deux types structurés: les listes et les objets exprimés comme des `dict` en `Python`)  
 un `dict` permet de décrire des objets `{attribut1: valeur1, attribut2: valeur2, ...}`

par exemple, la liste des animaux avec leur vitesse et leur longévité pourrait être représentée en `json` par le texte

```python
[
    {"name": "snail",    "speed": 0.1,  "lifespan": 2.0},
    {"name": "pig",      "speed": 17.5, "lifespan": 8.0},
    {"name": "elephant", "speed": 40.0, "lifespan": 70.0},
    {"name": "rabbit",   "speed": 48.0, "lifespan": 1.5},
    {"name": "giraffe",  "speed": 52.0, "lifespan": 25.0},
    {"name": "coyote",   "speed": 69.0, "lifespan": 12.0},
    {"name": "horse",    "speed": 88.0, "lifespan": 28.0}
 ]
```

ou encore sous une autre forme, toujours en `json`

```python
{"speed":
     {"snail":0.1,
      "pig":17.5,
      "elephant":40.0,
      "rabbit":48.0,
      "giraffe":52.0,
      "coyote":69.0,
      "horse":88.0},
  "lifespan":
      {"snail":2.0,
       "pig":8.0,
       "elephant":70.0,
       "rabbit":1.5,
       "giraffe":25.0,
       "coyote":12.0,
       "horse":28.0}
}
```

``````

+++ {"tags": ["framed_cell"]}

## format `json` pour le covid

````{admonition} →
revenons au covid  
le fichier <https://pomber.github.io/covid19/timeseries.json> contient un objet `dict` dont

* les clés sont les pays du monde
* chaque valeur est une liste de `dict`  
  chacun décrivant **une** mesure de covid avec les 4 clés: `date`, `confirmed`, `deaths` et `recovered`

```python
{
  "Afghanistan": [
    {
      "date": "2020-1-22",
      "confirmed": 0,
      "deaths": 0,
      "recovered": 0
    },
    {
      "date": "2020-1-23",
      "confirmed": 0,
      "deaths": 0,
      "recovered": 0
    },
    {
      "date": "2020-1-24",
      "confirmed": 0,
      "deaths": 0,
      "recovered": 0
    }, ...
```
````

+++

## acquisition des données `json`

+++

### avec `pd.read_json`

````{admonition} →
on ne va pas faire comme ça ici, mais sachez que c'est la méthode la plus rapide (à écrire):

```python
data = pd.read_json(json_url)
```

par contre ça peut être franchement long, surtout si votre **connexion réseau** n'est pas au top  
c'est pourquoi on va voir aussi une autre méthode - que vous pouvez sauter si vous êtes pressés de voir le traitement des données *per se*
````

+++ {"tags": ["framed_cell"]}

### caching avec `requests`

en utilisant la librairie `requests` on peut implémenter un *caching* pour nos données

````{admonition} caching ?
dans le cas présent le terme *caching* suggère que l'on sauverait le fichier sur disque après l'avoir *download* depuis Internet; de cette façon on n'attend qu'une seule fois la durée du *download*  
par contre, c'est bien d'être malin et de, par exemple, considérer que les fichiers qui ont plus de 1 jour ne sont plus valides et qu'il faut retourner les chercher; mais bon, *let's keep it simple*, on ne va pas aller jusque là...
````

+++ {"tags": ["framed_cell"]}

`````{admonition} → requests.get pas à pas
:class: dropdown

````{div}
le module `requests` permet de *récupérer* des fichiers sur Internet  
utiliser cette approche permet de toujours avoir des **données récentes**  
**mais** demande une bonne connexion à Internet  
sinon allez à la slide (l'encadré) suivante qui utilise des données figées

`requests` n'est pas dans la librairie standard  
il faut donc l'installer comme d'habitude avec `pip`  
(on fait comment déjà ?)

une fois que c'est fait on peut l'importer

```python
import requests
```

avec la fonction `requests.get` on envoie la *requête* d'une URL  
et on reçoit une réponse  
**attention** la requête suivante (`requests.get`) demande une bonne connexion  
ou beaucoup de patience...

```python
json_url = "https://pomber.github.io/covid19/timeseries.json"
response = requests.get(json_url)
```

on peut vérifier que l'échange s'est bien passé

```python
response.ok
-> True
```

la méthode `json()` sur l'objet `Response` décode le format JSON  
et renvoie les données prêtes pour des traitements en Python  

```python
by_country = response.json()
```

on voit bien une structure Python de **`dict`** et de `list`  
correspondant au contenu du fichier `json` vu ci-dessus

```python
by_country
-> {'Afghanistan': [
      {'date': '2020-1-22', 'confirmed': 0, 'deaths': 0,'recovered': 0},
      {'date': '2020-1-23', 'confirmed': 0, 'deaths': 0, 'recovered': 0},
      {'date': '2020-1-24', 'confirmed': 0, 'deaths': 0, 'recovered': 0},
    ...
```

si votre connexion ne vous permet pas la requête  
voir la prochaine cellule de cours
````
`````

```{code-cell} ipython3
# pensez à bien installer le module requests

import requests

json_url = "https://pomber.github.io/covid19/timeseries.json"
by_country = None
```

```{code-cell} ipython3
# mettez cette variable à True si vous avez une bonne connexion

#good_connection = False
good_connection = True
```

```{code-cell} ipython3
# le code UNIQUEMENT SI VOUS AVEZ UNE BONNE CONNEXION INTERNET

if good_connection:
    response = requests.get(json_url)
    print(response.ok)

    by_country = response.json()

    print(type(by_country))

else:
    print('pas bonne connexion - pas grave...')
```

+++ {"tags": ["framed_cell"]}

### chargement avec la lib `json`

````{admonition} →
si l'accès Internet n'est pas possible, sachez que nous exposons une copie des données  
faite il y a quelque temps, dans le fichier `data/covid-frozen.json`

le module `json` de la librairie standard permet de *lire* des fichiers en format JSON;  
on l'importe comme d'habitude avec

```python
import json
```

après avoir ouvert un fichier en lecture, 
la fonction `json.load` lit le contenu dans un objet Python

```python
json_file = 'data/covid-frozen.json'

with open(json_file) as f:
    by_country = json.load(f)
```

et on obtient une structure Python de `dict` et de `list`

```python
by_country
-> {'Afghanistan': [
      {'date': '2020-1-22', 'confirmed': 0, 'deaths': 0, 'recovered': 0},
      {'date': '2020-1-23', 'confirmed': 0, 'deaths': 0, 'recovered': 0},
      {'date': '2020-1-24', 'confirmed': 0, 'deaths': 0, 'recovered': 0},
    ...
```
````

```{code-cell} ipython3
# le code

if by_country is not None:
    print('on utilise les données déjà chargées')
else:
    import json
    json_file = 'data/covid-frozen.json'

    with open(json_file) as f:
        by_country = json.load(f)

    print(type(by_country))
```

```{code-cell} ipython3
# regardons un peu la première clé de ce dictionnaire

# les 4 premières clés
list(by_country.keys())[:4]
```

## une dataframe globale

qui consolide les données covid *monde*

+++ {"tags": ["framed_cell"]}

### exercice (version avancé)

````{admonition} →
il s'agit de construire une **unique** dataframe contenant toutes les données covid *monde*  
à partir de l'objet python `by_country`

vous devez obtenir quelque chose comme cela

```python
          date  confirmed  deaths  recovered      country
0    2020-1-22          0       0          0  Afghanistan
1    2020-1-23          0       0          0  Afghanistan
2    2020-1-24          0       0          0  Afghanistan
3    2020-1-25          0       0          0  Afghanistan
4    2020-1-26          0       0          0  Afghanistan
..         ...        ...     ...        ...          ...
???  2021-8-29     124437    4401          0     Zimbabwe
???  2021-8-30     124581    4416          0     Zimbabwe
???  2021-8-31     124773    4419          0     Zimbabwe
???   2021-9-1     124960    4438          0     Zimbabwe
???   2021-9-2     125118    4449          0     Zimbabwe

[115050 rows x 5 columns]
```

**attention**

* les `???` peuvent être différents suivant ce que vous faites
* `115050` dépend de la date à laquelle le fichier a été récupéré  
  (et de combien de données étaient alors disponibles)

**indications**

* les élèves avancés peuvent travailler sans indications supplémentaires
* pour les autres élèves, on vous propose une méthode pas-à-pas
````

```{code-cell} ipython3
# votre code
# rangez votre résultat dans la variable global_df

# global_df = ...
```

+++ {"tags": ["framed_cell"]}

### exercice (méthode pas-à-pas)

````{admonition} →
**exercice (méthode pas-à-pas)** de construction de la dataframe globale

nous allons commencer par créer les dataframes de 2 pays `'France'` et `'Italy'`  
puis les concaténer en une unique dataframe globale  
et ensuite généraliser à tous les pays

**rappel** l'objet Python `by_country` est un `dict` dont:

* les clés `keys()` sont les noms des pays
* les valeurs `values()` sont des séries temporelles (`list`) d'observations sur le covid  
* chaque observation est un objet exprimé sous la forme d'un `dict`  
  avec 4 mesures indiquées par les attributs `'date'`, `'confirmed'`, `'deaths'` et `'recovered'`  
  en `2021-8-31` au `Zimbabwe` on a `124773` cas confirmés, `4419` morts et `0` guéris
````

+++ {"tags": ["framed_cell"]}

````{admonition} exo

1. prenez la clé `'France'`  
construisez la dataframe à partie de la valeur de cette clé  
(la liste des enregistrements temporels de cas de covid)
````

```{code-cell} ipython3
# votre code
dict_fr = by_country['France']
df = pd.DataFrame(dict_fr)
df
```

+++ {"tags": ["framed_cell"]}

````{admonition} exo

2. quelles sont les colonnes de cette dataframe ?  
combien y-a-t-il d'entrées (de mesures différentes)
````

```{code-cell} ipython3
# votre code
df.value_counts()
#1143 mesures différentes
```

+++ {"tags": ["framed_cell"]}

````{admonition} exo

3. vous remarquez que cette dataframe ne contient plus l'information sur le pays  
ajoutez à cette dataframe une colonne de nom `'country'` contenant `'France'` à chaque ligne
````

```{code-cell} ipython3
# votre code
df['country'] = 'France'
df
```

```{code-cell} ipython3

```

+++ {"tags": ["framed_cell"]}

````{admonition} exo

4. faites de même avec la clé `'Italy'`  
et utilisez la fonction `pandas.concat` pour concaténer les deux dataframes
````

```{code-cell} ipython3
# votre code
dict_it = by_country['Italy']
df_it = pd.DataFrame(dict_it)
df_it['country'] = 'Italy'
df_it
```

```{code-cell} ipython3
pd.concat([df, df_it])
```

+++ {"tags": ["framed_cell"]}

````{admonition} exo
5. généralisez et construisez une dataframe avec tous les pays  
vous aurez sans doute besoin d'utiliser un `for` python
````

```{code-cell} ipython3
# votre code
cles = list(by_country.keys())
df_list = []
for i in range(len(by_country)) :
    df_count = pd.DataFrame(by_country[cles[i]])
    df_count['country'] = cles[i]
    df_list.append(df_count)
global_df = pd.concat(df_list)
global_df
    
```

## index de la dataframe globale

+++

### les index ne sont pas forcément uniques

si vous avez appelé `pd.concat()` sans paramètre particulier, vous pouvez sans doute observer ceci:

```{code-cell} ipython3
:tags: [raises-exception]

# si on essaie d'accéder à la ligne d'index 0
# on remarque qu'en fait on obtient .. plein de lignes
global_df.loc[0]
```

+++ {"tags": ["framed_cell"]}

````{admonition} → les index ne sont pas toujours uniques

ce qui s'est passé c'est que :  
chacune de nos dataframe par pays a été construite à partir d'un index **séquentiel**  
i.e. un `RangeIndex` qui commence à chaque fois à 0  
et lors du `concat` on a conservé ces valeurs  
ce qui crée une multitude de lignes indexées par 0 (un par pays)

c'est un trait de `pandas`  
contrairement aux dictionnaires Python - où une clé est forcément unique  
il est possible de **dupliquer plusieurs entrées dans les index**  
ligne ou colonne - d'une dataframe

même si ça n'est en général pas souhaitable  
c'est souvent commode de pouvoir le faire  
pendant la phase de construction / mise au point de la dataframe  
quitte à adopter par la suite un index plus approprié  
(comme on va le faire bientôt)
````

+++

***

+++ {"tags": ["framed_cell"]}

### les dates en `pandas`

+++ {"tags": ["framed_cell"]}

convertissez les dates en format `pandas`

```{code-cell} ipython3
# votre code
global_df['date'] = pd.to_datetime(global_df['date'], format = '%Y-%m-%d')
```

+++ {"tags": ["framed_cell"]}

### un index plus idoine

+++ {"tags": ["framed_cell"]}

utilisez `pivot_table()` pour construire une nouvelle  dataframe indexée par les pays et les dates    
   **variante** on peut aussi utiliser `set_index()` 
   pour aboutir au même résultat

rangez votre résultat dans une variable `clean_df`

```{code-cell} ipython3
# votre code
clean_df = pd.pivot_table(global_df, index = ['country', 'date'], aggfunc = 'sum')
clean_df
```

```{code-cell} ipython3

```

+++ {"tags": ["framed_cell"]}

### accéder via un *MultiIndex*

+++ {"tags": ["framed_cell"]}

extrayez de la dataframe la série des 3 mesures  
   faites en France le 1er Janvier 2021  
attention un multi-index est exprimé avec un tuple

```{code-cell} ipython3
# votre code
extrac = clean_df.loc[('France', '2021-01-01'),['confirmed','deaths','recovered']]
extrac
```

+++ {"tags": ["framed_cell"]}

extrayez de    cette dataframe toutes les données relatives à la France

```{code-cell} ipython3
# votre code
extrac_fr = clean_df.loc['France',['confirmed','deaths','recovered']]
extrac_fr
```

+++ {"tags": ["framed_cell"]}

même question pour la France et l'Italie

```{code-cell} ipython3
# votre code
extrac_fr_it = clean_df.loc[['France','Italy'],['confirmed','deaths','recovered']]
extrac_fr_it
```

+++ {"tags": ["framed_cell", "level_intermediate"]}

### un exemple de slicing (très) avancé

````{admonition} →
pour illustrer la puissance de pandas, et la pertinence de notre choix d'index  
voyons comment utiliser du **slicing** (*très très avancé*)  
pour extraire cette fois les données relatives à

* deux pays au hasard - disons `France` et `Italy`
* à la période 1er Juillet - 15 Août 2021 inclus

pour ça on va tirer profit de la structure de l'index  
et aussi de la puissance du type `datetime64`

on va fabriquer :

* `countries`: une liste de pays - c'est facile
* `time_slice`: un slice sur le temps  
  qui en temps normal pourrait s'écrire `'july 2021' : '15 august 2021'`  
  (bornes inclusives puisque `.loc[]`)

* un slice sur les colonnes  
  mais au fait on les veut toutes, on peut utiliser `:`

l'idée serait ensuite d'écrire simplement

```python
clean_df.loc [ (countries, time_slice), :]
```

tout ça fonctionne *presque* très bien,  
**sauf pour** la création de `time_slice` qui, pour de sombres raisons de syntaxe,  
ne **peut pas** se faire ici avec la notation `start:stop`  
(parce que pas dans des `[]`)  
et du coup on utilise la fonction *builtin* `slice()` pour créer `time_slice`
````

```{code-cell} ipython3
:tags: [level_intermediate, raises-exception]

# ce qui nous donne le code suivant
# plutôt subtil, mais vraiment puissant

### pour slicer sur les deux composantes de l'index

# NB: si on voulait tous les pays on pourrait faire
# countries = slice(None)
# qui est équivalent à utiliser ::
# sauf qu'à nouveau ce n'est pas possible syntaxiquement ici
countries = ['France', 'Italy']
time_slice = slice('july 2021', '15 aug 2021')

clean_df.loc[
    # les lignes: c'est un 2-index donc on peut passer 2 slices
    (countries, time_slice),
    # les colonnes: on les veut toutes
    :]
```

***

+++

## dessinons

+++ {"tags": ["framed_cell"]}

### plot d'une dataframe

````{admonition} →
plutôt que d'utiliser directement la mécanique de `matplotlib.pyplot` (tendance à être fastidieux)  
il est préférable d'utiliser les méthodes comme `plot()` mais **directement** sur la dataframe

la logique de `df.plot()` est de dessiner **autant de courbes que de colonnes**  
et de plus pandas se charge de tous les labels! bref c'est recommandé, car plus rapide
````

+++ {"tags": ["framed_cell"]}

### sur un pays

````{admonition} →
du coup on a souvent seulement besoin de **mettre en forme** les données pour  
qu'elles puissent être directement plottées par cette logique simple

imaginons que dans notre cas on veuille comparer sur un graphique l'évolution de

* 2 mesures : `deaths`, `confirmed`
* entre 3 pays: `France`, `Italy` et `Germany`  

il nous faut donc construire une dataframe qui a:

* six colonnes - le produit cartésien des 2 mesures et 3 pays  
* et autant de lignes que de dates - indexé par les dates 
````

+++ {"tags": ["framed_cell"]}

affichez sur un graphique les 3 mesures pour la France au cours du temps

```{code-cell} ipython3
# votre code
extrac_fr.plot()
```

+++ {"tags": ["framed_cell"]}

idem avec seulement 2 mesures `deaths` et `confirmed`

```{code-cell} ipython3
# votre code
extrac_fr2 = clean_df.loc['France',['confirmed','deaths']]
extrac_fr2.plot()
```

+++ {"tags": ["framed_cell"]}

### sur plusieurs pays

+++ {"tags": ["framed_cell"]}

extrayez les données pour les 2 mesures et les 3 pays (appelons là `df3`)

```{code-cell} ipython3
# votre code
df3 = clean_df.loc[['France','Italy','Germany'],['confirmed','deaths']]
df3
```

```{code-cell} ipython3
:tags: [raises-exception]

# que du coup il n'y a plus qu'à plotter
# 
# à vous
df3.plot()
```

### fonction d'extraction

+++

écrivez une fonction `extract()` qui prend en paramètres

* la liste des pays concernés
* la liste des mesures concernées
* et les dates de début et de fin

et qui retourne une dataframe *prête à être affichée*

```{code-cell} ipython3
:jp-MarkdownHeadingCollapsed: true

# votre code
def extract(pays,mesures,dates):
    deb,fin = dates
    time_slice = slice(deb,fin)
    return clean_df.loc[(pays, time_slice), mesures]
```

en utilisant cette fonction, plottez sur un même graphique les données de deux pays

```{code-cell} ipython3
# votre code
extract(['France', 'Japan'], ['confirmed', 'recovered', 'deaths'], ('january 2020', 'april 2020')).plot()
```

## proposez des analyses personnelles sur ces données

```{code-cell} ipython3
# votre code
extract(['China'],['confirmed', 'recovered','deaths'], ('january 2020', 'january 2021')).plot()
```

```{code-cell} ipython3
# votre code
extract(['China'],['confirmed', 'recovered','deaths'], ('january 2020', 'january 2022')).plot()
```

```{code-cell} ipython3
# votre code
extract(['China'],['confirmed', 'recovered','deaths'], ('january 2020', 'january 2023')).plot()
```

```{code-cell} ipython3
# votre code
extract(['China'],['confirmed', 'recovered','deaths'], ('january 2022', 'january 2023')).plot()
```

```{code-cell} ipython3
# .../...
#malgré un parallèle flagrant entre la croissance des cas confirmés et soignés au début de la pandémie, à partir de 2022 en Chine, le nombre de cas positifs explosent 
#devantle nombre de personnes rétablies. Peut etre dû à des variantes de la maladie.
```

```{code-cell} ipython3

```
