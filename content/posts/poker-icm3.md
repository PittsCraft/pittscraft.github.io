+++
draft = false
date = 2021-03-31T08:14:35+01:00
title = "Poker : MTT et ICM - Méthode de Monte-Carlo"
description = "Un Monaco-Picon s'il vous plaît. En pinte."
slug = "poker_mtt_icm_montecarlo"
authors = ["Pitt"]
tags = ["Poker", "AI", "ICM", "MTT", "Game Theory", "Monte Carlo"]
categories = ["Game Theory"]
externalLink = ""
series = ["Beating ICM"]
+++

*Résumé des épisodes précédents:*
- En théorie des jeux, [les fonctions d'évaluation]({{< ref "/poker-icm1" >}}) permettent d'évaluer des situations, de les comparer, et donc de faire des choix renseignés
- l'Independent Chip Model (ICM) est la fonction d'évaluation de référence pour les tournois de poker, et on cherche à trouver mieux, notamment pour un grand nombre de joueurs
- l'ICM est long à calculer pour un grand nombre de joueurs. [Après pas mal d'optimisations]({{< ref "/poker-icm2" >}}), on a des temps humainement raisonnables pour une vingtaine de joueurs payés maximum. Si on veut utiliser l'ICM de manière intensive comme dans une simulation de tournoi, il faudra viser plutôt autour de dix joueurs payés.

Du coup©™ peut-on obtenir des valeurs correctes pour un plus grand nombre de joueurs ?

## Monte-Carlo : le tâtonnement convergent

Une méthode commune d'approximation de valeur est la [méthode de Monte-Carlo (MC)](https://en.wikipedia.org/wiki/Monte_Carlo_method). On va évaluer différents points d'une fonction complexe choisis aléatoirement pour déduire une approximation probabiliste de sa moyenne. En l'occurence pour l'ICM on peut typiquement essayer de tirer aléatoirement le classement des joueurs selon les hypothèses de l'ICM ce qui n'est somme-toute pas évident. 

Naïvement on devrait tirer le premier joueur classé selon une probabilité proportionnelle à son stack, puis le second parmi les joueurs restants et les stacks restants et ainsi de suite. Autant vous dire qu'on ne va pas bien loin avec ça car la complexité est alors assez violente : pour chaque tirage on doit calculer la distribution de probabilité et effectuer un tirage, ce qui au minimum va nous mener sur du *O(n<sup>2</sup>)* pour un tirage, et il en faut pas mal.

Heureusement des gens ont réfléchi, et notamment le développeur d'IA [Tysen Streib](https://twitter.com/tysenstreib) qui a beaucoup contribué à l'analyse stratégique au poker (il a co-écrit *Kill everyone* notamment). Il a donc très justement analysé en 2011 qu'il était possible d'effectuer un tirage de la sorte en tirant aléatoirement la durée de vie de chaque joueur selon *uniform_random[O:1]<sup>1/stack</sup>*, puis en ordonnant les durées de vie pour obtenir l'ordre de classement des joueurs.

