# You Don't Know JS: Escopos e Encerramentos
# Capítulo 1: O que é Escopo?

Um dos paradigmas mais fundamentais de quase todas as linguagens de programação é a capacidade de armazenar valores em variáveis e, posteriormente, obter ou modificar esses valores. Na verdade, a habilidade de armazenar e pegar valores de variáveis é o que fornece *estado* a um programa.

Sem esse conceito, um programa pode realizar algumas tarefas, mas elas seriam extremamente limitadas e extremamente desinteressantes.

Mas a inclusão de variáveis em nossos programas gera perguntas mais interessantes como: onde essas variáveis *vivem*? Em outras palavras, onde elas são armazenadas? E, mais importante, como nossos programas as encontram quando eles precisam delas?

Estas perguntas mostram a necessidade de um conjunto bem definido de regras para armazenar variáveis em algum lugar, e para encontrar essas variáveis posteriormente. Nós iremos chamar esse conjunto de regras: *Escopo*.

Mas, onde e como essas regras de *Escopo* são definidas?

## Teoria de Compiladores

Talvez seja evidente, ou pode ser que seja uma novidade, dependendo do seu nível de interação com linguagens diversas, mas apesar do fato de Javascript ser geralmente colocada na categoria de linguagens "dinâmicas" ou "interpretadas", ela é de fato uma linguagem compilada. Ela *não* é compilada com muita antecedência, como são muitas outras linguagens tradicionalmente compiladas, e nem os resultados da compilação são portáteis entre vários sistemas distribuídos.

Mas, mesmo assim, o mecanismo do Javascript realiza muitos passos semelhantes, embora de maneiras mais sofisticadas do que nós estamos acostumados a ver na maioria dos compiladores tradicionais.

Em um processo tradicional de uma linguagem compilada, um pedaço de código fonte, seu programa, vai tipicamente passar por três passos *antes* de ser executado, grosseiramente chamado "compilação":

1. **Tokenizing/Lexing:** quebrar uma string de caracteres em pedaços com algum significado (para a linguagem), chamados tokens. Por exemplo, considere o programa: `var a = 2;`. Esse programa provavelmente seria ser quebrado nos seguintes tokens: `var`, `a`, `=`, `2`, e `;`. Espaço em branco pode ou não ser mantido como um token, dependendo se tem ou não algum significado.

    **Nota:** A diferença entre tokenizing e lexing é sutil e teórica, mas centraliza-se no fato desses tokens serem ou não identificados de uma maneira *stateless* ou *stateful*. Colocando de maneira simples, se o tokenizer fosse invocar regras de análise stateful para saber se `a` deve ser considerado um token distinto ou apenas parte de outro token, *isso* seria **lexing**.

2. **Parsing:** pegar um conjunto (array) de tokens e transformar isso numa árvore de elementos aninhados, que juntos representam a estrutura gramática do programa. Essa árvore é conhecida como "AST" (<b>A</b>rvore <b>S</b>intática <b>A</b>bstrata).

    A árvore para `var a = 2;` pode começar com um nó de nível superior chamado `VariableDeclaration`, que tem um nó filho chamado `Identifier` (cujo valor é `a`), e outro nó filho chamado `AssignmentExpression` que por sua vez tem um filho chamado `NumericLiteral` (cujo valor é `2`).

3. **Geração de código:** o processo de obter uma AST e transformar isso em código executável. Esta parte varia muito dependendo da linguagem, da plataforma-alvo, etc.

    Então, em vez de focar em detalhes, nós vamos apenas olhar superficialmente e dizer que existe uma forma de obter nossa AST descrita acima para `var a = 2;` e transformá-la em instruções de máquina para de fato *criar* uma variável chamada `a` (incluindo a reserva de memória, etc), e então armazenar um valor em `a`.

    **Note:** Os detalhes de como o mecanismo administra recursos do sistema estão além do que iremos cobrir, então nós vamos apenas considerar que esse mecanismo é capaz de criar e armazenar variáveis conforme necessário.

O mecanismo de Javascript é vastamente mais complexo do que *apenas* aqueles três passos, da mesma maneira do que os outros compiladores. Por exemplo, no processo de análise e geração de código, há com certeza passos para otimizar o desempenho da execução, incluindo tratar elementos redundantes, etc.

Então, eu estou mostrando de forma bem grosseira aqui. Mas eu acho que vocês verão rapidamente porque *esses* detalhes que nós *cobrimos*, mesmo que superficialmente, são relevantes.

Por um lado, o mecanismo de Javascript não tem o luxo (como compiladores de outras linguagens) de ter uma grande disponibilidade de tempo para otimização, porque a compilação de Javascript não acontece numa etapa de preparação anterior, como em outras linguagens.

