+++
draft = false
date = 2020-12-09T14:14:35+01:00
title = "SwiftUI & Combine Feedback #1 : architecture et grains de sable"
description = ""
slug = "swiftui_combine1"
authors = ["Pitt"]
tags = ["iOS", "Combine", "SwiftUI"]
categories = ["Application mobile"]
externalLink = ""
series = ["SwiftUI/Combine Feedback"]
+++

Après [mon premier petit TP](/posts/attestation-ios) autour de SwiftUI et Combine pour générer mes attestations de déplacement ~à la barbe de la maréchaussée~ à la volée voire en retard, j'ai profité de l'adaptation au second format d'attestation pour faire des explorations un peu plus poussées de mon architecture autour de Combine.

J'en sors une petite liste de considérations techniques que j'espère d'intérêt, et voici les premières !


# Architecture: MVVM+

Pour une petite app comme celle-ci, je m'autorise des entorses à nombre de principes stricts *overkill* que je n'estime pas pertinents ici, avec des gains principalement en concision et lisibilité.
Le [MVVM](https://fr.wikipedia.org/wiki/Mod%C3%A8le-vue-vue_mod%C3%A8le) est tout à fait indiqué pour un cloisonnement minimal, et en l'occurence ça ressemblait à ça

![Image description](/images/swiftui1/MVVM.png)

Avec des injection par construction et donc cette instanciation initiale :

```
let store = Store(context: moContext)
let model = MainViewModel(store: store, router: Router())
return MainView(model: model)
```

On peut remarquer le petit `Router` qui s'est avéré fort utile pour éviter trop de plomberie. Son rôle est juste de publier des propriétés destinées à contrôler et rendre compte de la navigation. Il ne fait donc clairement pas partie du modèle ni des vues, et j'ai du mal à le considérer comme un modèle de vue étant donnée sa nature transverse.

Tant qu'il ne dépasse pas ce rôle de navigation, ne stocke qu'un minimum de données transitoires au besoin (immuables de préférence, la struct d'une personne à éditer par exemple), ça reste très lisible et on évite les chaînages de `@Published` orthodoxes.

```swift
enum ActiveSheet {
    case attestationPresentation(person: PersonStruct), addPerson, edit(person: PersonStruct)
}

enum ActiveAlert {
    case confirmAttestation, detail(reason: Reason)
}

class Router: ObservableObject {
    
    @Published
    var showSheet = false
    @Published
    var showAlert = false
    
    private(set) var activeSheet = ActiveSheet.addPerson
    private(set) var activeAlert = ActiveAlert.confirmAttestation
    
    func showAttestationCreationAlert() {
        activeAlert = .confirmAttestation
        showAlert = true
    }
    
    func startAddPerson() {
        activeSheet = .addPerson
        showSheet = true
    }
    
    func startEdit(person: PersonStruct) {
        activeSheet = .edit(person: person)
        showSheet = true
    }
    
    func showReasonDetail(_ reason: Reason) {
        activeAlert = .detail(reason: reason)
        showAlert = true
    }
    
    func showAttestationView(person: PersonStruct) {
        activeSheet = .attestationPresentation(person: person)
        showSheet = true
    }
    
    func closeSheet() {
        showSheet = false
    }
}
```

Plutôt concis, ça vaut clairement le coup plutôt que de perdre ces quelques variables dans des chemins trop tortueux.

Tous mes proches le savent, les enums Swift [c'est ma grande passion](https://www.dailymotion.com/video/x2xl6r8). Et ceux du petit routeur ci-dessus me permettent de faire des fonctions SwiftUI bien compactes :

```swift
    func alert() -> Alert {
        switch vm.router.activeAlert {
        case .confirmAttestation:
            return Alert(title: coldFeetTitle,
                         message: coldFeetMessage,
                         primaryButton: .default(Text("Je certifie")) {
                vm.generateNewAttestation()
            }, secondaryButton: .cancel(Text("Annuler")))
        case .detail(reason: let reason):
            return Alert(title: Text(reason.niceString),
                         message: Text(reason.detail),
                         dismissButton: .default(Text("Ok")))
        }
    }
    
    func sheet() -> AnyView {
        switch vm.router.activeSheet {
        case .attestationPresentation(let person):
            return AnyView(AttestationView(vm: vm.attestationViewModel(person: person)))
        case .addPerson:
            return AnyView(AddOrEditPersonSheet(vm: vm.addPersonViewModel))
        case .edit(let person):
            return AnyView(AddOrEditPersonSheet(vm: vm.editPersonViewModel(person: person)))
        }
    }
```

Et enfin, le `body` de ma `View` principale sera très concis :

```swift
    var body: some View {
        NavigationView {
            VStack(alignment: .center, spacing: 5) {
                MainListsView(model: vm.mainListsViewModel).environment(\.editMode, $editMode)
                BottomMenu(model: vm.bottomMenuViewModel)
            }
            .sheet(isPresented: $vm.router.showSheet, onDismiss: {
                vm.checkShouldShowPinnedAttestation()
            }, content: sheet)
						.navigationBarTitle("", displayMode: .inline)
            .navigationBarItems(leading: navigationBarLeadingItem, trailing: EditButton(editMode: $editMode))            
        }
        .alert(isPresented: $vm.router.showAlert, content: alert)
    }
```

([on peut comprendre](https://stackoverflow.com/questions/58837007/multiple-sheetispresented-doesnt-work-in-swiftui) d'ailleurs pourquoi je sépare le `showSheet` et `showAlert` des enums, au lieu de déclarer des `.none`)

La vue principale est la principale consommatrice du routeur, cependant de multiples vues viennent agir dessus.

Evidemment, cette architecture est adaptée à ce projet particulier, ne cherchez pas à reproduire ça à la maison.

# Les petits trucs pénibles
## Type erasure
Ci-dessus, vous pouvez voir que j'utilise `AnyView(...)` pour renvoyer un type consistant de `View`. Pour tous ceux qui ont joué un peu en profondeur avec les protocoles et génériques en Swift, on atteint vite des obstacles mystérieux particulièrement brainboiling.

Heureusement on observe un effort de type erasure dans les bibliothèques système avec ces `AnyView`, `AnyCancellable`...

![Complétion sur "Any" dans XCode](/images/swiftui1/Capture-decran-2020-12-09-a-15.35.03.png)

Ainsi que de nouveaux mots clé mystérieux comme `some` qui est la réponse directe à la sentence :

> Protocols 'WouldBeSoNice' can only be used as a generic constraint because it has Self or associated type requirements

Si ça vous intéresse je vous conseille [ce petit article](https://learnappmaking.com/some-swift-opaque-types-how-to/).

Ceci-dit, même si ça disparaît vite, je pense que c'est un frein assez considérable notamment pour des débutants.

## Observer des objets imbriqués

Un `ViewModel` en mode Combine doit avoir cette allure :

```swift
class MyViewModel : ObservableObject {
	@Published
	var someProperty = "Coucou copaing !"
}
```

Le wrapper `@Published` est tout à fait adaptée aux structs puisque toute mutation d'une struct est un changement de valeur. Mais les classes si elles sont faites pour être mutées ne remonteront point l'évènement au wrapper.

Or pour observer une propriété imbriquée au deuxième niveau dans SwiftUI, comme `.sheet(isPresented: $vm.router.showSheet) {…}`, on peut essayer :

- d'observer le routeur qui serait une **struct** et de prendre sa valeur `showSheet`
- avec le routeur en `ObservableObject`, observer directement `showSheet`

Eh bien aucune des deux options ne fonctionne directement depuis une vue SwiftUI.
Ce petit `$` qui désigne la `projectedValue` d'une propriété encapsulée par un `@Published` ou un `@State` n'est pas magique, et ça ne fonctionne qu'au premier niveau, c'est à dire un `@Published` propriété d'un `ObservableObject`.

Et la feinte officielle n'est pas bien glorieuse :

```swift
class MyViewModel : ObservableObject {
	@Published
	var router = Router
	
	private var cancellables = Set<AnyCancellable>()
	
	init(router: Router) {
        self.router = router
        router.objectWillChange.sink { [weak self] _ in
            self?.objectWillChange.send()
        }.store(in: &cancellables)
    }
		
	deinit {
        cancellables.forEach({ $0.cancel() })
    }
}
```

Il-y-a des variantes plus concises au prix de sacrifices discutables, mais voilà le principe. A chaque fois que le routeur va changer, on va propager l'évènement pour indiquer à l'UI de se rafraîchir. C'est un peu large, un peu "Mario fait du Combine", mais ne soyons pas obtus, si ça roule après tout...

Une bonne alternative est de mettre tout ça à plat dans la vue :

```swift
struct MyView: View {
    
    @ObservedObject
    private var vm: MainViewModel
		
		var body: some View {...}
}
```

deviendrait

```swift
struct MyView: View {
    
    @ObservedObject
    private var vm: MyViewModel
		
    @ObservedObject
    private var router: Router
		
		var body: some View {...}
}
```

C'est plus élégant je trouve, mais imaginons que j'ai 8 entités un peu complexes à embarquer dans mon VM, ça commence alors à foisonner plus que de raison.

Autre point en passant, impossible de sécuriser l'instance embarquée du routeur dans le view-model avec un `private(set)` modifier dans la première version. C'est de l'ordre du TOC - c'est bien d'en être conscient - mais ça me gène 😅

## Propager des @Published, Model -> VM -> View

```swift
class Store : ObservableObject {
	@Published
	private(set) var someUrl: URL?
}

class MyViewModel : ObservableObject {
	@Published
	private(set) var someUrl: URL?
	
	init(store: Store) {
		store.$someUrl.assign(to: &$someUrl)
	}
}
```

Mais c'est très raisonnable, super ! Oui mais iOS14+ seulement.

Et voici la version iOS 13 :

```swift
class Store : ObservableObject {
	@Published
	private(set) var someUrl: URL?
}

class MyViewModel : ObservableObject {
	@Published
	private(set) var someUrl: URL?
	
	private var cancellables = Set<AnyCancellable>()
	
	init(store: Store) {
		store.$someUrl.sink { [weak self] url in
			self?.someUrl = url
		}.store(in: &cancellables)
	}
	
	deinit {
        cancellables.forEach({ $0.cancel() })
    }
}
```

\* *whip sound* *

Pas mal hein ?

\* *whip sound* *

## Sink twice
*silence génant*

Quand on observe un sujet avec `sink`, sachez que la valeur qui vous est passée en closure est celle qui **va** être attribuée, comme lorsqu'on utilise `willSet` sur une propriété (quelques détails [ici](https://stackoverflow.com/questions/58403338/is-there-an-alternative-to-combines-published-that-signals-a-value-change-afte)). 

Pas de problème pour l'update d'UI, c'est fait pour. Mais pour les autres besoins, comme par exemple quand on a des mécanismes complexes intermédiaires qui ne se résument pas à fusionner deux valeurs publiées, la meilleure chose à faire est encore de créer son publisher.

Et même si j'aurais encore beaucoup à dire autour de ce sujet, je réserve à un futur article une petite contribution autour de ce sujet, des property wrappers et consorts.



N'hésitez pas à m'écrire si vous avez un avis quelconque sur ce que j'ai écrit, mon email est (?) dans le footer ;)