Je vous recommande fortement la lecture de [son post original ici](https://forumserver.twoplustwo.com/15/poker-theory/new-algorithm-calculate-icm-large-tournaments-1098489/) et du thread qui suit, où l'algorithme tant que les raisons de la convergence sont bien expliqués, [à un niveau intuitif](https://forumserver.twoplustwo.com/showpost.php?p=28773129&postcount=17), avec [une interprétation théorique](https://forumserver.twoplustwo.com/showpost.php?p=28900308&postcount=27) et même des optimisations.

## Implémentation

Les gros points de consommation de CPU sont :
- le tirage aléatoire
- le calcul de puissance
- le tri

Comme suggéré dans le fil du post, on obtient de meilleures performance en utilisant le logarithme de la formule de tirage, `log` est strictement croissant donc l'ordre reste le même. 

Pour le tirage aléatoire, on a besoin d'un aléatoire statistique correct mais sans qualité cryptographique particulière. On gagne pas mal en utilisant [SFMT](http://www.math.sci.hiroshima-u.ac.jp/m-mat/MT/SFMT/index.html), une variante de l'algorithme de Mersenne-Twister plus rapide sur les microprocesseurs modernes.

Enfin pour le tri il faut noter que selon la distribution des prix, on peut éventuellement chercher à classer les premiers joueurs pour les places ayant un prix non-nul. Ce n'est pas une optimisation systématique mais lorsqu'elle est pertinente il serait dommage de l'ignorer.

Voici une version directe, sans autre optimisation.

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
        resultsArray[i] = 0;s
    }
    monteCarloIcm(stacksArray, payoutsArray, nbPlayers, trials, relevantRanksCount, resultsArray);
    for (int i = 0; i < nbPlayers; i++) {
        results[i] = resultsArray[i];
    }
    return results;
}
```

## Autres pistes d'optimisation

Voici quelques optimisations que j'imagine pertinentes :

- l'échantillonnage préférentiel : plus d'échantillons pour les cas les plus probables. On pourrait par exemple fixer successivement chaque joueur premier du classement, et estimer ce cas avec plus ou moins de précision par exemple avec un nombre d'échantillon proportionnel à sa probabilité au carré (la contribution de chaque cas restant proportionnelle au stack du joueur bien sûr). Pourquoi pas aller un ou plusieurs crans plus loin en envisageant toutes les combinaisons des k premiers joueurs.
- le [quasi-Monte-Carlo](https://fr.wikipedia.org/wiki/M%C3%A9thode_de_quasi-Monte-Carlo), une répartition plus homogène peut apporter une convergence plus rapide
- pour le tri et pour les très grands nombre de joueurs, comme les payouts sont souvent par palliers, utiliser quand c'est pertinent une succession de tris partiels par palier (plus précisément des [algorithmes de sélection](https://cs.stackexchange.com/questions/136619/batch-partial-sorting-algorithm)).

Si vous explorez ces optimisations (ou d'autres), faites-m'en part, je suis curieux !


## Convergence

On sait qu'on a une convergence en *O(1/sqrt(n))* (*note : ajouter une extension LateX à mon site builder*). Mais ça ne nous dit pas quand il faut arrêter l'échantillonnage !
Par le passé j'utilisais de petits algos observant la variation sur des lots de puissances de deux d'échantillons, ce qui n'est pas fiable : on peut avoir une convergence très lente, et un fort ralentissement ne permet pas de conclure en une précision donnée. "En pratique, ça marche" oui, enfin peut-être, mais ce n'est pas forcément beaucoup plus dur d'implémenter des garanties statistiques sûres.

Après quelques recherches j'ai trouvé une méthode assez générique dans ce (vieux) papier[^papier1]. Le calcul se base sur un plafond en dessous duquel la variance d'un échantillon satisfait une garantie statistique : avec une probabilité `α`, la moyenne de l'échantillon est dans l'intervalle de confiance de taille `d` autour de la moyenne à la limite.

J'ai donc réorganisé un peu mon code pour l'implémenter. Le coût du calcul étant assez élevé, il convient de ne pas l'appliquer à chaque étape de l'échantillonage, mais plutôt régulièrement, tous les 100 échantillons par exemple. De plus, j'ai extrait les définitions du RNG (pour un futur quasi-Monte-Carlo) et du tri (pour un futur tri partiel par palliers) histoire de pouvoir plus facilement implémenter d'autres versions de l'algorithme - *voir le code en fin d'article*.

Un point important à considérer est que la variance d'un calcul d'ICM par la méthode de Monte-Carlo dépend fortement de la distribution des stacks des joueurs et de celle des prix. En effet, pour prendre un extrême, si tous les prix sont égaux la variance est simplement nulle. On va évidemment souhaiter éviter le coût de calcul de cette *stopping rule* si on en opère en nombre, mais il faudra être prudent pour garder des garanties correctes. De plus, la taille `d` de l'intervalle de confiance est liée linéairement aux prix. Si on distribue deux fois plus de prix aux joueurs, pour une même précision relative on devra multiplier `d` par deux. L'idéal est alors d'exprimer `d` en fonction d'une grandeur pertinente comme la somme des prix distribués et d'en tenir compte lors du traitement d'échantillons variés pour au moins connaître la précision des résultats calculés.

 Si les échantillons sont assez homogènes, on pourra par exemple faire une petite tambouille comme calculer le nombre d'itérations nécessaires sur un petit ensemble de situations extraites du paquet à traiter, prendre le maximum et le multiplier par deux pour traiter tout le paquet (à vue de nez, "en pratique, ça marche").

## Performance

Je me suis intéressé au cas de 50 joueurs, les 40 premiers étant payés. Pour construire la structure de prix, j'ai notamment lu [cette présentation](https://www.chrismusco.com/dfsPresentation.pdf) qui m'a encouragé à utiliser une *power law fall-off* (une chute de loi de puissance..?) telle que le joueur au rang `i` obtienne un prix proportionnel à `1 / pow(i, a)`. J'ai négligé les étapes d'humanisation car je suspecte qu'au mieux elles font baisser la variance en créant des plateaux. Soyons-donc joyeusement pessimistes !

J'ai ensuite pris une distribution de stacks aléatoires selon une loi normale [tronquée](https://fr.wikipedia.org/wiki/Loi_tronqu%C3%A9e). Je ne pense pas que c'est vraiment représentatif d'une situation particulière, les distributions de stack sont assez difficiles à modéliser. Enfin pour le moment ça suffira, on cherche juste à connaître la performance de notre algorithme.

Dans ces circonstances, l'évaluation de la *stopping rule* pour une probabilité de `0.9` avec un intervalle de confiance de taille `1` (sachant que la somme des prix vaut 1000) me donne de manière assez stable une taille d'échantillon de **15500**.

Et une fois débarrassés de cette évaluation, le calcul prend `27ms` pour une moyenne sur 15500 échantillons. C'est très raisonnable en soi, mais si j'envisage de simuler quelques milliers de tournois comportant des milliers de mains elles-mêmes incluant plusieurs calculs d'ICM, ce ne sera pas suffisant. Damned.

Mais j'ai une idée un peu taquine pour résoudre ça...

## Next step

Tout ça pour... pour quoi déjà ? Pour défier l'ICM, parce qu'on se demande s'il n'y a pas mieux, et sinon pour se convaincre que c'est une bonne approximation.

Je pense qu'on pourrait déjà aller vers des simulations intéressantes avec un nombre réduit de joueurs pour mettre en évidence certains comportements. Mais je suis sur une bonne lancée, on va pas s'arrêter en si bon chemin !

Ce sera donc pour mon prochain article, en attendant n'hésitez pas à me contacter pour toute critique ou discussion via mon email en pied de page !

## Bonus : le code

Tout d'abord on a besoin d'une fonction qui n'existe pas dans les bibliothèques standard : `norminv` qui permet de déterminer pour quelle abscisse on obtient à gauche une certaine fraction de la population d'une distribution normale unitaire centrée en 0.

Un certain John D. Cook l'a fait pour nous, on va donc utiliser son code.

`util/math.cpp`

*Collez le code de cette page [https://www.johndcook.com/blog/cpp_phi_inverse/](https://www.johndcook.com/blog/cpp_phi_inverse/)*

`util/math.hpp`
```cpp
#ifndef MTT_EXPERIMENTS_C_MATH_HPP
#define MTT_EXPERIMENTS_C_MATH_HPP
double NormalCDFInverse(double p);