Para Javascript, a compilação que ocorre acontece, em muitos casos, somente alguns microsegundos (ou menos!) antes do código ser executado. Para garantir o mais alto desempenho, o mecanismo JS utiliza todos os tipos de truques (como JITs, que compilam de maneira preguiçosa e até mesmo recompilam rapidamente, etc.) que são além do escopo da nossa discussão aqui.

Vamos apenas dizer, para fins de simplicidade, que qualquer pedaço de Javascript tem que ser compilado antes (geralmente *exatamente* como dito antes!) de ser executado. Então, o compilador JS vai obter o programa `var a = 2;` e compilá-lo *antes*, e então estará pronto para executá-lo, de imediato.

## Entendendo Escopo

A forma como nós vamos abordar o aprendizado sobre escopo é imaginar o processo como uma conversa. Mas, *quem* está tendo essa conversa?

### O Elenco

Vamos conhecer o elenco dos personagens que interagem para processar o programa `var a = 2;`, assim entenderemos a conversa que vamos ouvir em breve:

1. *Mecanismo*: responsável pela compilação do começo ao fim e pela execução do nosso programa Javascript.

2. *Compilador*: um dos amigos do *Mecanismo*; gerencia todo o trabalho sujo da análise e da geração de código (veja a seção anterior).

3. *Escopo*: outro amigo do *Mecanismo*; coleta e mantém uma lista de consultas a todos os identificadores declarados (variáveis), e impõe um rigoroso conjunto de regras sobre a maneira como estes identificadores são acessíveis pelo código que está em execução.

Para o seu *completo entendimento* sobre como Javascript funciona, você precisa começar a *pensar* como o *Mecanismo* (e amigos) pensa, fazer as perguntas que eles fazem, e responder essas perguntas.

### Back & Forth

When you see the program `var a = 2;`, you most likely think of that as one statement. But that's not how our new friend *Engine* sees it. In fact, *Engine* sees two distinct statements, one which *Compiler* will handle during compilation, and one which *Engine* will handle during execution.

So, let's break down how *Engine* and friends will approach the program `var a = 2;`.

The first thing *Compiler* will do with this program is perform lexing to break it down into tokens, which it will then parse into a tree. But when *Compiler* gets to code-generation, it will treat this program somewhat differently than perhaps assumed.

A reasonable assumption would be that *Compiler* will produce code that could be summed up by this pseudo-code: "Allocate memory for a variable, label it `a`, then stick the value `2` into that variable." Unfortunately, that's not quite accurate.

*Compiler* will instead proceed as:

1. Encountering `var a`, *Compiler* asks *Scope* to see if a variable `a` already exists for that particular scope collection. If so, *Compiler* ignores this declaration and moves on. Otherwise, *Compiler* asks *Scope* to declare a new variable called `a` for that scope collection.

2. *Compiler* then produces code for *Engine* to later execute, to handle the `a = 2` assignment. The code *Engine* runs will first ask *Scope* if there is a variable called `a` accessible in the current scope collection. If so, *Engine* uses that variable. If not, *Engine* looks *elsewhere* (see nested *Scope* section below).

If *Engine* eventually finds a variable, it assigns the value `2` to it. If not, *Engine* will raise its hand and yell out an error!

To summarize: two distinct actions are taken for a variable assignment: First, *Compiler* declares a variable (if not previously declared in the current scope), and second, when executing, *Engine* looks up the variable in *Scope* and assigns to it, if found.

### Compiler Speak

We need a little bit more compiler terminology to proceed further with understanding.

When *Engine* executes the code that *Compiler* produced for step (2), it has to look-up the variable `a` to see if it has been declared, and this look-up is consulting *Scope*. But the type of look-up *Engine* performs affects the outcome of the look-up.

In our case, it is said that *Engine* would be performing an "LHS" look-up for the variable `a`. The other type of look-up is called "RHS".

I bet you can guess what the "L" and "R" mean. These terms stand for "Left-hand Side" and "Right-hand Side".

Side... of what? **Of an assignment operation.**

In other words, an LHS look-up is done when a variable appears on the left-hand side of an assignment operation, and an RHS look-up is done when a variable appears on the right-hand side of an assignment operation.

Actually, let's be a little more precise. An RHS look-up is indistinguishable, for our purposes, from simply a look-up of the value of some variable, whereas the LHS look-up is trying to find the variable container itself, so that it can assign. In this way, RHS doesn't *really* mean "right-hand side of an assignment" per se, it just, more accurately, means "not left-hand side".

Being slightly glib for a moment, you could also think "RHS" instead means "retrieve his/her source (value)", implying that RHS means "go get the value of...".

Let's dig into that deeper.

When I say:

```js
console.log( a );
```

