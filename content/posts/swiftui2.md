+++
draft = false
date = 2020-12-10T14:14:35+01:00
title = "SwiftUI & Combine Feedback #2 : property wrapper pour UserDefaults et @Published"
description = ""
slug = "swiftui_combine2"
authors = ["Pitt"]
tags = ["iOS", "Combine", "SwiftUI"]
categories = ["Application mobile"]
externalLink = ""
series = ["SwiftUI/Combine Feedback"]
+++

J'avais déjà croisé un exemple d'implémentation de property wrapper et devant leur simplicité, je m'étais mis en tête de créer un property wrapper accélérant le stockage de valeurs dans les `UserDefaults`. 

Il se trouve que dans mes expérimentations autour de `Combine` j'avais bien envie de créer un wrapper publiant les nouvelles valeurs seulement après les avoirs stockées ([voir ici](/posts/swiftui_combine1)).

Et puis j'ai finalement voulu faire tout en même temps !

## Qu'est-ce donc qu'un property wrapper ?
Un property wrapper très simple s'écrit de cette manière :

```swift
@propertyWrapper
public class Print<Value> {ss
    private var val: Value
    
    public init(wrappedValue value: Value) {
        val = value
    }
    
    public var wrappedValue: Value {
        set {
            print("Setting '\(val)'")
            val = newValue
        }
        get { 
            print("Returning '\(val)'")
            return val 
        }
    }
}
```

et s'utilise ensuite ainsi :

```swift
class SomeClass {
 @Print
 var myString: String = "Coucou copaing !"
 
 func doThings() {
	myString = "Bye friend"
	var something = myString
 }
}

SomeClass().doThings()
```

ce qui imprimera

```
> Setting 'Bye friend'
> Getting 'Bye friend'
```

ce qui est plutôt nul.

Ceci-dit, si on décidait de faire des choses plus intéressantes que des print, on pourrait bien se faciliter la vie !

## Got UserDefaults ?

### Stocker des propriétés dans les UserDefaults

Voilà un usage fréquent et intéressant.

```swift
@propertyWrapper
public struct UserDefaultsBacked<Value> {
    private let key: String
    private let defaultValue: Value
    private let storage: UserDefaults
    
    public init(wrappedValue value: Value, _ key: String, storage: UserDefaults = .standard) {
        defaultValue = value
        self.key = key
				self.storage = storage
    }
    
    public var wrappedValue: Value {
        get { storage.value(forKey: key) as? Value ?? defaultValue}
        set { storage.setValue(newValue, forKey: key) }
    }
}
```

Et une implémentation un peu naïve comme celle ci-dessus peut faire l'affaire.

**Un petit warning** quand même, on n'oublie pas que chaque *setValue* sur une clé de `UserDefaults` réécrit le dictionnaire complet. On utilisera donc tout ça en connaissance de cause, et n'oublions pas qu'on peut aussi réduire leur taille en n'utilisant pas systématiquement le `.standard`.

On peut noter la présence de nouveaux arguments du constructeur, la `key` et le `storage`. Ils peuvent être fournis via la déclaration de l'annotation, ou doivent l'être pour ceux qui n'ont pas de valeur par défaut comme la key.

```swift
@UserDefaultsBacked("int-key") // Smells like Java
var someInt = 8

@UserDefaultsBacked("my-data", storage: UserDefaults("DBName"))
var someData: Data? = nil

struct SomeStruct {
 var prop = "Yup"
}

@UserDefaultsBacked("my-struct")
var myStruct = SomeStruct() // Wait... What ?

```

### Stocker uniquement les types compatibles

Ce wrapper fonctionnera très bien avec tous les types supportés par les `UserDefaults` mais attendez-vous à de bons crashes pour tous les autres cas, comme `SomeStruct`.

Restreignons-donc déjà l'usage aux bonnes valeurs grâce à un *flag protocol* :

