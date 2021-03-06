# STLR - A Native Swift Lexer and Parser Generation Language

# Introduction
STLR is a language for specifying grammars that is inspired by many of the principles that Swift uses. It is intended to be as simple and readable as possible, and although the vision is for generation of multiple output languages, my goal is where there is not a clear overriding convention used for language recognition, to adopt a Swift like approach. 

Like many modern language recongition tools, STLR allows you describe not just rules for tokenisaiton, but rules for parsing that can be directly used for infering structure. However it has a number of 
key simplifications:

 - __Scanning__ 
   - You do not need to differentiate between scanner rules or grammar rules, meaning you are free to express your language without thinking about the underlying implementation. 
   - An optimizer is avilable and integrated into the STLR interpreter that will attempt to make scanning and parsing into an AST as efficient as possible
   - When you do wish to express a complex scanning rule, regular expressions are available to do this
 - __Parsing__
   - STLR can automatically generate a homogenous tree builder from your grammar. This is very similar to a dictionary, and can be easily traversed. 
   - In addition STLR can also automatically generate Swift intermediate representations drawing on your grammar to give it meaningful names, and where possible type safety. In more simple terms, STLR will generate classes and structs to represent your parsed language, so you don't have to navigate a dictionary. 

STLR eats its own dog food, and its own parser and intermediate representation are generated by STLR. As Stephen Hawking said... it's turtles all the way down. 

# Usage

OysterKit provides a framework for lexical analysis and parsing the resultant tokens into an intermediate representation (such as an Abstract Syntax Tree, or a stream of validated tokens). STLR builds on this by providing a language that can be used to create parsers using the OysterKit framework. This can be done by loading and "compiling" files or to directly create rules from strings in your code. 

Once compiled (in memory) STLR can then create a parser directly in memory, or generate Swift source code to capture the language you have defined. 

## Defining a rule in STLR

Rules can be defined very simply 

_token-identifier_ = _expression_

For example, if we wanted to define a simple token `digit` that was any decimal digit (there are much more efficient ways of doing this using STLR, but this is given as an illustrative example). 

    digit = "0" | "1" | "2" | "3" | "4" | "5" | "6" | "7" | "8" | "9" 
    
A token, `digit` is defined as being any one of the strings separated by the `|` character. We can also define sequences, very simply. This token can then be referenced by another token definition. For example, 

    threeDigitNumber = digit digit digit
    
When STLR compiles this it creates a rule for each token. 

### Defining identifier type

STLR (the library and ```stlrc``` the command line tool) is able to generate the source code to parse and build the abstract syntax tree for your grammar. When it does this it must make a number of inferences about the types (if any) each identifier represents. You can explicitly control this by providing the type name after the identifier of a rule. For example:

    wholeNumber : Int = .decimalDigit+
    
The field generated would then have the ```Int``` datatype. The following types are predefined and are treated in output language sensitive way (that is, the source code generator will ensure they
are an appropriate type for the language being generated)

- Int: An integer
- Double: A floating point number
- String: A string
- Boolean: A boolean value

You may optionally declare any other type you wish, but note that the type name must begin with an upper-case letter. You will also be responsible for ensuring that that type is availble in the build environment of the target language (either it is part of the language or you have defined it yourself such that it can be accessed). 

For Swift there generation the type must also implement the ```Codable``` protocol as decoding this is used for building and persisting the AST. 

## Expressions 

Expressions are made up of elements and other expressions. Elements can be terminals, groups, or identifiers and can have one or more modifiers that alter the meaning of the element (such as `!` or not) or the number of the element that is expected (for example `+` means one or more). 

We'll start by looking at groups, as these are the most fundamental elements.

### Groups
Groups are composed as a sequence or choice of other elements. For example 

     "http" ":" "//"
     
Captures a sequence of strings (or terminals) that would match `http://`. If we wanted to specify this as a choice we would use the or operator `|`.

    "a" | "b" | "c"
    
Would match `a` or `b` or `c` in a string. When you wish to combine these two group types, you may use brackets to do so. For example we might refine our first sequence to match either `http` or `https` by nesting a choice at the start of the group. 

    ("http" | "https") ":" "//"
   
 Would match either `http://` or `https://` 
 
 ### Terminals