#endif //MTT_EXPERIMENTS_C_MATH_HPP
```

`monte-carlo-icm.cpp`
```cpp
#include "monte-carlo-icm.hpp"
#include <random>
#include <sfmt.hpp>
#include <util/math.hpp>

using namespace std;

namespace icm {
    /**
     * The Random Number Generator type : functions denoted by this alias are expected to draw doubles between 0 and 1.
     * Abstracted in order to plug a proper RNG for quasi-Monte-Carlo in the future.
     */
    using RNG = std::function<double()>;
    /**
     * The sorter functions are expected to sort the indexes of the draw array in reverse order according to the values
     * of the array.
     */
    using Sorter = std::function<void(double *draw, int *destination, int size)>;
    /**
     * Private section
     */
    namespace {
        /**
         * Compute a permutation according to the sorting of random values put to the power of given weights.
         * In fact, logarithm is used for performance, and logarithm tables could certainly be used to fasten the
         * computation.
         *
         * @param weights
         * @param destination where the permutation will be written to
         * @param size size of weights and destination array
         * @param rng random number generator
         * @param sorter function to sort the permutation
         */
        void monteCarloPermutation(const double *weights, int *destination, const int size, const RNG &rng,
                                   const Sorter &sorter) {
            double draw[size];
            for (int i = 0; i < size; i++) {
                draw[i] = log2(rng()) * weights[i];
                destination[i] = i;
            }
            sorter(draw, destination, size);
        }

