# [S]imple [J]ava [F]lows [![Maven Central](https://img.shields.io/maven-central/v/com.simplj.flows/sjf.svg?label=Maven%20Central)](https://search.maven.org/search?q=g:%22com.simplj.flows%22%20AND%20a:%22sjf%22) [![javadoc](https://javadoc.io/badge2/com.simplj.flows/sjf/javadoc.svg)](https://javadoc.io/doc/com.simplj.flows/sjf)

## Motivation
  From my experience in coding java, I have felt that,
  > we write more or less 60% of repetative code on any project. _(The number is purely based on my experience and can go a little high or down for others but overall I believe, we all will agree that we write repetative code)_

  > Another big problem that I faced when joining to an existing project, is to get hold of the different flows that the project has. To understand this either, we need to get a KT from another person or spend a good amount of time and navigate through various classes/methods in the project

  > Last but certainly not the least is debugging - when we deploy our code in a enviromment and some flow breaks. We have to rely on logs that are already present, if they are not sufficient putting further logs and redeploy and replay the flow to identify the issue. I know there is a way of remote-debugging but not every where it can be used (for example if the environment is running behind a firewall) and even in remote-debugging we have to replay the flow to identify the issue.

  I am sure most (if not all) of us has faced these problems some time or the other with an impact of medium to heavy, and this motivated me to develop a framework which would resolve these problems fully (or as much as possible). Let's take a quick look at how does it help solving the problems.
  * This framework enables one to construct a business flow (or a feature) simply by configuring individual steps (that are part of the enitre flow) in various way as needed. Thus the name **Simple Java Flows** or **SJF**. This **increases readability** of the flow meaning a new comer can quickly get a overall picture of what the flow is doing just by looking at it's definition
  * When a flow is executed, it returns the execution's context comprising of _a) what was the input for the step, b) how much time the step took and c) whether the step produced an output or threw exception_. This gives an end-to-end knowlwedge of how the flow executed which **eases debugging**.
  * The framework comes with capabilities of performing common operations by itself, enabling developer to focus only on writing the business logic part and thus **gaining individual and team productivity**. The list of **common operations** that SJF offers currently are not exhaustive, because covering all operations which may sound redundant is not a quick work but they are in progress and will be published as a plug-and-play fashion gradually. Nonetheless, the framework already covers quite a few basic operations which will be discussed in the later section, to name a few: retryable step, bookmarking within a flow (helps in writing [idempotent operations](https://en.wikipedia.org/wiki/Idempotence)), triggering async step, parallel steps, pausing for a condidtion to satisfy etc.

Sounds exciting? Let's jump right in to get familiar with the framework then 😉

## Maven Dependency
```
<dependency>
  <groupId>com.simplj.flows</groupId>
  <artifactId>sjf</artifactId>
  <version>0.1</version>
</dependency>
```

Table of contents
=================
<!--ts-->
  * [Key Concepts](#key-concepts)
    - [Step\<I, O>](#stepi-o)
    - [Flow\<I, O>](#flowi-o)
    - [ExecutionResult\<O>](#executionresulto)
    - [ExecutionEngine](#executionengine)
    - [StepFactory](#stepfactory)
  * [Various steps and their usages](#various-steps-and-their-usages)
    - [Executing A Step Conditionally](#executing-a-step-conditionally)
    - [Executing Either of the Steps Conditionally](#executing-either-of-the-steps-conditionally)
    - [AsyncStep](#asyncstep)
    - [BookmarkStep](#bookmarkstep)
    - [ConditionalStep](#conditionalstep)
    - [ComparableStep](#comparablestep)
  * [Bonus: Injecting other Steps](#bonus-injecting-other-steps)
<!--te-->


## Key Concepts
Let's kick-off by understanding the key concepts of SJF.

* #### Exectuable\<I, O>
  > SJF uses Executable from [jlx](https://github.com/simplj/jlx) library. Please refer to [jlx](https://github.com/simplj/jlx) for details about Executable and other lambda funcionalities.

* #### Step\<I, O>
  > When we execute an `Executable<I, O>` with an `I`, we get just the result i.e. `O`. We don't have any other way to know how much time it took to execute. (_Yes, we can use startTs and endTs before and after the execution to calculate the time, but as I said earlier, that would be a repetitive code_) Here comes `Step<I, O>`. A Step is nothing but an enhanced version of Executable. We can lift an executable to a step using the `Step.lift(Executable<I, O> x)` method. A step can also be named by using the overloaded `lift` method and passing a name for the step. When we execute a step, we get `ExecutionResult<O>` and from this we can know the result (of type `O` if the execution was successful), error (if the execution failed), and the duration of the execution.

* #### Flow\<I, O>
  > Now that we know about step, would a single step be ever useful? Any complex operation is performed by executing multiple individual steps serially or parallely or mix of both according to the need. `Flow<I, O>` represents a collection of steps i.e. a `Flow<I, O>` is constructed by joining multiple steps that starts by taking `I` as input and ends producing `O` as output. A step can be converted to a flow using the `toFlow()` method of step. A step can be added to a flow using the `<R> then(Step<O, R> next)` method of flow. A step can be added to a flow in various ways which I will describe later in this article. Like step, flow also returns `ExecutionResult<O>` when executed.

* #### ExecutionResult\<O>
  > `ExecutionResult<O>` is produced either by executing a step or a flow. This contains a boolean value informing if the execution was successful, the result (of type `O` if the execution succeeded), error (if the execution failed), and the duration of the execution.

* #### ExecutionEngine
  > We need an engine to execute a step or a flow. This also distinguishes the construction and execution of a flow/step. We can also pass pre-hooks and/or post-hooks in a execution engine which if passed will be executed before executing a step and after executing a step accordingly. Let's see some example code to understand what we saw so far.
  > ```java
  > //Let's construct a sample flow which
  > // calculates the length of a string -> converts to a binary string -> calculates the length -> returns if the length is even
  > Step<String, Integer> lengthStep = Step.lift("length", String::length); // defined a step separately as this is used twice in the following flow.
  > Flow<String, Boolean> sampleFlow = lengthStep.toFlow()
  >         .then("binary", Integer::toBinaryString)
  >         .then(lengthStep)
  >         .then("isEven", l -> l % 2 == 0);
  > 
  > //Execute in default execution engine.
  > ExecutionResult<Boolean> result = ExecutionEngine.defaultEngine().execute(sampleFlow, "Simple Java Flows");
  > System.out.println(result);
  > result.printExecution();
  > 
  > /** Output:
  >  * Success[false :: java.lang.Boolean]
  >  * Flow Execution:
  >  * - {main: Simple Java Flows -> Success[false :: java.lang.Boolean] (2ms)}
  >  *   - {length: Simple Java Flows -> 17 (1ms)}
  >  *   - {binary: 17 -> 10001 (0ms)}
  >  *   - {length: 10001 -> 5 (0ms)}
  >  *   - {isEven: 5 -> false (0ms)}
  >  */
  > 
  > //Construct custom execution engine with pre-hooks and post-hooks.
  > ExecutionEngine engine = ExecutionEngine.custom()
  >       .addPreHooks((s, i) -> System.out.println(s + ".preHook: " + i))
  >       .addPostHooks(s -> System.out.println(s.name() + ".postHook: " + s))
  >       .getEngine();
  > 
  > //Execute in the custom execuetion engine
  > result = engine.execute(sampleFlow, "Simple Java Flows");
  > System.out.println("Result: " + result);
  > result.printExecution();
  > 
  > /** Output:
  >  * length.preHook: Simple Java Flows
  >  * length.postHook: {length: Simple Java Flows -> 17 (0ms)}
  >  * binary.preHook: 17
  >  * binary.postHook: {binary: 17 -> 10001 (0ms)}
  >  * length.preHook: 10001
  >  * length.postHook: {length: 10001 -> 5 (0ms)}
  >  * isEven.preHook: 5
  >  * isEven.postHook: {isEven: 5 -> false (0ms)}
  >  * Result: Success[false :: java.lang.Boolean]
  >  * Flow Execution:
  >  * - {main: Simple Java Flows -> Success[false :: java.lang.Boolean] (2ms)}
  >  *   - {length: Simple Java Flows -> 17 (1ms)}
  >  *   - {binary: 17 -> 10001 (0ms)}
  >  *   - {length: 10001 -> 5 (0ms)}
  >  *   - {isEven: 5 -> false (0ms)}
  >  */
  > ```

💡 _It is advised to always provide a name to a step. This will increase readability of flow execution log (as shown above in the output section) and help in debugging._

* #### StepFactory
  > This `factory` class helps to create the various types of step that SJF offers. These steps will be discussed in detail in the later section of this article.



## Various steps and their usages
  The examples that are used to explain the usages of various steps might look vague and I promise to come up with a real-life-kind flow at the end of this article. Till then, please put up with the fuzzy examples. Though the examples are a little vague, but I believe, the examples did explain the intent/usage of the corresponding steps clearly


### Executing A Step Conditionally
  It is possible to execute a step within a flow only if a condition satisfies. The `branch` method of `Flow` class takes a condition and a step and joins the step to the flow such that the step gets executed only if the condition satisfies. This method is overloaded and can take either an `Executable` or a `Flow` to execute conditionally.

  In the following flow, `s.replaceAll` will be executed only if the string `s` from the previous step contains any space in it.
  ```java
  Step.lift("randStrGenerator", (Integer n) -> generateStr(n)).toFlow()
    .branch(s -> s.contains(" "), s -> s.replaceAll(" ", ""))
    .then("length", String::length)
  ```


### Executing Either of the Steps Conditionally
  It is possible to execute either of two steps based on a condition. The `either` method of `Flow` class takes a condition and two steps and joins them to the flow such that if the condition satisifes, first step is executed otherwise second step. This method is overloaded and can take either an `Executable` or a `Flow` to execute conditionally.

  In the following flow, the length value from previous step will be divided by 2 if the it is even otherwise multiplied by 2.
  ```java
  Step.lift("length", String::length).toFlow()
    .branch(x -> x % 2 == 0, x -> x / 2, x -> x * 2)
  ```


### AsyncStep
  This step is used to execute an operation asynchronously. If an instance of `ExecutorService` is passed in, then the execution will happen on a thread from the same `ExecutorService` otherwise a new thread will be spawned to execute the step. There can be 2 ways of performing an asynchronous operation:
  * Fire and Forget Asynchronous Operation: the operation will be executed in a different thread (or in executor if provided) and no way to get the result (if any). Output type of this step will be the same as the input.
  ```java
  public void logEvent(Event event) {
    //event logging code goes here...
  }

  //To use logEvent asynchronously, StepFactory
  StepFactory.asyncStep(x -> logEvent(new EventStep(/*Event Parameters*/)))

  //This can also be used directly within a flow using flow's `async` api.
  Step.lift("first operation", firstOp).toFlow()
    .then("second operation", secondOp)
    .async(x -> logEvent(new Event(/*Event Parameters*/)))
  ```

  * Completable Asynchronous Operation: the operation will be executed in a different thread (or in executor if provided) and the result can be used later in the flow. This step returns a `Completable<O>` object which in turns return a `CompletionResult<O>` instance as result. `CompletionResult<O>` contains information like if the operation timedOut, if the operation was successful, the result if successful and the error if failed. To use the result later, it must be retained by a name and obtained by the same name when needed.
  ```java
  public PersistResult persistLog(LogDetail detail) {
    //actual persisting code goes here...
  }

  //To use logEvent asynchronously, StepFactory
  StepFactory.completableAsyncStep(x -> persistLog(new LogDetail(/*Log Parameters*/))).retainCompletable("logPersistResult").toFlow()
    .then("operation1", op1)
    .then("operation2", op2)
    .obtainCompletable("logPersistResult", PersistResult.class)
    .then(completable -> completable.waitForCompletion(2, TimeUnit.SECONDS))
  ```


### BookmarkStep
  This step is used to bookmark a (or multiple) step(s) in a flow. When bookmark step is executed in a flow it returns an indempotentId (if not already provided) in the resultant `ExecutionResult`. This idempotentId is used to recognize identical flows and resume the flow from the bookmarked point. Hence, if a flow with `BookmarkStep` fails, and needs to be re-executed, then the idempotentId from the result must be passed in to the `ExecutionEngine` along with the input. Please note, idempotentId will only be generated by the framework if none is passed, so, if using your own idempotentId is preferred, then pass the same while executing flows all the time. Currently, bookmarked values are stored in-memory, other options of persisting into cache or db is work-in-progress and can be used as a plug-and-play fashion when released.
  ```java
  //Utility methods
  private static final AtomicInteger ATOMIC_INTEGER = new AtomicInteger(0);
  private String erroneousToBinaryString(int num) {
      if (ATOMIC_INTEGER.incrementAndGet() == 1) {
          throw new RuntimeException("Step not yet ready to operate!");
      }
      return Integer.toBinaryString(num);
  }
  private int countOnes(String s) {
      int res = 0;
      char[] chars = s.toCharArray();
      for (char c : chars) {
          if (c == '1') {
              res++;
          }
      }
      return res;
  }

  //Defining the flow
  Flow<String, Integer> flow = Step.lift("length", String::length).toFlow()
      .either(x -> x % 2 == 0, x -> x / 2, x -> x * 2)
      .bookmark("modifiedLength")
      .then("binary", m::erroneousToBinaryString)
      .then("ones", m::countOnes);

  //Executing the flow
  ExecutionEngine engine = ExecutionEngine.defaultEngine();
  String idempotentId = UUID.randomUUID().toString(); //Generating idempotentId.
  ExecutionResult<Integer> res = engine.execute(flow, "simple java flows", idempotentId);

  System.out.println("Result: " + res);
  res.printExecution();
  if (!res.isSuccess()) {
      res = engine.execute(flow, "simple java flows", idempotentId);
      System.out.println("Result: " + res);
      res.printExecution();
  }

  /**
   * Result: Failure[Error: Step not yet ready to operate!]
   * Flow Execution:
   * - {main: simple java flows -> Failure[Error: Step not yet ready to operate!] (4ms)}
   *   - {length: simple java flows -> 17 (1ms)}
   *   - {either-condition: 17 -> false (0ms)}
   *   - {anonymous: 17 -> 34 (0ms)}
   *   - {modifiedLength: Bookmarked value!}
   *   - {binary: 34 -> Error: Step not yet ready to operate!}
   *
   * Result: Success[2 :: java.lang.Integer]
   * Flow Execution:
   *  - {main: simple java flows -> Success[2 :: java.lang.Integer] (1ms)}
   *    - {length: simple java flows -> 17 (0ms)}
   *    - {either-condition: 17 -> false (0ms)}
   *    - {anonymous: 17 -> 34 (0ms)}
   *    - {modifiedLength: Bookmarked value!}
   *    - {modifiedLength: Fetched bookmarked value!}
   *    - {binary: 34 -> 100010 (0ms)}
   *    - {ones: 100010 -> 2 (0ms)}
  ```
  In the above example, the flow is executed with a pre-generated idempotentId. If the step fails at first attempt (which will eventually fail because of the `erroneousToBinaryString` step logic), it retries second time with the same idempotentId. The output shows that in the second attempt, it fetch the bookmarked value and resumed from there.

💡 _Same idempotentId must be passed to `ExecutionEngine` when re-executing same flow, otherwise framework won't recognize the flow as identical the flow will be executed from the beginning again i.e. bookmarked step won't work as expected if different idempotentId is passed for same flow._


### ConditionalStep
  This step is used choose and execute a step among many based on condition. This resembles to the `if...else` ladder. This starts evaluating the conditions one by one until one is satiesfied or reached at the end of the step. If a condition is satisfied then the corresponding step is executed. If no condition is satisfied then the default step mentioned in `otherwise` is executed. If default step is not provided using `otherwise` and no condition is satisfied then `PatternExhaustedException` is thrown.
  ```java
  enum WeekDays {
    Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
  }

  StepFactory.newConditionalStep("idxToWeekDay")
    .when((Integer i) -> i == 0).then(x -> WeekDays.Sunday)
    .when(i -> i == 1).then(x -> WeekDays.Monday)
    .when(i -> i == 2).then(x -> WeekDays.Tuesday)
    .when(i -> i == 3).then(x -> WeekDays.Wednesday)
    .when(i -> i == 4).then(x -> WeekDays.Thursday)
    .when(i -> i == 5).then(x -> WeekDays.Friday)
    .otherwise(x -> WeekDays.Saturday)
  ```


### ComparableStep
  This step is used choose and execute a step among many based on a value. This resembles to the `switch...case` block. Unlike `ConditionalStep`, this compares the target value in constant time and executes the corresponding step. If no values matched then the default step mentioned in `otherwise` is executed. If default step is not provided using `otherwise` and no values matched then `PatternExhaustedException` is thrown.
  ```java
  enum WeekDays {
    Sunday, Monday, Tuesday, Wednesday, Thursday, Friday, Saturday
  }

  StepFactory.newComparableStep("idxToWeekDay")
    .match(0).then(x -> WeekDays.Sunday)
    .match(1).then(x -> WeekDays.Monday)
    .match(2).then(x -> WeekDays.Tuesday)
    .match(3).then(x -> WeekDays.Wednesday)
    .match(4).then(x -> WeekDays.Thursday)
    .match(5).then(x -> WeekDays.Friday)
    .otherwise(x -> WeekDays.Saturday)
  ```

TO BE UPDATED WITH MORE STEPS

## Bonus: Injecting Steps
  > As SJF uses our home grown dependency injection framework [SDF](https://github.com/simplj/sdf), it is possible to inject and run steps/sub-flows dynamically in a flow. This can help running different sub-flows according to the need.
  > For example, we may want to execute a pariticular step for dev environment and another step for all other environments. In this case, we can configure the steps as dependency using `@DependencyProvider` and assign id to the individual steps. Then, we can resolve the step by the corresponding id inside another flow to inject and run different steps in different environment.
