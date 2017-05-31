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


#### Running tests and specifying JVM arguments

* `multi-jvm:test-only sample.Spec`
* `SampleMultiJvmNode1.opts`
    ```
-Dakka.remote.port=9991 -Xmx256m    
    ```


#### Lessons learned

* Every test is executed in its own JVM
* Useful to spin off **some** tests in parallel
* Useful for testing static/class-loading time code




### `akka-multi-node-testkit`

```scala
libraryDependencies += 
    "com.typesafe.akka" %% "akka-multi-node-testkit" % "2.5.2"
```

[Docs](http://doc.akka.io/docs/akka/2.4.18/dev/multi-node-testing.html)


```scala
class DistributedSystemSpec 
      extends MultiNodeSpec(MyMultiNodeConfig) {
    "An Akka Persisted Cluster" must {

    "setup shared journal" in {
      // start the Persistence extension
    }

    "join cluster" in within(20.seconds) { … }

    "run test" { … }

    "kill a few nodes, maybe at random" { … }

    "run test again" { … }
}
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


#### Chaos Functions

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
* All specs run seperately, they don't need to be the same spec!
* Writing these tests is actually fun




### Discussion

* Docker + blockade
* Run Gatling in one node


### Conclusion

* Orchestration can be made easier
* Writing good tests is still hard