        /**
         * Get the index of the first zero payout after which all payouts are zero as well
         * @param payouts the payouts array
         * @param size the size of the array
         * @return the index of the first zero payout
         */
        int firstZeroPayout(const double *payouts, int size) {
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
         * Compute the mean of a batch of samples
         * @param samples the samples
         * @param nbPlayers the size of each sample aka the number of players
         * @return a vector containing the mean of the samples
         */
        vector<double> mean(vector<double *> samples, const int nbPlayers) {
            const long long nbSamples = samples.size();
            vector<double> result(nbPlayers, 0);
            for (int i = 0; i < nbSamples; i++) {
                double *sample = samples[i];
                for (int j = 0; j < nbPlayers; ++j) {
                    result[j] += sample[j];
                }
            }
            for (int i = 0; i < nbPlayers; ++i) {
                result[i] /= (double) nbSamples;
            }
            return result;
        }

        /**
         * Compute the variance of samples
         * @param samples the samples
         * @param nbPlayers the size of each sample aka the number of players
         * @return the variance
         */
        double variance(vector<double *> samples, const int nbPlayers) {
            const long long nbSamples = samples.size();
            vector<double> meanSample = mean(samples, nbPlayers);
            double result = 0;
            for (int i = 0; i < nbSamples; i++) {
                double *sample = samples[i];
                for (int j = 0; j < nbPlayers; ++j) {
                    double diff = sample[j] - meanSample[j];
                    result += diff * diff;
                }
            }
            // Not sure about nbSamples - 1, should be nbSamples but the paper about the stopping rule states n-1...
            return result / (double) (nbSamples - 1);
        }

        /**
         * Compute the threshold for the variance under which the stopping limit is considered reached
         * @param alphaProbability confidence probability
         * @param confidenceIntervalLength confidence interval length (root mean square)
         * @param nbSamples number of samples
         * @return the variance threshold
         */
        double stoppingRuleLimit(double alphaProbability, double confidenceIntervalLength, long long nbSamples) {
            double zAlpha = NormalCDFInverse(alphaProbability + (1 - alphaProbability) / 2); // Symmetrical distribution
            return (double)nbSamples * confidenceIntervalLength * confidenceIntervalLength / zAlpha;
        }

        /**
         * Pick the appropriate sorter depending on payouts. We can use a partial sort if most of the payouts are zero
         * @param payouts the payouts
         * @param nbPlayers the size of the payouts array aka the number of players
         * @return a sorter
         */
        Sorter chooseSorter(const double *payouts, int nbPlayers) {
            // Partial sort vs sort : interesting only if we have many zero payouts at the end
            const int relevantRanksCount = firstZeroPayout(payouts, nbPlayers);
            const float partialSortThreshold = 0.2; // Magic number <3
            const bool partialSort = (float) relevantRanksCount / (float) nbPlayers <= partialSortThreshold;
            if (partialSort) {
                return [=](const double *draw, int *destination, int size) {
                    partial_sort(destination, destination + relevantRanksCount, destination + size,
                                 [&](const int &a, const int &b) {
                                     // Reverse order
                                     return (draw[a] > draw[b]);
                                 }
                    );
                };
            }
            return [](const double *draw, int *destination, int size) {
                sort(destination, destination + size,
                     [&](const int &a, const int &b) {
                         // Reverse order
                         return (draw[a] > draw[b]);
                     });
            };
        }

        /**
         * Create a RNG instance (using SFMT)
         * @return a RNG instance
         */
        RNG defaultRng() {
            // Initialize random distribution
            // https://stackoverflow.com/questions/19665818/generate-random-numbers-using-c11-random-library
            // SFMT variant
            random_device rd;
            wtl::sfmt19937 mt(rd());
            shared_ptr<wtl::sfmt19937> mtPtr = std::make_shared<wtl::sfmt19937>(mt);
            uniform_real_distribution<double> dist(0, 1);
            return [mtPtr = make_shared<wtl::sfmt19937>(mt),
                    distPtr = make_shared<uniform_real_distribution<double>>(dist)]() -> double {
                return (*distPtr)((*mtPtr));
            };
        }

        /**
         * Check if we can stop the trials according to the confidence probability and interval
         * @param alphaProbability the confidence probability
         * @param confidenceIntervalLength confidence interval length (root mean square)
         * @param samples the samples
         * @param nbPlayers the number of players
         * @return true if the stopping rule criteria are met
         */
        bool canStop(const double alphaProbability, const double confidenceIntervalLength, const vector<double *> &samples,
                     const int nbPlayers) {
            const long long nbSamples = samples.size();
            if (nbSamples < 2) {
                return false;
            }
            double limit = stoppingRuleLimit(alphaProbability, confidenceIntervalLength, nbSamples);
            return variance(samples, nbPlayers) <= limit;
        }

        /**
         * Normalize stacks (not really necessary but cheap) and invert to weights
         * @param stacks the stacks
         * @param nbPlayers the number of players
         * @param weights the destination array
         */
        void fillWeights(const double *stacks, int nbPlayers, double *weights) {
            const double stacksAvg = accumulate(stacks, stacks + nbPlayers, 0.) / (double) nbPlayers;
            for (int i = 0; i < nbPlayers; i++) {
                weights[i] = stacksAvg / stacks[i];
            }
        }
    }