```swift
public protocol UserDefaultsStorable {}

extension String : UserDefaultsStorable {}
extension Int: UserDefaultsStorable {}
extension Double: UserDefaultsStorable {}
extension Float: UserDefaultsStorable {}
extension Date: UserDefaultsStorable {}
extension Data: UserDefaultsStorable {}
extension Array: UserDefaultsStorable where Element: UserDefaultsStorable {}
extension Dictionary: UserDefaultsStorable where Key == String, Value: UserDefaultsStorable {}

@propertyWrapper
public struct UserDefaultsBacked<Value> {
    private let key: String
    private let defaultValue: Value
    private let storage: UserDefaults
    
    public init(wrappedValue value: Value, key: String, storage: UserDefaults = .standard) where Value: UserDefaultsStorable {
        defaultValue = value
        self.key = key
        self.storage = storage
    }
    
    public var wrappedValue: Value {
        get { storage.value(forKey: key) as? Value ?? defaultValue}
        set { storage.setValue(newValue, forKey: key) }
    }
}
```

Ok, on a maintenant un property wrapper sélectif et une belle erreur de compilation dans le cas où le type ne se conforme pas à `UserDefaultsStorable`.

```swift
@UserDefaultsBacked("my-struct")
var myStruct = SomeStruct()
// Error: Initializer 'init(wrappedValue:_:storage)' requires that 'SomeStruct' conform to 'UserDefaultsStorable'
```

### Stocker les Codables

Bien mais `SomeStruct` n'est pas bien mystérieux, ce serait bien sympa de pouvoir le stocker aussi. En fait, tout codable est a priori stockable puisque `Data` l'est. Seulement pour éviter de se faire la conversion à chaque accès, profitons donc de notre wrapper.

Généralisons : on peut avoir à mapper les données dans un sens et dans l'autre pour pouvoir les stocker. L'interface de `UserDefaults` utilise le très générique `Any?`, on va donc définir les mappers :

```swift
typealias StoringMapper<Value> = (Value) -> Any?
typealias StoreReadingMapper<Value> = (Any?) -> Value?
```

Puis:
- définir des propriétés de ce type dans notre wrapper.
- définir un constructeur privé aveugle qui prend de bonne fois n'importe quel type de propriété avec des mappers
- définir des constructeurs publics stricts sur le type qui vont fournir des mappers au constructeur privé

```swift
@propertyWrapper
public class UserDefaultsBacked<Value> {
    private let key: String
    private let storage: UserDefaults
    private let defaultValue: Value
    private var storingMapper: (Value) -> Any?
    private var storeReadingMapper: (Any?) -> Value?
    
    private init(wrappedValue value: Value,
                 _ key: String ,
                 storage: UserDefaults,
                 storingMapper: @escaping StoringMapper<Value>,
                 storeReadingMapper: @escaping StoreReadingMapper<Value>) {
        var initValue = value
        if let storedValue = storeReadingMapper(storage.object(forKey: key)) {
           initValue = storedValue
        }
        defaultValue = initValue
        self.key = key
        self.storage = storage
        self.storingMapper = storingMapper
        self.storeReadingMapper = storeReadingMapper
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value: UserDefaultsStorable {
        self.init(wrappedValue: value,
                  key, storage: storage,
                  storingMapper: {$0},
                  storeReadingMapper: { $0 as? Value})
    }
    
    
    
    public var wrappedValue: Value {
        get { storage.value(forKey: key) as? Value ?? defaultValue}
        set { storage.setValue(newValue, forKey: key) }
    }
}
```

Mais pour mes `Codable`s les mappers ne sont pas si évidents.

```swift
public extension Encodable {
    func mapForStorage() -> Any? {
        do {
            return try JSONEncoder().encode(self)
        } catch {
            print("Couldn't encode \(self)", error)
            return nil
        }
    }
}
```

