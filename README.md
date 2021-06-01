# System.Text.Languages
May 31, 2021

## An experiment in minimalism for interpreters of LISP-ish languages
(in about 200 lines of C#)

***

### Table of contents
- [API](#api)
  - [Symbol](#class-symbol) class
  - [ISymbolProvider](#interface-isymbolprovider) interface
  - [IEnvironment](#interface-ienvironment) interface
  - [ILanguage](#interface-ilanguage) interface
  - [IEvaluator](#interface-ievaluator) interface
  - [Closure](#delegate-closure) delegate
- [Base and default implementations](#base-and-default-implementations)
  - [SymbolProviderBase](#abstract-class-symbolproviderbase) abstract class
  - [DefaultSymbolProvider](#class-defaultsymbolprovider) class
  - [Environment](#class-environment) class
  - [Evaluator](#class-evaluator) class
- [Deriving a sample interpreter](#deriving-a-sample-interpreter)
- [A more advanced example](#a-more-advanced-example)
- [Complete sources](#complete-sources)
  - [Namespace.cs](#namespacecs-52-lines)
  - [Runtime.cs](#runtimecs-150-lines)
- [License and disclaimer](#license-and-disclaimer)
- [Contact info](#contact-info)

***

## API

Complete source: [Namespace.cs](#namespacecs-52-lines)

The API consists in:

- the [Symbol](#class-symbol) class
- the [ISymbolProvider](#interface-isymbolprovider) interface
- the [IEnvironment](#interface-ienvironment) interface
- the [ILanguage](#interface-ilanguage) interface
- the [IEvaluator](#interface-ievaluator) interface
- the [Closure](#delegate-closure) delegate

### class Symbol
A **`Symbol`** interns all the literal occurrences that denote it, during both parsing and evaluation of the final S-expression.

Examples may include the "**`true`**" and "**`false`**" boolean constants, or the "**`null`**" special value, or arithmetic operators such as "**`+`**", etc.

The true identity of a symbol is its **`Index`**, allocated (during parsing) and maintained (during evaluation), by an implementation of [**`ISymbolProvider`**](#interface-isymbolprovider).

The sign of **`Index`** indicates the linguistic origin of the **`Symbol`**: either negative or zero for language builtins (ie, operators, special signs, keywords, etc, that pertain to the language's definition)
or strictly positive for programmer-defined symbols (ie, identifiers encountered in programs).

There are exactly 8 language builtins (aka "core symbols") common to all LISP-ish languages, 2 of which are optionally defined *semantically*:

#### core symbols

- **`Unknown`** is the "unknown symbol" which can be used to signal an unexpected character during parsing and/or an undefined identifier during evaluation
- **`Open`** (resp. **`Close`** ) is the open (resp. close) parenthesis, used to build non-atomic S-expressions
- **`Quote`** is the quote symbol, used to build complex, unevaluated S-expressions, eg, for representations in "code as data" scenarios
- **`Params`** is the optional "**`params`**" keyword, whose role could be similar to JavaScript's "**`arguments`**"
- **`This`** is the optional "**`this`**" keyword, whose role could be similar to C++/Java/JavaScript/C#'s "**`this`**"
- **`Let`** is the "**`let`**" binding keyword, used to define bindings from symbols to values in lexical scopes
- **`Lambda`** is the "**`=>`**" lambda abstraction keyword, used to define first-class anonymous functions, playing the same role as lambda abstractions in lambda calculus

Both **`Params`** and **`This`** are optionally defined *semantically* in the sense that it is the responsibility of an implementation of [**`IEvaluator`**](#interface-ievaluator) to decide whether a specific semantics is attached to them or not.

Obviously, if either or both of **`Params`** and **`This`** is/are optional, there won't be any need to have **`ISymbolProvider`** assign it/them the corresponding literal(s) either, although their unused **`Index`** will still remain unusable for anything else.

Symbols are just that: atoms which intern multiple occurrences of the same literals, be they builtins or programmer-defined identifiers.

They do not (nor does the implementation of **`ISymbolProvider`** either) know or care about which actual values and/or semantics they are attached to, at any particular point in time of the parsing or evaluation phases - this knowledge being the sole responsibility of an implementation of [**`IEnvironment`**](#interface-ienvironment).

Finally, note that not all "seemingly atomic" literals of the target language being represented are good candidates / suitable to have a **`Symbol`** attached to them, even as programmer-written only ones:
numeric constants such as "**`1792`**" or string literals such as " **`"Hello, world!"`** " aren't actually atomic from the **`ISymbolProvider`** implementer's standpoint.
Both of these are in fact composite constructs for the latter (of a presumed integral type represented in base 10 and of a character string type whose delimeter happens to be the double quote, resp.)
which are interpretable as constants only relatively to the host language, per se, of the implementation of **`ISymbolProvider`**, but arguably not relatively to that very implementation.

In the first case, "**`1792`**", indeed it wouldn't make much sense (at all) to try denoting for the target language a distinct **`Symbol`** for each and every of the concrete values of the **`System.Int32`** type coming from the CLR-based host language after parsing some user input.

```
public class Symbol
{
    public static readonly Symbol
        Unknown = new Symbol(0),
        Open = new Symbol(-1),
        Close = new Symbol(-2),
        Quote = new Symbol(-3),
        Params = new Symbol(-4),
        This = new Symbol(-5),
        Let = new Symbol(-6),
        Lambda = new Symbol(-7);
    public Symbol(int index) => Index = index;
    public override bool Equals(object obj) => ReferenceEquals(this, obj);
    public override int GetHashCode() => Index;
    public override string ToString() => $"[{GetType().Name}({Index})]";
    public int Index { get; }
}
```

### interface ISymbolProvider
**`ISymbolProvider`** is the contract for a bidirectional (ie, bijective, invertible) and mutable mapping between symbols and the literals that denote them.
The mutation of the mapping defined by an implementation of **`ISymbolProvider`** is append-only: one can only "append" a new bidirectional link between a [**`Symbol`**](#class-symbol) and the literal that denotes it.
In other words, one cannot change the literal that denotes a symbol already known to the **`ISymbolProvider`**, nor one can change the symbol denotated by a literal already known to the same.
The use of **`ISymbolProvider`** will fall into either one of two cases:

- during the ***initialization*** phase of a parser or evaluator, an implementation of **`ISymbolProvider`** is populated with all the bidirectional links of the language's builtin symbols and their corresponding literals;
such symbols will have their **`Index`** growing "downward" in the negative integers
- during the ***utilization*** phase of a parser or evaluator, the same implementation of **`ISymbolProvider`** can be used to either append a new bidirectional link between a programmer-defined symbol and its literal encountered for the first time, or to retrieve an already known symbol or literal (be it a builtin or not);
new programmer-defined symbols will have their **`Index`** growing "upward" in the (strictly) positive integers

Ideally, an implementation of **`ISymbolProvider`** should be the only piece of mutable state injected into (and owned by) an implementation of [**`ILanguage`**](#interface-ilanguage) or [**`IEvaluator`**](#interface-ievaluator) at construction time of the latter.
Thus, a thread-safe implementation of **`ISymbolProvider`** could - possibly, but not necessarily - allow to make its owner **`ILanguage`** or **`IEvaluator`** thread-safe as well, while a non-thread-safe implementation of the former will ***necessarily*** make the latter non-thread-safe, by very construction.

The **`ISymbolProvider`** contract's rationale is as follows:

- the method **`bool Contains(string literal)`** is for the client to determine whether a specific **`literal`** already denotes a symbol or not (be it a builtin or programmer-defined one)
- the method **`ISymbolProvider Include(string literal, out Symbol symbol, bool asBuiltin)`** is for the client to retrieve a specific **`symbol`** through its corresponding **`literal`**, after ensuring the bidirectional link between the two has been appended, if necessary and according to the (optional) **`asBuiltin`** hint; note the method's signature enables the client to make chained calls into the same (current) implementation of **`ISymbolProvider`** to define or retrieve multiple symbols "at once"
- the method **`Symbol Symbol(string literal, bool asBuiltin)`** has the same semantics as the **`Include`** method, but isn't chainable and instead directly returns the symbol of interest through its corresponding **`literal`** and (optional) **`asBuiltin`** hint
- the method **`string NameOf(Symbol symbol)`** is for the client to retrieve the specific literal which denotes a given **`symbol`**, under the assumption that the bidirectional link between the two ***must*** already exist

The overall practical utility of this **`ISymbolProvider`** contract's rationale will appear more clearly [when we derive our first interpreter](#deriving-a-sample-interpreter) from the default implementation of **`IEvaluator`** (ie, class [**`Evaluator`**](#class-evaluator)).

```
public interface ISymbolProvider
{
    bool Contains(string literal);
    ISymbolProvider Include(string literal, out Symbol symbol, bool asBuiltin = false);
    Symbol Symbol(string literal, bool asBuiltin = false);
    string NameOf(Symbol symbol);
}
```

### interface IEnvironment
As we have seen, the [**`Symbol`**](#class-symbol) class and [**`ISymbolProvider`**](#interface-isymbolprovider) contract are concerned with 3 things, and these 3 things only:

- the interning of all the occurrences of a literal that denotes a specific symbol (the purpose of the **`Symbol`** class)
- the allocation and persistence (if only in memory) of the unique **`Index`** held by instances of the **`Symbol`** class through a bidirectional and append-only mutable mapping between these symbols and their corresponding literals (the purpose of the **`ISymbolProvider`** contract)
- the distinction between builtin symbols that exclusively pertain to the language's definition itself (ie, special symbols or keywords, algebraic operators, etc) and programmer-defined symbols found in an input program to be parsed or interpreted (ie, identifiers for values or functions encountered in various lexical scopes) - a distinction inferred from the sign (strictly positive vs negative or zero) of the same **`Index`** identifying property

They say or do nothing about the binding of identifiable **`Symbol`** occurrences into actual values in the host language (eg, C# over the CLR) - either at:

- the language's definition time (ie, during the construction of, or the parsing performed by, a parser implementing the [**`ILanguage`**](#interface-ilanguage) contract), or
- the language's run-time (ie, during the construction of, or the evaluation performed by, an interpreter implementing the [**`IEvaluator`**](#interface-ievaluator) contract).

Thus, be it at language's definition time or at language's run-time, this binding from symbols to values is precisely the sole responsibility of the **`IEnvironment`** contract.

At the language's definition time, a "global" environment (ie, an implementation of **`IEnvironment`**) may (or may not) be provided by the host language to the parser (ie, **`ILanguage`**) or to the interpreter (ie, **`IEvaluator`**) being constructed. If no such "global" environment is explicitly provided by the host, it is up to the **`ILanguage`** or **`IEvaluator`** implementation to "infer" (ie, decide about) a suitable one, which shall include at least bindings for the [6 up to 8 core symbols](#core-symbols) aforementioned.

At the language's run-time (in the case of an interpreter implementing **`IEvaluator`**), implementations of **`IEnvironment`** are used to represent what is generally known as the tree of "stack frames" (or "activation records"), dynamically growing and shrinking after taking into account the lexical scoping rules of the language being interpreted vs its current run-time context which originated (or, was "seeded") from the initial, so-called "global environment".

The tree of **`IEnvironment`** implementations that are created at the language's run-time is induced by a child-to-parent relationship only:
an environment only knows about its parent environment, if any (there is no such parent for the global environment, which serves as the top-most ancestor, or root of this tree).

The **`IEnvironment`** contract's rationale is as follows:

- the method **`bool Contains(string literal)`** is for the client to determine whether a specific **`literal`** denotes, or not, a symbol that is bound to a value in the current environment (or in the nearest of its ancestors)
- the method **`bool Contains(Symbol symbol)`** is for the client to determine whether a specific, already known **`symbol`** is bound, or not, to a value in the current environment (or in the nearest of its ancestors)
- the method **`bool TryGet(string literal, out object value)`** is for the client to attempt retrieving a specific symbol's **`value`** through its corresponding **`literal`**, if and only if such a symbol exists ***and*** is bound to a value in the current environment (or in the nearest of its ancestors)
- the method **`bool TryGet(Symbol symbol, out object value)`** is for the client to attempt retrieving a specific, already known **`symbol`**'s **`value`**, if and only if that symbol is bound to a value in the current environment (or in the nearest of its ancestors)
- the method **`IEnvironment Set(Symbol symbol, object value)`** is for the client to forcefully bind, in the current environment, a specific, already known **`symbol`** to a given **`value`**, thus possibly "overriding" (or, "shadowing") all the symbol-to-value binding(s) previously known in one or more of the environment's ancestors
- the property **`ISymbolProvider SymbolProvider`** is for the client to access the underlying **`ISymbolProvider`** which is used by the current environment to resolve literals into symbols whenever necessary

Note that unlike the stakes at hands for an implementation of **`ISymbolProvider`**, an implementation of **`IEnvironment`** may choose to be thread-safe, but ideally, this should never be required, as the run-time tree of environment implementations should never be meant to support sharing between (possibly concurrent) calls into the **`IEvaluator`** implementation spawning them.

Finally, the single most fundamental semantic assumption (and actual hard requirement) for an implementation of **`IEnvironment`** should be that any child environment ***must*** be constructed to use the same run-time implementation of **`ISymbolProvider`** as is already in use by its parent,
most likely the very same implementation which was used to construct the initial global environment while initializing the ambient **`ILanguage`** or **`IEvaluator`** implementation.

```
public interface IEnvironment
{
    bool Contains(string literal);
    bool Contains(Symbol symbol);
    bool TryGet(string literal, out object value);
    bool TryGet(Symbol symbol, out object value);
    IEnvironment Set(Symbol symbol, object value);
    ISymbolProvider SymbolProvider { get; }
}
```

### interface ILanguage
There isn't much to say about the fairly loose **`ILanguage`** interface beyond the fact that it is the contract to implement (partially or in full) for a general purpose language pre- and/or post-, or sole processor.

The **`ILanguage`** contract's rationale is as follows:

- a common "pre-processing" operation performed over a textual language **`input`** phrase is to parse it into another intermediate form, prior to further processing; hence, the **`object Parse(string input)`** method
- since it is contemplated that the client may also wish, in some cases, to provide contextual, or "out-of-band" (eg, configuration) information for the parsing operation to execute as expected, the contract makes for this provision through the **`object Parse(object context, string input)`** method overload, with its additional **`context`** parameter
- a common "post-processing" operation performed over a known intermediate form (**`value`**) of what once possibly was a textual language input phrase, or the result of an evaluation thereof, is to turn it back into a serializable representation (**`type`**); hence the **`object Serialize(Type type, object value)`** method
- finally, turning such a value back into a textual form, say, through the **`string ToString(object value)`** method, can quite naturally be seen (and implemented accordingly) as just a special case of serialization where the first (**`type`**) argument to **`Serialize`** (the serialization target type) would be implied and equal to **`typeof(string)`** - thus making **`ToString`** more of a convenient, specialized helper around **`Serialize`** than a truly distinct facility

But we will see that a specific implementation of the [**`IEvaluator`**](#interface-ievaluator) contract, which derives from **`ILanguage`**, can also afford making a few more assumptions, which are meaningful especially from the standpoint of the implementer of an interpreter.

```
public interface ILanguage
{
    object Parse(string input);
    object Parse(object context, string input);
    object Serialize(Type type, object value);
    string ToString(object value);
}
```

### interface IEvaluator
Before we delve into the purpose of the **`IEvaluator`** contract, it is now a good time to try being more precise about what we mean exactly by a facility for the implementation of "interpreters of LISP-ish languages" in C#.

Assuming the reader is reasonably familiar with the origins and notion of S-expressions, it is worth pointing out right away that, for the sake of both the implementation simplicity (ie, straightforwardness) and the design flexibility, we will ***not*** restrict ourselves to only the historical/canonical form of S-expressions based solely on ordered pairs and the special **`NIL`** symbol used in the original LISP:

**`( x ( y ( z NIL ) ) )`**

rather, by "S-expressions", we will refer more specifically to what in our natural managed execution environment (ie, the CLR) can be recursively constructed from these three fairly simple construction rules:

1. a S-expression is either an "***atom***" or a "***list of***" S-expressions
2. an "***atom***" is either the special **`null`** value as known to the CLR, or any other non-**`null`** value which can be boxed on the heap ***and isn't*** an instance of **`object[]`**
3. a "***list of***" S-expressions is an instance of **`object[]`** (possibly empty) whose every element (if any) is itself a S-expression

Thus,

**`null`** (which is the **`null`** ***atom***)

or

**`new object[] { 123, "456", null, Symbol.Unknown, true }`** (which is a ***list of*** valid S-expressions)

or

**`new object[0]`** (which is an empty ***list of*** S-expressions)

or

**`1789`** (which is an ***atom*** whose run-time type is **`System.Int32`**)

or

**`new List<int>()`** (which is an ***atom*** whose run-time type is **`System.Collections.Generic.List<int>`**)

or

**`typeof(void)`** (which is an ***atom*** whose run-time type is **`System.Type`**)

are all valid S-expressions that could be considered (or computed) eventually by some specific implementation of **`IEvaluator`**, while

**`new byte[100].AsSpan(50)`** (forbidden)

clearly isn't one, [for obvious reasons](https://docs.microsoft.com/en-us/dotnet/api/system.span-1) (wrt. CLR types whose values can vs. cannot live on the heap).

We should note then that, unlike **`NIL`** in the original LISP, the special **`null`** value isn't meant to be an alias for something else just for the sake of terminating a S-expression, but rather is simply thrown into the mix of allowed atomic values that can appear anywhere in a non-empty list.

With these design choices and not-too-limited representational flexibility in mind, we can now look at the **`IEvaluator`** contract's rationale per se:

- the **`object Quote(object expression)`** method is simply required to return exactly **`new object[] { Symbol.Quote, expression }`**, that is, a quoted S-expression from the one provided; this, in order to follow the long LISP tradition of representing "code as data" in a homoiconic manner
- the **`object Evaluate(string input)`** method is simply required to be equivalent to calling **`Evaluate(null, input)`**
- the **`object Evaluate(IEnvironment environment, string input)`** method shall:
  1. accept a possibly null global **`environment`** (and infer its own if that's the case), then
  2. parse the **`input`** by delegating this to a call into the implementation of the **`object Parse(object context, string input)`** (inherited from the **`ILanguage`** contract) where the **`context`** is set to be the **`environment`**, and
  3. finally, rely on the implementation of the **`object Evaluate(IEnvironment environment, object expression)`** method to perform the sort of evaluation that has been pertaining for long already to LISP's **`eval`**
- the **`ISymbolProvider SymbolProvider`** property is simply required to return the necessarily non-null [**`ISymbolProvider`**](#interface-isymbolprovider) which was injected at construction time of the current **`IEvaluator`** in the first place

```
public interface IEvaluator : ILanguage
{
    object Quote(object expression);
    object Evaluate(string input);
    object Evaluate(IEnvironment environment, string input);
    object Evaluate(IEnvironment environment, object expression);
    ISymbolProvider SymbolProvider { get; }
}
```

### delegate Closure
The **`Closure`** delegate type is merely a convenience facility which allows clients of [**`IEvaluator`**](#interface-ievaluator) that have obtained a reference to a closure in the target language (after evaluating the whole input program), to invoke, *in the host language* (ie, C#) the said closure with thus a somewhat deferred execution in the same run-time environment which gave birth to the implementation of **`IEvaluator`**.

```
public delegate object Closure(IEnvironment environment, params object[] args);
```

## Base and default implementations

Complete source: [Runtime.cs](#runtimecs-150-lines)

The base and default implementations consist in:

- the [SymbolProviderBase](#abstract-class-symbolproviderbase) abstract class
- the [DefaultSymbolProvider](#class-defaultsymbolprovider) class
- the [Environment](#class-environment) class
- the [Evaluator](#class-evaluator) class

### abstract class SymbolProviderBase
```
public abstract class SymbolProviderBase : ISymbolProvider
{
    protected abstract void Add(string literal, Symbol symbol);
    protected abstract bool TryGet(string literal, out Symbol symbol);
    protected SymbolProviderBase(IEnumerable<KeyValuePair<string, Symbol>> core)
    {
        if (core != null)
            foreach (var item in core)
            {
                if (item.Value?.Index == -SymbolCount)
                    Add(item.Key, item.Value);
                else
                    throw new InvalidOperationException();
            }
    }
    public bool Contains(string literal) => TryGet(literal, out var ignored);
    public ISymbolProvider Include(string literal, out Symbol symbol, bool asBuiltin = false)
    {
        symbol = Symbol(literal, asBuiltin);
        return this;
    }
    public Symbol Symbol(string literal, bool asBuiltin = false)
    {
        Symbol symbol;
        if (!TryGet(literal, out symbol)) Add(literal, symbol = new Symbol(asBuiltin ? -SymbolCount : SymbolCount));
        return symbol;
    }
    public abstract string NameOf(Symbol symbol);
    public abstract int SymbolCount { get; }
}
```

### class DefaultSymbolProvider
```
public class DefaultSymbolProvider : SymbolProviderBase
{
    private readonly IDictionary<string, Symbol> symbols = new Dictionary<string, Symbol>();
    private readonly IDictionary<Symbol, string> names = new Dictionary<Symbol, string>();
    protected override void Add(string literal, Symbol symbol)
    {
        symbols.Add(literal, symbol);
        names.Add(symbol, literal);
    }
    protected override bool TryGet(string literal, out Symbol symbol) =>
        symbols.TryGetValue(literal, out symbol);
    public DefaultSymbolProvider(IEnumerable<KeyValuePair<string, Symbol>> core = null) : base(core) { }
    public override string NameOf(Symbol symbol) => names[symbol];
    public override int SymbolCount => symbols.Count;
}
```

### class Environment
```
public class Environment : Dictionary<Symbol, object>, IEnvironment
{
    private readonly IEnvironment parent;
    public Environment(IEnvironment parent, ISymbolProvider symbolProvider = null) =>
        SymbolProvider = (this.parent = parent)?.SymbolProvider ?? symbolProvider ?? throw new ArgumentNullException(nameof(symbolProvider), "cannot be null");
    public bool Contains(string literal) => TryGet(literal, out var ignored);
    public bool Contains(Symbol symbol) => TryGet(symbol, out var ignored);
    public bool TryGet(string literal, out object value) =>
        TryGet(SymbolProvider.Symbol(literal), out value);
    public virtual bool TryGet(Symbol symbol, out object value) =>
        TryGetValue(symbol, out value) || (parent != null && parent.TryGet(symbol, out value) && Set(symbol, value) != null);
    public virtual IEnvironment Set(Symbol symbol, object value)
    {
        this[symbol] = value;
        return this;
    }
    public ISymbolProvider SymbolProvider { get; }
}
```

### class Evaluator
```
public class Evaluator : IEvaluator
{
    internal class Builtin { internal readonly Closure Closure; internal Builtin(Closure closure) => Closure = closure; }
    protected static readonly object[] Empty = new object[0];
    protected static object Clone(object expression) => expression is object[]? ((object[])expression).Select(value => Clone(value)).ToArray() : expression;
    protected static object Eval(IEnvironment environment, object expression)
    {
        if (expression is object[])
        {
            var list = (object[])expression;
            object lambda = null;
            Symbol symbol;
            if (1 < list.Length)
            {
                int ip;
                if ((symbol = list[0] as Symbol) != null && symbol == Symbol.Quote) return list[1];
                if ((list[ip = 0] is Builtin) || (list[ip = 1] is Builtin)) return ((Builtin)list[ip]).Closure(environment, list);
                if (((symbol = list[ip = 1] as Symbol) != null && symbol.Index < Symbol.This.Index) || ((symbol = list[ip = 0] as Symbol) != null && symbol.Index < Symbol.This.Index)) return ((Builtin)(list[ip] = new Builtin((Closure)Eval(environment, symbol)))).Closure(environment, list);
                if ((expression = (list[0] as Closure) ?? (lambda = Eval(environment, list[0]))) is Closure)
                {
                    var args = (object[])Array.CreateInstance(typeof(object), list.Length - 1);
                    for (var i = 0; i < args.Length; i++) args[i] = Eval(environment, list[i + 1]);
                    return ((Closure)(lambda != null ? list[0] = lambda : expression))(environment, args);
                }
                return list.Select((arg, at) => 0 < at ? Eval(environment, arg) : expression).Last();
            }
            if (0 < list.Length)
            {
                if (list[0] is Builtin) return ((Builtin)list[0]).Closure(environment, list);
                if ((symbol = list[0] as Symbol) != null && symbol.Index < Symbol.This.Index) return ((Builtin)(list[0] = new Builtin((Closure)Eval(environment, symbol)))).Closure(environment, list);
                return (expression = (list[0] as Closure) ?? (lambda = Eval(environment, list[0]))) is Closure ? ((Closure)(lambda != null ? list[0] = lambda : expression))(environment, Empty) : expression;
            }
            return Empty;
        }
        return expression is Symbol ? (environment.TryGet((Symbol)expression, out var value) ? value : Symbol.Unknown) : expression;
    }
    protected Evaluator(ISymbolProvider symbolProvider) =>
        SymbolProvider = symbolProvider ?? throw new ArgumentNullException(nameof(symbolProvider), "cannot be null");
    protected virtual IEnvironment NewScope(IEnvironment parent, ISymbolProvider symbolProvider = null) =>
        new Environment(parent, symbolProvider);
    protected virtual IEnvironment Builtins(IEnvironment global, object expression) =>
        global.Set(Symbol.Let, (Closure)Definition).Set(Symbol.Lambda, (Closure)Abstraction);
    protected virtual object Tokenize(object context, string input, out int matched, ref int offset)
    {
        matched = 0;
        return null;
    }
    protected object ParseSExpression(object context, string input, object lookAhead, int matched, ref int offset)
    {
        object value = lookAhead;
        if (lookAhead != null && Symbol.Quote.Equals(value))
        {
            offset += matched; lookAhead = Tokenize(context, input, out matched, ref offset);
            value = Quote(ParseSExpression(context, input, lookAhead, matched, ref offset));
        }
        else if (lookAhead != null && Symbol.Open.Equals(value))
        {
            var list = new List<object>();
            offset += matched; while ((lookAhead = Tokenize(context, input, out matched, ref offset)) != null && !Symbol.Unknown.Equals(value = lookAhead) && !Symbol.Close.Equals(value)) list.Add(ParseSExpression(context, input, lookAhead, matched, ref offset));
            if (lookAhead != null) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
            else throw new Exception($"unexpected EOF at {offset}");
            value = list.ToArray();
        }
        else if (lookAhead != null) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
        else throw new Exception($"unexpected EOF at {offset}");
        return value;
    }
    protected virtual object Parse(IEnvironment environment, string input)
    {
        var offset = 0; var parse = ParseSExpression(environment = environment ?? NewScope(null, SymbolProvider), input, Tokenize(environment, input, out var matched, ref offset), matched, ref offset);
        return Tokenize(environment, input, out var ignored, ref offset) == null ? parse : throw new Exception($"unexpected '{input[offset]}' at {offset}");
    }
    protected virtual object Params(IEnvironment environment, params object[] args) => Symbol.Unknown;
    protected virtual object This(IEnvironment environment, params object[] args) => Symbol.Unknown;
    protected virtual object Definition(IEnvironment environment, params object[] args)
    {
        var scope = ((object[])args[1]).Cast<object[]>().Aggregate(NewScope(environment), (local, let) => local.Set((Symbol)let[0], Eval(local, let[1])));
        return args.Skip(2).Select(arg => Eval(scope, arg)).LastOrDefault();
    }
    protected virtual object Abstraction(IEnvironment environment, params object[] args)
    {
        Closure @this = null;
        return @this =
            (scope, @params) =>
            {
                var local = new Environment(environment);
                var formals = args[0] is Symbol ? new object[] { args[0] } : (object[])args[0];
                var varargs = 0 < formals.Length && formals[formals.Length - 1] is object[]? formals.Length - 1 : -1;
                var named = varargs < 0 ? formals.Length : varargs;
                for (var i = 0; i < named; i++) local.Set((Symbol)formals[i], i < @params.Length ? @params[i] : Symbol.Unknown);
                if (0 <= varargs) local.Set((Symbol)((object[])formals[varargs])[0], varargs < @params.Length ? @params.Skip(varargs).ToArray() : (object)Symbol.Unknown);
                local.Set(Symbol.This, This(local, @this)).Set(Symbol.Params, Params(local, new[] { (object)@params }));
                return Eval(local, args[2]);
            };
    }
    public static readonly IReadOnlyList<KeyValuePair<string, Symbol>> DefaultCore = new[]
    {
        new KeyValuePair<string, Symbol>(string.Empty, Symbol.Unknown),
        new KeyValuePair<string, Symbol>("(", Symbol.Open),
        new KeyValuePair<string, Symbol>(")", Symbol.Close),
        new KeyValuePair<string, Symbol>("`", Symbol.Quote),
        new KeyValuePair<string, Symbol>($"__{Guid.NewGuid():N}__", Symbol.Params),
        new KeyValuePair<string, Symbol>($"__{Guid.NewGuid():N}__", Symbol.This),
        new KeyValuePair<string, Symbol>("let", Symbol.Let),
        new KeyValuePair<string, Symbol>("=>", Symbol.Lambda)
    };
    public object Parse(string input) => Parse(null, input);
    public object Parse(object context, string input) => Parse(context as IEnvironment, input);
    public virtual object Serialize(Type type, object value) => throw new NotImplementedException();
    public string ToString(object value) => (string)Serialize(typeof(string), value);
    public object Quote(object expression) => new[] { Symbol.Quote, expression };
    public object Evaluate(string input) => Evaluate(null, input);
    public object Evaluate(IEnvironment environment, string input) =>
        Evaluate(environment = NewScope(environment, SymbolProvider), Parse(environment, input));
    public object Evaluate(IEnvironment environment, object expression) =>
        Eval(Builtins(environment ?? throw new ArgumentNullException(nameof(environment), "cannot be null"), Clone(expression)), expression);
    public ISymbolProvider SymbolProvider { get; }
}
```

## Deriving a sample interpreter
TBC...

## A more advanced example
TBC...

## Complete sources

### Namespace.cs (52 lines)
```
using System;
using System.Collections.Generic;

namespace System.Text.Languages
{
    public class Symbol
    {
        public static readonly Symbol Unknown = new Symbol(0), Open = new Symbol(-1), Close = new Symbol(-2), Quote = new Symbol(-3), Params = new Symbol(-4), This = new Symbol(-5), Let = new Symbol(-6), Lambda = new Symbol(-7);
        public Symbol(int index) => Index = index;
        public override bool Equals(object obj) => ReferenceEquals(this, obj);
        public override int GetHashCode() => Index;
        public override string ToString() => $"[{GetType().Name}({Index})]";
        public int Index { get; }
    }

    public interface ISymbolProvider
    {
        bool Contains(string literal);
        ISymbolProvider Include(string literal, out Symbol symbol, bool asBuiltin = false);
        Symbol Symbol(string literal, bool asBuiltin = false);
        string NameOf(Symbol symbol);
    }

    public interface IEnvironment
    {
        bool Contains(string literal);
        bool Contains(Symbol symbol);
        bool TryGet(string literal, out object value);
        bool TryGet(Symbol symbol, out object value);
        IEnvironment Set(Symbol symbol, object value);
        ISymbolProvider SymbolProvider { get; }
    }

    public interface ILanguage
    {
        object Parse(string input);
        object Parse(object context, string input);
        object Serialize(Type type, object value);
        string ToString(object value);
    }

    public interface IEvaluator : ILanguage
    {
        object Quote(object expression);
        object Evaluate(string input);
        object Evaluate(IEnvironment environment, string input);
        object Evaluate(IEnvironment environment, object expression);
        ISymbolProvider SymbolProvider { get; }
    }

    public delegate object Closure(IEnvironment environment, params object[] args);
}
```

### Runtime.cs (150 lines)
```
using System;
using System.Collections.Generic;
using System.Linq;

namespace System.Text.Languages.Runtime
{
    public abstract class SymbolProviderBase : ISymbolProvider
    {
        protected abstract void Add(string literal, Symbol symbol);
        protected abstract bool TryGet(string literal, out Symbol symbol);
        protected SymbolProviderBase(IEnumerable<KeyValuePair<string, Symbol>> core) { if (core != null) foreach (var item in core) { if (item.Value?.Index == -SymbolCount) Add(item.Key, item.Value); else throw new InvalidOperationException(); } }
        public bool Contains(string literal) => TryGet(literal, out var ignored);
        public ISymbolProvider Include(string literal, out Symbol symbol, bool asBuiltin = false) { symbol = Symbol(literal, asBuiltin); return this; }
        public Symbol Symbol(string literal, bool asBuiltin = false) { Symbol symbol; if (!TryGet(literal, out symbol)) Add(literal, symbol = new Symbol(asBuiltin ? -SymbolCount : SymbolCount)); return symbol; }
        public abstract string NameOf(Symbol symbol);
        public abstract int SymbolCount { get; }
    }

    public class DefaultSymbolProvider : SymbolProviderBase
    {
        private readonly IDictionary<string, Symbol> symbols = new Dictionary<string, Symbol>(); private readonly IDictionary<Symbol, string> names = new Dictionary<Symbol, string>();
        protected override void Add(string literal, Symbol symbol) { symbols.Add(literal, symbol); names.Add(symbol, literal); }
        protected override bool TryGet(string literal, out Symbol symbol) => symbols.TryGetValue(literal, out symbol);
        public DefaultSymbolProvider(IEnumerable<KeyValuePair<string, Symbol>> core = null) : base(core) { }
        public override string NameOf(Symbol symbol) => names[symbol];
        public override int SymbolCount => symbols.Count;
    }

    public class Environment : Dictionary<Symbol, object>, IEnvironment
    {
        private readonly IEnvironment parent;
        public Environment(IEnvironment parent, ISymbolProvider symbolProvider = null) => SymbolProvider = (this.parent = parent)?.SymbolProvider ?? symbolProvider ?? throw new ArgumentNullException(nameof(symbolProvider), "cannot be null");
        public bool Contains(string literal) => TryGet(literal, out var ignored);
        public bool Contains(Symbol symbol) => TryGet(symbol, out var ignored);
        public bool TryGet(string literal, out object value) => TryGet(SymbolProvider.Symbol(literal), out value);
        public virtual bool TryGet(Symbol symbol, out object value) => TryGetValue(symbol, out value) || (parent != null && parent.TryGet(symbol, out value) && Set(symbol, value) != null);
        public virtual IEnvironment Set(Symbol symbol, object value) { this[symbol] = value; return this; }
        public ISymbolProvider SymbolProvider { get; }
    }

    public class Evaluator : IEvaluator
    {
        internal class Builtin { internal readonly Closure Closure; internal Builtin(Closure closure) => Closure = closure; }
        protected static readonly object[] Empty = new object[0];
        protected static object Clone(object expression) => expression is object[]? ((object[])expression).Select(value => Clone(value)).ToArray() : expression;
        protected static object Eval(IEnvironment environment, object expression)
        {
            if (expression is object[])
            {
                var list = (object[])expression;
                object lambda = null;
                Symbol symbol;
                if (1 < list.Length)
                {
                    int ip;
                    if ((symbol = list[0] as Symbol) != null && symbol == Symbol.Quote) return list[1];
                    if ((list[ip = 0] is Builtin) || (list[ip = 1] is Builtin)) return ((Builtin)list[ip]).Closure(environment, list);
                    if (((symbol = list[ip = 1] as Symbol) != null && symbol.Index < Symbol.This.Index) || ((symbol = list[ip = 0] as Symbol) != null && symbol.Index < Symbol.This.Index)) return ((Builtin)(list[ip] = new Builtin((Closure)Eval(environment, symbol)))).Closure(environment, list);
                    if ((expression = (list[0] as Closure) ?? (lambda = Eval(environment, list[0]))) is Closure)
                    {
                        var args = (object[])Array.CreateInstance(typeof(object), list.Length - 1);
                        for (var i = 0; i < args.Length; i++) args[i] = Eval(environment, list[i + 1]);
                        return ((Closure)(lambda != null ? list[0] = lambda : expression))(environment, args);
                    }
                    return list.Select((arg, at) => 0 < at ? Eval(environment, arg) : expression).Last();
                }
                if (0 < list.Length)
                {
                    if (list[0] is Builtin) return ((Builtin)list[0]).Closure(environment, list);
                    if ((symbol = list[0] as Symbol) != null && symbol.Index < Symbol.This.Index) return ((Builtin)(list[0] = new Builtin((Closure)Eval(environment, symbol)))).Closure(environment, list);
                    return (expression = (list[0] as Closure) ?? (lambda = Eval(environment, list[0]))) is Closure ? ((Closure)(lambda != null ? list[0] = lambda : expression))(environment, Empty) : expression;
                }
                return Empty;
            }
            return expression is Symbol ? (environment.TryGet((Symbol)expression, out var value) ? value : Symbol.Unknown) : expression;
        }
        protected Evaluator(ISymbolProvider symbolProvider) => SymbolProvider = symbolProvider ?? throw new ArgumentNullException(nameof(symbolProvider), "cannot be null");
        protected virtual IEnvironment NewScope(IEnvironment parent, ISymbolProvider symbolProvider = null) => new Environment(parent, symbolProvider);
        protected virtual IEnvironment Builtins(IEnvironment global, object expression) => global.Set(Symbol.Let, (Closure)Definition).Set(Symbol.Lambda, (Closure)Abstraction);
        protected virtual object Tokenize(object context, string input, out int matched, ref int offset) { matched = 0; return null; }
        protected object ParseSExpression(object context, string input, object lookAhead, int matched, ref int offset)
        {
            object value = lookAhead;
            if (lookAhead != null && Symbol.Quote.Equals(value))
            {
                offset += matched; lookAhead = Tokenize(context, input, out matched, ref offset);
                value = Quote(ParseSExpression(context, input, lookAhead, matched, ref offset));
            }
            else if (lookAhead != null && Symbol.Open.Equals(value))
            {
                var list = new List<object>();
                offset += matched; while ((lookAhead = Tokenize(context, input, out matched, ref offset)) != null && !Symbol.Unknown.Equals(value = lookAhead) && !Symbol.Close.Equals(value)) list.Add(ParseSExpression(context, input, lookAhead, matched, ref offset));
                if (lookAhead != null) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
                else throw new Exception($"unexpected EOF at {offset}");
                value = list.ToArray();
            }
            else if (lookAhead != null) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
            else throw new Exception($"unexpected EOF at {offset}");
            return value;
        }
        protected virtual object Parse(IEnvironment environment, string input)
        {
            var offset = 0; var parse = ParseSExpression(environment = environment ?? NewScope(null, SymbolProvider), input, Tokenize(environment, input, out var matched, ref offset), matched, ref offset);
            return Tokenize(environment, input, out var ignored, ref offset) == null ? parse : throw new Exception($"unexpected '{input[offset]}' at {offset}");
        }
        protected virtual object Params(IEnvironment environment, params object[] args) => Symbol.Unknown;
        protected virtual object This(IEnvironment environment, params object[] args) => Symbol.Unknown;
        protected virtual object Definition(IEnvironment environment, params object[] args)
        {
            var scope = ((object[])args[1]).Cast<object[]>().Aggregate(NewScope(environment), (local, let) => local.Set((Symbol)let[0], Eval(local, let[1])));
            return args.Skip(2).Select(arg => Eval(scope, arg)).LastOrDefault();
        }
        protected virtual object Abstraction(IEnvironment environment, params object[] args)
        {
            Closure @this = null;
            return @this =
                (scope, @params) =>
                {
                    var local = new Environment(environment);
                    var formals = args[0] is Symbol ? new object[] { args[0] } : (object[])args[0];
                    var varargs = 0 < formals.Length && formals[formals.Length - 1] is object[]? formals.Length - 1 : -1;
                    var named = varargs < 0 ? formals.Length : varargs;
                    for (var i = 0; i < named; i++) local.Set((Symbol)formals[i], i < @params.Length ? @params[i] : Symbol.Unknown);
                    if (0 <= varargs) local.Set((Symbol)((object[])formals[varargs])[0], varargs < @params.Length ? @params.Skip(varargs).ToArray() : (object)Symbol.Unknown);
                    local.Set(Symbol.This, This(local, @this)).Set(Symbol.Params, Params(local, new[] { (object)@params }));
                    return Eval(local, args[2]);
                };
        }
        public static readonly IReadOnlyList<KeyValuePair<string, Symbol>> DefaultCore = new[]
        {
            new KeyValuePair<string, Symbol>(string.Empty, Symbol.Unknown),
            new KeyValuePair<string, Symbol>("(", Symbol.Open),
            new KeyValuePair<string, Symbol>(")", Symbol.Close),
            new KeyValuePair<string, Symbol>("`", Symbol.Quote),
            new KeyValuePair<string, Symbol>($"__{Guid.NewGuid():N}__", Symbol.Params),
            new KeyValuePair<string, Symbol>($"__{Guid.NewGuid():N}__", Symbol.This),
            new KeyValuePair<string, Symbol>("let", Symbol.Let),
            new KeyValuePair<string, Symbol>("=>", Symbol.Lambda)
        };
        public object Parse(string input) => Parse(null, input);
        public object Parse(object context, string input) => Parse(context as IEnvironment, input);
        public virtual object Serialize(Type type, object value) => throw new NotImplementedException();
        public string ToString(object value) => (string)Serialize(typeof(string), value);
        public object Quote(object expression) => new[] { Symbol.Quote, expression };
        public object Evaluate(string input) => Evaluate(null, input);
        public object Evaluate(IEnvironment environment, string input) => Evaluate(environment = NewScope(environment, SymbolProvider), Parse(environment, input));
        public object Evaluate(IEnvironment environment, object expression) => Eval(Builtins(environment ?? throw new ArgumentNullException(nameof(environment), "cannot be null"), Clone(expression)), expression);
        public ISymbolProvider SymbolProvider { get; }
    }
}
```

## License and disclaimer
```
Public Domain.

NO WARRANTY EXPRESSED OR IMPLIED. USE AT YOUR OWN RISK.
```

## Contact info
ysharp [dot] design {at} gmail (dot) com