    /**
     * Perform one MC trial
     *
     * @param results destination array to add trial contribution into
     * @param payouts the contribution to add to the results per rank
     * @param weights something proportional to the inverse of the stacks of the players
     * @param permutation an array that will be used to store the permutation (permutation[i] = player that reaches rank i)
     * @param nbPlayers the number of players
     * @param rng random number generator
     * @param sorter function to sort the permutation
     */
    void monteCarloIcmTrial(double *results, const double *payouts, double *weights, int *permutation, int nbPlayers,
                            const RNG &rng, const Sorter &sorter) {
        monteCarloPermutation(weights, permutation, nbPlayers, rng, sorter);
        // Cumulate payouts according to the random permutation
        for (int j = 0; j < nbPlayers; j++) {
            results[permutation[j]] += payouts[j];
        }
    }

    /**
     * Monte-Carlo ICM Ranking algorithm.
     *
     * @param stacks the stacks
     * @param payouts the payouts
     * @param nbPlayers the number of players
     * @param trials the number of trials to perform
     * @param results destination array for ICM EV
     */
    void monteCarloIcm(const double *stacks, const double *payouts, const int nbPlayers, const long long trials,
                       double *results) {
        // Algorithm : https://forumserver.twoplustwo.com/15/poker-theory/new-algorithm-calculate-icm-large-tournaments-1098489/
        RNG rng = defaultRng();
        int permutation[nbPlayers];

        double weights[nbPlayers];
        fillWeights(stacks, nbPlayers, weights);

        double contrib[nbPlayers];
        // Prepare each trial ranking contribution to the total
        for (int i = 0; i < nbPlayers; i++) {
            contrib[i] = payouts[i] / (double) trials;
        }

        const Sorter sorter = chooseSorter(payouts, nbPlayers);
        for (long i = 0; i < trials; i++) {
            monteCarloIcmTrial(results, contrib, weights, permutation, nbPlayers, rng, sorter);
        }
    }