Ci-dessus la fonction d'encodage, avec absolument pas de check sur le cas où l'objet est également stockable nativement, car la discrimination s'effectuera en amont. Bon, c'est l'équivalent de `try? JSONEncoder().encode(self)` mais avec une trace pour ne pas être complètement dans le brouillard en cas de problème. Evidemment on préfèrera certainement donner de meilleures options de traçage, et certains se sentent mal (à raison) de ne pas `throw` quoi que ce soit, mais ce n'est pas le débat ici. Et puis comme on dit parfois : "Quand tout va bien, tout va bien !".

```swift
/// Simple type opening using a static function to allow JSON decoding with erased types conforming to `Decodable` protocol
fileprivate extension Decodable {
    static func openedJSONDecode(using decoder: JSONDecoder, from data: Data) throws -> Self {
        return try decoder.decode(self, from: data)
    }
}
```
*Joyeusement pompé de [ce post SO](https://stackoverflow.com/questions/54963038/codable-conformance-with-erased-types)*

Tiens je parlais dans mon poste précédent de type erasure, mais saviez vous que l'inverse est le *type opening* ? Eh bien pareil, je me suis couché moins bête. Et pour tous ceux qui ont déjà joué avec un `JSONDecoder` dans des contextes génériques un peu poussés, la feinte ci-dessus est plutôt cool à retenir : une fonction statique a toujours accès au type concret, ce qui satisfait le compilo.

```swift
public extension Decodable {
    static func mapOutOfStorage(_ value: Any?) -> Self? {
        guard let data = value as? Data else { return nil }
        do {
            return try Self.openedJSONDecode(using: JSONDecoder(), from: data)
        } catch {
            print("Couldn't decode \(String(describing: value))", error)
            // Very opinionated choice to ignore thrown errors
            return nil
        }
    }
}
```

Et voilà, sur tout type se conformant à `Decodable` on a désormais cette fonction `mapOutOfStorage`.

Je rappelle que `typealias Codable = Decodable & Encodable`, donc pour tout type `Codable` on a nos deux mappers \o/

Rajoutons donc ce petit constructeur à notre property wrapper `UserDefaultsBacked` :

```swift
    convenience init(wrappedValue value: Value,
                            _ key: String,
                            storage: UserDefaults = .standard) where Value : Codable {
        self.init(wrappedValue: value,
                  key,
                  storage: storage,
                  storingMapper: Value.mapForStorage,
                  storeReadingMapper: Value.mapOutOfStorage)
    }
```

Nice, et maintenant qu'est ce qui se passe si je déclare quelque chose comme :

```
@UserDefaultsBacked("myKey")
var myInt = 0
// Error: Ambiguous use of 'init(wrappedValue:_:storage:sendAfterStore:)'
```

Eh bien les `Int` étant `Codable` mais aussi `UserDefaultsStorable`, le compilateur ne saura quel constructeur choisir. Deux solutions : enlever l'ambiguité en changeant la signature d'un des constructeurs (ce qui est un peu minable, même simplement d'y avoir pensé), ou donner un constructeur qui match encore mieux, avec `where Value : Codable & UserDefaultsStorable`. Encore un bon trick, vous êtes bienvenus.

### Checkpoint

Voici un petit bilan :

```swift
/// Any type implementing this protocol can be stored natively in UserDefaults
public protocol UserDefaultsStorable {}

/**
 Declare proper flag protocol conformance for all types natively compatible with UserDefaults storage
*/
extension String : UserDefaultsStorable {}
extension Int: UserDefaultsStorable {}
extension Double: UserDefaultsStorable {}
extension Float: UserDefaultsStorable {}
extension Date: UserDefaultsStorable {}
extension Data: UserDefaultsStorable {}
extension Array: UserDefaultsStorable where Element: UserDefaultsStorable {}
extension Dictionary: UserDefaultsStorable where Key == String, Value: UserDefaultsStorable {}

/// `Encodable` mapping for storage
public extension Encodable {
    func mapForStorage() -> Any? {
        do {
            return try JSONEncoder().encode(self)
        } catch {
            print("Couldn't encode \(self)", error)
            return nil
        }
    }
}

/// Simple type opening using a static function to allow JSON decoding with erased types conforming to `Decodable` protocol
fileprivate extension Decodable {
    // Useful trick from https://stackoverflow.com/questions/54963038/codable-conformance-with-erased-types
    static func openedJSONDecode(using decoder: JSONDecoder, from data: Data) throws -> Self {
        return try decoder.decode(self, from: data)
    }
}

/// `Decodable` mapping for reading from storage
public extension Decodable {
    static func mapOutOfStorage(_ value: Any?) -> Self? {
        guard let data = value as? Data else { return nil }
        do {
            return try Self.openedJSONDecode(using: JSONDecoder(), from: data)
        } catch {
            print("Couldn't decode \(String(describing: value))", error)
            // Very opinionated choice to almost ignore thrown errors
            return nil
        }
    }
}

@propertyWrapper
public class UserDefaultsBacked<Value> {
    private let key: String
    private let storage: UserDefaults
    private let defaultValue: Value
    private var storingMapper: (Value) -> Any?
    private var storeReadingMapper: (Any?) -> Value?
    
    private init(wrappedValue value: Value,
                 _ key: String,
                 storage: UserDefaults,
                 storingMapper: @escaping StoringMapper<Value>,
                 storeReadingMapper: @escaping StoreReadingMapper<Value>) {
        defaultValue = value
        self.key = key
        self.storage = storage
        self.storingMapper = storingMapper
        self.storeReadingMapper = storeReadingMapper
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String,
                            storage: UserDefaults = .standard) where Value: UserDefaultsStorable {
        self.init(wrappedValue: value,
                  key,
                  storage: storage,
                  storingMapper: {$0},
                  storeReadingMapper: { $0 as? Value})
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value : Codable {
        self.init(wrappedValue: value,
                  key,
                  storage: storage,
                  storingMapper: Value.mapForStorage,
                  storeReadingMapper: Value.mapOutOfStorage)
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value : Codable & UserDefaultsStorable {
        self.init(wrappedValue: value,
                  key,
                  storage: storage,
                  storingMapper: {$0},
                  storeReadingMapper: {$0 as? Value})
    }
    
    public var wrappedValue: Value {
        get { storage.value(forKey: key) as? Value ?? defaultValue}
        set { storage.setValue(newValue, forKey: key) }
    }
}
```

*Fun fact* : j'ai retrouvé le même flag protocol sur un post (je l'ai plus sous la main là), puis vu des implémentations proches pour les `Codable`, mais tout ça bien sûr après avoir réinventé la roue. Et même si on dit souvent ça péjorativement, dans une démarche d'apprentissage ça a tout son sens de regarder les solutions seulement ensuite. Et en plus c'était vachement moins bien fait, [genre là](https://swiftsenpai.com/swift/create-the-perfect-userdefaults-wrapper-using-property-wrapper/) pas de type checking, aucune cohabitation entre les `Codable` et les types natifs... Nan mais jvous jure...

Et puis de toute façon je voulais aussi m'occuper de faire ...

## Un publisher un peu custom

Lorsqu'on utilise SwiftUI et Combine simplement, on va naturellement devoir taper des expression comme `$myState` ou `$myObject.prop`. Une fois passée mon aversion forte pour le PHP, j'ai creusé rapidement pour constater que ce n'était qu'une syntaxe un peu flippante pour accéder à la valeur projetée d'une wrapped property.

### Projected value ?

En bref, n'importe quel property wrapper peut déclarer une `var projectedValue: SomeType {...}` dont le type n'est pas nécessairement le même que celui de sa `wrappedValue`. Et cette `projectedValue` est accessible grâce au dollar américain (comme tellement de choses).
```swift
@propertyWrapper
public class Print<Value> {
    private var val: Value
    
    public init(wrappedValue value: Value) {
        val = value
    }
    
    public var wrappedValue: Value {
        set {
            print("Setting '\(val)'")
            val = newValue
        }
        get { 
            print("Returning '\(val)'")
            return val 
        }
    }
		
    public var projectedValue: Int {
        print("42")
        return 8
    }
}

@Print
var myString: String = "5"

myString = "40"
print("Coucou \($myString)")
```

printera donc

```
> Setting '40'
> 42
> Coucou 8
```

Aaah, je pense que j'ai fait le pire exemple qui soit, "service !" comme on dit dans l'est.

Bref, ce `$` n'a en théorie pas forcément grand chose à voir avec Combine excepté qu'on l'y utilise en permanence. 

### Quelques notions de Combine

Typiquement, le property wrapper `Published` a pour type projeté **la struct** `Published<Value>.Publisher` ([doc](https://developer.apple.com/documentation/combine/published/publisher)) qui respecte notamment le **protocole** `Publisher` (et bien plus, [doc](https://developer.apple.com/documentation/combine/publisher)), et peut donc envoyer des `Value` à des `Subscriber`. 

Donc quand j'accède à un `@Published var myString: String` via `$myString` j'obtiens en gros une propriété dont je peux écouter les valeurs successives.

Et quand je fais du SwiftUI ainsi : `.sheet(isPresented: $vm.router.showSheet, ...)`, je passe donc un `Publisher` à la fonction `sheet` qui se fera un plaisir d'écouter si on doit où non présenter cette `sheet`.

**Rappel :** lorsqu'on utilise `sink` pour écouter les valeurs d'une var `@Published` via son `Publisher`, on reçoit la nouvelle valeur alors que la `wrappedValue` est encore l'ancienne valeur. Et je voudrais contourner ça dans certains cas (voir [mon post précédent](/posts/swiftui_combine1)).

Maintenant, ce qui m'intéresserait ce serait d'avoir ça, un `Publisher` que je pourrais contrôler finement pour lui envoyer des valeurs.
Eh bien Combine nous fournit gracieusement le protocole `Subject` qui hérite de `Publisher` et qui présente de surcroit une fonction `send(_:)` permettant d'envoyer des valeurs à publier. Ses implémentations sont :

- `CurrentValueSubject` qui détient une valeur courante
- `PassthroughSubject` qui au contraire ne retient rien

On devrait s'en sortir avec ça !

### DidSet Publisher property wrapper

Pour reproduire le comportement de `@Published` on pourrait écrire comme ça comme ça.

```swift
@propertyWrapper
public class BasicPublished<Value> {
    private let subject: CurrentValueSubject<Value, Never>
    
    public init(wrappedValue value: Value) {
        subject = CurrentValueSubject(value)
        wrappedValue = value
    }
    
    public var wrappedValue: Value {
        set {
            subject.send(newValue)
        }
        get { subject.value }
    }
    
    public var projectedValue: CurrentValueSubject<Value, Never> {
        get { subject }
    }
}
```

Le problème étant que lorsque le `send` va provoquer l'exécution de toutes les closures des subscribers, `subject.value` aura toujours l'ancienne valeur.

Assez simple **du coup** d'y remédier :

```swift
@propertyWrapper
public class DidSetPublished<Value> {
    private var val: Value
    private let subject: CurrentValueSubject<Value, Never>
    
    public init(wrappedValue value: Value) {
        val = value
        subject = CurrentValueSubject(value)
        wrappedValue = value
    }
    
    public var wrappedValue: Value {
        set {
            val = newValue
            subject.send(val)
        }
        get { subject.value }
    }
    
    public var projectedValue: CurrentValueSubject<Value, Never> {
        get { subject }
    }
}
```
(je crois que cette implémentation [vient de là](https://stackoverflow.com/questions/58403338/is-there-an-alternative-to-combines-published-that-signals-a-value-change-afte), et je crois aussi qu'il serait plus pertinent d'utiliser `PassthroughSubject` *a priori*)

## Tout ensemble

Vous le saviez, mon but ultime était ~la conquête de la Suède en lama~ de combiner tout ça. Pas juste pour le fun, mais parce que dans mon archi, à un moment, j'avais besoin d'une propriété `Codable` (un enum) stockée dans les `UserDefaults` et qui ne publierait son changement de valeur qu'après l'avoir affecté à sa `wrappedValue`.

Et puis d'autres besoins avec des variations : pas de stockage mais publication après affectation, stockage natif mais publication avant affectation...

J'aurais bien pu contourner tout ça, ou encore essayer de faire fonctionner des [wrappers imbriqués](https://noahgilmore.com/blog/nesting-property-wrappers/), mais je voulais me frotter à cette implémentation spécifique.

Et voilà, je vous colle juste l'ensemble là dessous, chaque détail étant expliqué dans les parties précédente (enfin faut savoir quelques trucs en amont quand même, oui).

N'oubliez pas que la perf n'est **pas** l'objectif de ce wrapper (du tout). 

Je vous suggère d'ailleurs de jeter un oeil à l'implémentation de `Published` [chez OpenCombine](https://github.com/OpenCombine/OpenCombine/blob/7286336b28585a594bb4769a680f6f03e375b6ba/Sources/OpenCombine/Published.swift), qui est particulièrement élégante (ya un enum <3).

```swift
//
//  SmartPublished.swift
//  attestation
//
//  Created by Pierre Mardon on 01/01/1970. Trust me.
//
import Foundation
import Combine

/// We need functions to map values before storing them to user defaults
fileprivate typealias StoringMapper<Value> = (Value) -> Any?
/// We need functions to map values read from user defaults storage to an expected type
fileprivate typealias StoreReadingMapper<Value> = (Any?) -> Value?

/// Any type implementing this protocol can be stored natively in UserDefaults
public protocol UserDefaultsStorable {}

/**
 Declare proper flag protocol conformance for all types natively compatible with UserDefaults storage
 */

extension String : UserDefaultsStorable {}
extension Int: UserDefaultsStorable {}
extension Double: UserDefaultsStorable {}
extension Float: UserDefaultsStorable {}
extension Date: UserDefaultsStorable {}
extension Data: UserDefaultsStorable {}
extension Array: UserDefaultsStorable where Element: UserDefaultsStorable {}
extension Dictionary: UserDefaultsStorable where Key == String, Value: UserDefaultsStorable {}

/// `Encodable` mapping for storage
public extension Encodable {
    func mapForStorage() -> Any? {
        do {
            return try JSONEncoder().encode(self)
        } catch {
            print("Couldn't encode \(self)", error)
            return nil
        }
    }
}

/// Simple type opening using a static function to allow JSON decoding with erased types conforming to `Decodable` protocol
fileprivate extension Decodable {
    // Useful trick from https://stackoverflow.com/questions/54963038/codable-conformance-with-erased-types
    static func openedJSONDecode(using decoder: JSONDecoder, from data: Data) throws -> Self {
        return try decoder.decode(self, from: data)
    }
}

/// `Decodable` mapping for reading from storage
public extension Decodable {
    static func mapOutOfStorage(_ value: Any?) -> Self? {
        guard let data = value as? Data else { return nil }
        do {
            return try Self.openedJSONDecode(using: JSONDecoder(), from: data)
        } catch {
            print("Couldn't decode \(String(describing: value))", error)
            // Very opinionated choice to almost ignore thrown errors
            return nil
        }
    }
}

/**
 Property wrapper that provides some common use cases options.
 
 Do NOT use for heavy performance demanding components.
 
 - `UserDefaults` storage is activated when a `key` is provided for all natively handled types and `Codable` ones
 - option `sendAfterStore` makes the subject send the value only after it has effectively been affected to the property itself: WARNING this is not recommended for UI bindings. Disabled by default.
 
 Usage:
 
 \```
 
 @SmartPublished("someKey")
 var myProp = "Bonjoir !"
 \```
 The string property will be backed in UserDefaults.standard for the key "someKey". It will be effectively stored only if the value of the var is affected after its initialization, until then UserDefaults entry will stay
 untouched.
 The initial value of the property will be `"Bonjoir !"` if there's not value in the store.
 
 \```
 @SmartPublished("myCodableValueUserDefaultsKey")
 var myProp = someValueOfCodableType
 \```
 
 The codables are stored the same way except they are JSON encoded if they're not natively handled by UserDefaults.
 
 \```
 @SmartPublished(sendAfterStore = true)
 var myProp = 8
 
 $myProp.sink {
 print("property: \(myProp), received: \($0)")
 }
 
 myProp = 1
 
 \```
 
 Will print `property: 1, received: 1`, while with `@Published` or `sendAfterStore = false` it would be `property: 8, received: 1`.
 It is not recommended to use this `sendAfterStore = true` for UI-bound properties.
 
 \```
 @SmartPublished
 \```
 
 Why would you do that, just use `@Published` then!
 
 */
@propertyWrapper
public class SmartPublished<Value> {
    private let key: String?
    private let storage: UserDefaults
    private let sendAfterStore: Bool
    private let subject: CurrentValueSubject<Value, Never>
    private var val: Value
    private var storingMapper: (Value) -> Any?
    private var storeReadingMapper: (Any?) -> Value?
    
    private init(wrappedValue value: Value,
                 _ key: String? ,
                 storage: UserDefaults,
                 sendAfterStore: Bool,
                 storingMapper: @escaping StoringMapper<Value>,
                 storeReadingMapper: @escaping StoreReadingMapper<Value>) {
        var initValue = value
        if let key = key, let storedValue = storeReadingMapper(storage.object(forKey: key)) {
            initValue = storedValue
        }
        val = initValue
        subject = CurrentValueSubject(initValue)
        self.key = key
        self.storage = storage
        self.sendAfterStore = sendAfterStore
        self.storingMapper = storingMapper
        self.storeReadingMapper = storeReadingMapper
        subject.send(val)
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String? = nil,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value: UserDefaultsStorable {
        self.init(wrappedValue: value,
                  key, storage: storage,
                  sendAfterStore: sendAfterStore,
                  storingMapper: {$0},
                  storeReadingMapper: { $0 as? Value})
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String?,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value : Codable {
        self.init(wrappedValue: value,
                  key,
                  storage: storage,
                  sendAfterStore: sendAfterStore,
                  storingMapper: {$0.mapForStorage()},
                  storeReadingMapper: Value.mapOutOfStorage)
    }
    
    public convenience init(wrappedValue value: Value,
                            _ key: String?,
                            storage: UserDefaults = .standard,
                            sendAfterStore: Bool = false) where Value : Codable & UserDefaultsStorable {
        self.init(wrappedValue: value,
                  key, storage: storage,
                  sendAfterStore: sendAfterStore,
                  storingMapper: {$0},
                  storeReadingMapper: { $0 as? Value})
    }
    
    public convenience init(wrappedValue value: Value, sendAfterStore: Bool = false) {
        self.init(wrappedValue: value,
                  nil,
                  storage: .standard,
                  sendAfterStore: sendAfterStore,
                  storingMapper: {$0},
                  storeReadingMapper: {$0 as? Value})
    }
    
    public var wrappedValue: Value {
        set {
            if sendAfterStore {
                subject.send(newValue)
            }
            if let key = self.key {
                storage.setValue(storingMapper(newValue), forKey: key)
            } else {
                val = newValue
            }
            if !sendAfterStore {
                subject.send(newValue)
            }
        }
        get {
            if let key = self.key {
                return storeReadingMapper(storage.value(forKey: key)) ?? val
            }
            return val
        }
    }
    
    public var projectedValue: CurrentValueSubject<Value, Never> {
        get { subject }
    }
}
```

N'hésitez pas à me contacter pour toute remarque, insulte ou éloge, mon email est dans le footer ;)