The reference to `a` is an RHS reference, because nothing is being assigned to `a` here. Instead, we're looking-up to retrieve the value of `a`, so that the value can be passed to `console.log(..)`.

By contrast:

```js
a = 2;
```

The reference to `a` here is an LHS reference, because we don't actually care what the current value is, we simply want to find the variable as a target for the `= 2` assignment operation.

**Note:** LHS and RHS meaning "left/right-hand side of an assignment" doesn't necessarily literally mean "left/right side of the `=` assignment operator". There are several other ways that assignments happen, and so it's better to conceptually think about it as: "who's the target of the assignment (LHS)" and "who's the source of the assignment (RHS)".

Consider this program, which has both LHS and RHS references:

```js
function foo(a) {
	console.log( a ); // 2
}

foo( 2 );
```

The last line that invokes `foo(..)` as a function call requires an RHS reference to `foo`, meaning, "go look-up the value of `foo`, and give it to me." Moreover, `(..)` means the value of `foo` should be executed, so it'd better actually be a function!

There's a subtle but important assignment here. **Did you spot it?**

You may have missed the implied `a = 2` in this code snippet. It happens when the value `2` is passed as an argument to the `foo(..)` function, in which case the `2` value is **assigned** to the parameter `a`. To (implicitly) assign to parameter `a`, an LHS look-up is performed.

There's also an RHS reference for the value of `a`, and that resulting value is passed to `console.log(..)`. `console.log(..)` needs a reference to execute. It's an RHS look-up for the `console` object, then a property-resolution occurs to see if it has a method called `log`.

Finally, we can conceptualize that there's an LHS/RHS exchange of passing the value `2` (by way of variable `a`'s RHS look-up) into `log(..)`. Inside of the native implementation of `log(..)`, we can assume it has parameters, the first of which (perhaps called `arg1`) has an LHS reference look-up, before assigning `2` to it.

**Note:** You might be tempted to conceptualize the function declaration `function foo(a) {...` as a normal variable declaration and assignment, such as `var foo` and `foo = function(a){...`. In so doing, it would be tempting to think of this function declaration as involving an LHS look-up.

However, the subtle but important difference is that *Compiler* handles both the declaration and the value definition during code-generation, such that when *Engine* is executing code, there's no processing necessary to "assign" a function value to `foo`. Thus, it's not really appropriate to think of a function declaration as an LHS look-up assignment in the way we're discussing them here.

### Engine/Scope Conversation

```js
function foo(a) {
	console.log( a ); // 2
}

foo( 2 );
```

Let's imagine the above exchange (which processes this code snippet) as a conversation. The conversation would go a little something like this:

> ***Engine***: Hey *Scope*, I have an RHS reference for `foo`. Ever heard of it?

> ***Scope***: Why yes, I have. *Compiler* declared it just a second ago. He's a function. Here you go.

> ***Engine***: Great, thanks! OK, I'm executing `foo`.

> ***Engine***: Hey, *Scope*, I've got an LHS reference for `a`, ever heard of it?

> ***Scope***: Why yes, I have. *Compiler* declared it as a formal parameter to `foo` just recently. Here you go.

> ***Engine***: Helpful as always, *Scope*. Thanks again. Now, time to assign `2` to `a`.

> ***Engine***: Hey, *Scope*, sorry to bother you again. I need an RHS look-up for `console`. Ever heard of it?

> ***Scope***: No problem, *Engine*, this is what I do all day. Yes, I've got `console`. He's built-in. Here ya go.

> ***Engine***: Perfect. Looking up `log(..)`. OK, great, it's a function.

> ***Engine***: Yo, *Scope*. Can you help me out with an RHS reference to `a`. I think I remember it, but just want to double-check.

> ***Scope***: You're right, *Engine*. Same guy, hasn't changed. Here ya go.

> ***Engine***: Cool. Passing the value of `a`, which is `2`, into `log(..)`.

> ...

### Quiz

Check your understanding so far. Make sure to play the part of *Engine* and have a "conversation" with the *Scope*:

```js
function foo(a) {
	var b = a;
	return a + b;
}

var c = foo( 2 );
```

1. Identify all the LHS look-ups (there are 3!).

2. Identify all the RHS look-ups (there are 4!).

**Note:** See the chapter review for the quiz answers!

## Nested Scope

We said that *Scope* is a set of rules for looking up variables by their identifier name. There's usually more than one *Scope* to consider, however.

Just as a block or function is nested inside another block or function, scopes are nested inside other scopes. So, if a variable cannot be found in the immediate scope, *Engine* consults the next outer containing scope, continuing until found or until the outermost (aka, global) scope has been reached.

Consider:

```js
function foo(a) {
	console.log( a + b );
}

var b = 2;

foo( 2 ); // 4
```

The RHS reference for `b` cannot be resolved inside the function `foo`, but it can be resolved in the *Scope* surrounding it (in this case, the global).

