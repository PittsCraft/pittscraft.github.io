+++
draft = false
date = 2021-03-03T14:14:35+01:00
title = "Poker : MTT et ICM - la question"
description = ""
slug = "poker_mtt_icm_question"
authors = ["Pitt"]
tags = ["Poker", "AI", "ICM", "Gambler's Ruin", "Game Theory"]
categories = ["Game Theory"]
externalLink = ""
series = ["Beating ICM"]
+++

Comme beaucoup, j’ai eu ma période “poker”. Elle s’est très vite traduite chez moi par la conception d’outils divers d’aide et d’analyse de jeu ([parce que je suis trop mauvais](http://george-abitbol.fr/v/6d59a4c2)), et a donné naissance à [cette petite bibliothèque Java](https://github.com/PierreMardon/gametheory). J’ai tiré beaucoup de cette expérience, que ce soit en théorie des jeux, en algorithmique théorique et pratique, en intelligence artificielle ou même en architecture logicielle. Alors bon, je n’assume pas tout à fait cette vieille lib, “si je devais la recoder je le ferais différemment”™ - pas en Java notamment - mais ça fait partie de ces projets tentaculaires qui vous portent pendant des années et vous font monter en compétence sur un large front dans l’allégresse.


Ça me titillait depuis longtemps de trouver un sujet dans le domaine et de bonnes raisons de m’y recoller. Quand soudain, je tombai sur le [paradoxe de Saint-Pétersbourg](https://fr.wikipedia.org/wiki/Paradoxe_de_Saint-P%C3%A9tersbourg) qui relança avec un ami une vieille conversation sur l’évaluation de l’EV ([expected value](https://en.wikipedia.org/wiki/Expected_value)) en MTT (multi-table tournament).

Il se trouve que la question *“Faut-il jouer toutes les mains EV+ ICM en tournoi?”* impliquait un long périple avec dedans du machine-learning, des simulations de jeu et des modélisations intéressantes - tout ce que j’adore.

Mais revenons au début, je vais commencer dans ce billet par tenter de vous expliquer la question.

## Théorie des jeux

Au poker, on cherche souvent à jouer GTO (Game Theory Optimal). Derrière ce terme, beaucoup d’implicite. La théorie des jeux étudie les interactions stratégiques d’agents à cheval entre les mathématiques et les sciences sociales, avec de très intéressantes questions qu’on va oublier bien vite :
Comment définit-on [la rationalité](https://www.youtube.com/watch?v=PFjX5tgu0iQ) des agents ?
Est-elle normative ou prescriptive ?
Quelles qualités descriptives a-t-elle ?

Voilà, on oublie, et maintenant on constate que dans la plupart des cas, tout le monde cherche à trouver la meilleure manière de jouer pour gagner - que ce soit aux échecs, au poker, au go ou à Starcraft[^different-games].

Dans des jeux comme les échecs, les algorithmes n’explorent pas l’ensemble de l’arbre de jeu mais évaluent un certain nombre de coups à l’avance. Si je regarde les 5 prochains coups possibles et que sur cette base je dois faire un choix, il me faut évaluer la valeur de chaque situation future résultant de chaque combinaison possible de 5 actions. On appelle fonction d’évaluation cette estimation de la valeur d’une situation qui n’est pas une situation finale. Dans des jeux faisant intervenir la chance (l’aléatoire), on la confond souvent avec la valeur statistique d’espérance (EV, Expected Value : l’espérance), car on va prendre les décisions en fonction de l’EV calculé grâce la fonction d’évaluation. EV+ signifie ainsi “d’espérance positive”.

## Poker

Le poker est un jeu de taille considérable du fait du nombre de combinaisons de cartes multiplié par le nombre de séquences d’action (mises) de jeu possibles. Je parle ici uniquement du No-Limit Hold’em (NLHE) qui est la variante la plus commune.

Dans le cas du cash-game où les jetons misés valent une somme déterminée d’argent, le problème de l’optimisation de la stratégie est circonscrit à une main. Cela reste énorme en complexité mais permet aux explorations algorithmiques de ne pas recourir à des fonctions d’évaluation intermédiaire, et d’interpréter directement les gains et pertes futurs résultant des actions en terme d’argent.

En effet, lorsqu’on va chercher à évaluer si telle action est préférable à telle autre, on va faire ce genre de calcul :

- Si je me couche, à la fin de la main j’ai un stack (un tapis) de 900 jetons.
- Si je suis la mise, selon les cartes qui vont être révélées :
   - j’ai 40% de chances de finir avec un stack de 1300 jetons
   - j’ai 60% de chances de finir avec un stack de 500 jetons

Je peux donc calculer mon espérance :
- Me coucher : 900 jetons
- Suivre : 0,4 x 1300 + 0,6 x 500 = 820 jetons

Ok, je dois donc rationnellement me coucher, car plus (+) de jetons, c’est mieux.

## Tournoi

Dans le cas d’un tournoi cependant, il-y-a la notion de prix, et donc que les rétributions sont attribuées selon la place finale d’un joueur.
Prenons le cas de trois joueurs qui doivent partager 1000€ de prix et disposent en tout de 1000 jetons. Nous ne connaissons rien des joueurs, si bien qu’on les considère aveuglément à égalité stratégique.

Si seul le premier joueur remporte les 1000€ il paraît raisonnable d’estimer leurs chances de gagner proportionnellement à leurs stacks. On peut d’ailleurs vérifier ça très facilement en leur faisant échanger des jetons aléatoirement (ce qui simule bien l’égalité stratégique).
Si les stacks sont 500, 300 et 200, alors en moyenne le premier joueur remportera la mise 50% du temps, le second 30% et le dernier 20%. Leurs espérances de gain sont donc 500€, 300€ et 200€ respectivement.

Petite simulation pour vérifier ça :

```python
import numpy as np
import random

nb_simulations = 100000
players_stacks = [5, 3, 2] # We'll exchange chips one by one, so let's speed-up by setting smaller stacks
payouts = [1000, 0, 0]

nb_players = len(players_stacks)

podiums = []

for i in range(nb_simulations):
    game_stacks = players_stacks.copy()
    podium = []
    while True:
        in_game_players = [i for i, stack in enumerate(game_stacks) if stack > 0]
        if len(in_game_players) == 1:
            # We have a winner
            podium = in_game_players + podium
            break
        # Exchange chips until one player is broke
        while True:
            # Randomly select two players
            random.shuffle(in_game_players)
            p1 = in_game_players[0]
            p2 = in_game_players[1]
            # And make them exchange one chip
            game_stacks[p1] += 1
            game_stacks[p2] -= 1
            # Is the losing player broke ?
            if game_stacks[p2] == 0:
                podium = [p2] + podium
                break  # Recompute in_game_players
    podiums.append(podium)

result = [0.0] * nb_players

for podium in podiums:
    for i in range(nb_players):
        player = podium[i]
        result[player] += payouts[i]
result = np.divide(result, nb_simulations)

print(result)

```
Output :
```
> [499.611 299.829 200.56 ]
```

Ce qui m'a bien l'air de converger vers le résultat attendu.

Maintenant, si le premier joueur remporte 600€ et le second 400€, a-t-on les mêmes espérances de gain ? Faisons donc tourner cette simulation avec ces payouts :

```python
payouts = [600, 400, 0]
```

Output :
```
> [445.71 330.83 223.46]
```

Il est tout de suite moins intuitif d’estimer ça à vue de nez.

## ICM

Le jeu simulé ci-dessus est le [Gambler’s Ruin](https://en.wikipedia.org/wiki/Gambler%27s_ruin) (qui est en soi plus un problème théorique qu’un jeu).

La performance de cette simulation est très faible pour de nombreux joueurs avec de nombreux jetons. Il existe des techniques de calcul avancées que je n’ai pas encore explorées, car en pratique les joueurs de poker préfèrent modéliser leur espérance de classement grâce à l’Independent Chip Model (ICM) dont le calcul est bien plus simple.

L’ICM est un modèle qui statue que la contribution à l’accès à la première place du tournoi est la même pour chaque jeton, puis récursivement pour les places suivantes. On retrouve donc la proportionnalité entre les jetons et l’espérance lorsque seul le premier joueur remporte le prix.

*Attention cependant, l’ICM n’est **pas** une solution du classement du problème du N-players Gambler’s Ruin. C’est par contre une approximation commune.*

Pour la situation ci-dessus, l’ICM nous donnera donc avec les stacks [500, 300, 200]:

En notant X_Y = le joueur X finit en Yième position.

P1_1 = 500 / 1000 = 0.5
P2_1 = 300 / 1000 = 0.3
P3_1 = 200 / 1000 = 0.2

Puis récursivement, en appliquant la formule des probabilités totales, on calcule la probabilité que le joueur 1 soit deuxième :
P1_2 = P2_1 * P(1_2 | 2_1) + P3_1 * P(1_3 | 3_1)
P1_2 = 0.3 * (500 / 700) + 0.2 * (500 / 800) = 0,339

De même pour les autres joueurs et ainsi de suite pour la troisième place.
Pour vous éviter les calculs à la main, il existe des calculateurs en ligne. [Celui d’HoldemResources](https://www.holdemresources.net/icmcalculator) et [celui d’ICMIzer](https://www.icmpoker.com/icmcalculator/).

Pour notre situation, l’ICM nous donne les valeurs :

```
> [435.71 330.00 234.29]
```

Pour comparer les résultats, la simulation de Gambler’s Ruin nous donnait :
```
> [445.71 330.83 223.46]
```

On retrouve un biais classique de l’ICM qui est de sous-estimer la valeur des plus gros stack et de surestimer les plus petits, tout en restant très correct pour les autres.


## Etat de l’art et ajustement d’ICM

L’ICM est reconnu comme un modèle plutôt fiable. Il donne des valeurs très satisfaisantes dans la majorité des cas. Cependant les joueurs lui reconnaissent de grandes faiblesses sur les très petits et très gros stacks (comme vu ci-dessus), dans le cas de blinds très élevées ou encore autour de la bulle - ce moment lors duquel il ne reste qu’un joueur à éliminer avant d’être certain que tous joueurs restant remportent un prix.

Il existe déjà des variantes de l’ICM. Celle que je décris est le modèle de Malmuth-Harville, alors que le modèle de Malmuth-Weitzman procède similairement en estimant les probabilités d’être éliminé en premier (plutôt que de terminer premier).

Ensuite dans les correctifs connus on a la technique [FGS](https://www.icmpoker.com/en/blog/how-fgs-future-game-simulation-calculator-works/) (Future Game Simulation) qui ajuste la valeur ICM selon des simulations de mains futures simplifiées.


## Récapitulatif

Prenons un tout petit peu de recul. Pourquoi a-t-on besoin de cet ICM déjà ?

Parce que lorsqu’on cherche une stratégie optimale, on doit estimer la valeur des situations vers lesquelles nous mènent nos actions potentielles. Dans le cas du cash-game, la valeur est immédiatement disponible en fin de main car elle est directement proportionnelle à la quantité de jetons. Mais pour un tournoi, il faudrait aller jusqu’à la résolution du tournoi entier pour observer le gain obtenu. Et autant dire que lorsqu’on doit dérouler toutes les possibilités de tirages et d’actions futures, on préfère circonscrire le problème à la main en cours pour avoir un résultat avant la fin des temps dans le cas d’un calcul informatique.

On préfèrerait savoir calculer les espérances de classement selon le modèle du Gambler’s Ruin, hélas il est beaucoup plus compliqué d’avoir une performance correcte dans ce calcul (ceci dit à ce sujet, j’ai dans ma pile de lectures quelques papiers que j’explorerai je l’espère un de ces jours[^gamblers-ruin-papers]).

La fonction d’évaluation est un outil sur lequel on va soit entrainer des IA, soit faire des analyses stratégiques avec des joueurs réels, mais elle ne représente pas l’espérance concrète des joueurs la plupart du temps. Et c’est d’ailleurs en général ce qu’on souhaite, car essayer de se rapprocher des valeurs concrète c’est essayer de revenir à la résolution du tournoi entier.

Pourtant, la fonction d’évaluation est un compromis qu’on souhaite toujours faire tendre vers plus de réalisme. Si on sait par exemple qu’un tournoi typique regroupe des joueurs de différents niveaux distribués d’une manière assez régulière, modéliser cet fait statistique peut produire une fonction d’évaluation plus adaptée à l’entraînement pour ce contexte de jeu. Car pour une IA, le jeu sur laquelle on l’entraîne inclut la fonction d’évaluation. Sa confrontation au jeu réel sera lourdement affecté par le choix de cette fonction. On peut d'ailleurs aussi arbitrairement considérer que le niveau des joueurs fait partie des données d'entrée de la fonction. Cependant on a tendance à reporter ce genre de données contextuelles vers les algorithmes stratégiques pour garder une fonction d'évaluation générique.

## Conclusion

La question *“Faut-il jouer toutes les mains EV+ ICM en tournoi?”* prise littéralement a une réponse immédiate : non, il-y-a même des ajustements que l’on sait bénéfiques.
Tout d’abord il est fort probable qu’une meilleure approximation du classement du Gambler’s Ruin donnerait de meilleurs résultats.

Cependant après cette question vient la suivante : peut-on trouver un meilleur modèle que l’ICM **calculable efficacement** ? Expérimentalement par exemple, pourrait on appliquer des techniques modernes de machine learning pour trouver une fonction d’évaluation qui surpasse l’ICM ? Et comment faire tout ça en pratique et pour un grand nombre de joueurs sachant que le calcul de l’ICM a une complexité factorielle ?


Des réponses dans un prochain billet !

***

PS: dans ce billet, j'essaye de ne pas trop rentrer dans les détails ce qui vaut des imprécisions et des raccourcis volontaires. N'hésitez cependant pas à m'écrire si vous trouvez à redire, ou s'il-y-a des manques criants !

PS bis: en cadeau, [un petit papier](https://arxiv.org/pdf/0911.3100.pdf) sympa qui montre deux théorèmes contre-intuitifs.
> Theorem 1. Suppose a tournament has prize money for nth place which is at least that for (n + 1)st place and that at least one player still in the tournament will not earn as much as second place prize money. Under the Independent Chip Model, any fair bet in which only one other player can gain or lose chips in the hand being played will lower the player’s expected prize money.

> Theorem 2. Suppose a tournament has prize money for nth place which is at least that for (n + 1)st place and that at least one player still in the tournament will not earn as much as second place prize money. Under the Independent Chip Model, the expected prize money of any player not involved in a fair bet between two players will increase*

Traduit à la pelle par :
- Si t’es en duel avec une équité de 50% contre un seul adversaire au sein d’un tournoi avec des prix croissants, tu es perdant selon l’ICM.
- Si deux joueurs sont en duel dans ces mêmes circonstances, tous les autres joueurs sont gagnants en ICM.

[^different-games]: Ces jeux sont de natures très variées. C’est ce qui fait d’ailleurs qu’à l’heure actuelle, [les meilleures IA de poker](https://ai.facebook.com/blog/pluribus-first-ai-to-beat-pros-in-6-player-poker/) sont basées sur des techniques très différentes de l’impressionnant [MuZero](https://deepmind.com/blog/article/muzero-mastering-go-chess-shogi-and-atari-without-rules) de DeepMind (j’espère avoir le temps de faire un petit papier “CFR vs Reinforcement Learning pour les nuls” tiens).

[^gamblers-ruin-papers]: les papiers en question : [Swan, Y., & Bruss, F. (2006). A matrix-analytic approach to the N-player ruin problem. Journal of Applied Probability, 43(3), 755-766. doi:10.1239/jap/1158784944](https://www.cambridge.org/core/journals/journal-of-applied-probability/article/matrixanalytic-approach-to-the-nplayer-ruin-problem/9FA680080D2FABB21564680B15E24858), et [Gambler's Ruin and the ICM - arXiv:2011.07610v2](https://arxiv.org/abs/2011.07610v2). Et bien sûr toutes leurs références 🙃
