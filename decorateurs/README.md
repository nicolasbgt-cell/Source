# Les décorateurs Python — Guide complet

## 1. Rappel : qu'est-ce qu'un décorateur ?

Un décorateur est une fonction qui **prend une fonction (ou une classe) en entrée et renvoie quelque chose à la place** — le plus souvent une version enrichie de l'original.

```python
def shout(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs).upper()
    return wrapper

@shout
def greet(name):
    return f"hello {name}"

greet("bob")   # "HELLO BOB"
```

`@shout` au-dessus de `greet` est un raccourci syntaxique pour `greet = shout(greet)`. C'est tout ce qu'un décorateur fait fondamentalement : **remplacer** une fonction par une autre, généralement une version qui appelle l'originale en ajoutant du comportement avant/après/autour.

---

## 2. Décorateurs natifs (built-in)

### `@staticmethod`
Méthode qui ne dépend d'aucune instance ni de la classe — une fonction "rangée" dans une classe par cohérence thématique.
```python
class MathTools:
    @staticmethod
    def add(a: int, b: int) -> int:
        return a + b

MathTools.add(2, 3)   # pas besoin d'instance
```

### `@classmethod`
Méthode qui reçoit la classe elle-même (`cls`) plutôt qu'une instance — typiquement pour des constructeurs alternatifs.
```python
class Pizza:
    def __init__(self, toppings):
        self.toppings = toppings

    @classmethod
    def margherita(cls):
        return cls(["tomato", "mozzarella"])

Pizza.margherita()   # crée une Pizza sans appeler __init__ directement avec la liste
```

### `@property`
Transforme une méthode en attribut : elle se lit **sans parenthèses**, comme un simple attribut, tout en exécutant du code à chaque accès.
```python
class Circle:
    def __init__(self, radius):
        self._radius = radius

    @property
    def area(self):
        return 3.14159 * self._radius ** 2

c = Circle(5)
c.area   # pas c.area() — se comporte comme un attribut, mais recalculé à chaque lecture
```
C'est la version "propre" de l'encapsulation vue dans Code Cultivation (`get_height()`) : au lieu d'une méthode `get_area()` qu'il faut appeler avec `()`, `@property` permet d'écrire `c.area` directement, tout en gardant la logique de calcul cachée derrière.

**`@x.setter`** — complète une `@property` pour permettre l'écriture, avec validation :
```python
class Circle:
    @property
    def radius(self):
        return self._radius

    @radius.setter
    def radius(self, value):
        if value < 0:
            raise ValueError("Radius can't be negative")
        self._radius = value

c.radius = 10   # passe par le setter, qui valide avant d'assigner
```

### `@abstractmethod` (module `abc`)
Vu dans Code Nexus / Data Deck : force toute sous-classe à implémenter cette méthode, sinon elle reste abstraite et non-instanciable.
```python
from abc import ABC, abstractmethod

class Shape(ABC):
    @abstractmethod
    def area(self) -> float: ...
```

### `@dataclass` (module `dataclasses`)
Génère automatiquement `__init__`, `__repr__` et `__eq__` à partir des attributs déclarés — évite d'écrire ce code répétitif à la main.
```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

p1 = Point(1.0, 2.0)
p2 = Point(1.0, 2.0)
p1 == p2        # True — __eq__ généré automatiquement, compare les attributs
print(p1)       # Point(x=1.0, y=2.0) — __repr__ généré automatiquement
```
C'est l'équivalent déclaratif de ce que tu codais à la main dans Code Cultivation (`__init__` avec `self.x = x`, etc.) — dans le même esprit que Pydantic (Cosmic Data), mais sans validation : `@dataclass` économise juste l'écriture, il ne vérifie pas les types à l'exécution.

*(Remarque : je n'ai trouvé ce décorateur dans aucun des sujets 42 que tu m'as donnés jusqu'ici — il est ici parce qu'il fait partie du langage standard, pas parce qu'un module de la piscine l'exige.)*

---

## 3. Décorateurs de `functools`

### `@functools.wraps`
Préserve les métadonnées (`__name__`, `__doc__`) d'une fonction décorée — indispensable dans **tout** décorateur "fait maison" (vu en détail dans FuncMage) :
```python
from functools import wraps

def logger(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper
```
Sans `@wraps(func)`, `ma_fonction.__name__` deviendrait `"wrapper"` au lieu du vrai nom — gênant pour le debug, la documentation, ou tout code qui inspecte les fonctions.

### `@functools.lru_cache` / `@functools.cache`
Mémoïse le résultat d'une fonction selon ses arguments — évite de recalculer ce qui a déjà été calculé (vu dans FuncMage ex3).
```python
from functools import lru_cache

@lru_cache(maxsize=128)
def fibonacci(n):
    return n if n < 2 else fibonacci(n-1) + fibonacci(n-2)
```
`@cache` (Python 3.9+) est un raccourci pour `@lru_cache(maxsize=None)` — un cache sans limite de taille.

