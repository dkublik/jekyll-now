---
layout: post
title: The Magic Behind the Spock
comments: true
---
Spock rulez. Writing tests has never been simpler and more pleasant. All thanks to concise and informative syntax. But how is Spock syntax made possible?

Expressions like
* _ValueProvider valueProvider = Stub()_
* _valueProvider.provideValue() >> 21_
* _then:_

what makes them work? If you're interetsed follow my plan which is to focus on simple Spec like:
```groovy
class MagnifyingProxySpec extends Specification {

    ValueProvider valueProvider = Stub()

    @Subject
    MagnifyingProxy magnifyingProxy = new MagnifyingProxy(valueProvider)

    def "should magnify value"() {
        given:
            valueProvider.provideValue() >> 21
        when:
            int result = magnifyingProxy.provideMagnifiedValue()
        then:
            result == 210
    }
}
```

and a present high level view of what makes it tick.

&nbsp;

#### It all compiles

Let's start with noticing that none of the expresions above are against Groovy syntax (in which Spock specifications are written).

* _ValueProvider valueProvider = Stub()_  
Is a _MethodCallExpression_. Namely calling _Stub()_ method and assigning it to a valueProvider variable (method name starting with capital letter is only a disguise).

* _valueProvider.provideValue() >> 21_  
Is a BinaryExpression using _>>_ operator on _valueProvider.provideValue()_ and _21_

