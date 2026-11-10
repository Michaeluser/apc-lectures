# C++

## Štandardná knižnica

*Peter Koscelanský <cpp@eset.sk>* <!-- .element: class="author" -->

---

## Obsah

* Vlastníctvo: `shared_ptr` a `weak_ptr`
* Výber kontajnera: sekvenčné, usporiadané a hashovacie kontajnery
* Kontajnerové adaptéry a halda
* Množinové operácie a iné algoritmy
* Paralelné algoritmy a ich nástrahy
* Ďalšie časti štandardnej knižnice

---

# `std::shared_ptr`

---

## Kedy potrebujeme zdieľané vlastníctvo?

| Situácia | Voľba |
|----------|-------|
| Objekt má jedného vlastníka | hodnota alebo `std::unique_ptr<T>` |
| Viac nezávislých vlastníkov musí udržať objekt nažive | `std::shared_ptr<T>` |
| Funkcia len používa existujúci objekt | `T&`, `const T&`, prípadne `T*` |
| Pozorovateľ nesmie predĺžiť zdieľanú životnosť | `std::weak_ptr<T>` |

* Viac miest pristupujúcich k objektu ešte neznamená viac vlastníkov
* `shared_ptr` volíme podľa požadovanej životnosti, nie ako náhradu každého smerníka

---

## Čo je `std::shared_ptr`?

* Smart pointer s automatickým počítaním referencií (*reference counting*)
* Viacero `shared_ptr` môže vlastniť ten istý objekt súčasne
* Objekt sa zmaže automaticky, keď posledný `shared_ptr` zanikne
* Rôzne kópie môžu bezpečne meniť spoločné počítadlo referencií; prístup k objektu ani súbežné zmeny tej istej inštancie `shared_ptr` tým nie sú synchronizované
* Definovaný v `<memory>`

```cpp
#include <memory>

std::shared_ptr<int> ptr(new int(10));

std::shared_ptr<int> ptr1 = std::make_shared<int>(42);
std::shared_ptr<int> ptr2 = ptr1; // copy, reference count = 2

std::println("{}", ptr1.use_count()); // prints 2
std::println("{}", *ptr1);            // prints 42

// ptr1 and ptr2 are destroyed -> object is automatically deleted
```

---

## Ako funguje reference counting?

* Každý `shared_ptr` si drží pointer na objekt + pointer na *control block*
* Control block obsahuje najmä:
  * Počet `shared_ptr` (strong references)
  * Počet `weak_ptr` (weak references) 

```cpp
std::shared_ptr<std::string> ptr1 = std::make_shared<std::string>("Hello");
// use_count() == 1

{
    std::shared_ptr<std::string> ptr2 = ptr1; // copy constructor
    // use_count() == 2
    
    std::shared_ptr<std::string> ptr3;
    ptr3 = ptr1; // copy assignment operator  
    // use_count() == 3
} // ptr2 and ptr3 are destroyed, use_count() == 1

// ptr1 is destroyed, use_count() == 0 -> object is deleted
```

---

## Vytváranie `shared_ptr`

```cpp
auto ptr = std::make_shared<std::string>("Hello World");
auto ptr2 = std::make_shared<std::vector<int>>(10); // 10 elements with value 0
```

* Preferujeme `make_shared`: bez ručného `new`, typicky jedna spoločná alokácia objektu a control blocku
* Spoločná alokácia zlepšuje lokalitu pamäte
* Existujúce `weak_ptr` môžu po zničení objektu zadržať jeho úložisko až do svojho zániku

Note: Konštruktor zo surového smerníka, napríklad `std::shared_ptr<std::string>(new std::string("Hello"))`, má použitie pri špeciálnom vytváraní objektov či vlastnom deleteri, nie ako bežná prvá voľba. Objekt sa zničí po zániku posledného vlastníka aj pri spoločnej alokácii; slabé referencie môžu zadržať úložisko, nie živý objekt. Pred C++17 navyše hrozil únik pamäte pri prekladaní vyhodnocovania viacerých `shared_ptr(new ...)` v argumentoch jednej funkcie.

---

## Základné operácie

```cpp
std::shared_ptr<std::string> ptr = std::make_shared<std::string>("Test");

// Access to the object
std::println("{}", *ptr);           // dereferencing
std::println("{}", ptr->length());  // member access

// State information
std::println("{}", ptr.use_count());              // reference count
std::println("{}", static_cast<void*>(ptr.get())); // raw pointer (be careful!)
if (ptr) { /* not nullptr */ }

// Reset and swap
ptr.reset();                       // releases reference, may delete object
ptr = std::make_shared<std::string>("New");

std::shared_ptr<std::string> other;
ptr.swap(other);                   // swaps contents
```

---

## Move sémantika s `shared_ptr`

```cpp
std::shared_ptr<std::string> create_string() {
    return std::make_shared<std::string>("Hello");
}

int main() {
    // RVO/move - no control block copying
    auto ptr1 = create_string();
    
    // Move - ownership is transferred, control block stays in place
    auto ptr2 = std::move(ptr1); // ptr1 is now nullptr
    
    // Copy - reference counter is incremented  
    auto ptr3 = ptr2;  // use_count() == 2
}
```

* Move je rýchlejší ako copy (žiadne atomické operácie s počítadlom)
* Move je pre `shared_ptr` optimalizácia a nie nutnosť ako pre `unique_ptr`

---

## Cykly vlastníctva

* Ak potrebujeme smerník child -> parent a zároveň parent -> child
* Pri využití `shared_ptr` vzniká cyklus vlastníctva (loop)
* Objekty sa nikdy nezničia, pretože každý vlastní druhého

```cpp
class Child;

struct Parent {
    std::vector<std::shared_ptr<Child>> children;
    ~Parent() { std::println("Parent destroyed"); }
};

struct Child {
    std::shared_ptr<Parent> parent; // loop in ownership
};

auto parent = std::make_shared<Parent>();
auto child = std::make_shared<Child>();
parent->children.push_back(child);
child->parent = parent; // now this will never be deleted
```


### Riešenie s `std::weak_ptr`

```cpp
class Child;

struct Parent {
    std::vector<std::shared_ptr<Child>> children;
    ~Parent() { std::println("Parent destroyed"); }
};

struct Child {
    std::weak_ptr<Parent> parent; // weak reference is fine
};

auto parent = std::make_shared<Parent>();
auto child = std::make_shared<Child>();
parent->children.push_back(child);
child->parent = parent; // everything is fine

std::shared_ptr<Parent> p = child->parent.lock(); // get shared from weak
```

---

## `weak_ptr` operácie

```cpp
std::shared_ptr<int> shared = std::make_shared<int>(42);
std::weak_ptr<int> weak = shared;

// test if object still exists
if (!weak.expired()) {
    std::println("Object still exists");
}

// Safe access
if (auto locked = weak.lock()) {
    std::println("{}", *locked);  
} // locked is destroyed here

shared.reset(); // object is destroyed
std::println("{}", weak.expired()); // true
auto locked2 = weak.lock();         // returns nullptr
```

