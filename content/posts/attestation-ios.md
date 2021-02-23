+++ 
draft = true
date = 2020-11-09T20:38:09+01:00
title = "Attestation 2s iOS "
description = ""
slug = ""
authors = ["Pitt"]
tags = ["iOS", "combine", "SwiftUI", "confinement", "covid"]
categories = []
externalLink = ""
series = []
+++

Il m'est arrivé plusieurs fois d'oublier mon attestation de sortie (c'est mal), de la générer au volant en panique (c'est très mal), de prendre du retard en tapant le formulaire avant de partir... Loin de moi l'idée de débattre du bien-fondé du confinement et de ses modalités, cependant j'étais confronté à un inconfort mineur. Et comme tout bon ingé, j'ai cherché et évalué des solutions complètement superflues - toutefois avec un indiscutable sérieux et un professionnalisme inébranlable.

## La voie des anciens

*Imprimer des attestations préremplies et ne laisser que le motif, la date, l'heure et la signature à remplir.*

Oui ça marche, mais la matérialisation est une contrainte forte. Si on oublie de remplir son attestation (c'est mal) et qu'on prend la voiture on n'a pas de moyen de gérer la situation sans un 180° bien crissant (ce qui est certes classe mais dangereux).

On me dit dans l'oreillette qu'on peut tout à fait écrire une attestation à la main sur papier libre... Oui mais bon, change pas de sujet, j'ai pas de stylo ni de papier sur moi, voilà, et puis niveau ergonomie, c'est so *les millénaires passés* l'écriture...

## La voie officielle #1 (web)

J'ai essayé de faire avec la page web gouvernementale. Et franchement, c'est correct sur ordinateur avec le bon équipement logiciel. J'utilise personnellement un navigateur Chromium avec le plugin de mon gestionnaire de mots de passe [Dashlane](https://www.dashlane.com/fr). Le seul inconfort évident est la lecture des motifs du déplacement. J'ai commencé à faire un petit plugin Chrome pour retravailler ça avant de me raviser rapidement : les plugins ne fonctionnent pas sur les navigateurs Chromium iOS et la génération sur portable est bien plus pratique.

## La voie officielle #2 (TousAntiCovid)

On monte en qualité avec le générateur intégré à l'application [TousAntiCovid](https://www.gouvernement.fr/info-coronavirus/tousanticovid). Il est possible de faire retenir mes coordonnées par l'appli et les motifs ont un titre en gras qui permet d'y voir un peu plus clair. Cependant je ne suis pas intéressé par la fonctionnalité de traçage de cette application. Donc je n'ai pas apprécié quand j'ai dû impérativement autoriser l'app à utiliser le bluetooth à la première ouverture. Et puis en regardant ça, je commençais à avoir ma petite idée de l'appli idéale donc toutes les petites frictions du parcours pour générer mon autorisation me faisaient tiquer. Ca fait quand même pas mal d'étapes après ouverture de l'application :

- **scroll tout en bas**
- **tap sur *Attestation de déplacement***
- **tap sur *Nouvelle attestation***
- entrer mes données - ok ça s'enregistre on ne le compte pas
- tap (optionnel) sur l'heure pour régler l'heure de ma sortie - je n'ai jamais touché à la date jusque là
- **tap pour choisir le motif de déplacement (qui ne s'enregistre pas)**
- sur mon iPhone, il faut 3 hauteurs d'écran pour lire intégralement la liste des motifs, on ajoute donc souvent un scroll ou deux
- **tap sur le motif**
- **tap sur *Générer***
- **alerte de confirmation : tap sur *Je certifie***

Donc dans le cas idéal (je pars maintenant, c'est bien moi qui gènère l'attestation et je suis la dernière personne à avoir utilisé le générateur sur cet appareil, et mon motif est mon caractère laborieux) j'ai donc **7 actions** avant d'obtenir le QR Code tant convoité. **C'est beaucoup.**

## La voie des ptits malins (app iOS Raccourcis)

Certaines boîtes comme [Luko](https://www.luko.eu/blog/confinement-astuce-remplir-attestation-derogatoire-automatiquement) ou [Newzik](https://newzik.com/fr/attestation-covid/) vous proposent de générer un lien qui contient vos données, et qui mène à un générateur automatique qui affiche l'attestation générée avec l'heure de sortie actuelle.

L'idée est notamment d'utiliser un raccourci déclenché par Siri par exemple pour commander son attestation à la voix ou encore la faire ouvrir automatiquement dès qu'on quitte son domicile. On n'est pas loin de la solution idéale, seulement je ne peux pas générer une attestation à la bourre.

Ceci-dit, ce lien est un bon exemple de ce qu'on peut faire rapidement pour se faciliter la vie. Quelques bookmarks, un peu de configuration et pour les gens pas trop technophobes on s'en sort.

## L'app de mes rêves
### Cahier des charges

L'app de mes rêves
- dans le meilleur des cas, nécessite **une seule action** pour générer une attestation
- si le motif change, une à deux actions supplémentaires sont tolérables.
- demande le plus rarement possible des actions supplémentaires.
- permet à ma compagne de faire son attestation sur mon téléphone sans effacer mes données préremplies
- gère *très efficacement* mon cas pathologique d'oubli. Elle doit donc me permettre de générer mon attestation lorsque je me rends compte après 20 minutes de trajet que j'ai oublié mon attestation : pour être dans les clous, **mon heure de sortie doit alors être 20 minutes dans le passé**
- n'embarque aucune autre fonctionnalité non souhaitée
- n'envoie aucune donnée à qui que ce soit (pas de tracking publicitaire ou autre)
- n'utilise aucune bibliothèque tierce non maîtrisée à 100%

Je me suis auto-défié, et au bout d'une petite journée de développement j'avais un prototype fonctionnel, ce qui m'a encouragé à continuer. Au bout de trois jours de développement j'avais une app présentable, les aspects légaux étaient confirmés, et la plupart des raffinements majeurs étaient implémentés.

### Réalisation !

Je suis bien content d'annoncer que j'ai respecté \*presque\* tous les points de l'app de mes rêves. Seule entorse, comme il faut quand même être un peu sérieux, j'ai ajouté une confirmation de véracité des données à la génération de l'attestation, on a donc deux actions pour générer l'attestation.

Tout comme pour la solution de génération par liens, j'ai récupéré [le code publié par le ministère de l'intérieur](https://github.com/LAB-MI/attestation-deplacement-derogatoire-q4-2020) pour générer les PDFs en inspectant son intégralité, en extrayant uniquement les parties nécessaires, puis en le modifiant pour son intégration dans l'app.

Je ne vais pas vous cacher que je n'aurais pas fait cette application juste pour me faciliter les sorties. Je voulais également expérimenter [SwiftUI](https://developer.apple.com/xcode/swiftui/), la bibliothèque déclarative d'interface utilisateur d'Apple qui me tend les bras depuis plusieurs années, et c'était une bonne occasion.

Et voici le résultat :

![Deux taps pour une attestation](/images/attestation_2s/attestation_2s.gif)

#### Vue principale

![Vue principale](/images/attestation_2s/main_view.jpeg)

Ici on peut :

- ajouter / supprimer / réordonner les personnes
- sélectionner un ou plusieurs motifs de sortie
- sélectionner la date de sortie en temps relatif par rapport à l'heure actuelle : les boutons du Stepper (+ / -) ajoutent ou retirent 10 minutes
- et surtout aller vers l'attestation !

#### Vue d'édition de personne

![Formulaire de personne](/images/attestation_2s/person_form.png)

Un simple formulaire tout bête :)

#### Présentation de l'attestation

![Image description](/images/attestation_2s/attestation_presentation.png)

Très simple, on peut juste :

- partager le PDF
- épingler l'attestation : dans ce cas la présentation en carte ne se laisse pas fermer comme d'habitude par swipe vertical, la croix de fermeture disparaît, et au cas où l'utilisateur ferme l'app, l'attestation sera restaurée à la réouverture

#### Temps de génération de l'attestation : 3s

Dans mon usage quotidien, avec ma (vraie) identité de remplie, je mets environ 3s à remplir mon attestation entre l'ouverture de l'app, le choix ou la vérification du motif et le réglage ou la vérification de l'heure. Oui je suis lent, mais mon objectif est atteint, je peux sans risque générer mon attestation dans des situation d'urgence et d'oubli \o/

![Cas d'oubli - c'est mal](/images/attestation_2s/30min_earlier.png)

### App Store ?

Eh bien malgré ma gestion paranoïaque des données utilisateur, il semble que ce ne soit pas suffisant pour Apple qui (je pense) n'autorise simplement aucune app avec cette fonctionnalité sauf celle du gouvernement.

> We found in our review that your app provides services or requires sensitive user information related to the COVID-19 pandemic. Since the COVID-19 pandemic is a public health crisis, services and information related to it are considered to be part of the healthcare industry. In addition, the seller and company names associated with your app are not from a recognized institution, such as a governmental entity, hospital, insurance company, non-governmental organization, or university.
> 
> Per section 5.1.1 (ix) of the App Store Review Guidelines, apps that provide services or collect sensitive user information in highly-regulated fields, such as healthcare, should be submitted by a legal entity that provides these services, and not by an individual developer.

J'ai évidemment fait appel mais je ne pense pas qu'ils cèderont, tant pis, je ne partagerai donc mon app qu'avec mes proches (du moins ceux qui possèdent un iPhone) !

### Développement : retour d'XP

#### SwiftUI + Combine

Tout d'abord SwiftUI est très agréable à utiliser. On a évidemment les traditionnelles errances de XCode 12, que ce soit niveau compilation, complétion, aperçu de l'UI... Mais il convient de saluer la prouesse qu'est l'implémentation de ce framework, un très bon exemple de DSL sur Swift, qui s'y prête particulièrement bien.

```swift {linenos=inline}
struct AddEditPersonSheet: View {

		// Local state
    @State
    private var tappedOkButton = false
		
	  // The ViewModel data and callbacks / Yeah I should have created a struct for this
    @Binding
    var editingPerson: EditingPerson
    let isCreation: Bool
    var cancelAddPerson: () -> Void
    var endAddOrEditPerson: () -> Void
    
		// No constructor (structs are cool)
		
    private var title: String {
        isCreation ? "Créer" : "Modifier"
    }
    
    private func isEmpty(_ str: String) -> Bool {
        str.trimmingCharacters(in: .whitespacesAndNewlines).isEmpty
    }
    
    var body: some View {
        NavigationView {
            Form {ss
                Section(header: Text("Identité")) {
                    if tappedOkButton && isEmpty(editingPerson.firstName) {
                        Text("Prénom manquant").foregroundColor(.red)
                    }
                    TextField("Prénom", text: $editingPerson.firstName).textContentType(.givenName)
                    if tappedOkButton && isEmpty(editingPerson.lastName) {
                        Text("Nom manquant").foregroundColor(.red)
                    }
                    TextField("Nom", text: $editingPerson.lastName).textContentType(.familyName)
                }
               // ... other sections
            }
            .navigationBarItems(leading: Button(action: cancelAddPerson) {
                Text("Annuler")
            }, trailing: Button(action: {
                withAnimation {
                    tappedOkButton = true
                    if (editingPerson.isValid) {
                        endAddOrEditPerson()
                    }
                }
            }) {
                Text("Enregistrer")
            }.disabled(tappedOkButton && editingPerson.isValid))
            .navigationBarTitle(Text(title), displayMode: .inline)
        }
    }
}
```
*Pas de critiques les puristes, j'ai fait du monolingue et du gros inline volontairement.*

***Développement rapide :*** Grâce à cette approche DSL, on se retrouve avec du code à l'imbrication proche de l'UI, facilement intelligible, avec de très bons comportements par défaut. Comme c'était mon premier test j'ai forcément un peu ramé, mais j'aurais pris bien plus de 3 jours de développement pour une petite app complète, fonctionnelle et propre si j'avais dû rapprendre UIKit ou pire : HTML + CSS. 

***Le couplage avec [Combine](https://developer.apple.com/documentation/combine)*** permet de s'engager sur le chemin des state-driven apps. Pour avoir pas mal joué avec [React](https://fr.reactjs.org/) + [Redux](https://redux.js.org/), je ne peux que vous inciter à adopter ce paradigme. Evidemment, quand on n'a jamais fait que de la programmation impérative, beaucoup de petites choses peuvent être frustrantes au premier abord. Mais ça dégraisse tellement ! 
Et pour ceux qui ont déjà eu des interrogations philosophiques sur les architectures logicielles en iOS - entre MVC = Massive View Controller par exemple) et le trop souvent overkill [VIPER](https://mutualmobile.com/resources/meet-viper-fast-agile-non-lethal-ios-architecture-framework) -, SwiftUI + Combine incitent très naturellement à dérouler le code en MVVM, apportant enfin une alternative moderne et structurante.

***RIP UIKit ? :*** Bien sûr que non, SwiftUI est principalement une surcouche d'UIKit qui a encore de beaux jours devant lui. On peut d'ailleurs palier assez facilement l'absence de nombreux composants essentiels de SwiftUI en encapsulant une `UIView`, comme j'ai dû le faire pour le lecteur PDF et la webview qui appelle le code de génération du document PDF.

C'est tout, ce fut un bon petit défi sympa et enrichissant !

J'espère avoir l'occasion de récrire sur Swift qui reste un de mes langages préférés, mais pour le moment je replonge dans mes projets TypeScript qui se positionne franchement pas mal non plus et a l'avantage d'être largement adopté en dehors du petit monde Apple.
