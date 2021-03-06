<div class="pagetitle">
![](img/titles/extended_parser.png)
</div>

<p class="halfbreak">
</p>

Extended Parser
===============

Up until now we've been using parser combinators to build our parsers. Parser
combinators are a top-down parser formally in the $\mathtt{LL}(k)$ family of
parsers. The parser proceeds top-down, with a sequence of $k$ characters used to
dispatch on the leftmost production rule. Combined with backtracking (i.e.  try
combinator) this is simultaneously both an extremely powerful and simple model
to implement as we saw before with our simple 100 line parser library.

However there are a family of grammars that include left-recursion that
$\mathtt{LL}(k)$ can be inefficient and often incapable of parsing.
Left-recursive rules are the case where the left-most symbol of the rule
recurses on itself. For example:

$$
\begin{aligned}
e ::=\ e\ \t{op}\ \t{atom}
\end{aligned}
$$

Now we demonstrated a way before that we could handle these cases using the
parser combinator ``chainl1`` function, and while this is possible sometimes it
can in many cases be inefficient use of parser stack and lead to ambiguous
cases. 

The other major family of parsers $\mathtt{LR}$ are not plagued with the same
concerns over left recursion. On the other hand $\mathtt{LR}$ parser are
exceedingly more complicated to implement, relying on a rather sophisticated
method known as Tomita's algorithm to do the heavy lifting. The tooling can
around the construction of the *production rules* in a form that can be handled
by the algorithm is often handled a DSL that generates the code for the parser.
While the tooling is fairly robust, there is a level of indirection between us
and the code that can often be a bit of brittle to extend with custom logic.

The most common form of this toolchain is the Lex/Yacc lexer and parser
generator which compile into efficient C parsers for $\mathtt{LR}$ grammars.
Haskell's **Happy** and **Alex** are roughly the Haskell equivalent of these
tools.

Toolchain
---------

Our parser logic will be spread across two different modules.

* Lexer.x
* Parser.y

The code in each of these modules is a hybrid of the specific Alex/Happy grammar
syntax and arbitrary Haskell logic that is spliced in. Code delineated by braces
(``{}``) is regular Haskell, while code outside is parser/lexer logic. 


```haskell
-- **Begin Haskell Syntax**
{
{-# OPTIONS_GHC -w #-}

module Lexer (
  Token(..),
  scanTokens
) where

import Syntax
}
-- **End Haskell Syntax**

-- **Begin Alex Syntax**
%wrapper "basic"

$digit = 0-9
$alpha = [a-zA-Z]
$eol   = [\n]
-- **End Alex Syntax**
```

The files will be used during the code generation of the two modules ``Lexer``
and ``Parser``. The toolchain is accessible in several ways, first via the
command-line tools ``alex`` and ``happy`` will will generate the resulting
modules by passing the appropriate input file to the tool.

```haskell
$ alex Lexer.x    # Generates Lexer.hs
$ happy Parser.y  # Generates Parser.hs
```

Or inside of the cabal file using the ``build-tools`` command.

```haskell
  Build-depends:       base, array
  build-tools:         alex, happy
  other-modules:       
    Parser,
    Lexer
```

So the resulting structure of our interpreter will have the following set of
files.

* **Lexer.hs**
* **Parser.hs**
* Eval.hs
* Main.hs
* Syntax.hs

Alex
----

Our lexer module will export our Token definition and a function for converting
an arbitrary String into a *token stream* or a list of Tokens.

```haskell
{
module Lexer (
  Token(..),
  scanTokens
) where

import Syntax
}
```

The tokens are simply an enumeration of the unique possible tokens in our
grammar.

```haskell
data Token 
  = TokenLet
  | TokenTrue
  | TokenFalse
  | TokenIn
  | TokenLambda
  | TokenNum Int
  | TokenSym String
  | TokenArrow
  | TokenEq
  | TokenAdd
  | TokenSub
  | TokenMul
  | TokenLParen
  | TokenRParen
  | TokenEOF
  deriving (Eq,Show)

scanTokens :: String -> [Token]
scanTokens = alexScanTokens
```

The token definition is list of function definitions mapping atomic character
and alphabetical sequences to constructors for our ``Token`` datatype.