So, revisiting the conversations between *Engine* and *Scope*, we'd overhear:

> ***Engine***: "Hey, *Scope* of `foo`, ever heard of `b`? Got an RHS reference for it."

> ***Scope***: "Nope, never heard of it. Go fish."

> ***Engine***: "Hey, *Scope* outside of `foo`, oh you're the global *Scope*, ok cool. Ever heard of `b`? Got an RHS reference for it."

> ***Scope***: "Yep, sure have. Here ya go."

The simple rules for traversing nested *Scope*: *Engine* starts at the currently executing *Scope*, looks for the variable there, then if not found, keeps going up one level, and so on. If the outermost global scope is reached, the search stops, whether it finds the variable or not.

### Building on Metaphors

To visualize the process of nested *Scope* resolution, I want you to think of this tall building.

<img src="fig1.png" width="250">

The building represents our program's nested *Scope* rule set. The first floor of the building represents your currently executing *Scope*, wherever you are. The top level of the building is the global *Scope*.

You resolve LHS and RHS references by looking on your current floor, and if you don't find it, taking the elevator to the next floor, looking there, then the next, and so on. Once you get to the top floor (the global *Scope*), you either find what you're looking for, or you don't. But you have to stop regardless.

## Errors

Why does it matter whether we call it LHS or RHS?

Because these two types of look-ups behave differently in the circumstance where the variable has not yet been declared (is not found in any consulted *Scope*).

Consider:

```js
function foo(a) {
	console.log( a + b );
	b = a;
}

foo( 2 );
```

When the RHS look-up occurs for `b` the first time, it will not be found. This is said to be an "undeclared" variable, because it is not found in the scope.

If an RHS look-up fails to ever find a variable, anywhere in the nested *Scope*s, this results in a `ReferenceError` being thrown by the *Engine*. It's important to note that the error is of the type `ReferenceError`.

By contrast, if the *Engine* is performing an LHS look-up and arrives at the top floor (global *Scope*) without finding it, and if the program is not running in "Strict Mode" [^note-strictmode], then the global *Scope* will create a new variable of that name **in the global scope**, and hand it back to *Engine*.

*"No, there wasn't one before, but I was helpful and created one for you."*

"Strict Mode" [^note-strictmode], which was added in ES5, has a number of different behaviors from normal/relaxed/lazy mode. One such behavior is that it disallows the automatic/implicit global variable creation. In that case, there would be no global *Scope*'d variable to hand back from an LHS look-up, and *Engine* would throw a `ReferenceError` similarly to the RHS case.

Now, if a variable is found for an RHS look-up, but you try to do something with its value that is impossible, such as trying to execute-as-function a non-function value, or reference a property on a `null` or `undefined` value, then *Engine* throws a different kind of error, called a `TypeError`.

`ReferenceError` is *Scope* resolution-failure related, whereas `TypeError` implies that *Scope* resolution was successful, but that there was an illegal/impossible action attempted against the result.

## Review (TL;DR)

Scope is the set of rules that determines where and how a variable (identifier) can be looked-up. This look-up may be for the purposes of assigning to the variable, which is an LHS (left-hand-side) reference, or it may be for the purposes of retrieving its value, which is an RHS (right-hand-side) reference.

LHS references result from assignment operations. *Scope*-related assignments can occur either with the `=` operator or by passing arguments to (assign to) function parameters.

The JavaScript *Engine* first compiles code before it executes, and in so doing, it splits up statements like `var a = 2;` into two separate steps:

1. First, `var a` to declare it in that *Scope*. This is performed at the beginning, before code execution.

2. Later, `a = 2` to look up the variable (LHS reference) and assign to it if found.

Both LHS and RHS reference look-ups start at the currently executing *Scope*, and if need be (that is, they don't find what they're looking for there), they work their way up the nested *Scope*, one scope (floor) at a time, looking for the identifier, until they get to the global (top floor) and stop, and either find it, or don't.

Unfulfilled RHS references result in `ReferenceError`s being thrown. Unfulfilled LHS references result in an automatic, implicitly-created global of that name (if not in "Strict Mode" [^note-strictmode]), or a `ReferenceError` (if in "Strict Mode" [^note-strictmode]).

### Quiz Answers

```js
function foo(a) {
	var b = a;
	return a + b;
}

var c = foo( 2 );
```

1. Identify all the LHS look-ups (there are 3!).

	**`c = ..`, `a = 2` (implicit param assignment) and `b = ..`**

2. Identify all the RHS look-ups (there are 4!).

    **`foo(2..`, `= a;`, `a + ..` and `.. + b`**


[^note-strictmode]: MDN: [Strict Mode](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions_and_function_scope/Strict_mode)