Terminals are specific characters or sequences of characters, such as a string. String are simply defined by wrapping the desired text in quotations marks, eg: `"http:"`. 

If you wish to include special characters, or indeed a quotation mark, you can insert escaped characters using the backslash. At this point the following escapes are supported: 

  - `\\` A backslash (note you can also use the .backslash character set)
  - `\"` A quotation mark 
  - `\r` A carriage return
  - `\n` A new line
  - `\t` A tab

More will be added in the future. 

You can also use predefined character sets that match multiple characters. These are prefixed by a `.` and followed by the character set name. The following character sets are available at the moment:

  - `.decimalDigit` All decimal digits from 0 to 9
  - `.letter` All letters 
  - `.uppercaseLetter` All uppercase letters
  - `.lowercaseLetter` All lowercase letters
  - `.alphaNumeric` All letters and digits
  - `.whitespace` All whitespaces. For example tabs and spaces. 
  - `.newline` All newline characters, matching for example both newline and carriage return
  - `.whitespaceOrNewline` All white space and newline characters
  - `.backslash` The backslash character
  - `.endOfFile` The end of the file. NB: Normally you will use lookahead with this character class (e.g. if you want to ensure you are at the end of the file `>>.endOfFile` or that you  are not at the end of the file `>>!.endOfFile`)
  
For example, a rule to capture a co-ordinate might be

    coord = .decimalDigit "," .decimalDigit
    
This would match `3,4` for example. 

You may also specify terminals using regular expressions. To do this, enclose the regular expression in ```/``` characters. For example

        keyword = /for|next|while|wend/

Would match ```for```, ```next```, ```while```, or ```wend```. The syntax is the same as NSRegularExpression. 

### Identifiers 

We can also reference other tokens in a sequence or choice by using their identifier (or name). For example we might define a token for our different web protocols earlier and use it in a rule for a URL. 

    protocol = "http" | "https"
    url      = protocol "://" //I'll finish this later
    
When matching with `url` the rule will first evaluate the `protocol` rule and then if matched continue through the sequence. This allows complex rules to be built up. 

STLR fully supports left-hand recursion, meaning a rule can refer directly or indirectly to itself. However you should be aware that you must always advance the scanner position before doing so (otherwise you will enter an infinite loop). 

## Modifiers

We often want to change how an element is matched. For example repeating it, or just checking it is there without advancing the scanner position (lookahead). The following modifiers are available before or after any element (terminal, group, or identifier). 

  - `?` Optional (0 or 1 instances of the element): This might be used to simplify our protocol rule for example `protocol = "http" "s"?`. This is much closer to how we think about it, `http` followed optionally by an `s`. It is added as a suffix to the element. 
  - `*` Optional repeated (0 or any number of instances of the element): This is used when an element does not have to be there, but any number of them will be greedily consumed during matching. It is added as a suffic to the element. 
  - `+` Required repeated (1 or any number of instances of the element): This is used when an element MUST exist, and then once one has been matched, any number of subsequence instances will be greedily consumed. It is added as a suffix to the element. 
  - `!` Not: This is used to invert the match and is prefixed to the element and can be combined with any of the suffix operators. For example to scan until a newline we could specify `!.newlines*`. This would scan up to, but not including, the first newline character found
  - `>>` Look ahead: This is used when you want to lookahead (for example, skip the next three characters, but check the fourth). It is prefixed to the element and like the not modifier can be combined with any suffix modifier. Regardless of matching or failure the scanner position will not be changed. However the sequence will only continue to be evaluated if it is met. 
  
## Comments

Comments can be single line or multiline. Single line comments begin whenever `//` is seen, and all further input will be ignored by the compiler until the end of the line. Multiline comments begin with `/*` and end only when `*/` is matched. 

## Annotations

Annotations change how elements are interpreted and principally impact construction of the AST (Abstract Syntax Tree). The are prefixed by the `@` character and can also have a supplied value which can be a string, integer, or boolean. If no value is provided they are simply considered 'set'. 

