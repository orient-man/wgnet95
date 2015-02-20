## TODO: Monady

Marcin Malinowski

- Twitter: [@orientman](https://twitter.com/orientman)
- GitHub: https://github.com/orient-man
- Blog: https://orientman.wordpress.com/

---

## Abstrakt

TODO:

---

## O mnie

- Tata^2, mąż humanistki, mól książkowy, uparciuch, programista, konferencjoholik.  Don Kichot walczący z entropią. Kocha sprzeczności i humor. Wierzy w przypadek. Piwny filozof. W nielicznych wolnych chwilach harata w gałę (na bramce).

- Basic, Turbo Pascal/C, Assembler, Clipper, MS Access, Visual Basic, Java-XML :), C++, C#, JavaScript, F#...  i ze wszystkiego miałem frajdę, ale nie za wszystkim tęsknię.

- Absolwent informatyki i matematyki na UW. Tech lead w firmie Piątka.

***

## Spis rzeczy

TODO:

***

### Monadic null checking aka null propagator

F# dziś:
```fsharp
let (>>=) x y = Option.bind y x // generalnie mało przydatne
```

C# 2015:
```csharp
var bestValue = points?.FirstOrDefault()?.X ?? -1;
```

***

# Jeśli C# jest już językiem funkcyjnym to...

---

<!-- .slide: data-background="./images/skull.png" style="top: -50px !important;" -->
## Gdzie te Monady?

> "Monady – byty duchowe; nie mają charakteru czasowego ani przestrzennego"

Note:
 - zapytajmy eksperta

---

<!-- .slide: style="top: -100px !important;" -->
![Barbie](./images/barbie_monad.png)

---

### Tako rzecze źródło wszelkiej wiedzy

"Monada jest rodzajem <font color="#fa0">konstruktora abstrakcyjnego typu danych</font> [...] Monady pozwalają programiście <font color="#fa0">sprzęgać ze sobą kolejno wykonywane działania</font> i budować potoki danych, w których każda akcja jest materializacją wzorca <font color="#fa0">dekoratora z dodatkowymi regułami przetwarzającymi</font>.

Formalnie monadę tworzy się definiując dwie operacje – wiązanie (ang. <font color="#fa0">bind</font>) i powrót (ang. <font color="#fa0">return</font>) [...]"

http://pl.wikipedia.org/wiki/Monada_%28programowanie%29

---

<!-- .slide: data-transition="convex" -->
![Ginger](./images/ginger.jpeg)

---

### Ostatnia szansa: przez przykład
```csharp
public static Task<T> ToTask<T>(this T value) // aka "unit" lub "return"
{
    return Task<T>.Factory.StartNew(() => value);
}

public static Task<B> Bind<A, B>(this Task<A> a, Func<A, Task<B>> func)
{
    return a.ContinueWith(prev => func(prev.Result)).Unwrap();
}

public static Task<C> SelectMany<A, B, C>(
    this Task<A> a, Func<A, Task<B>> func, Func<A, B, C> select)
{
    return a.Bind(
        aval => func(aval).Bind(bval => select(aval, bval).ToTask()));
}
```

---

### Co to robi?

```csharp
Func<Task<int>> compute3 = () => 3.ToTask();
Func<int, int, Task<int>> prod = (x, y) => (x * y).ToTask();
Func<int, int, Task<int>> add = (x, y) => (x + y).ToTask();

var r =
    from a in compute3()
    from b in prod(a, 2)
    from c in add(b, 4)
    select c.ToString();

r.Result.Should().Be("10");
```

---

### Co kompilator tłumaczy na:

```csharp
var r =
    compute3()
        .SelectMany(a => prod(a, 2), (a, b) => new { a, b })
        .SelectMany(ab => add(ab.b, 4), (ab, c) => c.ToString());
```

---

### Przykłady typów monadycznych w C# ###

- ``Nullable<T>``
- ``Func<T>``
- ``Lazy<T>``
- ``Task<T>``
- ``IEnumerable<T>``

---

### Ograniczenia Monad w C# ###

