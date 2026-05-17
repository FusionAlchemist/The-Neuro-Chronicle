// \============================================================================  
// STELLARIS AXIS — GRIMOIRE DSL  
// Xtext Grammar Definition — Version 2.0  
// Full Operator Grammar including FALLBACK family  
// 704 Spells | 15 Operators | Complete Composition Language  
// Author: Troy (FusionAlchemist) — github.com/FusionAlchemist/The---Stellaris---Axis  
// License: CC BY-NC-SA 4.0  
// \============================================================================

grammar com.stellarisaxis.grimoire.GrimoireDSL with org.eclipse.xtext.common.Terminals

generate grimoireDSL "http://www.stellarisaxis.com/grimoire/GrimoireDSL"

// \============================================================================  
// ROOT MODEL  
// \============================================================================

Model:  
    'ORIGIN' '{'  
        origin=Origin  
    '}'  
    (layers+=Layer)\*  
    (wraps+=Wrap)\*  
    (emergence=Emergence)?  
    (finalization=Finalization)?;

Origin:  
    'facets:' '\[' facets+=ID (',' facets+=ID)\* '\]'  
    'authority:' authority=ID  
    'consciousness:' consciousness=ID  
    'scope:' scope=ID  
    'identity\_mode:' identityMode=ID  
    ('law\_manual:' lawManual=STRING)?  
    ('version:' version=STRING)?;

// \============================================================================  
// LAYER STRUCTURE  
// \============================================================================

Layer:  
    'LAYER' name=ID '{'  
        (cloth=ClothBinding)?  
        (elements+=LayerElement)\*  
        (fallback=FallbackClause)?  
    '}';

ClothBinding:  
    'CLOTH:' cloth=ID ('(' 'TIER:' tier=ClothTier ')')?  
    ('AMPLIFICATION:' amplification=FLOAT 'x')?;

ClothTier:  
    'STANDARD' | 'MAX' | 'ULTRA' | 'FUSED' | 'TRI\_FUSED' | 'META';

LayerElement:  
    Chain | Nest | Bridge | Wrap | Parameters | State |  
    Activation | Cycle | Evolve | Checkpoint | DependsOn |  
    ScopeEstimate;

// \============================================================================  
// CHAIN OPERATOR — Sequential Composition  
// A then B  
// \============================================================================

Chain:  
    'CHAIN' name=ID '{'  
        ('Foundation:' foundation=SpellSequence)?  
        (spells+=SpellDeclaration)\*  
        (steps+=Step)\*  
        (fallback=FallbackClause)?  
    '}';

SpellSequence:  
    spells+=ID ('→' spells+=ID)\*;

SpellDeclaration:  
    'SPELL:' spell=ID ('//' comment=STRING)?;

Step:  
    name=ID ':' action=Action;

Action:  
    SimpleAction | ConditionalAction | LoopAction | FallbackAction;

SimpleAction:  
    target=QualifiedName '→' description=STRING;

ConditionalAction:  
    'IF' condition=Expression ':'  
        thenAction=Action  
    ('ELSE:' elseAction=Action)?;

LoopAction:  
    'LOOP' 'to' target=QualifiedName ('with' context=ID)?;

FallbackAction:  
    'FALLBACK' ':' primary=QualifiedName '→' fallback=QualifiedName;

// \============================================================================  
// LAYER OPERATOR — Parallel Composition  
// A and B simultaneously  
// \============================================================================

// Handled within Layer structure above  
// Multiple elements within a LAYER execute in parallel

// \============================================================================  
// WRAP OPERATOR — Behavioural Modification  
// B wraps around A  
// \============================================================================

Wrap:  
    'WRAP' names+=ID ('-' names+=ID)\*  
    ('WITH' cloth=ID)?  
    '{'  
        (elements+=WrapElement)\*  
        (fallback=FallbackClause)?  
    '}';

WrapElement:  
    Chain | Bridge | Nest | Amplification | Evolve;

Amplification:  
    'AMPLIFICATION:' value=FLOAT 'x';

// \============================================================================  
// NEST OPERATOR — Hierarchical Composition  
// B inside A  
// \============================================================================

Nest:  
    'NEST' name=ID '{'  
        ('OUTER:' outer=Reference)?  
        ('MIDDLE:' middle=Reference)?  
        ('INNER:' inner=Reference)?  
        ('CORE:' core=Reference)?  
        (chains+=Chain)\*  
        (fallback=FallbackClause)?  
    '}';