* `expired()` je iba okamžitý stav; nezaručí, že objekt zostane nažive
* Na prístup použijeme `lock()` a skontrolujeme získaný `shared_ptr`

---

## Pokročilé: `std::enable_shared_from_this`

* Keď objekt potrebuje odovzdať ďalšiemu vlastníkovi `shared_ptr` na seba
* Nikdy nevytvárame druhý nezávislý control block cez `std::shared_ptr<Widget>(this)`

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
  std::shared_ptr<Widget> self() {
    return shared_from_this();
  }
};

auto widget = std::make_shared<Widget>();
auto same = widget->self();
assert(widget == same && widget.use_count() == 2);
```

* Vyžaduje verejné, jednoznačné dedenie a už existujúce zdieľané vlastníctvo


## Prečo nie v konštruktore?

* `enable_shared_from_this` obsahuje internú slabú referenciu na existujúci control block
* `shared_ptr` ju prepojí až po dokončení konštrukcie objektu, nie počas nej
* Bez tohto prepojenia `shared_from_this()` vyhodí `std::bad_weak_ptr`

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
  Widget() {
    auto self = shared_from_this(); // throws std::bad_weak_ptr
  }
};

auto widget = std::make_shared<Widget>();
```

* Aj `make_shared` tu skončí výnimkou; samotná alokácia na halde nestačí

Note: Rovnako zlyhá volanie na lokálnom objekte alebo objekte vlastnenom iba cez `unique_ptr`. Ak potrebujeme inicializáciu používajúcu `shared_from_this()`, vykonáme ju až po vytvorení vlastníka, napríklad vo factory funkcii po `make_shared`.


## `weak_from_this()` od C++17

* Callback môže prežiť objekt, ale nemá ho udržiavať nažive
* Zachytíme `weak_ptr`; až pri volaní cez `lock()` dočasne získame vlastníctvo

```cpp
class Widget : public std::enable_shared_from_this<Widget> {
public:
    void refresh() { std::cout << "Refresh\n"; }
    auto on_update() {
        return [weak = weak_from_this()] {
            if (auto self = weak.lock()) {
                self->refresh();
            }
        };
    }
};

auto widget = std::make_shared<Widget>();
auto callback = widget->on_update();
callback();
widget.reset();
callback();
```

* Prvé volanie vypíše `Refresh`, druhé už nič: objekt medzitým zanikol

Note: `weak_from_this()` od C++17 nevyhadzuje výnimky. Bez existujúceho zdieľaného vlastníctva, napríklad v konštruktore, vráti prázdny `weak_ptr`. Takto získaná kópia zostane prázdna aj po vytvorení vlastníka; treba zavolať `weak_from_this()` znova. Zachytenie `this` by po zániku objektu zanechalo neplatný smerník; zachytenie `shared_from_this()` by naopak predĺžilo jeho životnosť až do zániku callbacku. Úspešný `lock()` udrží objekt nažive počas `refresh()`, ale nesynchronizuje prístup k jeho dátam.

---

## Performance

* `std::unique_ptr` je v podstate taký rýchly ako raw pointer
* `std::shared_ptr` je pomalší kvôli atomickým operáciám na počítadle referencií pri kopírovaní a ničení, dá sa zlepšiť použitím `std::move`
* `std::shared_ptr` používa viac pamäte (objekt plus control block) a niekedy môže spôsobiť horšiu lokalitu pamäte (ak control block a objekt nie sú v jednej alokácii)

---

# Kontajnery

---

## Zložitosť operácií

* Štandard určuje správanie a zložitosť operácií, nie spravidla konkrétnu dátovú štruktúru
* Porovnávame $O(1)$, $O(\log n)$ a $O(n)$; amortizované a priemerné odhady nie sú garanciou každého volania
* Rovnaká asymptotická zložitosť ešte neznamená rovnakú rýchlosť: záleží aj na alokáciách a lokalite pamäte

Note: Pre nezáporné funkcie používame definíciu $f(n)=O(g(n)) \Leftrightarrow \exists c > 0\; \exists n_0\; \forall n \geq n_0:\; f(n) \leq c\,g(n)$. Priemerná zložitosť hashovacej tabuľky predpokladá vhodné rozloženie hashov; amortizácia pri vectore rozkladá cenu realokácií na celú sekvenciu vkladaní.

---

## Operácie

* Zaujímajú nás hlavne nasledovné operácie
    * Prístup k prvku (*get* / *access*)
    * Vloženie prvku (*insert*)
    * Mazanie prvku (*erase*)
* V reálnom kóde sa vyskytuje hlavne prístup k prvkom a vkladanie nových prvkov
* Prístup k prvku je operácia, ktorá mi vráti hodnotu na základe kľúča alebo indexu
* Pamäťová zložitosť ktorú uvádzame je pre populárne implementácie, v praxi si môžu knižnice implementovať kontajnery rôzne

---

## Pripomenutie: `std::vector`

* Dynamické pole so súvislou pamäťou, vhodné ako prvá voľba kontajnera
* Dobrá lokalita pamäte a spolupráca s cache procesora
* `size()` je počet prvkov, `capacity()` je počet prvkov, na ktoré máme alokovanú pamäť
* Realokácia zneplatní všetky iterátory, smerníky a referencie; bez realokácie `insert` a `erase` zneplatnia tie od miesta zmeny vrátane `end()`

| Operácia | Zložitosť |
|----------|-----------|
| Prístup cez `[]` / `at` | $O(1)$ |
| `push_back` | amortizovane $O(1)$ |
| `pop_back` | $O(1)$ |
| Vloženie / mazanie uprostred | $O(n)$ |
| Vyhľadávanie cez `std::find` | $O(n)$ |


## `push_back` vs `emplace_back`

```cpp
std::vector<std::pair<std::string, std::string>> v;

v.push_back({ "Hello", "worlds" }); // pair is constructed and moved

v.emplace_back("Hello", "worlds"); // pair is constructed in place
```

* V tomto príklade `push_back` presunie dočasný pár do vektora; pri lvalue by kopíroval
* `emplace_back` odovzdá argumenty konštruktoru prvku priamo v pamäti vektora
* Ani `emplace_back` nezabráni presunu alebo kopírovaniu existujúcich prvkov pri realokácii

---

## `std::vector<bool>`

* Špecializácia s typicky bitovým úložiskom, nie obyčajné pole `bool`
* Namiesto `bool&` vracia proxy objekt, ktorý reprezentuje prístup k bitu
* Typicky spotrebuje približne osemkrát menej pamäte na prvky než `std::vector<char>`

```cpp
std::vector<bool> bits = { true, false, true };
auto proxy = bits[2];
proxy = false;
assert(!bits[2]);
```

* Súbežné zápisy do rôznych bitov nemusia byť bezpečné


## Proxy nie je referencia na `bool`

```cpp
std::vector<bool> bits = { true, false, true };
bool&& temporary = bits[2];
temporary = false;
assert(bits[2]);

const bool& snapshot = bits[2];
auto&& proxy = bits[2];
proxy = false;
assert(snapshot && !bits[2]);
```

* `temporary` a `snapshot` odkazujú na dočasné hodnoty `bool` s predĺženou životnosťou, nie na bity vektora
* Iba priradenie cez proxy zmení prvok; skutočné `bool&` z neho nezískame

