# Reasoning

Forward reasoning on Ecore classes using compute graphs. Focus on documentation, visualizations, and explainability.

## Reasoning model

* ``Rule`` contains ``Inputs`` and ``Join Predicates``. It has ``conclusionTypes`` many reference with ``EClass`` type. It extends Ncore ``NamedDocumentedElement`` or ``NamedDocumentedElementWithId``.
* ``RuleCategory`` (or domain) is a grouping construct for rules.
* ``Input`` defines a single input to a rule - type (``EClass``), zero or more predicates. Alse extends ``NamedDocumentedElement`` or ``NamedDocumentedElementWithId``. Input may specify nullability. Nullable input would mean that joins of multiple inputs would include an implicit ``null`` entry for this element (outer join).
* ``Predicate`` explains a condition which a single input must meet. 
* ``Join Predicate`` explains conditions involving more than one input. References condition inputs.
* ``Conclusion`` (bi-directionally) references a rule and its inputs. Conclusions can be chained - form a inference tree/graph
* Fact is an EObject - input to rules. It may be an wrapper around an EObject. Facts may be derived from rules - in this case they have associated conclusions. They can be also provided to the rule engine as "axioms". An axiom is a type of conclusion which cannot be negated/invalidated.

## Execution

The metamodel does not specify rule and predicate execution logic and there are multiple ways to "wire" the logic to the rules including:

* Subclassing
* Adapter - the default rule implementation should implement this logic - looking for an adapter to delegate and throwing UnsupportedOperationException if the adapter is not found.

Rules can be authored in multiple ways and loaded from multiple sources. Examples:

* Author:
    * Tree editor in Eclipse
    * YAML
    * MS Excel
    * Map Drawio diagram elements to rules and predicates
