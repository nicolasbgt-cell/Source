# Le polymorphisme en Python — Guide complet

Référence sur le polymorphisme : ce que c'est, ses différentes formes en Python, et comment l'exploiter.

---

## 1. Qu'est-ce que le polymorphisme ?

Le polymorphisme (littéralement "plusieurs formes") désigne la capacité à **utiliser une même interface pour des objets de types différents**, sans que le code appelant ait besoin de savoir précisément à quel type il a affaire.

```python
class Dog:
    def speak(self):
        return "Woof"

class Cat:
    def speak(self):
        return "Meow"

for animal in [Dog(), Cat()]:
    print(animal.speak())   # même appel, comportement différent selon l'objet
```

La boucle ne fait aucun `if isinstance(animal, Dog)` — elle appelle juste `speak()`, et chaque objet sait comment se comporter. C'est le principe central : **le même code fonctionne pour des objets différents**, tant qu'ils respectent le même contrat (ici : posséder une méthode `speak()`).

Python propose plusieurs façons d'obtenir ce comportement, du plus souple (aucune contrainte formelle) au plus strict (contrat imposé par le langage).

---

## 2. Duck typing — le polymorphisme "informel"

Python ne vérifie **jamais** le type d'un objet avant d'appeler une méthode dessus — il essaie juste, et échoue à l'exécution si ça ne marche pas. C'est ce qu'on appelle le *duck typing* : *"si ça marche comme un canard et que ça cancane comme un canard, alors c'est un canard"*.

```python
class Duck:
    def quack(self):
        return "Quack!"

class Person:
    def quack(self):
        return "I'm quacking like a duck!"

def make_it_quack(thing):
    print(thing.quack())   # aucune vérification de type, juste un appel

make_it_quack(Duck())
make_it_quack(Person())   # fonctionne aussi, sans lien de parenté entre les deux classes
```