```haskell
%wrapper "basic"

$digit = 0-9
$alpha = [a-zA-Z]
$eol   = [\n]

tokens :-

  -- Whitespace insensitive
  $eol                          ;
  $white+                       ;

  -- Comments
  "#".*                         ;

  -- Syntax
  let                           { \s -> TokenLet }
  True                          { \s -> TokenTrue }
  False                         { \s -> TokenFalse }
  in                            { \s -> TokenIn }
  $digit+                       { \s -> TokenNum (read s) }
  "->"                          { \s -> TokenArrow }
  \=                            { \s -> TokenEq }
  \\                            { \s -> TokenLambda }
  [\+]                          { \s -> TokenAdd }
  [\-]                          { \s -> TokenSub }
  [\*]                          { \s -> TokenMul }
  \(                            { \s -> TokenLParen }
  \)                            { \s -> TokenRParen }
  $alpha [$alpha $digit \_ \']* { \s -> TokenSym s }
```

Happy
-----

We'll parse into a small untyped lambda calculus for our frontend language. 

```haskell
module Syntax where

type Name = String

data Expr
  = Lam Name Expr
  | App Expr Expr
  | Var Name
  | Lit Lit
  | Op Binop Expr Expr
  deriving (Eq,Show)

data Lit
  = LInt Int
  | LBool Bool
  deriving (Show, Eq, Ord)

data Binop = Add | Sub | Mul | Eql
  deriving (Eq, Ord, Show)
```

The token constructors are each assigned to a name that will be used in our
production rules.

```haskell
-- Lexer structure 
%tokentype { Token }

-- Token Names
%token
    let   { TokenLet }
    true  { TokenTrue }
    false { TokenFalse }
    in    { TokenIn }
    NUM   { TokenNum $$ }
    VAR   { TokenSym $$ }
    '\\'  { TokenLambda }
    '->'  { TokenArrow }
    '='   { TokenEq }
    '+'   { TokenAdd }
    '-'   { TokenSub }
    '*'   { TokenMul }
    '('   { TokenLParen }
    ')'   { TokenRParen }
```

The parser itself will live inside of a custom monad of our choosing. In this
simple case we'll just add error handling with the ``Except`` monad.

```haskell
-- Parser monad
%monad { Except String } { (>>=) } { return }
%error { parseError }
```

And finally our production rules, the toplevel entry point for our parser will
be the ``expr`` rule.  Notice how naturally we can right left recursive grammar
for our infix operators.

```haskell
-- Entry point
%name expr

-- Operators
%left '+' '-'
%left '*'
%%

Expr : let VAR '=' Expr in Expr    { App (Lam $2 $6) $4 }
     | '\\' VAR '->' Expr          { Lam $2 $4 }
     | Form                        { $1 }

Form : Form '+' Form               { Op Add $1 $3 }
     | Form '-' Form               { Op Sub $1 $3 }
     | Form '*' Form               { Op Mul $1 $3 }
     | Fact                        { $1 }

Fact : Fact Atom                   { App $1 $2 }
     | Atom                        { $1 }

Atom : '(' Expr ')'                { $2 }
     | NUM                         { Lit (LInt $1) }
     | VAR                         { Var $1 }
     | true                        { Lit (LBool True) }
     | false                       { Lit (LBool True) }
```

Syntax Errors
-------------

Parsec's default error reporting leaves a bit to be desired, but does in fact
contain most of the information needed to deliver better messages packed inside
the ParseError structure.

```haskell
showSyntaxError :: L.Text -> ParseError -> String
showSyntaxError s err = L.unpack $ L.unlines [
      "  ",
      "  " <> lineContents,
      "  " <> ((L.replicate col " ") <> "^"),
      (L.pack $ show err)
    ]
  where
    lineContents = (L.lines s) !! line
    pos  = errorPos err
    line = sourceLine pos - 1
    col  = fromIntegral $ sourceColumn pos - 1
```

Now when we enter an invalid expression the error reporting will point us
directly to the adjacent lexeme that caused the problem as is common in many
languages.

```bash
λ> \x -> x + 
  
  \x -> x + 
            ^
"<interactive>" (line 1, column 11):
unexpected end of input
expecting "(", character, literal string, "[", integer, "if" or identifier
```

Type Error Provenance
---------------------

Before our type inference engine would generate somewhat typical type inference
error messages. If two terms couldn't be unified it simply told us this and some
information about the toplevel declaration where it occurred. Leaving us with a
bit of a riddle about this error came to be.

```haskell
Cannot unify types: 
        Int
with 
        Bool
in the definition of 'foo'
```

