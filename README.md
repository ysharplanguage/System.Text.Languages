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

There are exactly 8 language builtins (aka "core symbols") common to all LISP-ish languages implemented thanks to the facility presented here, 2 of which are optionally defined *semantically*:

#### Core symbols

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

Also, note that not all "seemingly atomic" literals of the target language being represented are good candidates or suitable to have a **`Symbol`** attached to them, even as programmer-written only ones:
numeric constants such as "**`1792`**" or string literals such as " **`"Hello, world!"`** " aren't actually atomic from the **`ISymbolProvider`** implementer's standpoint.
Both of these are in fact composite constructs for the latter (of a presumed integral type represented in base 10 and of a character string type whose delimeter happens to be the double quote, resp.)
which are interpretable as constants only relatively to the host language, per se, of the implementation of **`ISymbolProvider`**, but arguably not relatively to that very implementation.

In the first case, "**`1792`**", indeed it wouldn't make much sense (at all) to try denoting for the target language a distinct **`Symbol`** for each and every of the concrete values of the **`System.Int32`** type coming from the CLR-based host language after parsing some user input.

Finally, there is in fact a 9th special/predefined symbol baked in as a public, static, read-only field of the **`Symbol`** class, ie, **`Symbol.EOF`**, but this one should be of no concern to anyone who isn't specifically busy with overriding the [**`Evaluator`**](#class-evaluator)'s **`Tokenize`** method.

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
        Lambda = new Symbol(-7),
        EOF = new Symbol(int.MaxValue);
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

