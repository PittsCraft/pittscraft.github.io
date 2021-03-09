+++
draft = false
date = 2021-03-08T14:14:35+01:00
title = "Poker : MTT et ICM - Le Calcul Brutal"
description = "Où on calcule avec brutalité les espérances des joueurs selon un modèle indépendantiste."
slug = "poker_mtt_icm_calcul"
authors = ["Pitt"]
tags = ["Poker", "AI", "ICM", "MTT", "Game Theory"]
categories = ["Game Theory"]
externalLink = ""
series = ["Beating ICM"]
+++

Dans [mon précédent billet]({{< ref "/poker-icm1" >}}), je vous parlais des fonctions d'évaluation en théorie des jeux, plus particulièrement dans les tournois de poker No-Limit Hold'Em (NLHE), et spécifiquement de l'Independent Chip Model (ICM).

Pour rappel une fonction d'évaluation donne une estimation probabiliste des gains de chaque joueur en fonction de l'état du jeu. L'ICM utilise la taille des stacks des joueurs pour évaluer leurs chances d'atteindre chaque rang dans le classement final du tournoi. Ces probabilités multipliées par les prix de chaque rangs donnent une espérance de gain en monnaie sonnante et trébuchante qui permettra aux joueurs de prendre des décisions selon les issues possibles de ses actions.

## Calcul de l'ICM

L'ICM suppose que la probabilité pour chaque joueur d'accéder à la première place est égale à la proportion représentée par son stack sur la totalité des jetons en jeu (la somme des stacks).
Récursivement la probabilité pour un joueur `A` d'accéder à la seconde place va être une somme pour chaque autre joueur `B` en première place.

```
SA = stack du joueur A
SB = stack du joueur B

P2A = probabilité du joueur A d'accéder à la seconde place, initialisé à 0

Pour chaque autre joueur B:
    P1B = probabilité du joueur B d'être premier = SB / Somme des stacks
    P2A_sachant_1B = probabilité du joueur considéré d'accéder à la seconde place quand le joueur B accède à la première place
    P2A_sachant_1B = SA / (Somme des stacks - SB)
    P2A = P2A + P1B x P2A_sachant_1B
```

Pour calculer le tableau de valeurs de l'ICM en fonction des stacks et des prix (payouts) on pourrait écrire un algorithme de ce type :

```
payouts = [Q1, Q2, Q3, ...] : prix pour la première, deuxième, troisième.. place.
résultat = [RA, RB, RC, ...] : tableau des résultats, avec chaque valeur initialisée à 0

pour tout classement possible des joueurs :
  (prenons le classement (A, B, C...) pour l'exemple)
  p = probabilité de ce classement 
    = P1A 
      x P2B_sachant_1A 
      x P3C_sachant_1A_et_2B 
      x ...
  RA = RA + p x Q1
  RB = RB + p x Q2
  RC = RC + p x Q3
  ...

```

Voici un bout de code python naïf qui fait ça :

```python
from itertools import permutations
import numpy as np


def icm_proba(permutation, stacks: np.ndarray, stacks_sum: int):
    result = 1
    for rank, player in enumerate(permutation):
        stack = stacks[player]
        result *= stack / stacks_sum
        stacks_sum -= stack
    return result


def icm(stacks: np.ndarray, payouts: np.ndarray) -> np.ndarray:
    if len(stacks) == 0:
        return np.array([])
    nb_stacks = len(stacks)
    stacks_sum = sum(stacks)
    result = np.zeros((nb_stacks,))
    for permutation in permutations(range(nb_stacks)):
        probability = icm_proba(permutation, stacks, stacks_sum)
        for rank, player in enumerate(permutation):
            result[player] += probability * payouts[rank]
    return result


if __name__ == '__main__':
    val = icm(np.array([500, 300, 200]), np.array([600, 400, 0]))
    print(val)
```

Output :

```
[435.71428571 330.         234.28571429]
```
*On note la finesse du formattage*

## Complexité

Prendre tout classement possible de N joueurs, c'est ce qu'on appelle une permutation ou encore un arrangement.
Et on calcule très facilement combien il existe d'arrangement pour un nombre donné de joueurs :

