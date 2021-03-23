+++
draft = true
date = 2021-03-09T14:14:35+01:00
title = "Poker : MTT et ICM - Méthode de Monte-Carlo"
description = "Un Monaco-Picon s'il vous plaît. En pinte."
slug = "poker_mtt_icm_montecarlo"
authors = ["Pitt"]
tags = ["Poker", "AI", "ICM", "MTT", "Game Theory"]
categories = ["Game Theory"]
externalLink = ""
series = ["Beating ICM"]
+++

*Résumé des épisodes précédents:*
- En théorie des jeux, [les fonctions d'évaluation]({{< ref "/poker-icm1" >}}) permettent d'évaluer des situations, de les comparer, et donc de faire des choix renseignés
- l'Independent Chip Model (ICM) est la fonction d'évaluation de référence pour les tournois de poker, et on cherche à trouver mieux, notamment pour un grand nombre de joueurs
- l'ICM est long à calculer pour un grand nombre de joueurs. [Après pas mal d'optimisations]({{< ref "/poker-icm2" >}}), on a des temps humainement raisonnables pour une vingtaine de joueurs maximum. Si on veut utiliser l'ICM de manière intensive comme dans une simulation de tournoi, il faudra viser plutôt autour de dix joueurs

DU COUP©™ peut-on obtenir des valeurs correctes pour un plus grand nombre de joueurs ?

## Monte-Carlo : le tâtonnement convergent

Une méthode commune d'approximation de valeur est la [méthode de Monte-Carlo (MC)](https://en.wikipedia.org/wiki/Monte_Carlo_method). On va évaluer différents points d'une fonction complexe choisis aléatoirement pour en tirer une approximation probabiliste de résultat. En l'occurence pour l'ICM on peut typiquement essayer de tirer aléatoirement le classement des joueurs selon les hypothèses de l'ICM ce qui n'est somme-toute pas évident. 

Naïvement on devrait tirer le premier joueur classé selon une probabilité proportionnelle à son stack, puis le second parmi les joueurs restants et les stacks restants et ainsi de suite. Autant vous dire qu'on ne va pas bien loin avec ça car la complexité est alors assez violente : pour chaque tirage on doit calculer la distribution de probabilité et effectuer un tirage, ce qui au minimum va nous mener sur du *O(n<sup>2</sup>)* pour un tirage, et il en faut pas mal.

Heureusement des gens ont réfléchi, et notamment le développeur d'IA [Tysen Streib](https://twitter.com/tysenstreib) qui a beaucoup contribué à l'analyse stratégique au poker (il a co-écrit *Kill everyone* notamment). Il a donc très justement analysé en 2011 qu'il était possible d'effectuer un tirage de la sorte en tirant aléatoirement la durée de vie de chaque joueur selon *uniform_random[O:1]<sup>1/stack</sup>*, puis en ordonnant les durées de vie pour obtenir l'ordre de classement des joueurs.

Je vous recommande fortement la lecture de [son post original ici](https://forumserver.twoplustwo.com/15/poker-theory/new-algorithm-calculate-icm-large-tournaments-1098489/) et du thread qui suit, où l'algorithme tant que les raisons de la convergence sont bien expliqués, [à un niveau intuitif](https://forumserver.twoplustwo.com/showpost.php?p=28773129&postcount=17) et avec [une interprétation théorique](https://forumserver.twoplustwo.com/showpost.php?p=28900308&postcount=27) et même des optimisations.

## Implémentation

Les gros points de consommation de CPU sont :
- le tirage aléatoire
- le calcul de puissance
- le tri

Comme suggéré dans le fil du post, on obtient de meilleures performance en utilisant le logarithme de la formule de tirage, `log` est strictement croissant donc l'ordre reste le même. 

Pour le tirage aléatoire, on a besoin d'un aléatoire statistique correct mais sans qualité cryptographique particulière. On gagne pas mal en utilisant [SFMT](http://www.math.sci.hiroshima-u.ac.jp/m-mat/MT/SFMT/index.html), une variante de l'algorithme de Mersenne-Twister plus rapide sur les microprocesseurs modernes.

Enfin pour le tri il faut noter que selon la distribution des prix, on peut éventuellement chercher à classer les premiers joueurs pour les places ayant un prix non-nul. Ce n'est pas une optimisation systématique mais lorsqu'elle est pertinente il serait dommage de l'ignorer.


Voici ma version actuelle, si j'ai raté une optim n'hésitez pas :)