* _then:_  
and 'when:' and 'given:' are Groovy labels - counstructs that (repeating after [doc](http://groovy-lang.org/semantics.html#_labeled_statements)) have no impact on the semantics of the code but can be used to make the code easier to read.

So nothing against Groovy compilator here, but a little against the Groovy runtime.
Run the above Spec without the Spock magic and 

* _Stub()_  
will throw _InvalidSpecException_ as this is as it is defined in [MockingApi](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/spock/lang/MockingApi.java) class

* _>>_  
will be call to [DefaultGroovyMethods.rightShift(Number self, Number operand)](https://github.com/groovy/groovy-core/blob/GROOVY_2_4_3/src/main/org/codehaus/groovy/runtime/DefaultGroovyMethods.java) and not stubbing a method call

* _then:_  
will just be a label with no extra functionality like asserting every comparision


So somewhere between a compilation and an execution must exist an extension point to which Spock reaches to perfom it's tricks.

&nbsp;

#### Groovy compiler and AST representation

Remember the _MethodCallExpression_ and _BinaryExpression_ terms I used above? These are just two elements of Groovy source code representation as the Abstract Synax Tree (AST).
The idea is to represent every block of code by some subtree of ASTNodes.

The AST abstraction is used in the compilation process.			
When Groovy compiler transforms _.groovy_ files into Java byte code, the overall work is divided into different phases, every of them performing different task.
These phases can be found in [Phases](https://github.com/groovy/groovy-core/blob/GROOVY_2_4_3/src/main/org/codehaus/groovy/control/Phases.java) class

```java
public static final int INITIALIZATION        = 1;   // Opening of files and such
public static final int PARSING               = 2;   // Lexing, parsing, and AST building
public static final int CONVERSION            = 3;   // CST to AST conversion
public static final int SEMANTIC_ANALYSIS     = 4;   // AST semantic analysis and elucidation
public static final int CANONICALIZATION      = 5;   // AST completion
public static final int INSTRUCTION_SELECTION = 6;   // Class generation, phase 1
public static final int CLASS_GENERATION      = 7;   // Class generation, phase 2
public static final int OUTPUT                = 8;   // Output of class to disk
public static final int FINALIZATION          = 9;   // Cleanup
public static final int ALL                   = 9;   // Synonym for full compilation
```
Basically the whole process built from these phases can be summarized as:
- read data from some input (source file, String script, etc)
- transform it to more robust representation - _AST (Abstract Syntax Tree)_
- complement and transform the _AST_ representation 
- generate _GroovyClass_ from _AST_
- write _GroovyClass_ as a _.class_ file 

The crucial point here is that the _AST_ is exposed not only for _Groovy's_ internal usage, but also for user defined external _AST_ transformations. And this is where the Spock comes in.

&nbsp;

#### Spock AST transformation

When _CompilationUnit_ is first created by _GroovyClassLoader_, it collects all global transformations it can find in _META-INF/services/org.codehaus.groovy.transform.ASTTransformation_ files.  
open [org.codehaus.groovy.transform.ASTTransformation](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/resources/META-INF/services/org.codehaus.groovy.transform.ASTTransformation) file in _spock-core_ jar and you'll find
```
org.spockframework.compiler.SpockTransform 
```
which is the Spock _AST_ transformation being the entry point for the whole process, as we can read in java doc above the class:

```java
/**
 * AST transformation for rewriting Spock specifications. Runs after phase
 * SEMANTIC_ANALYSIS, which means that the AST is semantically accurate
 * and already decorated with reflection information. On the flip side,
 * because types and variables have already been resolved,
 * program elements like import statements and variable definitions
 * can no longer be manipulated at will.
 *
 * @author Peter Niederwieser
 */
@GroovyASTTransformation(phase = CompilePhase.SEMANTIC_ANALYSIS)
public class SpockTransform implements ASTTransformation
```

_SpockTransform_ is a global transformation, meaning it will be executed once for every Groovy class being compiled (as opposite to local transformation performed only on marked classes, e.g. by an annotation).  
At the very beginning _SpockTransform_ checks if given class is derived from [Specification](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/spock/lang/Specification.java) and if so, the tranformation process begins.


Spock steps in during _SEMANTIC_ANALYSIS_ phase. Taking into consideration only the points we're investigating, the _AST representation_ of MagnifyingProxySpec class before _Spock_ transformation, can be simplified to:

```
ClassNode: MagnifyingProxySpec
  fields:
    [0]: name: valueProvider
         type: ValueProvider -> ValueProvider
         initialValueExpression:
             MethodCallExpression: Stub()
    [1]: name: magnifyingProxy
         type: MagnifyingProxy -> MagnifyingProxy
         initialValueExpression:
             ConstructorCallExpression: new MagnifyingProxy(valueProvider)
  methods:
    name: "should magnify value"
    code:
      statements:
        [0]: expresion: 
               BinaryExpression:
                 leftExpresion:  valueProvider.provideValue() 
                 rightExpresion: 21
                 operation: >>	
             statementLabels: given
        [1]: expresion: 
               DeclarationExpression:
                 leftExpresion: int result 
                 rightExpresion: magnifyingProxy.provideMagnifiedValue()	
                 operation: =
             statementLabels: when
        [2]: expresion:
               BinaryExpression:
                 leftExpresion: result
                 rightExpresion: 210
                 operation: == 
             statementLabels: then
```

&nbsp;

####  Rewriting the AST
					 
During the _SpockTransform_ a lot is happening.


* the above _ClassNode_ (Node representing class in AST) is taken by [SpecParser](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/compiler/SpecParser.java) and turned into [Spec](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/compiler/model/Spec.java) - more convinient representation where among others - labeled statements (by given, when, then labels) are turned into blocks for every encountered feature method.
(Check [BlockParseInfo](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/compiler/model/BlockParseInfo.java) class and you will see what labels are allowed)

* the same _ClassNode_ is being rewritten by [SpecRewriter](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/compiler/SpecRewriter.java) - this is where most of the AST transformation is happening

* As the _AST_ is rewritten, some information about original structure still needs to be kept - these are added to the new _AST_ structure in a form of annotations by [SpecAnnotator](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/compiler/SpecAnnotator.java)


After tranformation is finished the AST tree will look like:  
(it's a simplified view, so the changes to original are more vivid - e.g. all annotations info is removed and complex subtrees are flatten to String values)
```
ClassNode: MagnifyingProxySpec
  fields:
    [0]: name: valueProvider
         type: ValueProvider -> ValueProvider
         initialValueExpression: null
    [1]: name: magnifyingProxy
         type: MagnifyingProxy -> MagnifyingProxy
         initialValueExpression: null
  methods:
    [0]: name: "$spock_initializeFields"
         code:
           statements:
             [0]: expresion: 
                    FieldInitializationExpression:
                        leftExpresion:  valueProvider
                        rightExpresion: StubImpl('valueProvider', ValueProvider)
                        operation: =	
             [1]: expresion: 
                    FieldInitializationExpression:
                        leftExpresion: magnifyingProxy
                        rightExpresion: new MagnifyingProxy(valueProvider)
                        operation: =
    [1]: name: "$spock_feature_0_0"
         code:
           statements:
             [0]: expresion: 
                    DeclarationExpression:
                        leftExpresion:  $spock_valueRecorder
                        rightExpresion: new ValueRecorder()
                        operation: =	
             [1]: expresion: 
                    MethodCallExpression: this.getSpecificationContext().getMockController().addInteraction(new InteractionBuilder(14, 13, 'valueProvider.provideValue() >> 21').addEqualTarget(valueProvider).addEqualMethodName('provideValue').setArgListKind(true).addConstantResponse(21).build())
             [2]: expresion:
                    DeclarationExpression:
                        leftExpresion: Integer result
                        rightExpresion: magnifyingProxy.provideMagnifiedValue()
                        operation: = 
             [3]: expresion:
                    MethodCallExpression: SpockRuntime.verifyCondition($spock_valueRecorder.reset(), 'result == 210', 20, 13, null, $spock_valueRecorder.record(2, $spock_valueRecorder.record(0, result) == $spock_valueRecorder.record(1, 210)))
             [4]: expresion:
                    MethodCallExpression: getSpecificationContext().getMockController().leaveScope()
```

							
Instead of describing it, it would be better to compare source code from _Groovy AST Browser_ before and after the _SpockTransform_ transformation:

before:
```groovy
public class MagnifyingProxySpec extends Specification { 

    private ValueProvider valueProvider 
    @Subject
    private MagnifyingProxy magnifyingProxy 

    public Object should magnify value() {
        valueProvider.provideValue() >> 21
        Integer result = magnifyingProxy.provideMagnifiedValue()
        result == 210
    }
}
```
after:
```groovy
@SpecMetadata(filename = 'script1488219488396.groovy', line = 5)
public class MagnifyingProxySpec extends Specification { 

    @FieldMetadata(line = 7, name = 'valueProvider', ordinal = 0)
    private ValueProvider valueProvider 
    @Subject
    @FieldMetadata(line = 9, name = 'magnifyingProxy', ordinal = 1)
    private MagnifyingProxy magnifyingProxy 

    private Object $spock_initializeFields() {
        valueProvider = this.StubImpl('valueProvider', ValueProvider)
        magnifyingProxy = new MagnifyingProxy(valueProvider)
    }

    @FeatureMetadata(line = 12, blocks = [
      []org.spockframework.runtime.model.BlockKind.SETUPorg.codehaus.groovy.ast.AnnotationNode@138c8ce,
      []org.spockframework.runtime.model.BlockKind.WHENorg.codehaus.groovy.ast.AnnotationNode@7e7261e6,
      []org.spockframework.runtime.model.BlockKind.THENorg.codehaus.groovy.ast.AnnotationNode@34e6c430],
      name = 'should magnify value', parameterNames = [], ordinal = 0)
    public void $spock_feature_0_0() {
        Object $spock_valueRecorder = new ValueRecorder()
        this.getSpecificationContext().getMockController()
            .addInteraction(
                new InteractionBuilder(14, 13, 'valueProvider.provideValue() >> 21')
                    .addEqualTarget(valueProvider)
                    .addEqualMethodName('provideValue')
                    .setArgListKind(true)
                    .addConstantResponse(21)
                    .build())
        Integer result = magnifyingProxy.provideMagnifiedValue()
        SpockRuntime.verifyCondition(
            $spock_valueRecorder.reset(), 'result == 210', 20, 13, null,
            $spock_valueRecorder.record(2, $spock_valueRecorder.record(0, result) == $spock_valueRecorder.record(1, 210)))
        this.getSpecificationContext().getMockController().leaveScope()
    }
}
```

From the code above it should be clear that
- _Stub()_  
exception throwing method has been replaced by _StubImpl()_ from [SpecInternals](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/lang/SpecInternals.java) class
- _valueProvider.provideValue() >> 21_  
stubbing was replaced with [MockController](https://github.com/spockframework/spock/blob/spock-1.0/spock-core/src/main/java/org/spockframework/mock/runtime/MockController.java)
 interaction
- _then:_  
block was transformed to condition verification.

&nbsp;

If you want to read more about AST transformations (among others) check the great [Groovy metaprogramming guide](http://groovy-lang.org/metaprogramming.html)

&nbsp;