Effective error reporting in the presence of type inference is a difficult task,
effectively our typechecker takes our frontend AST and transforms it into a
large constraint problem but effectively destroys position information
information in the process. Even if the position information were tracked, the
nature of unification is that a cascade of several unifications can give rise to
invalid solution and the immediate two syntactic constructs that gave rise to a
unification fail are not necessarily the two that map back to human intuition
about how the type error arose. Very little research has done on this topic and
it remains a open topic with very immediate and applicable results to
programming. 

To do simple provenance tracking we will use a technique of track the "flow" of
type information through out typechecker and associate position information
associated with the inferred types back to their position information in the
source.

```haskell
type Name = String

data Expr
  = Var Loc Name
  | App Loc Expr Expr
  | Lam Loc Name Expr
  | Lit Loc Int

data Loc = NoLoc | Located Int
  deriving (Show, Eq, Ord)
```

So now inside of our parser we simply attach Parsec information on to each AST
node. For example for the variable term.

```haskell
variable :: Parser Expr
variable = do
  x <- identifier
  l <- sourceLine <$> getPosition
  return (Var (Located l) x)
```

Our type system will also include information, although by default it will use
the ``NoLoc`` type until explicit information is provided during inference. The
two functions ``getLoc`` and ``setLoc`` will be used to update and query the
position information from type terms.

```haskell
data Type
  = TVar Loc TVar
  | TCon Loc Name
  | TArr Loc Type Type
  deriving (Show, Eq, Ord)

newtype TVar = TV String
  deriving (Show, Eq, Ord)

typeInt :: Type
typeInt = TCon NoLoc "Int"

setLoc :: Loc -> Type -> Type
setLoc l (TVar _ a)   = TVar l a
setLoc l (TCon _ a)   = TCon l a
setLoc l (TArr _ a b) = TArr l a b

getLoc :: Type -> Loc
getLoc (TVar l _) = l
getLoc (TCon l _) = l
getLoc (TArr l _ _) = l
```

Our fresh variable supply now also takes a location field which is attached to
resulting type variable.

```haskell
fresh :: Loc -> Check Type
fresh l = do
  s <- get
  put s{count = count s + 1}
  return $ TVar l (TV (letters !! count s))
```

```haskell
infer :: Expr -> Check Type
infer expr = case expr of
  Var l n -> do
    t <- lookupVar n
    return $ setLoc l t

  App l a b -> do
    ta <- infer a
    tb <- infer b
    tr <- fresh l
    unify ta (TArr l tb tr)
    return tr

  Lam l n a -> do
    tv <- fresh l
    ty <- inEnv (n, tv) (infer a)
    return (TArr l (setLoc l ty) tv)

  Lit l _ -> return (setLoc l typeInt)
```

Now specifically at the call site of our unification solver, if we encounter a
unification fail we simply pluck the location information off the two type terms
and plug it into the type error fields.

```haskell
unifies t1 t2 | t1 == t2 = return emptyUnifer
unifies (TVar _ v) t = v `bind` t
unifies t (TVar _ v) = v `bind` t
unifies (TArr _ t1 t2) (TArr _ t3 t4) = unifyMany [t1, t2] [t3, t4]
unifies (TCon _ a) (TCon _ b) | a == b = return emptyUnifer
unifies t1 t2 = throwError $ UnificationFail t1 (getLoc t1) t2 (getLoc t2)

bind ::  TVar -> Type -> Solve Unifier
bind a t
  | eqLoc t a        = return (emptySubst, [])
  | occursCheck a t  = throwError $ InfiniteType a (getLoc t) t
  | otherwise        = return $ (Subst $ Map.singleton a t, [])
```

So now we can explicitly trace the provenance of the specific constraints that
gave rise to a given type error all the way back to the source that generated
them.

```haskell
Cannot unify types: 
        Int
        Introduced at line 27 column 5

          f 2 3

with 
        Int -> c
        Introduced at line 5 column 9

          let f x y = x y
```

This is of course the simplest implementation of the this tracking method and
could be further extended by giving an weighted ordering to the constraints
based on their likelihood of importance and proximity and then choosing which
location to report based on this information. This remains an open area of work.

Indentation
-----------

Haskell's syntax uses indentation blocks to delineated sections of code.  This
use of indentation sensitive layout to convey the structure of logic is
sometimes called the *offside rule* in parsing literature. At the beginning of
"laidout" block the first declaration or definition can start in any column, and
the parser marks that indentation level. Every subsequent top-level declaration
must have the same indentation.


