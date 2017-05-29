## Chaos-Testing Akka Applications with Multi JVM tests


### Distributed applications are hard

* To write
* To test
    * Deploying… Orchestrating… 
* What is a good test?
    * Asynchronicity


### Wouldn't it be nice if…

* We could run tests from our own machine?
* We would have a bit of control over the execution?


### How to make life a bit easier

* `sbt-multi-jvm`
* `akka-multi-node-testkit`




### `sbt-multi-jvm`

```scala
addSbtPlugin("com.typesafe.sbt" % "sbt-multi-jvm" % "0.3.11")
```

* Place files in `src/multi-jvm/..`
* Filename format: 

  `{test-name}MultiJvmNode{#no}`
* [sbt config](http://doc.akka.io/docs/akka/2.4/dev/multi-jvm-testing.html)


#### Run multiple nodes

```scala
package sample
 
object SampleMultiJvmNode1 extends App {
    println("Hello from node 1")
}
 
object SampleMultiJvmNode2 extends App {
    println("Hello from node 2")
}
 
object SampleMultiJvmNode3 extends App{
    println("Hello from node 3")
}
```

```
> multi-jvm:run sample.Sample
…
[info] * sample.Sample
[JVM-1] Hello from node 1
[JVM-2] Hello from node 2
[JVM-3] Hello from node 3
[success] Total time: …
```


#### Running tests

`multi-jvm:test-only sample.Spec`



#### Changing jvm arguments

``` SampleMultiJvmNode1.opts
-Dakka.remote.port=9991 -Xmx256m    
```



#### Lessons learned

* Every test is executed in its own jvm
* Useful to spin off **some** tests in parallel




### `akka-multi-node-testkit`

```scala
libraryDependencies += 
    "com.typesafe.akka" %% "akka-multi-node-testkit" % "2.5.2"
```

[Docs](http://doc.akka.io/docs/akka/2.4.18/dev/multi-node-testing.html)


```scala
class DistributedSystemSpec 
      extends MultiNodeSpec(MyMultiNodeConfig) {
    "An Akka Persited Cluster" must {

    "setup shared journal" in {
      // start the Persistence extension
      Persistence(system)
      runOn(controller) {
        system.actorOf(Props[SharedLeveldbStore], "store")
      }
      enterBarrier("started")
      …
      enterBarrier("configured")
    }

    "join cluster" in within(20.seconds) { … }
}
```
```scala
class MultiNodeSpecMultiJvmNode1 extends DistributedSystemSpec
class MultiNodeSpecMultiJvmNode2 extends DistributedSystemSpec
class MultiNodeSpecMultiJvmNode3 extends DistributedSystemSpec
class MultiNodeSpecMultiJvmNode4 extends DistributedSystemSpec
```



```
object MyMultiNodeConfig extends MultiNodeConfig {
  val controller: RoleName = role("controller")
  val first: RoleName = role("first")
  val second: RoleName = role("second")
  val third: RoleName = role("third")
  …
}
```



#### Functions

* `runOn(role: RoleName)`
* `enterBarrier(name: String)`
* `awaitAssert { … }`
* `testConductor.exit(first, 0).await`
* `throttle`, `blackhole`, [etc](http://doc.akka.io/api/akka/current/akka/remote/testconductor/TestConductorExt.html)…


#### Join
```scala

"join cluster" in within(20.seconds) { 
    join(first, first)
    join(second, first)
    join(third, first)
 }

def join(from: RoleName, to: RoleName): Unit = {
    runOn(from) {
        cluster join node(to).address
    }
    enterBarrier(from.name + "-joined")
}
```


#### Lessons learned
* Start your actor system!
    
    ```scala
class Spec with BeforeAndAfterAll {
    override def beforeAll(): Unit = multiNodeSpecBeforeAll()

    override def afterAll(): Unit = multiNodeSpecAfterAll()
}
    ```
* All specs run seperately




### Discussion

* Docker + blockage



### Conclusion

* Orchestration can be made easier
* Writing good tests is still hard

<!--
# Multi-JVM
    * Part SBT
    * Part Akka Toolkit
* Intro to sbt-multi-jvm
    * Add plugin
    * src/multi-jvm/...
    * SampleMultiJvmNode2
    * multi-jvm:run runs all nodes
    * Tests
        * Create node SpecMultiJvmNode1
        * Multi-jvm:test-only sample.Spec
            * IT tests usually take a long time so useful to just run one at a time
        * 
* Useful functions
* Akka
    * Adds controls for testing actors ("com.typesafe.akka" %% "akka-multi-node-testkit" % akkaVersion)
        * Start/stop api
        * Barriers
        * runOn - if statement
        * 
    * Custom
        * Join - one by one
def join(from: RoleName, to: RoleName): Unit = {
    runOn(from) {
      cluster join node(to).address
    }
    enterBarrier(from.name + "-joined")
  }
            * 
-->