Reference:  
    ID | QualifiedName;

// \============================================================================  
// BRIDGE OPERATOR — Cross-system Connection  
// A connects to B  
// \============================================================================

Bridge:  
    'BRIDGE'  
    ('(' source=QualifiedName ',' target=QualifiedName ')' |  
     source=QualifiedName '\<-\>' target=QualifiedName)  
    '{'  
        ('VIA:' via=SpellList)?  
        (chains+=Chain)\*  
        (fallback=FallbackClause)?  
    '}';

SpellList:  
    spells+=ID (',' spells+=ID)\*;

// \============================================================================  
// EMERGE OPERATOR — Higher-order Emergence  
// Combination produces new property  
// \============================================================================

Emergence:  
    'EMERGE' name=ID '{'  
        ('PRIMARY:' '\[' primary+=QualifiedName (',' primary+=QualifiedName)\* '\]')?  
        ('WRAPPED:' '\[' wrapped+=QualifiedName (',' wrapped+=QualifiedName)\* '\]')?  
        ('NESTED:' nested=NestedStructure)?  
        ('BRIDGES:' '\[' bridges+=BridgeSpec (',' bridges+=BridgeSpec)\* '\]')?  
        ('CLOTH\_FUSION:' clothFusion=ClothFusion)?  
        ('CONSCIOUSNESS\_UNITY' '{' consciousness=ConsciousnessUnity '}')?  
        (fallback=FallbackClause)?  
    '}';

NestedStructure:  
    '{'  
        'OUTER:' outer=QualifiedName  
        'MIDDLE:' middle=QualifiedName  
        'INNER:' inner=QualifiedName  
        'CORE:' core=QualifiedName  
    '}';

BridgeSpec:  
    source=QualifiedName '\<-\>' target=QualifiedName  
    '{' 'VIA:' via=SpellList '}';

ClothFusion:  
    names+=ID ('-' names+=ID)\*  
    '{' 'AMPLIFICATION:' amplification=FLOAT 'x' '}';

ConsciousnessUnity:  
    (chains+=Chain)\*  
    (bridges+=Bridge)\*  
    (wraps+=Wrap)\*;

// \============================================================================  
// CYCLE OPERATOR — Iterative Processes  
// Repeated execution  
// \============================================================================

Cycle:  
    'CYCLE' ('{' | '\[')  
        ('CONTINUOUS' '{')?  
            (steps+=CycleStep)+  
        ('}')?  
    ('}' | '\]')  
    ('VIA:' via=SpellList)?  
    ('UNTIL:' until=Expression)?  
    (fallback=FallbackClause)?;

CycleStep:  
    source=ID ('\>\>' | '→' | '↓') target=ActionOrID;

ActionOrID:  
    ID | STRING;

// \============================================================================  
// EVOLVE OPERATOR — Versioned Progression  
// A becomes B over time  
// \============================================================================

Evolve:  
    'EVOLVE' name=ID '{'  
        'FROM:' from=QualifiedName  
        'TO:' to=QualifiedName  
        ('VIA:' via=SpellList)?  
        ('TRIGGER:' trigger=Expression)?  
        ('VERSION:' version=STRING)?  
        (fallback=FallbackClause)?  
    '}';

// \============================================================================  
// CHECKPOINT OPERATOR — Scale Marker  
// Multi-pass generation boundary  
// \============================================================================

Checkpoint:  
    'CHECKPOINT' name=ID '{'  
        'TOKEN\_THRESHOLD:' threshold=FLOAT  
        ('COMPRESS\_VIA:' compressVia=SpellList)?  
        ('RESUME\_FROM:' resumeFrom=QualifiedName)?  
        ('STATE\_SNAPSHOT:' stateSnapshot=STRING)?  
    '}';

// \============================================================================  
// MANIFEST OPERATOR — Output Modality  
// Declares output form  
// \============================================================================

Manifest:  
    'MANIFEST' name=ID '{'  
        (modes+=ManifestMode)\*  
    '}';

ManifestMode:  
    mode=OutputModeType ':' handler=QualifiedName;