```haskell
-- Start of layout ( Column: 0 )
fib :: Int -> Int
fib x = truncate $ ( 1 / sqrt 5 ) * ( phi ^ x - psi ^ x ) -- (Column: > 0)
  -- Start of new layout ( Column: 2 )
  where
      -- Indented block ( Column: > 2 )
      phi = ( 1 + sqrt 5 ) / 2
      psi = ( 1 - sqrt 5 ) / 2
```

The Parsec monad is itself parameterized over a type variable ``s`` which stands
for the State layer baked into the monad allowing us to embed custom parser
state inside of our rules. To adopt our parser to handle sensitive whitespace we
will

```haskell
-- Indentation sensitive Parsec monad.
type IParsec a = Parsec Text ParseState a

data ParseState = ParseState
  { indents :: Column
  } deriving (Show)

initParseState :: ParseState
initParseState = ParseState 0
```

Inside of the Parsec the internal position state (SourcePos) is stored during
each traversal, and is accessible inside of rule logic via ``getPosition``
function.

```haskell
data SourcePos = SourcePos SourceName !Line !Column
getPosition :: Monad m => ParsecT s u m SourcePos
```

In terms of this function we can write down a set of logic that will allow us to
query the current column count and then either succeed or fail to match on a
pattern based on the current indentation level. The ``laidout`` combinator will
capture the current indentation state and push it into the ``indents`` field in
the State monad.

```haskell
laidout :: Parsec s ParseState a -> Parsec s ParseState a
laidout m = do
  cur <- indents <$> getState
  pos <- sourceColumn <$> getPosition
  modifyState $ \st -> st { indents = pos }
  res <- m
  modifyState $ \st -> st { indents = cur }
  return res
```

And then have specific logic which guard the parser match based on comparing the
current indentation level to the stored indentation level.

```haskell
indentCmp
  :: (Column -> Column -> Bool)
  -> Parsec s ParseState ()
indentCmp cmp = do
  col <- sourceColumn <$> getPosition
  current <- indents <$> getState
  guard (col `cmp` current)
```

We can then write two combinators in terms of this function which match on
either positive and identical indentation difference. 

```haskell
indented :: IParsec ()
indented = indentCmp (>) <?> "Block (indented)"

align :: IParsec ()
align = indentCmp (==) <?> "Block (same indentation)"
```

On top of these we write our two combinators for handling block syntax, which
match a sequence of vertically aligned patterns as a list.

```haskell
block, block1 :: Parser a -> Parser [a]
block  p = laidout (many (align >> p))
block1 p = laidout (many1 (align >> p))
```

GHC uses an optional layout rule for several constructs, allowing us to
equivalently manually delimit indentation sensitive syntax with braces. The most
common is for do-notation. So for example:

```haskell
example = do { a <- m; b }

example = do
  a <- m
  b
```

To support this in Parsec style we adopt implement a ``maybeBraces`` function.

```haskell
maybeBraces :: Parser a -> Parser [a]
maybeBraces p = braces (endBy p semi) <|> block p

maybeBraces1 :: Parser a -> Parser [a]
maybeBraces1 p = braces (endBy1 p semi) <|> block p
```

Extensible Operators
--------------------

Haskell famously allows the definition of custom infix operators, and extremely
useful language feature although this poses a bit of a challenge to parse! There
are several ways to do this and both depend on two properties of the operators.

* Precedence
* Associativity

1. The first is the way that GHC does is to parse all operators as left associative
and of the same precedence, and then before desugaring go back and "fix" the
parse tree given all the information we collected after finishing parsing.

2. The second method is a bit of a hack, and involves simply storing the collected
operators inside of the Parsec state monad and then simply calling
``buildExpressionParser`` on the current state each time we want to parse and
infix operator expression.

To do later method we set up the AST objects for our fixity definitions, which
associate precedence and associativity annotations with a custom symbol.

```haskell
data FixitySpec = FixitySpec
  { fixityFix :: Fixity
  , fixityName :: String
  } deriving (Eq, Show)

data Assoc
  = L
  | R
  | N
  deriving (Eq,Ord,Show)

data Fixity
  = Infix Assoc Int
  | Prefix Int
  | Postfix Int
  deriving (Eq,Ord,Show)
```

Our parser state monad will hold a list of the at Ivie fixity specifications and
whenever a definition is uncounted we will append to this list.

