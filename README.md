# System.Text.Languages

May 30, 2021

## An experiment in minimalism for interpreters of LISP-ish languages

(in about 200 lines of C#)

***

### Table of contents

- [API](#api) (complete source: [Namespace.cs](#namespacecs-50-lines))
  - [Symbol](#class-symbol) class
  - [ISymbolProvider](#interface-isymbolprovider) interface
  - [IEnvironment](#interface-ienvironment) interface
  - [ILanguage](#interface-ilanguage) interface
  - [IEvaluator](#interface-ievaluator) interface
- [Base and default implementations](#base-and-default-implementations) (complete source: [Runtime.cs](#runtimecs-150-lines))
  - [Closure](#delegate-closure) delegate type
  - [SymbolProviderBase](#abstract-class-symbolproviderbase) abstract class
  - [DefaultSymbolProvider](#class-defaultsymbolprovider) class
  - [Environment](#class-environment) class
  - [Evaluator](#class-evaluator) class
- [Deriving a sample interpreter](#deriving-a-sample-interpreter)
- [A more advanced example](#a-more-advanced-example)
- Complete sources
  - [Namespace.cs](#namespacecs-50-lines)
  - [Runtime.cs](#runtimecs-150-lines)
- [License and disclaimer](#license-and-disclaimer)
- [Contact info](#contact-info)

***

## API

### class Symbol

A **`Symbol`** interns all the literal occurrences that denote it, during both parsing and evaluation of the final S-expression.

Examples may include the "**`true`**" and "**`false`**" boolean constants, or the "**`null`**" special value, or arithmetic operators such as "**`+`**", etc.

The true identity of a symbol is its **`Index`**, allocated (during parsing) and maintained (during evaluation), by an implementation of [**`ISymbolProvider`**](#interface-isymbolprovider).

The sign of **`Index`** indicates the linguistic origin of the **`Symbol`**: either negative or zero for language builtins (ie, operators, special signs, keywords, etc, that pertain to the language's definition)
or strictly positive for programmer-defined symbols (ie, identifiers encountered in programs).

There are exactly 8 language builtins common to all LISP-ish languages, 2 of which are optionally defined *semantically*:

- **`Unknown`** is the "unknown symbol" which can be used to signal an unexpected character during parsing and/or an undefined identifier during evaluation
- **`Open`** (resp. **`Close`** ) is the open (resp. close) parenthesis, used to build non-atomic S-expressions
- **`Quote`** is the quote symbol, used to build complex, unevaluated S-expressions, eg, for representations in "code as data" scenarios
- **`Params`** is the optional "**`params`**" keyword, whose role could be the same as JavaScript's "**`arguments`**"
- **`This`** is the optional "**`this`**" keyword, whose role could be the same as C++/Java/JavaScript/C#'s "**`this`**"
- **`Let`** is the "**`let`**" binding keyword, used to define bindings from symbols to values in lexical scopes
- **`Lambda`** is the "**`=>`**" lambda abstraction keyword, used to define first-class anonymous functions, playing the same role as lambda abstractions in lambda calculus

Both **`params`** and **`this`** are optionally defined *semantically* in the sense that it is the responsibility of an implementation of [**`IEvaluator`**](#interface-ievaluator) to decide whether a specific semantics is attached to them or not.

Symbols are just that: atoms which intern multiple occurrences of the same literals, be they builtins or programmer-defined identifiers.

They do not know (nor does the implementation [**`ISymbolProvider`**](#interface-isymbolprovider) either) to which actual values and/or semantics they are attached to, at any particular point in time of the parsing or evaluation phases - this knowledge being the sole responsibility of an implementation of [**`IEnvironment`**](#interface-ienvironment).

```
public class Symbol
{
    public static readonly
        Symbol Unknown = new Symbol(0),
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
```
public interface IEnvironment : IDictionary<Symbol, object>
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

## Base and default implementations

### delegate Closure
```
public delegate object Closure(IEnvironment environment, params object[] args);
```

### abstract class SymbolProviderBase
```
public abstract class SymbolProviderBase : ISymbolProvider
{
    protected abstract void Add(string literal, Symbol symbol);
    protected abstract bool TryGet(string literal, out Symbol symbol);
    protected SymbolProviderBase(IEnumerable<KeyValuePair<string, Symbol>> core)
    {
        if (core != null) foreach (var item in core) { Add(item.Key, item.Value); }
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
                local.Set(Symbol.This, @this).Set(Symbol.Params, @params);
                return Eval(local, args[2]);
            };
    }
    public static readonly IReadOnlyList<KeyValuePair<string, Symbol>> DefaultCore = new[]
    {
        new KeyValuePair<string, Symbol>(string.Empty, Symbol.Unknown),
        new KeyValuePair<string, Symbol>("(", Symbol.Open),
        new KeyValuePair<string, Symbol>(")", Symbol.Close),
        new KeyValuePair<string, Symbol>("`", Symbol.Quote),
        new KeyValuePair<string, Symbol>("params", Symbol.Params),
        new KeyValuePair<string, Symbol>("this", Symbol.This),
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

### Namespace.cs (50 lines)
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

    public interface IEnvironment : IDictionary<Symbol, object>
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
}
```

### Runtime.cs (150 lines)
```
using System;
using System.Collections.Generic;
using System.Linq;

namespace System.Text.Languages.Runtime
{
    public delegate object Closure(IEnvironment environment, params object[] args);

    public abstract class SymbolProviderBase : ISymbolProvider
    {
        protected abstract void Add(string literal, Symbol symbol);
        protected abstract bool TryGet(string literal, out Symbol symbol);
        protected SymbolProviderBase(IEnumerable<KeyValuePair<string, Symbol>> core) { if (core != null) foreach (var item in core) { Add(item.Key, item.Value); } }
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
                    local.Set(Symbol.This, @this).Set(Symbol.Params, @params);
                    return Eval(local, args[2]);
                };
        }
        public static readonly IReadOnlyList<KeyValuePair<string, Symbol>> DefaultCore = new[]
        {
            new KeyValuePair<string, Symbol>(string.Empty, Symbol.Unknown),
            new KeyValuePair<string, Symbol>("(", Symbol.Open),
            new KeyValuePair<string, Symbol>(")", Symbol.Close),
            new KeyValuePair<string, Symbol>("`", Symbol.Quote),
            new KeyValuePair<string, Symbol>("params", Symbol.Params),
            new KeyValuePair<string, Symbol>("this", Symbol.This),
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