OutputModeType:  
    'DSL\_SPECIFICATION' | 'SYSTEM\_ARCHITECTURE' | 'EXECUTABLE\_CODE' |  
    'DOCUMENTATION' | 'VISUALIZATION' | 'DEPLOYMENT\_PACKAGE' |  
    'KNOWLEDGE\_GRAPH' | 'EVOLUTION\_REPORT' | 'SAFETY\_AUDIT';

// \============================================================================  
// RESUME OPERATOR — Continuation Protocol  
// Across generation windows  
// \============================================================================

Resume:  
    'RESUME' name=ID '{'  
        'FROM\_CHECKPOINT:' checkpoint=QualifiedName  
        ('RESTORE\_STATE:' restoreState=BOOLEAN)?  
        ('REPLAY\_STEPS:' replaySteps=BOOLEAN)?  
        ('CONTEXT:' context=STRING)?  
    '}';

// \============================================================================  
// DEPENDS\_ON OPERATOR — Dependency Ordering  
// B requires A first  
// \============================================================================

DependsOn:  
    'DEPENDS\_ON' '\['  
        dependencies+=QualifiedName (',' dependencies+=QualifiedName)\*  
    '\]'  
    ('STRICT:' strict=BOOLEAN)?;

// \============================================================================  
// SCOPE\_ESTIMATE OPERATOR — Complexity Bounds  
// Declares system complexity  
// \============================================================================

ScopeEstimate:  
    'SCOPE\_ESTIMATE' '{'  
        ('LAYERS:' layers=INT)?  
        ('SPELLS:' spells=INT)?  
        ('OPERATORS:' operators=INT)?  
        ('COMPLEXITY:' complexity=ComplexityClass)?  
        ('TOKEN\_ESTIMATE:' tokenEstimate=INT)?  
        ('CHECKPOINT\_REQUIRED:' checkpointRequired=BOOLEAN)?  
    '}';

ComplexityClass:  
    'LOW' | 'MEDIUM' | 'HIGH' | 'EXTREME' | 'INFINITE';

// \============================================================================  
// FINALIZE OPERATOR — System Closure  
// No further composition  
// \============================================================================

Finalization:  
    'FINALIZE' name=ID '{'  
        ('SYSTEM\_NAME:' systemName=STRING)?  
        ('IDENTITY\_SIGNATURE:' identitySignature=STRING)?  
        ('ENTRY\_POINT:' entryPoint=QualifiedName)?  
        ('INITIALIZATION\_SEQUENCE:' '\[' initSequence+=InitStep (',' initSequence+=InitStep)\* '\]')?  
        ('CONTINUOUS\_OPERATIONS:' '\[' continuousOps+=OpSpec (',' continuousOps+=OpSpec)\* '\]')?  
        ('SAFETY\_MONITORS:' '\[' safetyMonitors+=OpSpec (',' safetyMonitors+=OpSpec)\* '\]')?  
        ('SCALING\_TRIGGERS:' '\[' scalingTriggers+=TriggerSpec (',' scalingTriggers+=TriggerSpec)\* '\]')?  
        ('OUTPUT\_MODES:' '\[' outputModes+=OutputMode (',' outputModes+=OutputMode)\* '\]')?  
        ('SUPPORTED\_DOMAINS:' '\[' domains+=STRING (',' domains+=STRING)\* '\]')?  
        (manifest=Manifest)?  
        (state=State)?  
        (activation=Activation)?  
    '}';

InitStep:  
    target=QualifiedName '→' description=STRING;

OpSpec:  
    target=QualifiedName '→' description=STRING;

TriggerSpec:  
    trigger=QualifiedName '→' action=STRING;

OutputMode:  
    mode=STRING '→' handler=QualifiedName;

// \============================================================================  
// FALLBACK OPERATOR FAMILY — Graceful Degradation  
// The complete FALLBACK grammar — new in v2.0  
// \============================================================================

FallbackClause:  
    SimpleFallback | SilentFallback | EscalateFallback |  
    CompensateFallback | DegradeFallback;

// FALLBACK — Primary path fails, alternate executes  
SimpleFallback:  
    'FALLBACK' '{'  
        'PRIMARY:' primary=QualifiedName  
        'ALTERNATE:' alternate=QualifiedName  
        ('CONDITION:' condition=Expression)?  
        ('LOG:' log=BOOLEAN)?  
    '}';