- LINQ jest zaprojektowany do zapytań (zaskoczenie :)
    - Brak instrukcji sterujących: if/then/else, pętli etc.
- Brak uniwersalnego wsparcia dla typu "monadycznego" na poziomie języka np.: _notacja do_ w Haskellu, _computational expressions_ w F#

<!-- .element: class="fragment" -->
Stąd każdy typ monadyczny, aby w pełni się nim cieszyć wymaga zmian w składni C#.

---

## Computational Expressions w F# ##

Workflow _async_ będący pierwowzorem dla składni _async/await_ w C#:

```fsharp
open System.IO
open System.Net

let downloadUrl(url : string) = async {
    let request = HttpWebRequest.Create(url)
    use! response = request.AsyncGetResponse()
    let stream = response.GetResponseStream()
    use reader = new StreamReader(stream)
    return! reader.AsyncReadToEnd()
}
```

---

...co kompiltor przetłumaczy na:
```fsharp
async.Delay(fun () ->
    let request = HttpWebRequest.Create(url)
    async.Bind(request.AsyncGetResponse(), fun response ->
        async.Using(response, fun response ->
            let stream = response.GetResponseStream()
            async.Using(new StreamReader(stream), fun reader ->
                reader.AsyncReadToEnd()))))
```

<!-- .element: class="fragment" -->
![http://tia.mat.br/blog/html/2012/09/29/asynchronous_i_o_in_c_with_coroutines.html](./images/callbacks.jpg)

---

...gdzie ``async`` jest instancją klasy ``AsyncBuilder``:

```fsharp
type AsyncBuilder =
    class
        new AsyncBuilder : unit -> AsyncBuilder
        member this.Bind : Async<'T> * ('T -> Async<'U>) -> Async<'U>
        member this.Combine : Async<unit> * Async<'T> -> Async<'T>
        member this.Delay : (unit -> Async<'T>) -> Async<'T>
        member this.For : seq<'T> * ('T -> Async<unit>) -> Async<unit>
        member this.Return : 'T -> Async<'T>
        member this.ReturnFrom : Async<'T> -> Async<'T>
        member this.TryFinally : Async<'T> * (unit -> unit) -> Async<'T>
        member this.TryWith : Async<'T> * (exn -> Async<'T>) -> Async<'T>
        member this.Using : 'T * ('T -> Async<'U>) -> Async<'U>
        member this.While : (unit -> bool) * Async<unit> -> Async<unit>
        member this.Zero : unit -> Async<unit>
    end
```

<!-- .element: class="fragment" -->
Na szczęście, aby używać LINQ-a nie musimy znać w każdym szczególe implementacji _LINQ Providera_ - to samo dotyczy _async_ i innych workflowów w F#.

***

![Rękopis znaleziony w Saragossie](./images/rekopis.jpg)

***

## Bibliografia

- Blog: [F# for fun and profit](http://fsharpforfunandprofit.com/) - skarbiec!
- Artykuł: [Why Functional Programming Matters](http://www.cse.chalmers.se/~rjmh/Papers/whyfp.html)
- Książka: [Real-World Functional Programming: With Examples in F# and C# - Petricek & Skeet](http://www.amazon.com/Real-World-Functional-Programming-With-Examples/dp/1933988924)

Materiały:

- Slajdy: ?
- Źródłowce: ?

---

### Bibliografia (Monady)

- Video: [Mike Hadlow on Monads](http://vimeo.com/21705972) - prościej się nie da?
- Video: [Scott Wlaschin - Railway Oriented Programming -- error handling in functional languages](http://vimeo.com/97344498)
- Video: [Greg Meredith - Monadic Design Patterns for the Web - Introduction to Monads](http://channel9.msdn.com/Series/C9-Lectures-Greg-Meredith-Monadic-Design-Patterns-for-the-Web/C9-Lectures-Greg-Meredith-Monadic-Design-Patterns-for-the-Web-Introduction-to-Monads) - abstrakcyjnie, ale zjadliwie
- Blog: [Fabulous adventures in coding: Monads, parts 1-13](http://ericlippert.com/category/monads/) - wyczerpująco

***

# Aaa... pytania?
