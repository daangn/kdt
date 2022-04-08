# KDT Specification

Specification for KDT language.

## What is Design Token?

TBD

## What is KDT?

KDT is a language for describing and managing design tokens.

## Structure

KDT defines two types of tokens, **"scale tokens"** and **"semantic tokens"**.

The scale token is for low-level definition. Its purpose of the scale token is to define a finite number symbols and make it platform-agnostic by hiding display details behind of it.

The value bound to the scale token may depends on application context or user preferences. If it always need a fixed value regardless of context, you can define it as a **static token**.

The semantic token is for high-level definition. It abstracts the token by giving it a unique meaning so designers and developers can have same language.

![Overview of design token strcucture](docs/images/overview.png)

## Content Type

KDT is set of multiple token definitions(`$`) and macros(`%`). Its file extension should be `.kdt` and its media type should be `application/design-tokens+kdt`.

KDT can be serialized to text. Definitions in the serialized text can hold all information design token schema and actual values, but macros can be ignored by runtime.

## Language Concepts

### Prefixed

Every syntax in KDT should always start with special prefix such like `$` or `%`. These prefixes can be used to identify meaningful definitions earlier in a mixed content.

### Sequential

Tokens can be defined flat sequences, instead of relying deeply on nested object form. This makes interoperability with many tools, especially in design tools that don't have embedded code editors.

This helps design token integration with the native workflow of design tools without external service.

![Example: Using KDT in Figma frames](docs/images/kdt-in-figma.png)

### Expressive

The KDT should have the right scope of expressiveness to not only declare tokens, but also hint at integrations like supporting tools or synchronizing documents.

### Domain-specific

Instead of being compatible with every possible cases, KDT focuses more on the real problem. That means KDT would have some opinions on how to define and to manage design tokens.

## Syntax

### Scale Token Definitions (`$scale` and `$static`)

Scale tokens can be declared as `{prefix}.{target}.{name}`.

- `prefix` can be `$static` or `$scale`.
- `target` is the pre-defined name associated with the actual display. 
- `name` is *unique* id, can contain alphanumeric, and `-`.

```

$scale.color.carrot-100
$scale.color.carrot-200
$scale.color.carrot-300
$scale.color.carrot-400
$scale.color.carrot-500
```

#### Binding Target

Definitions can have value binding via `->` operator.

```
$scale.color.carrot-100 -> #FFF5F0
$scale.color.carrot-200 -> #FFE2D2
$scale.color.carrot-300 -> #FFD2B9
$scale.color.carrot-400 -> #FFBC97
$scale.color.carrot-500 -> #FF9E66
```

The available value bindings depend on the target.

| Target        | Available value formats                                                         |
|:------------- |:------------------------------------------------------------------------------- |
| `color`       | hex (`#FFFFFF`), RGB (`rgb(255, 255, 255)`), RGBA(`rgba(255, 255, 255, 1.0)`)   |
| `opacity`     | percentage (`70%`)                                                              |
| `font-family` | quoted string
| `font-size`   | integers (pt)                                                                   |
| `font-weight` | `thin`, `regular`, `bold`                                                       |
| `line-height` | integers (pt), percentage (`70%`)                                               |

It is intentionally defined to be web-like, but it should be kept in mind that they are not identical and may be used differently by different platforms.

#### Conditional Binding

The actual value pointed by a scale token can be changed according to user preference or some application context.

A scale token definition can includes a condition to specify the context.

```
$scale(theme=light).color.carrot-500 -> #ff7e36
$scale(theme= dark).color.carrot-500 -> #ed7735
```

Sometimes you need token that have a fixed value regardless the context. You can mark it as "static".

```
$static.color.white -> #fff;
```

The `$static` is semantically equivalent to a `$scale(*)`, but it is defined in a separate namespace so that it can have a name that is duplicated with a scale token.

### Semantic Token Definitions (`$semantic`)

The format of semantic token is `{prefix}.{group}.{name}`. `group` and `name` is unique identifier.

```
$semantic.color.primary
$semantic.color.secondary

$semantic.typography.title
$semantic.typography.subtitle
```

