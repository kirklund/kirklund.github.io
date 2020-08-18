---
layout: post
title: Using IgnoredException in Distributed Tests
date: 2020-08-14 00:00:00 -0000
categories: [Apache Geode, Testing]
tags: [Distributed Test, IgnoredException]
---
Goal: Use `IgnoredException` to prevent failures caused by an expected suspicious string or throwable logged by a background thread in any Distributed Test JVM.

After each test, `DistributedRule` greps the output of every Distributed Test JVM used by the test for suspicious strings or throwables logged at `error` or `fatal` level. If such a log statement is found, it will fail with something like this:
```
Found suspect string in log4j at line 574

[error 2020/08/14 09:11:07.140 PDT <main> tid=1] java.lang.IllegalStateException: bad thing happened

	at org.junit.Assert.fail(Assert.java:88)
	at org.apache.geode.test.dunit.internal.DUnitLauncher.closeAndCheckForSuspects(DUnitLauncher.java:395)
	at org.apache.geode.test.dunit.rules.DistributedRule$TearDown.doTearDown(DistributedRule.java:225)
	at org.apache.geode.test.dunit.rules.DistributedRule.after(DistributedRule.java:149)
	at org.apache.geode.test.dunit.rules.AbstractDistributedRule.afterDistributedTest(AbstractDistributedRule.java:81)
	at org.apache.geode.test.dunit.rules.AbstractDistributedRule$1.evaluate(AbstractDistributedRule.java:61)
	at org.junit.rules.RunRules.evaluate(RunRules.java:20)
	at org.junit.runners.ParentRunner.runLeaf(ParentRunner.java:325)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:78)
	at org.junit.runners.BlockJUnit4ClassRunner.runChild(BlockJUnit4ClassRunner.java:57)
	at org.junit.runners.ParentRunner$3.run(ParentRunner.java:290)
	at org.junit.runners.ParentRunner$1.schedule(ParentRunner.java:71)
	at org.junit.runners.ParentRunner.runChildren(ParentRunner.java:288)
	at org.junit.runners.ParentRunner.access$000(ParentRunner.java:58)
	at org.junit.runners.ParentRunner$2.evaluate(ParentRunner.java:268)
	at org.junit.runners.ParentRunner.run(ParentRunner.java:363)
	at org.junit.runner.JUnitCore.run(JUnitCore.java:137)
```
The above failure can be reproduced with the following Distributed Test. The log statement would normally be logged by a background thread in Geode product code, but we'll log it directly in the test to keep it simple and obvious:
```java
import org.apache.logging.log4j.Logger;
import org.junit.Rule;
import org.junit.Test;

import org.apache.geode.logging.internal.log4j.api.LogService;
import org.apache.geode.test.dunit.rules.DistributedRule;

public class SuspiciousDistributedTest {

  private static final Logger logger = LogService.getLogger();

  @Rule
  public DistributedRule distributedRule = new DistributedRule();

  @Test
  public void failsWithSuspectString() {
    logger.error(new IllegalStateException("bad thing happened"));
  }
}
```
The [IgnoredException API in geode-dunit](https://github.com/apache/geode/blob/develop/geode-dunit/src/main/java/org/apache/geode/test/dunit/IgnoredException.java) may be used if the cause of the failure is expected and needs to be suppressed. This API allows the test to pass by ignoring output containing the string or throwable. The specified string or throwable will be suppressed for every thread in every Distributed Test JVM.

One option is to use `addIgnoredException` in `setUp` or within a specific test:
```java
addIgnoredException(IllegalStateException.class);
```
Another option is to surround a specific block of code by using the try-with-resources syntax:
```java
try (IgnoredException ie = addIgnoredException(IllegalStateException.class)) {
  // code that might throw IllegalStateException
}
```
You can also surround code with multiple ignored exceptions using try-with-resources:
```java
try (IgnoredException ie1 = addIgnoredException(IllegalStateException.class);
     IgnoredException ie2 = addIgnoredException(AnotherException.class)) {
  // code that might throw IllegalStateException or AnotherException
}
```
You can even use a string or regex pattern:
```java
addIgnoredException("java.lang.IllegalStateException");
addIgnoredException("IllegalStateException");
addIgnoredException(".*Error.*");
addIgnoredException("bad thing happened");
```
The `DistributedRule` automatically removes all `IgnoredException`s during `tearDown` after grepping for suspect strings.

For this example, I've decided to use the try-with-resources syntax:
```java
import static org.apache.geode.test.dunit.IgnoredException.addIgnoredException;

import org.apache.logging.log4j.Logger;
import org.junit.Rule;
import org.junit.Test;

import org.apache.geode.logging.internal.log4j.api.LogService;
import org.apache.geode.test.dunit.IgnoredException;
import org.apache.geode.test.dunit.rules.DistributedRule;

public class SuspiciousDistributedTest {

  private static final Logger logger = LogService.getLogger();

  @Rule
  public DistributedRule distributedRule = new DistributedRule();

  @Test
  public void failsWithSuspectString() {
    try (IgnoredException ie = addIgnoredException(IllegalStateException.class)) {
      logger.error(new IllegalStateException("bad thing happened"));
    }
  }
}
```
`IgnoredException` should only be used in Distributed Tests.

Use `AssertJ`'s `catchThrowable` instead of `IgnoredException` if you need to catch an expected throwable for assertions in any JUnit tests including Distributed Tests.