```cpp

#include "monte-carlo-icm.hpp"
#include <random>
#include <unordered_map>
#include <sfmt.hpp>

/**
  * Compute a permutation according to the sorting of random values put to the power of given weights.
  * In fact, logarithm is used for performance, and logarithm tables could certainly be used to fasten the
  * computation.
  *
  * @param weights
  * @param destination where the permutation will be written to
  * @param size size of weights and destination array
  * @param dist random distribution
  * @param mt SFMT (fast Mersenne-Twister algorithm)
  */
void monteCarloPermutation(const double *weights, int *destination, const int size,
                            uniform_real_distribution<double> &dist, wtl::sfmt19937 &mt) {
    double draw[size];
    for (int i = 0; i < size; i++) {
        draw[i] = log2(dist(mt)) * weights[i];
        destination[i] = i;
    }
    sort(destination, destination + size,
          [&](const int &a, const int &b) {
              // Reverse order
              return (draw[a] > draw[b]);
          }
    );
}

/**
  * Same as `monteCarloPermutation` except the sorting process stops for performance
  * after the first values are sorted.
  *
  * @param weights
  * @param destination where the permutation will be written to
  * @param size size of weights and destination array
  * @param nbRelevant number of top values that should be strictly sorted. The other ones may not be sorted
  *          according to the random draw.
  * @param dist random distribution
  * @param mt SFMT (fast Mersenne-Twister algorithm)
  */
void monteCarloPermutationPartial(const double *weights, int *destination, const int size,
                                  const int nbRelevant,
                                  uniform_real_distribution<double> &dist, wtl::sfmt19937 &mt) {
    double draw[size];
    for (int i = 0; i < size; i++) {
        draw[i] = log2(dist(mt)) * weights[i];
        destination[i] = i;
    }
    partial_sort(destination, destination + nbRelevant, destination + size,
                  [&](const int &a, const int &b) {
                      // Reverse order
                      return (draw[a] > draw[b]);
                  }
    );
}

int firstZeroPayout(vector<double> payouts) {
    const int size = payouts.size();
    int firstZeroPayout = -1;
    for (int i = 0; i < size; ++i) {
        if (firstZeroPayout < 0) {
            if (payouts[i] == 0) {
                firstZeroPayout = i;
            }
        } else if (payouts[i] != 0) {
            firstZeroPayout = -1;
        }
    }
    return firstZeroPayout;
}

/**
  * Monte-Carlo ICM Ranking algorithm.
  *
  * @param stacks
  * @param payouts
  * @param trials
  * @param relevantRanksCount
  * @param results array of ICM EV
  */
void monteCarloIcm(const double *stacks, const double *payouts, const int nbPlayers, const long trials,
                    const int relevantRanksCount, double *results) {
    // Algorithm : https://forumserver.twoplustwo.com/15/poker-theory/new-algorithm-calculate-icm-large-tournaments-1098489/
    // Initialize random distribution
    // https://stackoverflow.com/questions/19665818/generate-random-numbers-using-c11-random-library
    // SFMT variant
    random_device rd;
    wtl::sfmt19937 mt(rd());
    uniform_real_distribution<double> dist(0, 1);
    int permutation[nbPlayers];
    const double stacksAvg = accumulate(stacks, stacks + nbPlayers, 0.) / (double) nbPlayers;
    double weights[nbPlayers];
    double contrib[nbPlayers];
    // - Normalize stacks (not really necessary but cheap) and invert to weights
    // - Prepare each trial ranking contribution to the total
    for (int i = 0; i < nbPlayers; i++) {
        weights[i] = stacksAvg / stacks[i];
        contrib[i] = payouts[i] / trials;
    }
    // Partial sort vs sort : interesting only if we have many zero payouts at the end
    const float partialSortThreshold = 0.2; // Magic number <3
    const bool partialSort =
            relevantRanksCount >= 1 && ((float) relevantRanksCount / (float) nbPlayers <= partialSortThreshold);


    for (long i = 0; i < trials; i++) {
        // Draw a permutation
        if (partialSort) {
            monteCarloPermutationPartial(weights, permutation, nbPlayers, relevantRanksCount, dist, mt);
        } else {
            monteCarloPermutation(weights, permutation, nbPlayers, dist, mt);
        }
        // Cumulate payouts according to the random permutation
        for (int j = 0; j < nbPlayers; j++) {
            results[permutation[j]] += contrib[j];
        }
    }
}

/**
  * Vector wrapper of MC ICM
  * @param stacks
  * @param payouts
  * @param trials
  * @return ICM EV
  */
vector<double> monteCarloIcm(const vector<double> &stacks, const vector<double> &payouts, const long trials) {
    const int nbPlayers = stacks.size();
    const int relevantRanksCount = firstZeroPayout(payouts);
    vector<double> results(nbPlayers);
    double stacksArray[nbPlayers];
    double payoutsArray[nbPlayers];
    double resultsArray[nbPlayers];
    for (int i = 0; i < nbPlayers; ++i) {
        stacksArray[i] = stacks[i];
        payoutsArray[i] = payouts[i];
        resultsArray[i] = 0;
    }
    monteCarloIcm(stacksArray, payoutsArray, nbPlayers, trials, relevantRanksCount, resultsArray);
    for (int i = 0; i < nbPlayers; i++) {
        results[i] = resultsArray[i];
    }
    return results;
}
```

# Convergence et performance

On sait qu'on a une convergence en *1/