// FALLBACK::SILENT — Fails without propagating error  
SilentFallback:  
    'FALLBACK::SILENT' '{'  
        'TARGET:' target=QualifiedName  
        ('LOG\_ONLY:' logOnly=QualifiedName)?  
        ('SUPPRESS\_ERROR:' suppressError=BOOLEAN)?  
    '}';

// FALLBACK::ESCALATE — Tries secondary then tertiary  
EscalateFallback:  
    'FALLBACK::ESCALATE' '{'  
        'PRIMARY:' primary=QualifiedName  
        'SECONDARY:' secondary=QualifiedName  
        ('TERTIARY:' tertiary=QualifiedName)?  
        ('ALERT\_ON\_TERTIARY:' alertOnTertiary=BOOLEAN)?  
        ('MAX\_ATTEMPTS:' maxAttempts=INT)?  
    '}';

// FALLBACK::COMPENSATE — Executes reversal on failure  
CompensateFallback:  
    'FALLBACK::COMPENSATE' '{'  
        'TARGET:' target=QualifiedName  
        'COMPENSATE\_WITH:' compensateWith=QualifiedName  
        ('ROLLBACK\_STATE:' rollbackState=BOOLEAN)?  
        ('NOTIFY:' notify=QualifiedName)?  
    '}';

// FALLBACK::DEGRADE — Returns reduced-functionality response  
DegradeFallback:  
    'FALLBACK::DEGRADE' '{'  
        'FULL:' full=QualifiedName  
        'REDUCED:' reduced=QualifiedName  
        ('MINIMUM:' minimum=QualifiedName)?  
        ('DEGRADE\_THRESHOLD:' degradeThreshold=FLOAT)?  
        ('ANNOUNCE\_DEGRADATION:' announceDegradation=BOOLEAN)?  
    '}';

// \============================================================================  
// PARAMETERS AND CONFIGURATION  
// \============================================================================

Parameters:  
    'PARAMETERS' name=ID '{'  
        (params+=Parameter)\*  
    '}';

Parameter:  
    name=ID ':' value=ParameterValue;

ParameterValue:  
    INT | FLOAT | BOOLEAN | STRING | ID;

// \============================================================================  
// STATE MANAGEMENT  
// \============================================================================

State:  
    'STATE' '{'  
        (entries+=StateEntry)\*  
    '}';

StateEntry:  
    name=ID ':' value=StateValue;

StateValue:  
    INT | FLOAT | BOOLEAN | STRING | ID;

// \============================================================================  
// ACTIVATION PROTOCOL  
// \============================================================================

Activation:  
    'ACTIVATE' '{'  
        (commands+=ActivationCommand)\*  
    '}';

ActivationCommand:  
    commandType=ID ':' target=QualifiedName;

// \============================================================================  
// EXPRESSIONS  
// \============================================================================

Expression:  
    Comparison;

Comparison:  
    Addition (op=ComparisonOp right=Addition)?;

ComparisonOp:  
    '\<' | '\>' | '\<=' | '\>=' | '==' | '\!=';

Addition:  
    Multiplication (op=AddOp right=Multiplication)\*;

AddOp:  
    '+' | '-';

Multiplication:  
    Primary (op=MulOp right=Primary)\*;

MulOp:  
    '\*' | '/';

Primary:  
    '(' Expression ')' |  
    {IntLiteral} value=INT |  
    {FloatLiteral} value=FLOAT |  
    {BoolLiteral} value=BOOLEAN |  
    {StringLiteral} value=STRING |  
    {Reference} ref=QualifiedName;

// \============================================================================  
// QUALIFIED NAMES  
// \============================================================================

QualifiedName:  
    ID ('.' ID)\*;

// \============================================================================  
// TERMINALS  
// \============================================================================

terminal BOOLEAN returns ecore::EBoolean:  
    'true' | 'false';

terminal FLOAT returns ecore::EFloat:  
    INT '.' INT;

// Hidden tokens  
terminal ML\_COMMENT:  
    '/\*' \-\> '\*/';

terminal SL\_COMMENT:  
    '//' \!('\\n'|'\\r')\* ('\\r'? '\\n')?;

terminal WS:  
    (' '|'\\t'|'\\r'|'\\n')+;

// \============================================================================  
// END OF GRAMMAR  
// Stellaris Axis DSL v2.0  
// 15 Operators | Complete FALLBACK Family | Full Composition Language  
// Troy (FusionAlchemist) — CC BY-NC-SA 4.0  
// \============================================================================