    /**
     * Monte-Carlo ICM Ranking algorithm with stopping rule.
     *
     * @param stacks the stacks
     * @param payouts the payouts
     * @param nbPlayers the number of players
     * @param results destination array for ICM EV
     * @param stoppingAlphaProbability the confidence probability
     * @param stoppingConfidenceIntervalLength confidence interval length (root mean square)
     * @param stoppingEvalLag the number of trials that are performed before each evaluation of the stopping rule
     */
    long long monteCarloIcmWithStoppingRule(const double *stacks, const double *payouts, const int nbPlayers,
                                            double *results, const double stoppingAlphaProbability,
                                            const double stoppingConfidenceIntervalLength, long long stoppingEvalLag) {
        // Algorithm : https://forumserver.twoplustwo.com/15/poker-theory/new-algorithm-calculate-icm-large-tournaments-1098489/
        // Stopping rule : http://www.lib.ncsu.edu/resolver/1840.4/5244 (first method for independent samples)
        RNG rng = defaultRng();
        int permutation[nbPlayers];

        double weights[nbPlayers];
        fillWeights(stacks, nbPlayers, weights);

        // For each trial we'll contribute the real payouts because we want to use them for the stopping rule
        // computation

        const Sorter sorter = chooseSorter(payouts, nbPlayers);

        vector<double *> samples;
        vector<double *> allocated;
        long long count = 0;
        while (true) {
            // While we don't stop according to the stopping rule, we create a batch of samples
            auto *memory = (double *) calloc(nbPlayers * stoppingEvalLag, sizeof(double));
            // Store the allocated memory area pointer
            allocated.push_back(memory);
            for (long i = 0; i < stoppingEvalLag; i++) {
                double *sampleMemory = memory + i * nbPlayers;
                monteCarloIcmTrial(sampleMemory, const_cast<double *>(payouts), weights, permutation, nbPlayers, rng,
                                   sorter);
                // Store the pointer to the sample
                samples.push_back(sampleMemory);
            }
            count += stoppingEvalLag;
            // Break if the stopping rule says it's enough
            if (canStop(stoppingAlphaProbability, stoppingConfidenceIntervalLength, samples, nbPlayers)) {
                break;
            }
        }
        // Sum the samples
        for (double *sample : samples) {
            for (int i = 0; i < nbPlayers; i++) {
                results[i] += sample[i];
            }
        }
        // Divide the sum to get the mean
        for (int i = 0; i < nbPlayers; i++) {
            results[i] /= (double) count;
        }
        // Free the allocated memory areas
        for (double *memory : allocated) {
            free(memory);
        }
        // Return the number of samples that were generated
        return count;
    }
}
```

`monte-carlo-icm.hpp`
*Déclarez les fonctions qu'il vous intéresse d'exposer !*

[^papier1]: A Brief Survey of Stopping Rules in Monte Carlo Simulations, 1968, Gilman - [https://repository.lib.ncsu.edu/handle/1840.4/5244](https://repository.lib.ncsu.edu/handle/1840.4/5244)