**`Symbol`** and **`ISymbolProvider`** do not say or do anything about what the binding of identifiable **`Symbol`** occurrences into actual values in the host language (eg, C# over the CLR) are supposed to be, whether it is at:

- the language's definition time (ie, during the construction of, or the parsing performed by, a parser implementing the [**`ILanguage`**](#interface-ilanguage) contract), or
- the language's run-time (ie, during the construction of, or the evaluation performed by, an interpreter implementing the [**`IEvaluator`**](#interface-ievaluator) contract).

Thus, be it at language's definition time or at language's run-time, this binding from symbols to values is precisely the sole responsibility of an implementation of the **`IEnvironment`** contract.

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
The fairly loose **`ILanguage`** contract is the one to implement (partially or in full) for, say, a "general purpose" language's pre- and/or post-, or sole processor.

Its rationale is as follows:

- a common "pre-processing" need which naturally arises from interpreting or compiling a textual language **`input`** phrase is to parse it first into another intermediate form, prior to further processing; hence, the **`object Parse(string input)`** method
- since it is contemplated that the client may also wish, in some cases, to provide contextual, or "out-of-band" (eg, configuration) information for the parsing operation to execute as expected, the contract makes for this provision through the **`object Parse(object context, string input)`** method overload, with its additional **`context`** parameter
- as a dual of the previous, a common "post-processing" need which also naturally arises from holding a known intermediate form (**`value`**) of what possibly once was given as a textual language input phrase, or the result of an evaluation thereof, is to turn it back into a serializable representation (**`type`**); hence the **`object Serialize(Type type, object value)`** method
- finally, turning such a value back into a textual form more specifically, say, through the **`string ToString(object value)`** method, can also quite naturally be seen (and implemented accordingly) as just a special case of the serialization process, where the first (**`type`**) argument to **`Serialize`** (ie, the serialization target type) happens to be implied and equal to **`typeof(string)`** - thus making **`ToString`** more of a convenient, specialized helper around **`Serialize`** than a truly distinct facility

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

rather, by "S-expressions", we will refer more specifically to what can be recursively constructed - in our natural managed execution environment (ie, the CLR) - from these 3 simple construction rules:

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

**`typeof(void)`** (which is an ***atom*** whose run-time type is **`System.Type`**... or, well, **`System.RuntimeType`** actually...)

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
The **`Closure`** delegate type is merely a convenience facility which allows clients of [**`IEvaluator`**](#interface-ievaluator) that have obtained a reference to a closure in the target language (eg, after evaluating a whole input program returning such a value), to invoke, *in the host language* (ie, C#) the said closure with thus a somewhat deferred execution, in the same run-time environment which gave birth to the implementation of **`IEvaluator`**.

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

Now, the attentive reader of the code provided herein will soon find that there isn't in fact much more to say about these base and default implementations beyond what has already been laid out as semantic expectations from [their respective contracts](#api).

Except for one most notably: the **`Evaluator`** base class, which, [as we'll see](#class-evaluator), does come with a few syntactic, semantic, and even (so to speak) "meta linguistic" convenience "perks" for the implementer of a language interpreter whose intermediate (and/or final) run-time representation of code and data is the particular flavor of S-expressions that we have discussed in the [**`IEvaluator`**](#interface-ievaluator) section.

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
Expectedly, **`Evaluator`** is the base class that one will want to derive from in order to implement an interpreter with relatively little effort.

And unsurprisingly, the lexing/tokenizing operation (which turns a textual input phrase of the target language into ***atom***s - or ***list of*** thereof - for the construction of S-expressions in the sense given in the [**`IEvaluator`**](#interface-ievaluator) section) will be the prerogative of an implementation override of the (protected) **`object Tokenize(object context, string input, out int matched, ref int offset)`** method, with the following assumptions and expectations:

#### Tokenize assumptions
- the **`context`** parameter can be assumed to be the same global **`IEnvironment`** implementation which was passed onto the **`Evaluate`** and **`Parse`** methods down the call stack
- at all time, the whole textual input phrase is held in the **`input`** parameter
- likewise, at all time, the current position in the input is known through the last **`offset`** parameter

#### Tokenize duties
- the lexer/tokenizer that is implemented through **`Tokenize`** must return, for the current position at **`offset`**, either:
  - before the end of the input is reached: a non-**`Symbol.EOF`** value as the resulting token that is recognized at this position, or **`Symbol.Unknown`** when stumbling upon an unexpected character, or
  - **`Symbol.EOF`** exactly when the end of input is detected
- it must also set, at all time, the **`out int matched`** parameter to the (strictly positive) number of characters matched in the input for the resulting token, or simply zero when either of **`Symbol.Unknown`** or **`Symbol.EOF`** is to be returned

#### Tokenize perk
Its **`ref int offset`** parameter allows for the implementation to silently consume whitespace (whenever it is reasonable to do so, and simply by incrementing the **`offset`** accordingly) prior to matching the input against actual tokens (either valid, **`Unknown`**, or **`EOF`**) which are only indicated through the combination of the **`matched`** parameter and the return value.

After lexing/tokenizing, the next topic that comes to mind is parsing.

And as far as parsing the general syntax of S-expressions is concerned, the implementer deriving his/her interpreter from the **`Evaluator`** base class has little to worry about, since all of what's needed is already baked in the (protected) **`object ParseSExpression(object context, string input, object lookAhead, int matched, ref int offset)`** method.

Thus, assuming the implementation of **`Tokenize`** can deal properly with (optional) whitespace and the only allowed lexical forms of atoms to return as **`Symbol`**'s or other values in the host language, there is no need to override anything to be able to parse something alike, say:

**`( <atom0> <atom1> ( ) <atom3> ( <atom40> <atom41> ( ) <atom43> ) <atom5> )`**

But there is a barely veiled perk to our **`Evaluator`**'s internals:

the **`object Parse(object context, string input)`** method does not actually call directly into the handy **`object ParseSExpression(...)`**, instead it calls into the (protected, virtual) **`object Parse(IEnvironment environment, string input)`** ***which then*** delegates the S-expressions parsing work to **`object ParseSExpression(...)`**.

This offers the implementer the flexibility of providing, if so wished, a completely different parsing algorithm - through that **`object Parse(IEnvironment environment, string input)`** intermediary - which would be meant to process any other language, while still relying on **`Tokenize`**'s lexing facility and the rest of **`Evaluator`**'s implementation to interpret an S-expression intermediate form.

After lexing and parsing are taken care of, one will soon start thinking about what is left to do to support the desired run-time evaluation semantics, and to begin with, how to bind this or that particular builtin symbol (devised during the language design phase) to its specific semantic value (be it a constant, a builtin language operator, or anything else really).

Remember that design and implementation choices around builtin symbols will generally have to support at least two distinct phases:

I. What are those symbols, their respective literals, and how do we make an implementation of **`ISymbolProvider`** know about them, to start with?
II. Once the symbols are known to the **`ISymbolProvider`** injected into the evaluator, how do we make them available to this or that implementation of **`IEnvironment`**, and under which conditions (or, "binding rules") exactly?

**`Evaluator`** provides a simple pattern to follow, covering both (I) and (II):

```
using System.Text.Languages;
using System.Text.Languages.Runtime;

public class MyLISP : Evaluator
{
    // Phase (I)
    //
    // define and cache our new builtins once and for all:
    public readonly Symbol NIL, PostalServiceActSigner, PostalServiceActYear;

    public MyLISP(ISymbolProvider symbolProvider = null) : base(symbolProvider ?? new DefaultSymbolProvider(DefaultCore)) =>
        SymbolProvider
        // literals <=> symbols mappings:
        .Include("NIL", out NIL, true)
        .Include("PSAS", out PostalServiceActSigner, true)
        .Include("PSAY", out PostalServiceActYear, true);

    // Phase (II)
    //
    // bind our new builtins in the global environment, if so desired
    protected override IEnvironment Builtins(IEnvironment global, object expression) =>
        !global.Contains(PostalServiceActYear) ? // Last known builtin not yet bound to its semantic value?
        base.Builtins(global, expression)
        .Set(PostalServiceActSigner, "George Washington")
        .Set(PostalServiceActYear, 1792)
        :
        global; // Already all set up

    // Last but not least, a very naive (but sufficient) tokenizer for the job
    protected override object Tokenize(object context, string input, out int matched, ref int offset)
    {
        var symbolProvider = ((IEnvironment)context).SymbolProvider;

        var OpenLiteral = symbolProvider.NameOf(Symbol.Open); // "("
        var CloseLiteral = symbolProvider.NameOf(Symbol.Close); // ")"
        var NILLiteral = symbolProvider.NameOf(NIL); // "NIL"
        var PSASLiteral = symbolProvider.NameOf(PostalServiceActSigner); // "PSAS"
        var PSAYLiteral = symbolProvider.NameOf(PostalServiceActYear); // "PSAY"

        // Silently eat any leading whitespace:
        while (offset < input.Length && char.IsWhiteSpace(input[offset])) offset++;

        if (offset < input.Length)
        {
            matched = OpenLiteral.Length;
            if (offset <= input.Length - matched && input.Substring(offset, matched) == OpenLiteral)
            {
                return Symbol.Open;
            }
            matched = CloseLiteral.Length;
            if (offset <= input.Length - matched && input.Substring(offset, matched) == CloseLiteral)
            {
                return Symbol.Close;
            }
            matched = NILLiteral.Length;
            if (offset <= input.Length - matched && input.Substring(offset, matched) == NILLiteral)
            {
                // The semantic decision about NIL can be made directly in the tokenizer,
                // not using a binding that'd be held in the environment (just for example):
                return null;
            }
            matched = PSASLiteral.Length;
            if (offset <= input.Length - matched && input.Substring(offset, matched) == PSASLiteral)
            {
                return PostalServiceActSigner;
            }
            matched = PSAYLiteral.Length;
            if (offset <= input.Length - matched && input.Substring(offset, matched) == PSAYLiteral)
            {
                return PostalServiceActYear;
            }
            else
            {
                matched = 0;
                return Symbol.Unknown;
            }
        }
        matched = 0;
        return Symbol.EOF;
    }
}
```

Which can then make this sort of tests possible:

```
    var mylisp = new MyLISP();
    var input1 = "  ( NIL   NIL       PSAY   PSAS  )  ";
    var input2 = " (   NIL     NIL   PSAS    NIL    PSAY  )  ";

    var exp1 = (object[])mylisp.Parse(input1);
    System.Diagnostics.Debug.Assert(exp1
        .SequenceEqual
        (
            new object[]
            {
                null,
                null,
                mylisp.PostalServiceActYear,
                mylisp.PostalServiceActSigner
            }
        ));
    var val1 = mylisp.Evaluate(input1);
    System.Diagnostics.Debug.Assert(val1 as string == "George Washington");

    var exp2 = (object[])mylisp.Parse(input2);
    System.Diagnostics.Debug.Assert(exp2
        .SequenceEqual
        (
            new object[]
            {
                null,
                null,
                mylisp.PostalServiceActSigner,
                null,
                mylisp.PostalServiceActYear
            }
        ));
    var val2 = mylisp.Evaluate(input2);
    System.Diagnostics.Debug.Assert((int)val2 == 1792);

```

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
        return Symbol.EOF;
    }
    protected object ParseSExpression(object context, string input, object lookAhead, int matched, ref int offset)
    {
        object value = lookAhead;
        if (!Symbol.EOF.Equals(lookAhead) && Symbol.Quote.Equals(value))
        {
            offset += matched; lookAhead = Tokenize(context, input, out matched, ref offset);
            value = Quote(ParseSExpression(context, input, lookAhead, matched, ref offset));
        }
        else if (!Symbol.EOF.Equals(lookAhead) && Symbol.Open.Equals(value))
        {
            var list = new List<object>();
            offset += matched; while (!Symbol.EOF.Equals(lookAhead = Tokenize(context, input, out matched, ref offset)) && !Symbol.Unknown.Equals(value = lookAhead) && !Symbol.Close.Equals(value)) list.Add(ParseSExpression(context, input, lookAhead, matched, ref offset));
            if (!Symbol.EOF.Equals(lookAhead)) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
            else throw new Exception($"unexpected EOF at {offset}");
            value = list.ToArray();
        }
        else if (!Symbol.EOF.Equals(lookAhead)) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
        else throw new Exception($"unexpected EOF at {offset}");
        return value;
    }
    protected virtual object Parse(IEnvironment environment, string input)
    {
        var offset = 0; var parse = ParseSExpression(environment = environment ?? NewScope(null, SymbolProvider), input, Tokenize(environment, input, out var matched, ref offset), matched, ref offset);
        return Symbol.EOF.Equals(Tokenize(environment, input, out var ignored, ref offset)) ? parse : throw new Exception($"unexpected '{input[offset]}' at {offset}");
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
        public static readonly Symbol Unknown = new Symbol(0), Open = new Symbol(-1), Close = new Symbol(-2), Quote = new Symbol(-3), Params = new Symbol(-4), This = new Symbol(-5), Let = new Symbol(-6), Lambda = new Symbol(-7), EOF = new Symbol(int.MaxValue);
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
        protected virtual object Tokenize(object context, string input, out int matched, ref int offset) { matched = 0; return Symbol.EOF; }
        protected object ParseSExpression(object context, string input, object lookAhead, int matched, ref int offset)
        {
            object value = lookAhead;
            if (!Symbol.EOF.Equals(lookAhead) && Symbol.Quote.Equals(value))
            {
                offset += matched; lookAhead = Tokenize(context, input, out matched, ref offset);
                value = Quote(ParseSExpression(context, input, lookAhead, matched, ref offset));
            }
            else if (!Symbol.EOF.Equals(lookAhead) && Symbol.Open.Equals(value))
            {
                var list = new List<object>();
                offset += matched; while (!Symbol.EOF.Equals(lookAhead = Tokenize(context, input, out matched, ref offset)) && !Symbol.Unknown.Equals(value = lookAhead) && !Symbol.Close.Equals(value)) list.Add(ParseSExpression(context, input, lookAhead, matched, ref offset));
                if (!Symbol.EOF.Equals(lookAhead)) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
                else throw new Exception($"unexpected EOF at {offset}");
                value = list.ToArray();
            }
            else if (!Symbol.EOF.Equals(lookAhead)) { if (!Symbol.Unknown.Equals(value)) offset += matched; else throw new Exception($"unexpected '{input[offset]}' at {offset}"); }
            else throw new Exception($"unexpected EOF at {offset}");
            return value;
        }
        protected virtual object Parse(IEnvironment environment, string input)
        {
            var offset = 0; var parse = ParseSExpression(environment = environment ?? NewScope(null, SymbolProvider), input, Tokenize(environment, input, out var matched, ref offset), matched, ref offset);
            return Symbol.EOF.Equals(Tokenize(environment, input, out var ignored, ref offset)) ? parse : throw new Exception($"unexpected '{input[offset]}' at {offset}");
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