The semantic tokens can have reference binding to scale tokens via `->` operator.

```
$semantic.color.primary -> $scale.color.carrot-500
```

Also, it can be compsition of multiple references.

```
$semantic.color.background -> $scale(theme=dark).color.gray-100
$semantic.color.background -> $static.color.white
```

### Token Description

You can leave a single line of comment on a specific token using `??` operator.

```
$semantic.color.primary ?? This is primary color
```

The description of every node is initialized to an empty string in the absence of an explicit binding.

## Design Token Scheme

Scheme is the source of truth, an intepreter including all information about the confirmed design tokens.

The AST of KDT is converted to a scheme to be used.

WIP:

```ts
type ScaleTokenDefinition = {
  type: 'scale' | 'static',
  description: string,
  deprecated: boolean,
};

type SemanticTokenDefinition = {
  type: 'semantic',
  description: string,
  deprecated: boolean,
};
```

### Resolving Value of a Semantic Token

A semantic token is defined as a composition of different scale tokens.

Only one value per scale target can be reflected in the actual program. The condition always chooses the more specific one declared last.

TBD

## Extensions & Macros (`%`)

KDT can direct additional runtime behavior via macros. Macros are defined as `%{extension}:{name}({arguments})`.

Macros are extended with user extensions based on runtime requirements. only `KDT` extension is reserved.

### Pre-defined extension

KDT:

KDT is built-in, and is the only extension can transforms the scheme. The scheme retained as read-only for all other extensions. It may be included in the syntax later.

- `%KDT:scale(cond)`: Specifies the default condition to be used by subsequent scale token definitions.
- `%KDT:deprecate(token)`: Mark a specific token has been deprecated.

### Example extension for Figma

- `%figma:printf(format, token)`: Sets the frame text to the token (property) in specific format. (for documentation purpose)
- `%figma:fill(token)`: Sets the frame fill color to the value of the specific token.

## ABNF (incomplete)

```abnf
KDT = *Line

Line = (TokenDefinition / TokenDescription) EOL

TokenDefinition =
  / SemanticTokenDefinition
  / ScaleTokenDefinition

TokenDescription = Token _ DescribeOperator _ Text

Token =
  / SemanticToken
  / ScaleToken

SemanticTokenDefinition = SemanticToken [_ BindOperator _ (ScaleToken / StaticToken)] EOL

SemanticToken = SemanticPrefix . group:Identifier . name:Identifier

SemanticPrefix = "$semantic"

ScaleTokenDefinition =
  / FontFamilyTokenDefinition
  / FontSizeTokenDefinition
  / FontWeightTokenDefinition
  / LineHeightTokenDefinition
  / ColorTokenDefinition
  / OpacityTokenDefinition

FontFamilyTokenDefinition = FontFamilyToken [_ BindOperator _ value:StringLit] EOL
FontFamilyToken = (ScalePrefix / StaticPrefix) . target:"font-family" . name:Identifier 

FontSizeTokenDefinition = FontSizeToken [_ BindOperator _ value:PointLit] EOL
FontSizeToken = (ScalePrefix / StaticPrefix) . target:"font-size" . name:Identifier 

FontWeightTokenDefinition = FontWeightToken [_ BindOperator _ value:WeightLit] EOL
FontWeightToken = (ScalePrefix / StaticPrefix) . target:"font-weight" . name:Identifier 

LineHeightTokenDefinition = LineHeightToken [_ BindOperator _ value:(PointLit / PercentLit)] EOL
LineHeightToken = (ScalePrefix / StaticPrefix) . target:"line-height" . name:Identifier 

ColorTokenDefinition = ColorToken [_ BindOperator _ value:ColorLit] EOL
ColorToken = (ScalePrefix / StaticPrefix) . target:"color" . name:Identifier 

OpacityTokenDefinition = OpacityToken [_ BindOperator _ value:PercentLit] EOL
OpacityToken = (ScalePrefix / StaticPrefix) . target:"opacity" . name:Identifier 

ScalePrefix = "$scale" [Condition]
StaticPrefix = "$static"

BindOperator = "->"
DescribeOperator = "?"
```

## Questions (may unresolved)

TBD