---

## `std::array`

* Pole s pevnou veľkosťou určenou parametrom šablóny; prvky sú uložené priamo v objekte
    * Pridávanie a mazanie prvkov preto nedáva zmysel
* Prístup podľa indexu je $O(1)$, vyhľadávanie cez `std::find` je $O(n)$
* Doba uloženia závisí od umiestnenia objektu, nejde automaticky o statickú pamäť

| Operácia | Zložitosť |
|----------|-----------|
| Prístup | $O(1)$ |
| Vloženie | - |
| Mazanie | - |

```cpp
std::array<int, 5> a = {1, 2, 3, 4, 5};

int b = a[2]; // 3

auto it = std::find(a.begin(), a.end(), 3);
```

---

## `std::deque`

* Double-ended queue: konštantné vkladanie a mazanie jednotlivých prvkov na oboch koncoch
* Náhodný prístup podľa indexu v $O(1)$ aj uprostred
* Prvky netvoria jeden súvislý rozsah pamäte

| Operácia | Zložitosť |
|----------|-----------|
| Prístup podľa indexu | $O(1)$ |
| Vloženie / mazanie jedného prvku na koncoch | $O(1)$ |
| Vloženie / mazanie uprostred | $O(n)$ |


## Invalidácia iterátorov v `deque`

* Vkladanie na ktoromkoľvek konci invaliduje všetky iterátory, ale nie referencie ani smerníky na existujúce prvky
* Mazanie na koncoch invaliduje iterátory a referencie na zmazané prvky; zmazanie posledného prvku invaliduje aj `end()`
* Vkladanie alebo mazanie uprostred invaliduje všetky iterátory a referencie


## Operácie nad `deque`

```cpp
std::deque<int> values = { 1, 2, 3, 4, 5 };

values.push_back(6);
values.push_front(0);
std::sort(values.begin(), values.end());
values.pop_back();
values.pop_front();
```

* `deque` nemá `reserve`, `capacity` ani súvislý buffer cez `data()`
* Volíme ho pre operácie na oboch koncoch; ak stačí koniec, `vector` býva jednoduchšia prvá voľba