```haskell
data ParseState = ParseState
  { indents :: Column
  , fixities :: [FixitySpec]
  } deriving (Show)

initParseState :: ParseState
initParseState = ParseState 0 defaultOps

addOperator ::  FixitySpec -> Parsec s ParseState ()
addOperator fixdecl = do
  modifyState $ \st -> st { fixities = fixdecl : (fixities st) }
```

The initial state will consist of the default arithmetic and list operators
defined with the same specification as the Haskell specification.

```haskell
defaultOps :: [FixitySpec]
defaultOps = [
    FixitySpec (Infix L 4) ">"
  , FixitySpec (Infix L 4) "<"
  , FixitySpec (Infix L 4) "/="
  , FixitySpec (Infix L 4) "=="

  , FixitySpec (Infix R 5) ":"

  , FixitySpec (Infix L 6) "+"
  , FixitySpec (Infix L 6) "-"

  , FixitySpec (Infix L 5) "*"
  , FixitySpec (Infix L 5) "/"
  ]
```

Now In our parser we need to be able to transform the fixity specifications into
Parsec operator definitions. This is a pretty straightforward sort and group
operation on the list.

```haskell
fixityPrec :: FixitySpec -> Int
fixityPrec (FixitySpec (Infix _ n) _) = n
fixityPrec (FixitySpec _ _) = 0

toParser (FixitySpec ass tok) = case ass of
    Infix L _ -> infixOp tok (op (Name tok)) Ex.AssocLeft
    Infix R _ -> infixOp tok (op (Name tok)) Ex.AssocRight
    Infix N _ -> infixOp tok (op (Name tok)) Ex.AssocNone

mkTable ops =
  map (map toParser) $
    groupBy ((==) `on` fixityPrec) $
      reverse $ sortBy (compare `on` fixityPrec) $ ops
```

Now when parsing a infix operator declarations we simply do a state operation
and add the operator to the parser state so that all subsequent definitions.
This differs from Haskell slightly in that operators must be defined before
their usage in a module.

```haskell
fixityspec :: Parser FixitySpec
fixityspec = do
  fix  <- fixity
  prec <- precedence
  op   <- operator
  semi
  let spec = FixitySpec (fix prec) op
  addOperator spec
  return spec
  where
    fixity = Infix L <$ reserved "infixl"
         <|> Infix R <$ reserved "infixr"
         <|> Infix N <$ reserved "infix"

precedence :: Parser Int
precedence = do
  n <- natural
  if n <= 10
  then return (fromInteger n)
  else empty
  <?> "Invalid operator precedence"

fixitydecl :: Parser Decl
fixitydecl = do
  spec <- fixityspec
  return $ FixityDecl spec
 <?> "operator fixity definition"
```

And now when we need to parser a infix expression term we simply pull our state
out and build the custom operator table, and feed this the build Expression
Parser just as before.

```haskell
term :: Parser Expr -> Parser Expr
term a = do
  st <- getState
  let customOps = mkTable (fixities st)
  Ex.buildExpressionParser customOps a
```

Full Source
-----------

* [Happy Parser](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter9/happy)
* [Imperative Language (Happy)](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter9/assign)
* [Layout Combinators](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter9/layout)
* [Type Provenance Tracking](https://github.com/sdiehl/write-you-a-haskell/tree/master/chapter9/provenance)

Resources
---------

The tooling and documentation for Alex and Happy is well-developed as it is used
extensively inside of GHC:

* [Alex User Guide](https://www.haskell.org/alex/doc)
* [Happy User Guide](https://www.haskell.org/happy/doc/html/)
* [A Tool for Generalized LR Parsing In Haskell](http://www.benmedlock.co.uk/Functional_GLR_Parsing.pdf)
* [Haskell Syntax Definition](https://www.haskell.org/onlinereport/haskell2010/haskellch10.html#x17-17500010)

Haskell itself uses Alex and Happy for it's parser infastructure. The resulting
parser is rather sophisicated. 

* [Lexer.x](https://github.com/ghc/ghc/blob/master/compiler/parser/Lexer.x)
* [Parser.y](https://github.com/ghc/ghc/blob/master/compiler/parser/Parser.y)

One of the few papers ever written in Type Error reporting gives some techniques
for presentation and tracing provenance:

* [Top Quality Type Error Messages](http://www.staff.science.uu.nl/~swier101/Papers/Theses/TopQuality.pdf)

\clearpage