* Logic:
    * [Spring Expression Language](https://docs.spring.io/spring-framework/reference/core/expressions.html) (SpEL)
    * Java
    * Scripted languages supported by the Java Scripting framwork, e.g. JavaScript.

The primary focus would be on Java logic for rules and Java and SpEL for predicates. 

Model rules can be loaded from annotated Java methods using ``Reflector``. Rule categories may be created from declaring class annotations. Reflector supports ``@Factory`` annotations and as such it would be possible to build hierarchies of rule categories. ``AnnotatedElementRecord`` would need to be extended with ``List<Object> factoryPath`` field. 
Loading rules from annotated methods would accelerate rule development - less code to write.
When a rule is create the annotated method's ``AnnotatedElementRecord`` would be added to the rule using an adapter, so the rule can invoke the method. 
This means that rules developed this way must be loaded from annotated methods to be executable. Once they are created and possibly executed they may be stored in XMI or other format for reporting. 
Rule may keep track of where they were loaded from by extending ``Marked`` and using ``Markers``, or by using an explicitly specified attribute.

Once rules are loaded, they would be transofrmed into an inference graph. Graph classes may be Java classes, or may be Ecore classes.
The second approach would allow to use Ecore documentation generation to document the inference network/graph metamodel and models. 

The inference graph would have the following types of nodes:

* Type predicate nodes passing only objects which are instances of a specific type. This would allow to use Ecore type system in inference by specifying which type of conclusions a rule emits and which type of inputs it expects. Type predicate nodes would leverage the existing EClass nodes to pass objects along inheritance connections from superclasses to subclasses. For example, Rule A emits facts of type Human/Person and Rule B takes an input of type Woman. Rule A inference node would have a connection to the Human/Person class node, which in turn would have a sub-class connection to the Woman node. The Woman node would have a connection to the Rule B input of type Woman. Whan a instance of Human/Person is emitted, it will be passed to the Human/Person node, which will check input type an if it is Human/Person will go over its outgoing connections and pass the input to them. Similarly, the Woman node will check the input type and if it is a Woman, then it will pass it to the outgoing connections and to the Rule B input as such.
* Input predicate evaluating a condition for a single input.  
* Join predicates evaluating a condition which uses more than one input.
* Rules receiving inputs and producing conclusions. Produced conclusions are passed to the type predicate nodes to route them to interested rule nodes via predicate nodes.

(Predicate) nodes may implement ``equals()``. In this case several equal nodes with incoming conections from the same set of nodes can be merged into a single node.

The compute graph can be built using classes/interfaces from ``org.nasdanika.graph.processor.function`` package.

### Commutative AND's and OR's

The model may support logical preciates such as AND and OR including commutative AND and OR where the order of evaluation is not significant.
Commutative AND and OR would allow to potentially optimize the inference graph by rearranging individual predicates to merge/share predicates for multiple rules.  

### Concurrency

Concurrent execution of nodes can be implemented using:

* Parallel streams - this would work well for CPU-bound nodes where most of the time is spend in computing predictates and conclusions with little or no I/O.
* CompletableFutureEndpoint and Executor - this approach is more flexible, yet a bit more complex. It may be used if nodes are I/O bound by making external calls or reading/writing files. 

In the first case whether or not to parallelize can be defined at the graph level (default) and at a node level. E.g. a type node may pass inputs to outgoing nodes using a parallel stream.
In the second case it may be decided at the connection level (endpoint). As such, the two approached may be combined - CPU-bound nodes would be executed using the built-in Java fork-join thread pool, and I/O bound nodes would be executed by a different executor.

#### Locking

Nodes may need to use the underlying model (Knowledge Base).
There should be a Read/Write lock and nodes (predicates and rules) shall be able to declare locking level required:

* None - no locking is required, or locking is explicit. For example, a predicate makes an external call and then needs to navigate the model. In this case the node lock declaration would be "none" and once the external call is completed the node would explicitly acquire a read lock to navigate the model.
* Read - a read lock is implicitly acquired and released.
* Write - a write lock is implicitly acquired and released.

### Probabilistic reasoning

One intereting feature to implement is to provide support for fact and conclusion probabilities and computing conclusion probabilities from derivation trees and source facts probabilities. 
E.g. if there is 50% probability that Joe is Jim's colleague and 50% probability that Jane is Jim's colleague then there is 25% probability that Joe is Jane's colleague.
Support of probabilities would allow to build rule sets with AI rules. E.g. an image recognition "rule" taking an image fact and producing a conclusion "Cat, 80%".

## Reporting

* Rules documentation with help context referencing the metamodel documentation
* Inference graphs - can be built without rule execution
* Conclusion graphs - explaining how a particular conclusion was made

## Demo

Family ties model, shall be in the [tutorial files](http://www.hammurapi.biz/hammurapi-biz/ef/xmenu/hammurapi-group/products/hammurapi-rules/xmenu436.html@parent=36&uplink=no.html)

## Use Case

An organization has multiple information systems and wants to build a holistic view of the organization "entities" and provide insights/recommendations.
For example:

* GitLab as a source repository with hundreds or even thousands of repositories. 
* People hierarchy with hudreds or thousands of people.
* The organizaiton uses Java/Maven. 

In this case the organization may use https://github.com/Nasdanika-Models/gitlab to load groups, projects, users, members, contributors, branches, ``CODEOWNER`` files, and ``pom.xml`` files from GitLab into a model. ``pom.xml`` files would be loaded parsed and loaded to the model leveraging https://github.com/Nasdanika-Models/maven. 
After that the organization may load the people hierarchy into the model and establish cross-referencing with GitLab users, members, contributors, CODEOWNERS, as well as developer entries in ``pom.xml`` files.
``pom.xml`` file would also be used to establish repository dependencies as well as dependencies on external Maven artifacts.

All of this information may be reported alighend to the people hierarchy. E.g. Joe Doe manages an org which owns X GitLab repositories and then a break-down by Joe Doe's reports.
Just reporting alone would provide valuable insights, but providing recommendations regarding how to make things better would be even more valuable.
Some type types of recommendations which might be provided using the above model:

* Branching - too many branches, old branches, branch naming conventions.
* Dormant projects to archive.
* Dependency on old versions of internal components. E.g. Repo B declares a dependency on version X of a Maven component in Repo A. Repo A's version of the component is Y and X is, say, a year behind Y.
* Hotspots such as [superchickens](https://en.wikipedia.org/wiki/Super-chicken_model) - people who own too much code while many others own little or nothing. In this case ownership shall be balanced.
* Circular dependencies in ownership.
* Discripancies in code owners and Maven developers.
* Maven oranization is missing or is not aligned to the organization owning the Maven component.

This is where the reasoning model comes into play:

* A set of rules is created. Initially it may have no execution logic, just descriptions.
* Rule set documentation is generated and published so everybody in the organization gets familiar with the rules and may provide feedback.
* Rules are gradually implemented. Rule recommendations are published to a report generated from the model with roll-up of stats along the people hierarchy. Recommendations reference the rules, inputs, and conclusion graphs for multi-step recommendations and recommendations with complex conditions.

Some rules may produce intermediate results to be used by other rules. Such rules may be excluded from the documentation.

Please keep in mind that the above is a very simple use case. There is much more to it in real life such as:

* Rules rollout in phases - published, warning level, error level.
* Rules context - different configuration, turning rules on and off
* Ruleset inheritance - org-wide rules, team specific rules inheriting from them
* "Waivers" - ability to suppress findings for a specific period of time or permanently
* Loading of data from multiple systems - issue tracking, binary repository, runtime environments, ...