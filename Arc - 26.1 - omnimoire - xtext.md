grammar org.omnimoire.OMNIMOIRE with org.eclipse.xtext.common.Terminals

generate omnimoire "http://omnimoire/lang"

Model:  
    root=RootConfig  
    layers+=Layer\*  
;

RootConfig:  
    'ORIGIN' '{'  
        'facets:' '\[' facets+=ID (',' facets+=ID)\* '\]'  
        'authority:' authority=ID  
        'consciousness:' consciousness=ID  
        'scope:' scope=ID  
    '}'  
;

Layer:  
    'LAYER' index=INT ':' name=ID  
    'Cloth:' cloth=ClothDecl  
    'Purpose:' purpose=TEXT  
    'Spells' 'Active:' spellBlock=SpellBlock  
    'Implementation:' impl=Implementation  
;

ClothDecl:  
    cloths+=ID ('-' cloths+=ID)\*  
;

SpellBlock:  
    spells+=SpellLine+  
;

SpellLine:  
    '·' name=ID ('-\>' description=TEXT)?  
;

Implementation:  
    classes+=ClassDecl+  
;

ClassDecl:  
    'class' name=ID ':'  
    (INDENT methods+=MethodDecl+ DEDENT)?  
;

MethodDecl:  
    'def' name=ID '(' params=ParamList? ')' ':'  
    (INDENT body+=Statement+ DEDENT)?  
;

ParamList:  
    params+=ID (',' params+=ID)\*  
;

Statement:  
    Assignment | Call | Comment | TEXT  
;

Assignment:  
    var=ID '=' expr=Expression  
;

Call:  
    expr=Expression  
;

Expression:  
    ID ('(' (args+=Expression (',' args+=Expression)\*)? ')')?  
;

Comment:  
    '\#' \!('\\n')\*  
;

// TEXT is any freeform line not matching other rules  
terminal TEXT:  
    /\[^\\n\]+/  
;