`Duck` et `Person` n'ont **aucune relation d'héritage** — elles ont juste, chacune de son côté, une méthode `quack()`. C'est la forme la plus permissive de polymorphisme en Python : seule compte la **forme** de l'objet (les méthodes qu'il possède), jamais sa généalogie.

---

## 3. Polymorphisme par héritage — redéfinition de méthode (overriding)

La forme la plus classique : une classe parente définit une méthode, chaque sous-classe la **redéfinit** avec son propre comportement.

```python
class Shape:
    def area(self) -> float:
        raise NotImplementedError

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self) -> float:
        return 3.14159 * self.radius ** 2

class Square(Shape):
    def __init__(self, side):
        self.side = side
    def area(self) -> float:
        return self.side ** 2

shapes: list[Shape] = [Circle(5), Square(4)]
for shape in shapes:
    print(shape.area())   # appelle la bonne version selon le type réel de l'objet
```

À l'exécution, Python appelle toujours la méthode `area()` définie sur la **classe réelle** de l'objet (`Circle` ou `Square`), jamais celle de `Shape`, même si la variable est annotée `Shape`. C'est ce mécanisme — la résolution de méthode au moment de l'appel, basée sur le type réel de l'objet — qui porte le nom de *dynamic dispatch*.

### `super()` pour étendre plutôt que remplacer

Une redéfinition peut réutiliser le comportement du parent au lieu de le dupliquer entièrement :

```python
class Shape:
    def describe(self) -> str:
        return "A generic shape"

class Circle(Shape):
    def describe(self) -> str:
        return super().describe() + " (a circle)"
```

---

## 4. Classes abstraites — imposer le contrat

Le duck typing et l'héritage simple n'obligent à rien : rien n'empêche une sous-classe d'oublier d'implémenter une méthode. Le module `abc` permet de **forcer** ce contrat.

```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...

class Circle(Shape):
    def __init__(self, radius):
        self.radius = radius
    def area(self) -> float:
        return 3.14159 * self.radius ** 2

Shape()    # TypeError : impossible d'instancier une classe abstraite
```

Si une sous-classe de `Shape` "oublie" d'implémenter `area()`, elle reste elle-même considérée comme abstraite et ne peut pas être instanciée — l'erreur est détectée **avant** même que le programme tourne vraiment, pas seulement au moment où on appellerait `area()` sur un objet mal formé.

---

## 5. `Protocol` — le contrat sans l'héritage

`Protocol` (module `typing`) formalise le duck typing : il définit une interface **sans obliger les classes qui la respectent à en hériter**.

```python
from typing import Protocol

class Speaker(Protocol):
    def speak(self) -> str: ...

class Dog:            # n'hérite PAS de Speaker
    def speak(self) -> str:
        return "Woof"

def announce(speaker: Speaker) -> None:
    print(speaker.speak())

announce(Dog())   # accepté, car Dog a bien une méthode speak() compatible
```

`Dog` est considérée compatible avec `Speaker` uniquement parce qu'elle possède une méthode `speak()` avec la bonne signature — aucun lien de parenté requis. C'est le duck typing du point 2, mais rendu **vérifiable par un outil de typage statique** (`mypy`), sans perdre la souplesse de ne pas exiger d'héritage.

| | `ABC` | `Protocol` |
|---|---|---|
| Lien avec l'interface | Héritage explicite obligatoire | Aucun héritage requis |
| Vérification | À l'instanciation (erreur si méthode manquante) | Seulement via un vérificateur de types statique |
| Cas d'usage | Famille de classes avec une vraie relation "est un" | Code indépendant qui n'a pas besoin de connaître une classe de base commune |

---

## 6. Surcharge d'opérateurs (operator overloading)

Le polymorphisme s'applique aussi aux **opérateurs** : `+`, `==`, `<`, `len()`, `str()`... sont eux-mêmes des appels de méthode déguisés, redéfinissables via les méthodes spéciales (*dunder methods*).

```python
class Vector:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __add__(self, other):
        return Vector(self.x + other.x, self.y + other.y)

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

    def __repr__(self):
        return f"Vector({self.x}, {self.y})"

    def __len__(self):
        return int((self.x ** 2 + self.y ** 2) ** 0.5)

v1 = Vector(1, 2)
v2 = Vector(3, 4)
v1 + v2        # appelle v1.__add__(v2) → Vector(4, 6)
v1 == v2       # appelle v1.__eq__(v2) → False
len(v1)        # appelle v1.__len__() → 2
```

`+` ne fait rien de magique : Python traduit `v1 + v2` en `v1.__add__(v2)`. C'est encore du polymorphisme — le même opérateur `+` se comporte différemment selon le type des objets (`int`, `str`, `list`, `Vector`...) parce que chaque type définit sa propre version de `__add__`.

| Méthode spéciale | Opérateur/fonction associé |
|---|---|
| `__add__` | `+` |
| `__eq__` | `==` |
| `__lt__` | `<` |
| `__len__` | `len(x)` |
| `__str__` | `str(x)` / `print(x)` |
| `__repr__` | représentation "développeur" (console, debug) |
| `__getitem__` | `x[i]` |
| `__iter__` | `for i in x` |
| `__call__` | `x()` — rend l'objet lui-même appelable |

---

## 7. Polymorphisme par fonction — `singledispatch`

Une fonction (pas une méthode de classe) peut changer d'implémentation selon le **type** de son argument, sans `if isinstance(...)` :

```python
from functools import singledispatch

@singledispatch
def render(value) -> str:
    return "unknown"

@render.register
def _(value: int) -> str:
    return f"number: {value}"

@render.register
def _(value: str) -> str:
    return f"text: {value}"

render(42)        # "number: 42"
render("hello")   # "text: hello"
```

C'est une forme de polymorphisme basée sur des **fonctions indépendantes par type**, plutôt que sur des méthodes de classes qui héritent d'une interface commune — utile quand on ne peut pas (ou ne veut pas) modifier les classes concernées pour leur ajouter une méthode commune.

---

## 8. Généricité (parametric polymorphism)

Une fonction ou une classe peut être écrite pour fonctionner avec **n'importe quel type**, sans redéfinir de version spécifique pour chacun — c'est le rôle des types génériques.

```python
from typing import TypeVar

T = TypeVar("T")

def first_element(items: list[T]) -> T:
    return items[0]

first_element([1, 2, 3])        # fonctionne avec des int
first_element(["a", "b"])       # fonctionne avec des str, sans réécrire la fonction
```

Contrairement aux formes précédentes (qui adaptent le *comportement* selon le type), la généricité garde un comportement **identique** quel que soit le type — seul le type manipulé change. C'est ce que font déjà nativement les collections natives (`list`, `dict`) : une même `list` fonctionne aussi bien avec des `int` qu'avec des `str`.

---

## Comparatif des différentes formes

| Forme | Contrat exigé | Vérifié... | Exemple typique |
|---|---|---|---|
| Duck typing | Aucun | jamais (échoue à l'exécution si absent) | fonctions génériques sur "n'importe quel objet avec la bonne méthode" |
| Héritage + overriding | Héritage d'une classe commune | à l'exécution | familles d'objets avec un comportement partagé mais spécialisé |
| `ABC` | Héritage + implémentation obligatoire | à l'instanciation | garantir qu'une interface est bien complète |
| `Protocol` | Aucun héritage, juste la bonne "forme" | par un vérificateur de types statique | plugins/extensions indépendants du code central |
| Surcharge d'opérateurs | Implémenter les bonnes méthodes spéciales | à l'exécution | objets qui doivent se comporter comme des types natifs |
| `singledispatch` | Aucun | à l'exécution, basé sur le type de l'argument | une fonction, plusieurs implémentations selon le type reçu |
| Généricité (`TypeVar`) | Aucun contrat sur le comportement | par un vérificateur de types statique | code qui traite n'importe quel type de la même façon |

## Ce qu'il faut retenir
- Le polymorphisme, c'est toujours la même idée : **le code appelant ignore le type exact** de ce qu'il manipule, il compte juste sur une interface commune.
- Python privilégie par défaut le duck typing (souple, non vérifié) ; `ABC` et `Protocol` ajoutent des degrés de contrainte différents quand on veut plus de garanties.
- Les méthodes spéciales (`__add__`, `__len__`...) sont la façon dont Python fait fonctionner ses propres opérateurs de façon polymorphe — et permettent à tes classes de s'intégrer aux mêmes opérateurs.