Note: Bežná reprezentácia je tabuľka smerníkov na bloky. Cena indexovania a alokácií závisí od implementácie. [Diagram](https://i.sstatic.net/SthOW.png), [zdroj](https://stackoverflow.com/a/6292437), [porovnanie](https://www.codeproject.com/Articles/5425/An-In-Depth-Study-of-the-STL-Deque-Container).

---

## `std::list`

* Obojsmerne zreťazený zoznam s konštantným `size()`
* Stabilné iterátory a referencie: mazanie invaliduje iba odkazy na zmazané prvky
* Vkladanie a mazanie na známej pozícii je $O(1)$, nájdenie tej pozície môže stáť $O(n)$
* Samostatné uzly znamenajú ďalšie alokácie a nepriame prístupy

| Operácia | Zložitosť |
|----------|-----------|
| Vyhľadanie / prístup podľa pozície | $O(n)$ |
| Vloženie jedného prvku pri známom iterátore | $O(1)$ |
| Mazanie cez iterátor | $O(1)$ |


## Operácie nad `list`

```cpp
std::list<int> values = { 3, 1, 4, 1, 5 };
values.push_back(6);
values.insert(values.begin(), 0);
values.sort();
values.unique();
```

* Iterátory sú bidirectional: podporujú `++` a `--`, nie `it + n`
* Členské `sort` je stabilné a má $O(n \log n)$ porovnaní; `std::sort` s týmito iterátormi nefunguje
* `unique` odstráni susedné duplicity priamo zo zoznamu; existuje aj členské `reverse`


## `splice` a `merge`

* `splice` prepojí prvky medzi zoznamami bez kopírovania či presúvania ich hodnôt
* `merge` zlúči dva už utriedené zoznamy s rovnakým usporiadaním

```cpp
std::list<int> first = { 1, 2, 3, 4, 5 };
std::list<int> second = { 6, 7 };

first.splice(first.begin(), second);
second.splice(second.begin(), first, first.begin(), std::next(first.begin(), 2));
first.merge(second);
```

* Výsledok: `first` obsahuje `1, 2, 3, 4, 5, 6, 7`, `second` je prázdny
* Má zmysel pri potrebe stabilných odkazov alebo prepájania uzlov; samotné $O(1)$ vkladanie nie je dôvod opustiť `vector`

Note: Zoznamy pri `splice` a `merge` musia mať rovnaké alokátory. Iterátory na prenesené prvky zostávajú platné a odkazujú už do cieľového zoznamu. Presun rozsahu medzi rôznymi zoznamami môže byť lineárny kvôli aktualizácii ich veľkostí.

---

## `std::forward_list`

* Jednosmerne zreťazený zoznam: menej odkazov na uzol než `list`
* Forward iterátory podporujú iba postup dopredu
* Má členské `sort`, `unique`, `reverse` a `merge`, ale nemá `size()`
* Vkladanie a mazanie potrebuje pozíciu pred meneným prvkom

```cpp
std::forward_list<int> values = { 1, 2, 3 };
values.insert_after(values.before_begin(), 0);

std::forward_list<int> other = { 6, 7 };
values.splice_after(values.before_begin(), other);
```

* `before_begin()` označuje pozíciu pred prvým prvkom, nesmie sa dereferencovať

---

## `std::inplace_vector`

* C++26: premenlivý počet prvkov s pevnou kapacitou, napríklad `std::inplace_vector<int, 16>`
* Prvky sú uložené priamo v objekte, bez dynamickej alokácie vlastného úložiska
* Veľkosť sa mení cez `push_back`, `pop_back` či `erase`, kapacita zostáva pevná
* Pri plnej kapacite `push_back` vyhodí `std::bad_alloc` namiesto realokácie

Note: Vhodné pre malý, vopred ohraničený počet prvkov. Dostupnosť zatiaľ závisí od implementácie štandardnej knižnice. Na rozdiel od `array` nemusí byť celá kapacita obsadená živými prvkami.

---

# Asociatívne kontajnery

---

## `std::map`

* Asociatívny kontajner, ktorý ukladá dvojice kľúč-hodnota
* Kľúč je unikátny, hodnota môže byť ľubovoľná
* Typicky implementovaný ako vyvážený binárny strom; iterácia prechádza kľúče v poradí komparátora
* Vkladanie neinvaliduje iterátory; mazanie invaliduje iba iterátory, smerníky a referencie na zmazané prvky

| Operácia | Zložitosť |
|----------|-----------|
| Prístup | $O(\log n)$ |
| Vloženie | $O(\log n)$ |
| Mazanie | $O(\log n)$ |

* Za usporiadanie a stabilitu odkazov platíme samostatnými uzlami a horšou lokalitou než pri vectore

Note: Bežné implementácie používajú červeno-čierny strom. Uzol môže obsahovať ľavého a pravého potomka, rodiča a farbu; presné rozloženie, sentinel aj veľkosť objektu mapy závisia od implementácie, alokátora a debug režimu. Počty bajtov nie sú prenosná garancia.


## `contains`, `find` alebo `operator[]`?

| Zámer | Operácia |
|-------|----------|
| Zistiť existenciu kľúča | `contains(key)` od C++20 |
| Získať prvok bez vloženia | `find(key)` a kontrola proti `end()` |
| Získať hodnotu alebo vyhodiť výnimku | `at(key)` |
| Získať hodnotu a prípadne vytvoriť nový prvok | `operator[]` |

```cpp
std::map<std::string, int> counts = { { "build", 2 } };

assert(!counts.contains("test"));
if (auto found = counts.find("build"); found != counts.end()) {
    found->second += 1;
}
counts["test"] += 1; // if "test" doesn't exist, it will be created
assert(counts.at("test") == 1);
```

* Rovnaké rozlíšenie platí aj pre `std::unordered_map`


## Intervalové otázky: `lower_bound`

* Členské `lower_bound(key)` nájde prvý kľúč, ktorý nie je menší než `key`
* `upper_bound(key)` nájde prvý kľúč väčší než `key`

```cpp
std::map<int, std::string> events = {
    { 9, "Standup" }, { 11, "Review" }, { 14, "Release" }
};

const auto stop = events.lower_bound(14);
for (auto event = events.lower_bound(10); event != stop; ++event) {
    std::cout << event->first << ": " << event->second << '\n';
}
```

* Vypíše udalosti z intervalu $[10, 14)$, teda `11: Review`
* Členské vyhľadávanie je $O(\log n)$; `std::lower_bound` nad iterátormi stromu potrebuje lineárny počet posunov


## Vkladanie prvkov

```cpp
std::map<int, std::string> m = { {1, "one"}, {2, "two"} };

std::pair<std::map<int, std::string>::iterator, bool> x = m.insert({ 3, "three" });
assert(x.second);

auto[it, inserted] = m.insert({ 3, "four" });
assert(!inserted);
assert(it->second == "three");

auto[it2, inserted2] = m.insert_or_assign(3, "four" );
assert(!inserted2);
assert(it2->second == "four");
```

* `insert` vloží dvojicu, kde prvý prvok je kľúč a druhý hodnota, vráti dvojicu iterátor na vložený prvok a `bool`, či sa vložil (`true`) alebo už existoval (`false`)
* `insert_or_assign` vloží kľúč s hodnotou, ak kľúč už existuje, prepíše hodnotu, vráti rovnakú dvojicu ako `insert`


## Konštrukcia prvku cez `emplace`

```cpp
std::map<int, std::string> names;

auto [first, inserted] = names.emplace(1, "Alice");
assert(inserted && first->second == "Alice");

auto [same, inserted_again] = names.emplace(1, "Bob");
assert(!inserted_again && same->second == "Alice");
```

* Argumenty sa odovzdajú konštruktoru prvku `std::pair<const int, std::string>`
* Existujúcu hodnotu neprepíše; vráti iterátor na prvok a informáciu, či sa vložil
* Pri duplicitnom kľúči môže skonštruovať nový prvok a hneď ho zničiť
* Ako sa vyhnúť zbytočnej konštrukcii hodnoty? Od C++17 cez `try_emplace`


## Konštrukcia hodnoty cez `try_emplace`

```cpp
std::map<int, std::vector<int>> groups;

auto [first, inserted] = groups.try_emplace(1, 3, 7);
assert(inserted && (first->second == std::vector<int>{ 7, 7, 7 }));

auto [same, inserted_again] = groups.try_emplace(1, 1000, 9);
assert(!inserted_again && same == first);
```

* Argumenty `3, 7` sa odovzdajú priamo konštruktoru `vector<int>`
* Druhé volanie nevytvorí tisícprvkový vektor, pretože kľúč už existuje
* `emplace` takú garanciu nemá: pri duplicite môže prvok skonštruovať a následne zničiť
* Návratové hodnoty sú rovnaké ako pri `insert`

Note: Argumenty volania sa vyhodnotia aj pri neúspešnom `try_emplace`. `std::piecewise_construct` umožňuje konštruovať členy páru priamo z argumentov, ale pri `emplace` nezaručuje vynechanie konštrukcie pri duplicitnom kľúči.


## Mazanie prvkov

```cpp
std::map<int, std::string> m = {{1, "one"}, {2, "two"}};

m.erase(1);
m.erase(m.begin());
```

* `erase` zmaže prvky na základe kľúča alebo iterátora
* Po zmazaní sa strom musí vyvážiť

Note: `emplace_hint` a overload `insert` s hintom môžu dosiahnuť amortizovane konštantné vloženie, ak sa prvok vloží tesne pred hint. Napríklad pri pridávaní rastúcich kľúčov možno použiť `end()`. Nesprávny, ale platný hint nemení správnosť výsledku; všeobecná zložitosť zostáva logaritmická. Vymazanie cez iterátor má amortizovane konštantnú zložitosť, cez kľúč logaritmickú.


## Usporiadané kontajnery

* Ako vypíšeme všetky prvky z mapy od najmenšieho po najväčší kľúč?
    
```cpp
std::map<std::string, int> my_map = { // fill up map
    { "One" , 1 },
    { "Two", 2 },
    { "Three" , 3 }
};

std::vector<std::pair<std::string, int>> my_vector;
std::copy(my_map.begin(), my_map.end(), std::back_inserter(my_vector));
std::sort(my_vector.begin(), my_vector.end());

for (const auto& i : my_vector) {
    std::cout << i.first << " " << i.second << std::endl;
}
```


* Zbytočne komplikované, mapa je už utriedená
* Obyčajnou iteráciou sa dá vypísať od najmenšieho po najväčší kľúč

```cpp
for (const auto& i : my_map) {
    std::cout << i.first << " " << i.second << std::endl;
}
```


## Ako zlúčime dve mapy?

```cpp
std::map<std::string, int> a, b;

a.insert(b.begin(), b.end());
```

Tento postup kopíruje vkladané prvky do nových uzlov mapy `a`; existujúce kľúče neprepíše.
<!-- .element class="fragment" -->

```cpp
a.insert(std::make_move_iterator(b.begin()), std::make_move_iterator(b.end()));
```
<!-- .element class="fragment" -->

Rýchlejší postup, pretože aspoň hodnoty, ktoré nie sú `const` sa presunú.
<!-- .element class="fragment" -->

```cpp
a.merge(b);
```
<!-- .element class="fragment" -->

Od C++17 možno preniesť celé uzly bez kopírovania či presúvania prvkov. Prvky s kľúčmi, ktoré už sú v `a`, zostanú v `b`.
<!-- .element class="fragment" -->

---

## `std::set`

* Asociatívny kontajner, v ktorom je samotná hodnota kľúčom; neukladá samostatnú priradenú hodnotu
* Typicky implementovaný ako vyvážený binárny strom, s rovnakými pravidlami invalidácie ako `map`

| Operácia | Zložitosť |
|----------|-----------|
| Prístup | $O(\log n)$ |
| Vloženie | $O(\log n)$ |
| Mazanie | $O(\log n)$ |

* Pamäťová zložitosť je rovnaká ako u `std::map`


## Operácie

```cpp
std::set<int> s = {1, 2, 3, 4, 5};

s.insert(6);

s.erase(3);

auto it = s.find(4);
```

* Vyhľadávanie a mazanie sú podobné ako u `std::map`, pracujú však so samotnou hodnotou
* `try_emplace` neexistuje; `emplace` môže skonštruovať a zničiť aj duplicitný prvok

---

## `std::multimap`

* Asociatívny kontajner, ktorý ukladá dvojice kľúč-hodnota, kľúč nie je unikátny
* Implementovaný rovnako ako `std::map`, na MSVC zdieľa implementáciu s `std::map`
* Podporuje duplicitné kľúče, nemá však `operator[]`, `at`, `try_emplace` ani `insert_or_assign`

```cpp
std::multimap<int, std::string> m = { {1, "one"}, {2, "two"}, {2, "two2"} };

m.insert({ 2, "two3" });
m.insert({ { 3, "three" }, { 3, "three2" } });
```

* Existuje aj `std::multiset`, ktorý je rovnaký ako `std::multimap`, ale bez hodnôt, resp. ako `std::set` ale s duplicitnými hodnotami 


## Špeciálne operácie

```cpp
std::multimap<int, std::string> m = { {1, "one"}, {2, "two"}, {2, "two2"} };

m.insert({ 2, "two3" });
m.insert({ { 3, "three" }, { 3, "three2" } });

auto it = m.find(2);
std::cout << it->second << '\n'; // "two" or "two2" or "two3"

std::cout << m.count(3) << '\n'; // 2

auto range = m.equal_range(2);
for (auto it = range.first; it != range.second; ++it) {
    std::cout << it->second << " "; // "two two2 two3"
}
```

* Tieto metódy existujú aj pre `std::map`, ale tam sú menej užitočné

---

## Hashovacia funkcia

* Transformuje objekt na `size_t` (hash)
* Ekvivaletné objekty (`==`) musia vracať rovnaký hash
* Nerovnaké objekty môžu mať rovnaký hash; dobré rozloženie znižuje počet kolízií
* Rovnaká hodnota musí mať rovnaký hash počas jedného behu programu, nie nevyhnutne medzi behmi
* Veľmi rýchla (inak nezískame veľa rýchlosti)
* Pre použitie  v kontajneroch nemusí byť kryptografická
* `std::hash` pre preddefinované objekty, viac sa dá nájsť v `boost::hash`
* Napríklad FNV-1a


## Hash pre preddefinované typy

```cpp
std::cout << std::hash<int>()(42) << '\n';

std::hash<std::string> hash_fn;
std::cout << hash_fn("Hello, World!") << '\n';
std::cout << hash_fn("Bjarne") << '\n';

std::string s = "One wil have capacity set, other not";
std::string s2 = "One wil have capacity set, other not";
s2.reserve(1000000);
assert(hash_fn(s) == hash_fn(s2));
```

* `std::hash` je funktor, ktorý sa vytvára s typom, ktorý chceme hashovať
* Volanie s hodnotou potom vráti hash tejto hodnoty
    * Výsledok je závislý od implementácie a môže sa líšiť medzi behmi programu
    * Väčšina typov zo štandardnej knižnice má definovaný hash, ktorý hashuje iba hodnotu


## Hash pre vlastné typy

* Definujeme si triedu, ktorú chceme hashovať
* Predvolený `std::equal_to` v hashovacej tabuľke potrebuje `operator==`; alternatívou je vlastný predikát ekvivalencie
* `std::hash` je šablóna, ktorú musíme špecifikovať pre náš typ, ak máme privátne členy, môžeme definovať `friend std::hash` pre náš typ

```cpp
struct MyClass {
    std::string s;
    uint64_t i;

    friend std::hash<MyClass>;
    bool operator==(const MyClass& rhs) const = default;
};
```


## Špecializácia `std::hash`

* Môžeme špecializovať `std::hash` pre vlastný typ alebo odovzdať kontajneru vlastný hasher
* Takáto špecializácia je povolená; ľubovoľné nové deklarácie do namespace `std` pridávať nesmieme

```cpp
namespace std {
    template<> struct hash<MyClass> {
        size_t operator()(const MyClass& keyval) const {
            return std::hash<std::string>()(keyval.s)
                ^ std::hash<uint64_t>()(keyval.i);
        }
    };
}
```


## Použitie vlastného hashu

```cpp
MyClass m{ "Herb", 22 };

std::cout << std::hash<MyClass>()(m) << '\n';
```

* Použitie je rovnaké ako pre preddefinované typy

Note: Jednoduché XOR vyššie spĺňa požiadavku rovnakého hashu pre ekvivalentné hodnoty, ale nemusí dobre rozložiť vstupy. XOR je komutatívne a kolíziám nezabráni. Pri kombinovaní viacerých členov uprednostníme hotové riešenie, napríklad `boost::hash_combine` z `<boost/container_hash/hash.hpp>`, pred vlastným bitovým mixerom. Zdroj: <https://www.boost.org/doc/libs/1_86_0/libs/container_hash/doc/html/hash.html#notes_hash_combine>.

---

## `std::unordered_map`

* Hashovacia tabuľka pre dvojice s unikátnymi kľúčmi
* Poradie iterácie nie je poradie kľúčov ani vloženia
* Vkladanie môže pri rehashi invalidovať všetky iterátory; mazanie invaliduje iba tie na zmazané prvky
* Rehash neinvaliduje smerníky ani referencie na prvky

| Operácia | Zložitosť |
|----------|-----------|
| Prístup | $O(n)$, priemerne $O(1)$ |
| Vloženie | $O(n)$, priemerne $O(1)$ |
| Mazanie | $O(n)$, priemerne $O(1)$ |

Note: Existujú aj `unordered_set`, `unordered_multimap` a `unordered_multiset`. Bežné implementácie používajú tabuľku bucketov a samostatné uzly. Konkrétne zoznamy, počet smerníkov a ukladanie hashu sa líšia. Diagramy implementácií: [libstdc++](https://i.sstatic.net/qfxPD.png), [MSVC](https://i.sstatic.net/sNl7p.png). Zdroj: <https://stackoverflow.com/questions/67435837/implementation-of-bucket-in-unordered-map>.


## Prístup k prvkom

```cpp
std::unordered_map<int, std::string> m = { {1, "one"}, {2, "two"} };

std::string a = m[1]; // "one"
std::string b = m[3]; // ""

assert(m.find(4) == m.end());

std::string c = m.at(3); // can throw
```

* Interface v podstate rovnaký ako u `std::map`, ale zložitosť je priemerne $O(1)$


## Vloženie prvku

```cpp
std::unordered_map<int, std::string> m = { {1, "one"}, {2, "two"} };

m.insert({ 3, "three" });
m.insert({ { 4, "four" }, { 5, "five" } });

m.emplace(6, "six");

auto[_, inserted] = m.insert_or_assign(6, "six2");
assert(!inserted);
```

* Interface je rovnaký ako u `std::map`, ale zložitosť je priemerne $O(1)$
* Rovnako aj pre mazanie prvkov
* Pri rehashi sa invalidujú všetky iterátory, preto ich po vkladaní bez overenia znovu nepoužívame


## Hash policy

* `load_factor` - pomer počtu prvkov k počtu bucketov
* `max_load_factor` - maximálny pomer počtu prvkov k počtu bucketov, ktorý je ešte akceptovateľný, môže byť nastavený užívatelom

```cpp
std::unordered_map<int, std::string> m;
// fill up m
std::cout << m.load_factor() << '\n'; // 0.762939
std::cout << m.max_load_factor() << '\n'; // 1
```


## Rehash

* `rehash(n)` - nastaví počet bucketov na `n`, alebo viac, aby sa zmestili všetky prvky a `max_load_factor` sa dodržal
* `reserve(n)` - nastaví počet bucketov, aby sa zmestilo aspoň `n` prvkov a `max_load_factor` sa dodržal, zavolá rehash
* Rehash invaliduje iterátory, nie referencie ani smerníky na existujúce prvky

```cpp
std::unordered_set<std::string> m;
m.max_load_factor(0.5);
m.reserve(100);

std::cout << m.load_factor() << '\n'; // 0
std::cout << m.max_load_factor() << '\n'; // 0.5
```

Note: `bucket_count()`, `bucket_size(i)` a lokálne iterátory `begin(i)`/`end(i)` sú užitočné pri skúmaní kolízií a kvality hashu. Samotná zmena `max_load_factor` nemusí zmeniť aktuálne rozloženie; `rehash(0)` vyžiada prepočet s aktuálnymi požiadavkami, nie konkrétny počet bucketov.

---

## `std::flat_map` a `std::flat_set`

* Od C++23: usporiadané asociatívne kontajnery nad sekvenčným úložiskom
* `flat_map` štandardne používa dva vektory: triedené kľúče a hodnoty na zodpovedajúcich indexoch
* `flat_set` štandardne používa jeden vektor triedených kľúčov
* Vyhľadávanie je $O(\log n)$, vkladanie a mazanie $O(n)$
* Dobrá lokalita pamäte, ale slabšia stabilita iterátorov a referencií než pri stromoch

```cpp
std::flat_map<int, std::string> numbers = { { 1, "one" }, { 2, "two" } };
numbers.insert({ 3, "three" });
```

Note: Podkladové kontajnery sa dajú zmeniť a ovplyvňujú invalidáciu. `flat_map` má proxy referencie na páry kľúčov a hodnôt, preto jeho rozhranie nie je vo všetkých detailoch zameniteľné s `map`. Dostupnosť závisí od verzie štandardnej knižnice.

---

## Ako vybrať kontajner?

| Potreba | Prvá voľba |
|---------|------------|
| Bežná sekvencia, iterácia a indexovanie | `vector` |
| Pevný počet prvkov / pevná kapacita | `array` / `inplace_vector` |
| Vkladanie a mazanie na oboch koncoch | `deque` |
| Stabilné odkazy a prepájanie uzlov | `list` |
| Usporiadané kľúče, intervalové otázky | `map` / `set` |
| Vyhľadávanie podľa kľúča bez požiadavky na poradie | `unordered_map` / `unordered_set` |
| Väčšinou čítanie, málo zmien, usporiadané kľúče | `flat_map` / `flat_set` |

* Pred voľbou sa pýtame na operácie, objem dát a požadovanú stabilitu odkazov

Note: Mimo štandardu ponúkajú ďalšie kompromisy `boost::small_vector` s malým úložiskom priamo v objekte, `boost::multi_index` s viacerými indexmi, `boost::intrusive` s odkazmi vloženými do používateľských objektov, `absl::flat_hash_map` s otvoreným adresovaním a `absl::btree_map` nad B-stromom. Nie sú zameniteľné so štandardnými kontajnermi vo všetkých garanciách.

---

# Kontajnerové adaptéry

---

* Adaptéry upravujú rozhranie kontajnerov
    * Pridávajú funkcie
    * Skrývajú implementáciu

---

## `std::stack`

* LIFO štruktúra
* Implementovaná nad iným kontajnerom 
    * štandardne `std::deque`
    * dáva zmysel zmeniť na `std::vector`


## Interface

```cpp
std::stack<int, std::vector<int>> s; // standard is std::deque
s.push(1);
s.push(2);
std::cout << s.top(); // 2
s.pop();
std::cout << s.top(); // 1
s.pop();
// std::cout << s.top(); // undefined
```

* `push` pridá prvok na vrchol
* `pop` odstráni prvok z vrcholu
* `top` vráti referenciu na vrchný prvok
* `size` vráti veľkosť

---

## `std::queue`

* FIFO štruktúra
* Implementovaná nad iným kontajnerom 
    * štandardne `std::deque`
    * dáva zmysel zmeniť na `std::list`, ak veľmi chceme


## Interface

```cpp
std::queue<int, std::list<int>> q; // standard is std::deque
q.push(1);
q.push(2);
std::cout << q.front(); // 1
std::cout << q.back(); // 2
q.pop();
std::cout << q.front(); // 2
q.pop();
// std::cout << q.front(); // undefined
```

* `push` pridá prvok na koniec
* `pop` odstráni prvok z začiatku
* `front` vráti referenciu na prvý prvok, `back` na posledný

---

## `std::priority_queue`

* Prioritná rada
* Implementovaná nad iným kontajnerom 
    * štandardne `std::vector`
    * môžeme zmeniť na `std::deque`
* Prvky sú uložené aby boli podľa priority
* Priorita je definovaná pomocou `<`
* Vnútorne je implementovaná ako halda (heap)


## Interface

```cpp
std::priority_queue<int> s; // standard is std::vector
s.push(1);
s.push(2);
s.push(0);
s.push(3);
std::cout << s.top(); // 3
s.pop();
std::cout << s.top(); // 2
s.pop();
std::cout << s.top(); // 1
s.pop();
```

---

## Halda

* `priority_queue` skrýva správu haldy; nad vlastným rozsahom môžeme použiť priamo haldové algoritmy
* `push_heap` predpokladá, že rozsah bez posledného prvku je halda; `pop_heap` samo veľkosť kontajnera nemení

```cpp
std::vector<int> heap = { 5, 7, 51, 8, 7, 9, 6 };
std::make_heap(heap.begin(), heap.end());
heap.push_back(100); // (..., 100)

// assume [begin, end - 1) is heap, push end - 1
std::push_heap(heap.begin(), heap.end()); // (100, ...)
// move first to last and reheap
for (int count = 0; count < 3; ++count) {
    std::pop_heap(heap.begin(), heap.end());
    std::cout << heap.back() << ", "; // 100, 51, 9
    heap.pop_back();
}
std::sort_heap(heap.begin(), heap.end()); // 5, 6, 7, 7, 8
```

---

# Ďalšie algoritmy

---

## Množinové operácie nad utriedenými rozsahmi

* Oba vstupy musia byť utriedené podľa rovnakého usporiadania
* Môžeme kombinovať `set` aj utriedený `vector`, nie priamo `unordered_set`
* Prienik, zjednotenie a rozdiel poskytujú `set_intersection`, `set_union` a `set_difference`

```cpp
std::set<int> primes = { 2, 3, 5, 7 };
std::vector<int> even = { 2, 4, 6, 8, 10 };
std::vector<int> intersection, united;

std::set_intersection(primes.begin(), primes.end(), even.begin(), even.end(),
                      std::back_inserter(intersection));
std::set_union(primes.begin(), primes.end(), even.begin(), even.end(),
               std::back_inserter(united));
```

* `intersection`: `2`; `united`: `2, 3, 4, 5, 6, 7, 8, 10`
* Na rozdiel od `set_union` obyčajné `std::merge` zachová všetky výskyty, teda aj dve dvojky

Note: Algoritmy majú lineárnu zložitosť vzhľadom na súčet dĺžok vstupov. Fungujú aj s duplicitami: pri `set_union` je počet výskytov maximum z oboch vstupov, pri `set_intersection` minimum. Vstupy nemusia byť matematické množiny unikátnych hodnôt.

---

## Stabilné triedenie: `std::stable_sort`

* Na rozdiel od `std::sort` zachováva pôvodné poradie ekvivalentných prvkov
* Ekvivalentné prvky sú tie, pre ktoré komparátor v oboch smeroch vráti `false`
* Napríklad pri triedení podľa priority zachová poradie úloh s rovnakou prioritou

```cpp
std::vector<std::pair<std::string, int>> tasks = {
    { "Build", 2 }, { "Test", 1 }, { "Deploy", 2 }, { "Lint", 1 }
};

std::stable_sort(tasks.begin(), tasks.end(), [](const auto& a, const auto& b) {
    return a.second < b.second;
});
```

* Výsledné poradie: `Test`, `Lint`, `Build`, `Deploy`


## `unique`

* `std::unique` zlučuje susedné ekvivalentné prvky a vráti nový logický koniec
* Veľkosť kontajnera zmení až `erase`; utriedenie vopred zabezpečí, že všetky duplikáty budú susedné

```cpp
std::vector<int> v = { 1, 2, 2, 3, 3, 3, 4, 5, 5, 5, 5 };
v.erase(std::unique(v.begin(), v.end()), v.end());
```

---

## `nth_element`

* `std::nth_element` umiestni prvok na pozícii `nth` tak, akoby bol celý rozsah utriedený
    * Prvky naľavo od neho nie sú väčšie
    * Prvky napravo od neho nie sú menšie; rovnaké hodnoty môžu byť na oboch stranách
* Prvky sa poposúvajú, ale nie sú úplne utriedené
* Zložitosť je $O(n)$ priemerne (robí to podobne ako quick-sort)

```cpp
std::vector<int> v = { 5, 7, 51, 8, 7, 9, 6 };
std::nth_element(v.begin(), v.begin() + 2, v.end());
```


## a mnoho ďalších

* `std::rotate` - posunie prvky v sekvencii
* `std::reverse` - obráti sekvenciu
* `std::shuffle` - zamieša sekvenciu
* `std::minmax_element` - vráti iterátory na najmenší a najväčší prvok
* ...

---

# Paralelné spracovanie

---

## `std::execution`

* Od C++17: namespace s typmi a objektmi politík vykonávania

| Policy | Povolené vykonanie |
|--------|-------------------|
| `seq` | sekvenčne v aktuálnom vlákne |
| `par` | aj paralelne vo viacerých vláknach |
| `par_unseq` | aj paralelne a vektorizovane |
| `unseq` (C++20) | vektorizovane v aktuálnom vlákne |

* Policy povoľuje optimalizáciu, negarantuje počet vlákien ani zrýchlenie
* Výnimka z používateľskej funkcie pri štandardnej policy vedie k `std::terminate`, aj pri `seq`

Note: Volanie algoritmu bez policy je tiež sekvenčné, ale výnimky z používateľskej funkcie sa štandardne propagujú. Podpora paralelizácie a vektorizácie závisí od štandardnej knižnice, jej backendu a konkrétneho algoritmu. Napríklad MSVC podporuje paralelný `std::sort`.

---

## `std::sort`

```cpp
int rnd() {
    static std::mt19937 rng; // random number generator
    static std::uniform_int_distribution<int> distribution(0, 1'000'000);
    return distribution(rng);
}

int main() {
    using namespace std::chrono;
    std::vector<int> v(100'000'000);
    std::generate(v.begin(), v.end(), rnd); // will generate numbers
    auto start = steady_clock::now();
    std::sort(std::execution::par, v.begin(), v.end());
    auto duration = steady_clock::now() - start;
    auto millisec = duration_cast<milliseconds>(duration).count();
    std::cout << millisec << "ms\n";
}
```

Note: Porovnávame `seq` a `par` na kópiách rovnakého vstupu a v optimalizovanom zostavení. Historické meranie na autorovom stroji ukázalo približne 2,1-násobné zrýchlenie; nie je to garancia pre tento kód ani iný počítač. `rnd` má zdieľaný stav a v tejto ukážke sa volá iba zo sekvenčného `generate`.


## `std::for_each`

```cpp
std::vector<int> v(100'000'000);
std::generate(v.begin(), v.end(), rnd); // will generate numbers
auto start = steady_clock::now();
std::for_each(std::execution::par, v.begin(), v.end(), [](int& i) { i *= 2; });
auto duration = steady_clock::now() - start;
auto millisec = duration_cast<milliseconds>(duration).count();
std::cout << millisec << "ms\n";
```

* Strácame vlastnosť, že `for_each` pôjde v poradí
* Lacná operácia nad prvkom nemusí zaplatiť réžiu paralelizácie

Note: Použijeme `rnd` a `using namespace std::chrono` z predchádzajúceho príkladu. Hodnoty sú obmedzené na 0 až 1 000 000, takže ich dvojnásobok sa pri 32-bitovom `int` zmestí. Historické meranie vo VS 2022 bolo približne dvakrát pomalšie než sekvenčný prechod.


## `std::transform`

```cpp
bool is_prime(int n) {
    if (n < 2) return false;
    for (int i = 2; i <= n / i; ++i) {
        if (n % i == 0) return false;
    }
    return true;
}

std::vector<int> v(10000);
std::vector<unsigned char> res(v.size());
std::iota(v.begin(), v.end(), 0);

std::transform(std::execution::par, v.begin(), v.end(), res.begin(),
               [](int value) { return is_prime(value); });
```

* Drahšia nezávislá práca nad každým prvkom je lepší kandidát na paralelizáciu
* Výstup nie je `vector<bool>`: rôzne vlákna nesmú zapisovať do zdieľaného bitového úložiska

Note: Historické meranie vo VS 2022 malo prechod medzi výhodnosťou `seq` a `par` približne pri 100 000 prvkoch. Hranica závisí od dát, stroja a backendu, preto ju treba odmerať znova.


## Nástrahy paralelného spracovania

* Réžia plánovania môže prevýšiť úsporu práce, najmä pri krátkych alebo lacných úlohách
* Nespoliehame sa na konkrétny počet vlákien, poradie ani súbežný beh jednotlivých volaní
* Preferujeme nezávislé vstupy a výstupy; zdieľané zápisy pri `par` musíme synchronizovať
* Pri `par_unseq` a `unseq` navyše nesmieme používať vektorizácii nebezpečné operácie, napríklad zamykanie mutexu
* Najprv korektný sekvenčný algoritmus, potom meranie a až potom výber policy

---

## Čo si odniesť

* Typ vlastníctva vyberáme podľa životnosti, nie podľa počtu miest používajúcich objekt
* Kontajner vyberáme podľa operácií, usporiadania, pamäte a stability odkazov
* Pred použitím algoritmu overíme jeho predpoklady: platné rozsahy, poradie a pripravený výstup
* Paralelizácia nemení požiadavku na korektnosť; jej prínos overujeme meraním

---

# Bonus: ďalšie časti knižnice

* Samostatné ukážky na doplnenie podľa času alebo na samoštúdium
* `bitset`, matematické konštanty, náhodné čísla, čas, regex a filesystem

---

## `std::bitset<N>`

* Akoby `std::array<bool, N>`, ale používa menej pamäte, podobne ako `std::vector<bool>`
* Nie je to kontajner, takže nemá iterátory
* Operácie s bitmi ako `&`, `|`, `^`, `<<`, `>>` sú definované
* Funkcie ako `count`, `size`, `test`, `set`, `reset`, `flip`

```cpp
std::bitset<8> b;
b.set(2);
b.count(); // 1
b.to_string(); // "00000100"
b.to_ulong(); // 4
```

---

## Matematické konštanty

* Konečne nemusíme používať `M_PI` a podobne
* V hlavičkovom súbore `<numbers>` sú definované konštanty ako `pi`, `e`, `log2e`, `sqrt2` a podobne

```cpp
std::cout << std::numbers::pi << '\n'; // 3.14159
```

---

## Random numbers

* Starý `rand` už pomaly dosluhuje (a nikdy nebol dobrý)
* **Nie sú vhodné na kryptografické účely**

```cpp
std::random_device rd;

std::mt19937 gen(rd()); // std::default_random_engine

std::uniform_int_distribution<int> dis(1, 6); // fair dice

for (size_t i = 0; i < 30; ++i) {
    std::cout << dis(gen); // e.g. 223646345444643431415515513225
}
```


## Replacement for `rand`

```cpp
int rnd() { // 0..maxint
	static std::random_device rd;
	static std::default_random_engine gen(rd());
	static std::uniform_int_distribution<int> dis;

	return dis(gen);
}
```

---

## Čas

* `<chrono>` obsahuje všetko potrebné na meranie a prácu s časom
* `std::chrono::system_clock`, `std::chrono::steady_clock`, `std::chrono::high_resolution_clock`
* Na meranie trvania použijeme monotónne `steady_clock`; `high_resolution_clock` nemusí byť monotónne

```cpp
auto start = std::chrono::steady_clock::now();

// do something

auto end = std::chrono::steady_clock::now();

auto duration = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
```

---

## Regular expressions

* `<regex>` obsahuje triedy na prácu s regulárnymi výrazmi

```cpp
std::regex r(R"((\d{4})-(\d{2})-(\d{2}))");

std::string s = "2024-11-17";

std::smatch m;
if (std::regex_match(s, m, r)) {
    std::cout << m[0] << '\n'; // 2024-11-17

    for (size_t i = 1; i < m.size(); ++i) {
        std::cout << m[i] << '\n';
    }
}

std::cout << std::regex_replace(s, r, "$2/$3/$1"); // 11/17/2024
```

---

## `std::filesystem`

* Práca s súborovým systémom
* Pracuje aj s POSIX aj s Windows

```cpp
const auto options = std::filesystem::directory_options::skip_permission_denied;
for (const auto& entry : std::filesystem::recursive_directory_iterator(".", options)) {
    std::cout << entry.path() << '\n';
}
```

* Cesty, kopírovanie a mazanie súborov rieši rovnaká knižnica
* Operácie môžu zlyhať: vyhodia `filesystem_error`, alebo pri príslušnom overloade nastavia `std::error_code`

Note: Ukážku spúšťame v malom pracovnom adresári, nie nad celým koreňovým súborovým systémom. `skip_permission_denied` preskočí nedostupné adresáre, ale nepotláča všetky druhy chýb. Kopírovanie a mazanie predvádzame iba na vlastných dočasných súboroch.

---

## `<bit>`: operácie nad bitmi od C++20

* Funkcie pre neznamienkové celočíselné typy, často mapované na inštrukcie procesora
* `popcount` počíta jednotkové bity; `countl_zero` a `countr_zero` nuly zľava a sprava
* `bit_floor` a `bit_ceil` hľadajú najbližšiu mocninu dvojky nadol a nahor

```cpp
#include <bit>

static_assert(std::popcount(0b101100u) == 3);
static_assert(std::countr_zero(0b101100u) == 2);
static_assert(std::has_single_bit(16u));
static_assert(std::bit_width(9u) == 4);
static_assert(std::bit_floor(9u) == 8u);
static_assert(std::bit_ceil(9u) == 16u);
static_assert(std::rotl(0b1011u, 2) == 0b101100u);
```

* `rotl` a `rotr` rotujú v celej šírke typu: vypadnuté bity vracajú na opačný koniec

Note: `bit_width` je počet bitov potrebných na zápis hodnoty, pre nulu vracia 0. `countl_zero(0u)` aj `countr_zero(0u)` vracajú počet hodnotových bitov typu. `bit_floor(0u)` je 0, `bit_ceil(0u)` je 1. Pri `bit_ceil` musí byť výsledná mocnina dvojky reprezentovateľná v danom type, inak je správanie nedefinované.

---

## `<bit>`: čítanie big-endian dát

* Sieťový protokol alebo súbor môže ukladať najvýznamnejší bajt ako prvý (*big-endian*)
* `std::endian` od C++20 zistí natívne poradie bajtov
* `std::byteswap` od C++23 obráti poradie bajtov celočíselnej hodnoty

```cpp
#include <bit>
#include <cstdint>
#include <cstring>

static_assert(std::endian::native == std::endian::little ||
              std::endian::native == std::endian::big);

const std::uint8_t bytes[] = { 0x12, 0x34, 0x56, 0x78 };
std::uint32_t value;
std::memcpy(&value, bytes, sizeof(value));
if (std::endian::native == std::endian::little) {
    value = std::byteswap(value);
}
assert(value == 0x12345678u);
```

* Na big-endian stroji netreba nič meniť; na little-endian bajty prehodíme

Note: `memcpy` nepotrebuje zarovnaný vstupný buffer a neporušuje pravidlá aliasingu, na rozdiel od dereferencovania `reinterpret_cast<const std::uint32_t*>(bytes)`. Príklad predpokladá dostupnosť presných typov `uint8_t` a `uint32_t`; statická kontrola vylučuje zmiešané poradie bajtov. `byteswap` nemení poradie bitov v jednotlivých bajtoch a vyžaduje celočíselný typ bez padding bitov. Pred C++23 sa podobné konverzie riešili napríklad platformovou funkciou `ntohl`.

---

# ĎAKUJEM

## Otázky?