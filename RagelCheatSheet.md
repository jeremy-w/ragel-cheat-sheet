# Ragel Concept Cheat Sheet
Jeremy W. Sherman

2010–11–23

All information comes from the Ragel guide for Ragel version 6.6, which
I guess makes this document **GPLv2-licensed**.

## Ragel compiles state machines with actions.
Ragel builds [Mealy machines](https://en.wikipedia.org/wiki/Mealy_machine):
Actions label arcs, so they are emitted on state change.
Ragel compiles machine descriptions into executable code.
Actions are written in the host language. Actions can manipulate machine
behavior in order to recognize non-regular languages.

Ragel preprocesses a .rl file to generate an output parser.
The file has embedded actions written in the target compilation language,
and scaffolding to set up the scanning environment for consumption by Ragel.

`%%` introduces a single-line directive.
`%%{…}%%` wraps a Ragel block.

You can include another Ragel file with a specific name as:

```
include FsmName "inputfile.rl"
```

You can import definitions of literal strings or numbers in particular
formats (`name = val` or `define name val`) using:

```
import "inputfile";
```

Note how these two formats match C enum tags and preprocessor \#defines.

`#` is treated as a line comment within a Ragel block.
There is no multi-line comment character;
just keep prefixing comment lines with `#`.

## Machines
The most basic machine is the regular expression.
More complicated machines are built by combining regular expressions
using operators.
Unwanted non-determinism can be removed
by prioritizing some transitions over others.

Name an entire Ragel block by including:

```
machine fsm_name;
```

Define submachines with:

```
name = expression;
```

Define and instantiate with:

```
name := expression;
```

## Builtin Machines
Strings as a concatenation of characters quoted with either single or
double quotes.

Named builtin machines are mostly the XXX parts of the C `isXXX`
functions (`grep '^is' /usr/include/ctype.h`),
but there's also `zlen` (zero-length string), `any`, and `empty`
(AKA `^any`).
`extend` is extended ASCII, whatever will fit in an 8-bit
(signed or unsigned as per your alphabet) value.

## Operators
The standard set operations:

- union: `|`
- intersection: `&`
- subtraction: `-` and `--` (see [Subtraction](#subtraction) note below)

Plus the standard regex operations:

- concatenation: `.`
- repetition, in its many guises: `*` `?` `+` `{n}` `{,n}` `{n,}` `{n,m}`
- negation: `!`
- character negation: `^`

- grouping with parentheses: `( … )`

### Concatenation Gotcha
Concatenation is the default operator joining two machines when none
other is specified. Watch out for `(M -7)`, which is understood as
`(M - (7))`. To work around this,
either add an explicit `.` or group the `(-7)`.

### Subtraction
There are two varieties of subtraction, **regular subtraction `m1 - m2`**
and **strong subtraction `m1 -- m2`,**
where `m1` and `m2` are any other machines.

Subtraction prevents input that would match both `m2` and `m1` from matching.

Strong subtraction prevents input that would match `m1`
and contains any *substring* that would match `m2` from matching.

Strong subtraction `m1 -- m2` is equivalent to `m1 - ( any* m2 any* )`.

## Actions
Actions look like:

```
action ActionName {
    code block
}
```

The name is optional; you can just embed a bare `{ code block }`.
But naming makes your patterns more readable and makes it easy to reuse the
same block.

### Transition action embeddings
Transition action embeddings are written as `(machine OP action)`,
where `OP` is one of:

- `>` entering – transition from initial state
- `$` all – all transitions within the machine
- `@` finishing – transition into final state
- `%` leaving – transition from final state

Can embed actions at multiple transitions for a machine,
such as `(M >ENTER $ALL @FINISH %LEAVE)`,
where `ENTER`, `ALL`, etc., are named actions defined elsewhere.

### State action embeddings
State action embeddings are specified in two parts,
**state class** and **embedding type**.
They are used to specify error/EOF actions for different states.

*NOTE:* These put actions in the state bubbles,
rather than on the transition arcs.

#### State classes
State class symbols and their descriptions:

- `<` – non-initial
- `>` – initial
- `$` – all
- `@` – non-final
- `%` – final
- `<>` – non-initial, non-final

#### Event types
Event types can be written using either a symbol or a keyword:

- `~` – `to`
- `*` – `from`
- `/` – `eof`
- `!` – `err` (global error)
- `^` – `leer` (local error)

#### Writing a state-action embedding
Write a state-action embedding in one of three ways:

- `>~action` – all symbols; either `>~ActionName` or `>~{ code block }`
  would work fine
- `>to(name)` – using the keyword; the action name must now be bracketed by
parens
- `>to{…}` – directly embedding a code block

Note how using an alphabetic keyword requires special syntax for the
action name case.

`EOF` means end of the current input, i.e., `p == pe`.
These can be used to twiddle `p` and jump to another state.

**Global error actions** start as actions attached to the labeled state,
but become transition actions to the error state
after compilation is complete.
The transition is triggered for all input characters
not handled by the state's other transitions.
It also becomes an EOF action if the state it was embedded in was non-final
(i.e., EOF was also an invalid "next character").

**Local error actions** are associated with machine names.
After compilation of the machine definition (rather than overall compilation),
all local errors with the same name as the machine definition get transferred
to the error transitions (and EOF actions for non-final states).
The name defaults to the current machine,
but it can be specified using `(name, action)` as the action.
They can be used to recover from within a submachine.

*TIP:* A common way to try to recover is to `fhold` (AKA `p--`)
then `fgoto` to a machine that eats input till a sync point, like a newline.

### Priorities
Priorities use literal numbers along with the transition action
embedding symbols, like `>1`.
Higher-priority transitions kill lower-priority when they have the same name.
The name defaults to the current machine,
but can be specified explicitly as `>(name, 1)`.
Prefer specifying the name to avoid unwanted interactions
between priorities introduced to solve one problem
and those introduced to specify another.

#### Guarded Operators
Priorities are powerful but also require a nuanced understanding at the
state machine level.
There is also the possibility of name collisions in larger machines.

Some operators, called **guarded operators**,
handle common uses of priorities for you:

- `<:` – left-guarded concatenation:
  prioritize M1's transitions
  (prefix one sequence that overlaps another):
  `M1 $(N,1) . M2 >(N,0)`
- `:>` – entry-guarded concatenation:
  prioritize M2's entering transitions
  (terminate M1 on entering M2):
  `M1 $(N,0) . M2 >(N,1)`
- `:>>` –  finish-guarded concatenation:
  prioritize M2's finishing transitions
  (terminate M1 when M2 hits a final state):
  roughly `M1 \$(N,0) . M2 @(N,1)` – will also prioritize leaving M2's start
  state if start is also final
- `**` – longest-match kleene star:
  prioritize all M1's transitions over M1's leaving transitions (greedy match):
  `(M1 $(N,1) %(N,0))*`

### Scanners
Scanners are created by jumping into an instantiated longest-match machine.
The longest-match syntax looks like this:

```
|* pattern => action; pattern => action; *|;
```

The `=> action` part is optional.
Longer matches are preferred over shorter.
If two patterns match the same length text,
the earlier one is preferred.

Scanners rely on the `ts` (token start `char*`),
`te` (token end `char*`), and
`act` (active pattern `int`) variables.

### State Charts
State charts provide a way to manually specify state machines.
You use the **join operator `,`** to combine machines without any transitions.
You label the machines so you can target them for transition.
In the machine description,
you transition from a submachine to a label using epsilon transitions.
Stupid example:

```
M =
label: (
    'a' -> b
),
b: 'b' -> final;
```

`final` seems to be an implicit state in the state chart meaning
"we're done!"

### Semantic Conditions
Semantic conditions make handling variable-length fields easy.
They look like `(M when test_action)`,
where `test_action` is an action block that returns `true` or `false`.
The condition is tested just before a transition is taken;
`true` takes the transition, `false` prevents it.
This works by transforming the transition label to include the
"condition true" information;
this effectively adds a new "compound symbol"
to the alphabet recognized by the machine.