### `@functools.singledispatch`
Une fonction qui change d'implémentation selon le **type** de son premier argument, sans `if isinstance(...)` (vu dans FuncMage ex3).
```python
from functools import singledispatch

@singledispatch
def render(value):
    return "unknown"

@render.register
def _(value: int):
    return f"number: {value}"

@render.register
def _(value: str):
    return f"text: {value}"
```

### `@functools.total_ordering`
Génère automatiquement tous les opérateurs de comparaison (`<`, `<=`, `>`, `>=`) à partir de seulement deux que tu définis (`__eq__` et un seul autre, typiquement `__lt__`).
```python
from functools import total_ordering

@total_ordering
class Version:
    def __init__(self, number):
        self.number = number
    def __eq__(self, other):
        return self.number == other.number
    def __lt__(self, other):
        return self.number < other.number

# __le__, __gt__, __ge__ sont déduits automatiquement
```

### `@functools.cached_property`
Combine `@property` et mise en cache : le calcul n'a lieu qu'**une seule fois**, à la première lecture, puis la valeur est réutilisée.
```python
from functools import cached_property

class Report:
    def __init__(self, data):
        self.data = data

    @cached_property
    def total(self):
        print("Computing...")   # ne s'affiche qu'une fois
        return sum(self.data)

r = Report([1, 2, 3])
r.total   # affiche "Computing...", renvoie 6
r.total   # ne recalcule pas, renvoie directement 6
```

---

## 4. Écrire ses propres décorateurs

### Décorateur simple
```python
def timer(func):
    import time
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.3f}s")
        return result
    return wrapper
```

### Décorateur paramétrable (decorator factory)
Vu dans FuncMage — un niveau d'imbrication en plus pour accepter un argument sur le décorateur lui-même :
```python
def retry(max_attempts):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    print(f"Attempt {attempt} failed: {e}")
            raise RuntimeError("All attempts failed")
        return wrapper
    return decorator

@retry(max_attempts=3)
def unstable_call():
    ...
```
Trois étages : `retry(3)` renvoie `decorator`, appliqué à `unstable_call` il renvoie `wrapper`, qui remplace `unstable_call`.

### Décorateur basé sur une classe
Un décorateur n'est pas obligé d'être une fonction — n'importe quel objet **appelable** (qui définit `__call__`) fonctionne :
```python
class CountCalls:
    def __init__(self, func):
        self.func = func
        self.count = 0

    def __call__(self, *args, **kwargs):
        self.count += 1
        print(f"Call #{self.count}")
        return self.func(*args, **kwargs)

@CountCalls
def say_hi():
    print("hi")

say_hi()   # Call #1 / hi
say_hi()   # Call #2 / hi
```
Utile quand le décorateur doit garder un **état** plus riche qu'une simple closure (ici, `self.count` persiste naturellement en tant qu'attribut d'instance).

### Empiler plusieurs décorateurs
Les décorateurs s'appliquent **de bas en haut**, mais s'exécutent au final **de haut en bas** :
```python
@timer
@retry(max_attempts=3)
def fetch_data():
    ...
```
Équivalent à `fetch_data = timer(retry(3)(fetch_data))` : `retry` enveloppe `fetch_data` en premier (le plus proche de la fonction), puis `timer` enveloppe le tout. À l'exécution, `timer` se déclenche en premier (il mesure *tout*, y compris les tentatives de retry), puis délègue à la version avec retry.

### Décorateur qui fonctionne sur des méthodes (avec `self`)
Rien de spécial à faire — `self` fait juste partie de `*args` :
```python
def log_call(func):
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        print(f"Calling {func.__name__} on {self}")
        return func(self, *args, **kwargs)
    return wrapper

class Robot:
    @log_call
    def move(self, direction):
        print(f"Moving {direction}")
```

---

## Tableau récapitulatif

| Décorateur | Source | Rôle |
|---|---|---|
| `@staticmethod` | natif | méthode sans `self` ni `cls` |
| `@classmethod` | natif | méthode qui reçoit `cls`, constructeur alternatif |
| `@property` | natif | méthode lue comme un attribut |
| `@x.setter` | natif | autoriser l'écriture d'une `@property`, avec validation |
| `@abstractmethod` | `abc` | méthode obligatoire pour toute sous-classe |
| `@dataclass` | `dataclasses` | génère `__init__`/`__repr__`/`__eq__` automatiquement |
| `@wraps` | `functools` | préserve les métadonnées dans un décorateur fait maison |
| `@lru_cache` / `@cache` | `functools` | mémoïsation |
| `@singledispatch` | `functools` | comportement différent selon le type d'argument |
| `@total_ordering` | `functools` | génère les opérateurs de comparaison manquants |
| `@cached_property` | `functools` | `@property` calculée une seule fois puis mise en cache |

## Ce qu'il faut retenir
- Un décorateur = une fonction qui reçoit une fonction et en renvoie une autre.
- Toujours `@wraps(func)` dans un décorateur fait maison, sauf raison précise de ne pas le faire.
- L'ordre d'empilement des décorateurs compte — celui collé à la fonction s'exécute "le plus près" d'elle.
- Un décorateur "à paramètres" a toujours un niveau d'imbrication de plus qu'un décorateur simple.