```
1 : 1 (ben oui)
2 : 2 x 1 = 2 (juré ça change après)
3 : 3 x 2 x 1 = 6
4 : 4 x 3 x 2 x 1 = 24
5 : 5 x 4 x 3 x 2 x 1 = 120
```

C'est la factorielle, et on peut voir que ça croit très rapidement (plus vite qu'une exponentielle).
Si on code ça comme ça on obtient une complexité tellement violente qu'il est difficile de calculer l'ICM au delà de 10 joueurs dans un temps raisonnable.

Par exemple avec le code ci-dessus j'ai obtenu ces temps :

```
3.695487976074219e-05 seconds for 2 players
2.8848648071289062e-05 seconds for 3 players
8.177757263183594e-05 seconds for 4 players
0.0006070137023925781 seconds for 5 players
0.0037360191345214844 seconds for 6 players
0.027889013290405273 seconds for 7 players
0.24523401260375977 seconds for 8 players
2.2415449619293213 seconds for 9 players
23.641671180725098 seconds for 10 players
```

Je vous rappelle qu'à l'origine, je m'intéresse à l'ICM en tournoi multi-table. C'est à dire dans des tournois où le nombre de joueur est entre dix et... quelques milliers.

De plus, quand je regarde par exemple sur [HoldemRessources](https://www.holdemresources.net/icmcalculator), leur calculateur me répond en environ 200ms pour une vingtaine de joueurs. Même si ils ont mis un serveur très performant, on est dans des ordres de grandeur très lointains. On peut faire beaucoup mieux.

## Première optimisation : passer en récursif et en C++

Eh oui car le Python n'est rapide que quand il appelle des bibliothèques compilées. Faisons-donc la notre !
Et j'en profite pour faire une version récursive de l'algorithme qui va nous économiser pas mal de multiplications pour le calcul de probabilité. En effet, on regroupe toutes les permutations pour lesquelles le premier joueur dans le classement est le joueur d'index 0 dans les stacks par exemple, mais voyez plutôt ci-dessous.

Au premier appel, la fonction `recursiveNaiveIcm` va énumérer tous les joueurs possibles en première place. Pour chacun, elle va s'appeler elle-même pour la seconde place, etc.

```cpp
void recursiveNaiveIcm(const vector<double> &stacks, const vector<double> &payouts, vector<double> &result,
                        vector<bool> &usedPlayers, const int rank,
                        const int size, const double factor, const double stacksSum) {
    if (rank == size) {
        // No player to rank left
        return;
    }
    for (int i = 0; i < size; i++) {
        if (usedPlayers[i]) { continue; } // ith player has already been considered
        // The probability to have this ith player at this rank knowing the previous ranking is stacks[i] / stacksSum
        // The total probability of this ranking chain is thus :
        const double newFactor = factor * stacks[i] / stacksSum;
        // Let's remember this player is ranked
        usedPlayers[i] = true;
        // Add his pondered payout to the result
        result[i] += newFactor * payouts[rank];
        // Rank all possible next players
        recursiveNaiveIcm(stacks, payouts, result, usedPlayers, rank + 1, size, newFactor,
                          stacksSum - stacks[i]);
        // Reset the flag
        usedPlayers[i] = false;
    }
}

// Entry point
vector<double> naiveIcm(const vector<double> &stacks, const vector<double> &payouts) {
    // Number of players
    const int size = stacks.size(); 
    vector<double> result(size);
    // Flags to note down players that were already considered
    vector<bool> usedPlayers(size); 
    double stacksSum = accumulate(stacks.begin(), stacks.end(), 0.);
    recursiveNaiveIcm(stacks, payouts, result, usedPlayers, 0, size, 1, stacksSum);
    return result;
}
```
*Je n'avais pas fait de C++ depuis plus de 10 ans alors on ne se moque pas. De mon temps on codait en C++98, on avait une orange à Noël, et on était content !*


Voyons ce qu'on gagne en temps d'exécution :

```
Duration 0ms for 2 players
Duration 0ms for 3 players
Duration 0ms for 4 players
Duration 0ms for 5 players
Duration 0ms for 6 players
Duration 0ms for 7 players
Duration 2ms for 8 players
Duration 18ms for 9 players
Duration 177ms for 10 players
Duration 2087ms for 11 players
Duration 25924ms for 12 players
```

Pas mal, on peut ajouter un onzième et un douzième joueurs pour presque le même temps de calcul. En gros, on a une accélération de 11 x 12, donc de l'ordre de la centaine !

Mais ça ne nous emmène pas si loin. Continuons de gratter !

## Deuxième optimisation : cache et C-moins-moins

Supposons que nous avons `n` joueurs. À un moment du calcul où on a défini les rangs des `k` premiers joueurs, il reste `n - k` joueurs dont on va énumérer les différentes classements possibles.

Si on prend ces mêmes `k` premiers joueurs **mais dans un autre ordre**, l'énumération des différents classements des `n - k` joueurs restants va être presque identique :

- `size`, `stacks` et `payouts` ne bougent pas
- `result` est la destination, ce n'est pas un paramètre du calcul
- `usedPlayers` a les mêmes indexes à `true` (correspondant aux `k` joueurs classés)
- `rank` = `k`, c'est le même
- `stacksSum` correspond à la somme des stacks des `n - k` joueurs restants et a la même valeur

En fait, seul le paramètre `factor` varie, car la probabilité d'avoir les `k` premiers joueurs classés dans un ordre ou un autre n'est pas la même.

Cependant, ce facteur est finalement appliqué à toutes les contributions des appels imbriqués. Mais on peut donc l'extraire et l'appliquer a posteriori.

Ce-faisant, on va calculer une seule fois cette sous-partie pour `n - k` joueurs au lieu de la calculer pour chaque classement des mêmes `k` premiers joueurs.

Si en plus on applique ça pour tous les `k` entre 0 et `n - 1`, on n'a en fait à calculer la sous partie que pour toutes les combinaisons (de taille 1 à `n`) de joueurs.

Hors on sait très bien dénombrer également le nombre de combinaisons possible de joueurs parmis `n`, et c'est 2<sup>n</sup>. J'évoquais tout à l'heure la complexité factorielle de l'algorithme naïf, la réduire à une complexité exponentielle semble prometteur !

On va indexer les sous-calculs à mettre en cache selon la liste non-ordonnée des joueurs déjà classés, qui implicitement définit le rang actuel et la somme des stacks restants. Pour ce faire, on va choisir un vecteur de bits représenté par un entier, avec à 1 les bits des joueurs déjà classés, et on a une indexation parfaite (et on retombe d'ailleurs sur la complexité en 2<sup>n</sup>). En bonus, on peut se servir de ce bitmask pour remplacer l'ancien tableau de booléens, ce qui va nous économiser quelques cycles de CPU.

Pour ajouter encore un peu de perf, je vous mets avec ça des pointeurs old-s**C**hool plutôt que des `vector`s, car oui, ça fait une vraie différence.

*Entendons-nous bien, le niveau d'optimisation dépend fortement de la nature du problème. Si on bosse sur une app de TODO list, le nombre d'opérations est tellement faible qu'il n'y a en général[^optim] aucune raison de chercher à économiser des cycles de CPU. Mais quand on multiplie un nombre d'opérations par 2<sup>20</sup> c'est à dire environ un million, on s'approche dangereusement de la seconde d'exécution.*

et voici ce que ça donne :

```cpp
void recursiveIcm(const double *stacks, const double *payouts,
                  double *result,
                  long long usedPlayers, int rank,
                  int size, double factor, double stacksSum,
                  double *cacheValues, bool *cacheFlags) {
    if (rank == size) {
        return;
    }
    // Point subResult to the cache entry
    double *subResult = cacheValues + usedPlayers * size;
    // If the subResult wasn't computed previously, let's do it
    if (!*(cacheFlags + usedPlayers)) {
        for (int i = 0; i < size; i++) {
            const long long iMask = 0x1ll << i;
            if (usedPlayers & iMask) { continue; }
            // Probability of ith player to have this rank *among remaining players* 
            // (ignoring previously ranked ones)
            const double newFactor = stacks[i] / stacksSum;
            // Fill the EV of this subresult for this player
            subResult[i] += newFactor * payouts[rank];
            recursiveIcm(stacks, payouts, subResult, usedPlayers | iMask, rank + 1,
                          size, newFactor, stacksSum - stacks[i], cacheValues, cacheFlags);
        }
        // Mark the cache entry as filled
        *(cacheFlags + usedPlayers) = true;
    }
    // Apply the factor accounting for previously ranked players and add to the destination array
    for (int i = 0; i < size; ++i) {
        result[i] += factor * subResult[i];
    }
}

vector<double> icm(const vector<double> &stacks, const vector<double> &payouts) {
    const int size = stacks.size();
    // Prepare the array that will receive the results from the recursive function
    double result[size];
    fill_n(result, size, 0);
    // Just translate vectors to C arrays
    double stacksArray[size];
    double payoutsArray[size];
    for (int i = 0; i < size; ++i) {
        stacksArray[i] = stacks[i];
        payoutsArray[i] = payouts[i];
    }
    // Compute the total stack sum
    const double stacksSum = accumulate(stacks.begin(), stacks.end(), 0.);

    // ## Cache ##
    const auto cacheEntrySize = size * sizeof(double); // One double for each player
    // We'll store all cached subresult in this memory area.
    auto cacheValues = (double *) calloc(pow(2, size), cacheEntrySize);
    // And we'll store the information of whether each one is filled or not.
    auto cacheFlags = (bool *) calloc(pow(2, size), sizeof(bool));

    // Ok let's go
    recursiveIcm(stacksArray, payoutsArray, result, 0, 0, size, 1, stacksSum,
                  cacheValues, cacheFlags);
    
    // Just translate back to vector
    vector<double> toReturn(size);
    for (int i = 0; i < size; ++i) {
        toReturn[i] = result[i];
    }
    // And free the allocated memory
    free(cacheValues);
    free(cacheFlags);
    return toReturn;
}
```

Mais surtout le résultat en termes de performance : 

```
Duration 0ms for 2 players
Duration 0ms for 3 players
Duration 0ms for 4 players
Duration 0ms for 5 players
Duration 0ms for 6 players
Duration 0ms for 7 players
Duration 0ms for 8 players
Duration 0ms for 9 players
Duration 0ms for 10 players
Duration 0ms for 11 players
Duration 1ms for 12 players
Duration 1ms for 13 players
Duration 3ms for 14 players
Duration 7ms for 15 players
Duration 19ms for 16 players
Duration 51ms for 17 players
Duration 130ms for 18 players
Duration 321ms for 19 players
Duration 737ms for 20 players
```

## Chemin d'optimisation

Je vous passe bien sûr le cheminement détaillé, car j'ai pris le temps d'appréhender tout ce que j'avais raté ou oublié en termes de gestion de mémoire et des bibliothèques standards en C++20. En bref, disons que je suis passé par des versions bien plus moderne et haut niveau de cet algorithme avec notamment une `std::map` puis `std::unordered_map` puis `robinhood::unordered_map` pour le cache, des `std::vector` (qui se copiaient à chaque affectation ofc) puis des `std::vector&` et ensuite des `std::shared_ptr<std::vector>` super hype pour finalement revenir à des pointeurs et `calloc` pour des questions de performance évidentes. 

Comme disait Sam Waters : "Le profiling, c'est ma vie".

## Conclusion et quoi-est-après

J'ai toujours en tête de comparer et de titiller les fonctions d'évaluation en MTT. J'ai maintenant une bibliothèque de calcul de l'ICM de qualité professionnelle, qui m'autorise des simulations de tournois pour un petit nombre de joueur. Ça peut être suffisant pour faire des tests qualitatifs sur quelques joueurs, mais j'ai envie de voir si je peux aller plus loin.

Comment calculer efficacement l'ICM pour 50 joueurs ? Ce sera très probablement le sujet de mon prochain billet !

[^optim]: Il-y-a pas mal d'écoles différentes sur ce sujet bien sûr, et cette question prend de plus en plus d'importance avec la montée en puissance de l'éco-conception. D'un côté on risque de produire du code plus complexe, moins maintenable et moins fiable en cherchant l'optimisation, ce qu'on traduit par l'injonction "early optimization is the root of all evil" qui nous somme de garder le code simple à moins qu'il ne soit constaté nécessaire de l'optimiser -, de l'autre on assiste facilement à un phénomène d'encrassement à mesure que le code grossit : si des opérations suboptimales sont disséminées partout, l'optimisation consiste alors à tout réécrire et on regrette de ne pas avoir anticipé. Comme disait Bouddha : "La voie du milieu c'est bien, enfin rarement en voiture quand même".