Annotations can be applied inline to elements in an expression, or to a token-identifier when it is first being defined. If both are provided the inline annotation will override the token-identifier's annotation. 

### @token(String) 

The next element will result in a token with the provided String value. 

### @transient

We do not always want tokens met to be included in the AST. If prefixed with @transient then the token will not be captured in the AST or stream. For example, if we do not want any newlines to be captured in our AST but we do need to capture both types of newline we could define the following rule

    @transient crlf = "\r\n" | "\n"
    
However the match will still be included in the range of the parent node and any non-transient child nodes of the annotated node will be adopted by the parent node. If you wish to completely ignore the match and proceed with evaluation you should use the @void annotation. 

You can also use the ```~``` as a prefix as a shortcut to this. It can be applied to either the declaration of a rule

    ~crlf = "\r\n" | "\n"
    
Or inline for an element of a rule. For example

    crlf = "\r\n" | "\n"
    rule = code ~crlf

### @void

When a token of this type is matched scanning will continue after the match but no nodes will be created and any children of a void node will be completely discarded. The range of this match will not be applied to the range of the parent. 

As with transient annotations you can use a prefix, in this case ```-```, to simplify annotating either the rule itself

    -quote = "'"
    
Or inline on an element of a rule's expression

    quote = "'"
    string = -quote stringBody -quote

### @error(String)

When your defined language is being parsed, if the element annotated with this fails the match the specified error message will be available in the results. For example 

    url = @error("You must specify a protocol") protocol "://" 
    
Will generate an error explaining why matching failed. 

### @pin

When an optional non-transient element is not matched a node will be created regardless of whether or not the optional element was matched. It will also ensure that if this is the only non-transient child created for a node that it will be not be hoisted (normally if a node has only one child, the child is raised to the level of the parent to reduce complexity of the AST). 

This can be convenient if you wish to depend on the order and structure of children for some node of your AST without needing to implement your own AST that could implement this behaviour. 

## Order of evaluation

It can be important to understand the order of evaluation of the various matching operators as it can impact both behaviour during matching and the generated structure. 

1. Grouping (i.e. ```( expression )```)
2. Negation  (i.e. ```!```)
2. Quantifiers (e.g. ```+```, ```*```,  ```+```, or ```?```)
3. Annotations 
4. Structural modifiers (e.g. ```~```, ```-```, or  ```token : <expression>```)
4. Lookahead (i.e. ```>>```)
5. Sequences & choices (e.g. ```|```)

For example consider the expression

    name   = @token("forename") !","+ "," @token("surname") !","+
    
Will first negate the ```","``` test, then require one or more of those not comma characters and finally if it finds one or more characters that are not whitespace, generate a ```forename``` then, if separated by a ```,```,  a ```surname``` token. Which will be children of a ```name``` token.  

    name - 'John,Appleseed`
        forename - `John`
        surname - `Appleseed`

A more complex (although somewhat unnecessarily so) example is

    letter = .letter
    word   = ~letter+
    
Will match multiple letters, discarding all of the ```letter``` tokens created but wrapping the match in a ```name``` token producing

    word - tree

## Examples

### Number Parser


    real		= sign? digits point digits exponent?
    exponent 	= "e" integer
    integer  	= sign? digits
    
    sign 		= "+" | "-"
    digits 		= .decimalDigit+
    spaces 		= .whitespaceOrNewline+
    point 		= "."



### HTTP Request

    //
    // HTTP 1.1 Request (Abridged)
    //
    // Created as an example
    //

    //
    // Constants 
    //
    method              = "GET" | "POST" 
    protocol            = "HTTP/1.1"
    @transient crlf     = .newline
    
    //
    // Requested path
    //
    path                = pathSep? pathElement? (pathSep pathElement?)* @transient pathSep = "/"
    pathElement         = (.letter | .decimalDigit)+
    
    //
    // Query
    //
    queryParameterName  = .letter+
    queryParameterValue = .letter | .decimalDigit 
    queryParameter      = queryParameterName ("=" queryParameterValue)?
    query               = "?" queryParameter ("&" queryParameter)* 
    
    //
    // All content before header
    //
    transaction         = method .whitespace+ path query? .whitespace+ protocol crlf
    
    //
    // Header
    //
    headerName          = (.letter | "-")+
    headerValue         = !(crlf)+
    header              = headerName ":" .whitespaces headerValue crlf
    
    headers             = header+
    
    //
    // Root rule
    //
    request             = transaction headers crlf



