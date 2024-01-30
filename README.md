# Autosubst 2, OCaml implementation

## Meta

- Authors:
  - Adrian Dapprich
- Contributors:
  - Andrej Dudenhefner
  - Yannick Forster
  - Meven Lennon-Bertrand
  - Audrey Seo
  - Chris Lam
  - Ana Borges
- Maintainer:
  - Yannick Forster ([**@yforster**](https://github.com/yforster))
- License: [MIT License](LICENSE)
- Compatible Coq versions: 8.19.0
- Related publication(s):
  - Adrian Dapprich's [bachelor thesis](https://www.ps.uni-saarland.de/~dapprich/bachelor.php) (Advisor: Andrej Dudenhefner, Supervisor: Gert Smolka)

## Description

This is an OCaml reimplementation of the [Autosubst 2 code generator](https://github.com/uds-psl/autosubst2) by Stark, Schäfer and Kaiser, which was the main focus of Stark's [doctoral dissertation](https://www.ps.uni-saarland.de/~kstark/thesis/). Autosubst 2 in turn is based on previous work on [Autosubst](https://github.com/coq-community/autosubst) by Schäfer, Tebbi and Smolka.

If you ever were in the situation of looking at metatheorems of languages modelled in Coq (e.g. progress and preservation of lambda calculi) and were bothered by the tediousness of formalizing substitution of de Bruijn indices again, this tool might be for you.
Autosubst is a tool that allows you to quickly generate boilerplate code to handle substitutions in languages with binders.

The output is Coq source code that contains 
1. an implementation of the language via (mutual) inductive types and de Bruijn indices for variables,
2. definitions for capture avoiding substitution and renaming on de Bruijn indices,
3. lemmas about the behavior and interaction of renaming and substitution,
4. and a tactic to solve assumption-free substitution equations.

Like Autosubst 2, the OCaml reimplementation supports well-scoped & unscoped syntax as well as variadic & polyadic syntax & functors.
It does not support Autosubst 2's modular syntax.
However, the lemmas generated by the OCaml implementation do not use functional extensionality.

Being implemented in OCaml, it uses Coq as a library to construct and pretty-print the code it generates.

Note that there is also a [reimplementation of Autosubst 2 in the form of a MetaCoq plugin](https://github.com/uds-psl/autosubst-metacoq), but it has less features than the OCaml version.

## Example
For a language like System F which we represent with a signature like the following
```
ty : Type
tm : Type
vl : Type

arr : ty -> ty -> ty
all : (bind ty in ty) -> ty

app  : tm -> tm -> tm
tapp : tm -> ty -> tm
vt   : vl -> tm

lam  : ty -> (bind vl in tm) -> vl
tlam : (bind ty in tm) -> vl
```

Autosubst Ocaml can generate inductive types without scopes to represent the language.
```
Inductive ty : Type :=
  | var_ty : nat -> ty
  | arr : ty -> ty -> ty
  | all : ty -> ty.

Inductive tm : Type :=
  | app : tm -> tm -> tm
  | tapp : tm -> ty -> tm
  | vt : vl -> tm
with vl : Type :=
  | var_vl : nat -> vl
  | lam : ty -> tm -> vl
  | tlam : tm -> vl.
```

and also with scopes, which designate the upper bound of free variables in a term. Note the increased scope level in a `lam` or `tlam` constructor.
```
Inductive ty (n_ty : nat) : Type :=
  | var_ty : fin n_ty -> ty n_ty
  | arr : ty n_ty -> ty n_ty -> ty n_ty
  | all : ty (S n_ty) -> ty n_ty.

Inductive tm (n_ty n_vl : nat) : Type :=
  | app : tm n_ty n_vl -> tm n_ty n_vl -> tm n_ty n_vl
  | tapp : tm n_ty n_vl -> ty n_ty -> tm n_ty n_vl
  | vt : vl n_ty n_vl -> tm n_ty n_vl
with vl (n_ty n_vl : nat) : Type :=
  | var_vl : fin n_vl -> vl n_ty n_vl
  | lam : ty n_ty -> tm n_ty (S n_vl) -> vl n_ty n_vl
  | tlam : tm (S n_ty) n_vl -> vl n_ty n_vl.
```

Optionally, users can control the name of the variable constructor using the syntax `ty(var_ty) : Type`.

Additionally, Autosubst generates a number of lemmas describing the behavior of substitutions of free variables and the composition of substitutions, e.g. that instantiation with extensionally equal substitutions yields the same term.

```
Fixpoint ext_ty (sigma_ty : nat -> ty) (tau_ty : nat -> ty) (Eq_ty : forall x, sigma_ty x = tau_ty x) (s : ty) :
    subst_ty sigma_ty s = subst_ty tau_ty s
```

Finally, it generates a tactic `asimpl` that uses these lemmas as a rewriting system to simplify and solve assumption free equations between terms containing substitutions, which often appear during proofs of metatheorems.
For example the following says that we can first substitute a term `t` for variable 0 and then substitute with `σ`, or do it the other way around (where we must also substitute `σ` in `t`).
```
s[t · id][σ] = s[⇑ σ][t[σ] · id]
```
For more examples and an extended explanation as well as an explanation of the notation we refer to Adrian's bachelor thesis or Stark's doctoral dissertation linked above.

## Setup
### Opam 
First, install opam following the [directions for your operating system](https://opam.ocaml.org/doc/Install.html).

It is best practice to create a new opam switch to not cause conflicts with any of your other installed packages.
We will also need to add the Coq repository and then we can install the `coq-autosubst-ocaml` package.
```bash
$ opam switch create autosubst-ocaml ocaml-base-compiler.4.11.1
$ eval $(opam env)
$ opam repo add coq-released https://coq.inria.fr/opam/released
$ opam install coq-autosubst-ocaml
```

This installs the package along with all dependencies.
An implicit dependency is the [monad library by Denommus](https://github.com/Denommus/monadic) which is not on opam, so we include the sources in the repo.

## Run
If you have installed the program, you can use `autosubst` in the following. Otherwise you can also follow the drections in the "Build & Run" section below.

To display the help, use
```bash
$ autosubst --help
```

En example invocation that shows how to generate code for the untyped lambda calculus.
```bash
$ autosubst signatures/utlc.sig -o output/utlc.v -s ucoq
```

### Signatures
You specify the input language using our input syntax.
The syntax is described in my thesis at https://www.ps.uni-saarland.de/~dapprich/bachelor.php
Example signature files are in the ./signatures directory.

The following example contains all possible rules.
```
-- comments are like in Haskell
tm(var_tm) : Type -- sort declarations
nat : Type -- preexisting sort declaration, it must not have any constructors

list : Functor -- functor declarations, only "list", "prod", "cod" and "option" are allowed

-- declaring some constructors for ty
arr : ty -> ty -> ty
all : (bind ty in ty) -> ty -- this declares "all" as constructor with a binder that binds a new "ty" variable in a "ty" argument and results in a "ty"
-- declaring some constructos for tm
app : tm -> list (tm) -> tm               -- application of the list functor to the sort "tm"
lam (p: nat) : (bind <p, tm> in tm) -> tm -- this declares "lam" as a constructor with parameter "p" which is then used in a variadic binder (i.e. it binds p-many values of "tm")
pair : tm -> (bind tm in "option" (tm)) -> tm -> tm -> tm -- string syntax to include arbitrary functors
const : nat -> tm
```

## Development
### Dev Dependencies
Some dependencies for developing in emacs.
```bash
$ opam install merlin utop ocp-indent ocamlformat
$ opam user-setup install
```

### Build & Run
Create the switch, install the dependencies, and then:
```bash
$ dune build
$ dune exec -- bin/main.exe --help
# e.g. to generate code for the untyped lambda calculus
$ dune exec -- bin/main.exe signatures/utlc.sig -o output/utlc.v -s ucoq
```