### Simple Basic   
    
    //
    // A moderately updated version of   
    // https://www.ecma-international.org/publications/files/ECMA-ST-WITHDRAWN/ECMA-55,%201st%20Edition%20January%201978.pdf     
    //
    
  
    //program					= (line endLine)* line?
    
    //program					= (block | remarkLine)* endLine
    //block						= ((line | forBlock) .whitespaceOrNewline*)*
    line						= lineNumber ows statement
    
    remarkLine					= lineNumber ws statementREM endOfLine?
    endLine						= lineNumber ws statementEND ows .newline?
    
    lineNumber					= digit+ 
    
    forBlock					= "SOMETHING FOR A FOR BLOCK"
    
    statement					= statementLET
    statementEND				= keywordEND ows
    statementREM			 	= keywordREM ( " "+ remarkString? )?
    statementLET				= keywordLET ws @error("expected variable") variable ows @error("expected '='") equalsSign ows @error("expected expression") expr
    
    
    expr						= operand (ows? operator ows? operand)*
    operator					= "+" | "-" | "/" | "*" | "^"
    operand						= (variable | literal | function) | (leftParenthesis expr rightParenthesis)
    
    function					= functionName argumentList?
    argumentList				= leftParenthesis argument rightParenthesis
    argument					= expr
    
    functionName				= "SIN" | "COS"
    
    
    variable					= arrayElement | variableName
    variableName				= letter (letter | digit)*
    arrayElement				= variableName subscript
    
    subscript					= leftParenthesis expr (comma expr)? rightParenthesis
    
    literal						= number | quotedString
    
    number						= sign? numericRepresentation
    sign						= plusSign | minusSign
    numericRepresentation		= significand exrad?
    significand					= (integer fullStop?) | (integer? fraction)
    integer						= digit+
    fraction					= fullStop digit+
    exrad						= "E" sign? integer
    
    keywordLET					= "LET"
    keywordREM					= "REM" | "rem" 
    keywordEND					= "END"
    
    // The very basics
    letter 						= .letters 
    digit  						= .decimalDigits
    
    
    
    // Specific characters
	    exclamationMark				= "!"
	    numberSign					= "???" // What's this??
	    dollarSign					= "$"
	    space						= " "
	    plusSign					= "+"
	    minusSign					= "-"
	    percentSign					= "%"
	    ampersand					= "&"
	    apostrophe					= "'"
	    leftParenthesis				= "("
	    rightParenthesis			= ")"
	    asterisk					= "*"
	    comma						= ","
	    solidus						= "/" 
	    colon						= ":"
	    semicolon					= ";"
	    lessThanSign				= "<"
	    greaterThanSign				= ">"
	    equalsSign					= "="
	    questionMark				= "?"
	    circumflexAccent			= "^"
	    underline					= "_"
	    quotationMark				= "\""
	    fullStop					= "."
	    endOfLine					= .newlines
    
    stringCharacter 			= quotationMark | quotedStringCharacter
    plainStringCharacter		= plusSign | minusSign | fullStop | digit | letter
    
    unquotedStringCharacter 	= space | plainStringCharacter
    quotedStringCharacter		= exclamationMark | numberSign | dollarSign |
 							    dollarSign | percentSign | ampersand | 
							    apostrophe | leftParenthesis | 
							    rightParenthesis | asterisk | comma | 
							    solidus | colon | semicolon | lessThanSign | 
							    equalsSign | greaterThanSign | questionMark |
							    circumflexAccent | underline | 
							    unquotedStringCharacter 
    
    remarkString				= stringCharacter*
    quotedString				= quotationMark quotedStringCharacter* quotationMark
    
    
    nl = .newline*
    ws = .whitespace+
    ows = .whitespace*
    
    consume = .whitespace*
