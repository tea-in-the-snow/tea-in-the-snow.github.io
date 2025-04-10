---
title: "JQF：从0到1的第一次实践"
description: 
date: 2025-03-21T14:01:48+08:00
categories:
    - Program Testing
tags:
    - JQF
hidden: false
comments: true
draft: false
---

[JQF](https://github.com/rohanpadhye/JQF) 是一个基于反馈导向的模糊测试平台，专为 Java 设计（可以理解为 JVM 字节码版的 AFL/LibFuzzer）。JQF 采用了基于属性测试的抽象概念，这使得编写模糊测试驱动程序就像编写参数化的 JUnit 测试方法一样简单。JQF 构建在 junit-quickcheck 之上，能够以覆盖引导的模糊测试算法（如 Zest）的强大功能来运行 junit-quickcheck 风格的参数化单元测试。

[Zest](http://arxiv.org/abs/1812.00078) 是一种算法，它将覆盖引导的模糊测试偏向于生成语义上有效的输入，即满足结构和语义属性同时最大化代码覆盖率的输入。Zest 的目标是发现传统模糊测试工具无法找到的深层语义错误，这些工具通常只侧重于测试错误处理逻辑。默认情况下，JQF 通过简单的命令 mvn jqf:fuzz 来运行 Zest。

下面是对利用 JQF 和 Zest 进行模糊测试的第一次实践，也是了解 JQF 框架的一个开始。(参考：[tutorial1](https://github.com/rohanpadhye/jqf/wiki/Fuzzing-with-Zest), [tutorial2](https://github.com/rohanpadhye/jqf/wiki/Fuzzing-a-Compiler))

## 运行环境

- Ubuntu 22.04 LTS
- openjdk 11.0.26 2025-01-21
- Apache Maven 3.6.3

## 利用 JQF 对简单程序进行测试

首先在 test 目录下 clone JQF 的源码并进行编译构建:

```shell
git clone https://github.com/rohanpadhye/jqf
cd jqf
mvn package
```

接着在 test 目录下新建一个 tutorial 目录，在 tutorial 目录下新建待测的 java 文件 `CalendarLogic.java`:

```java
import java.util.GregorianCalendar;
import static java.util.GregorianCalendar.*;

/* Application logic */
public class CalendarLogic {
    // Returns true iff cal is in a leap year
    public static boolean isLeapYear(GregorianCalendar cal) {
        int year = cal.get(YEAR);
        if (year % 4 == 0) {
            if (year % 100 == 0) {
                return false;
            }
            return true;
        }
        return false;
    }

    // Returns either of -1, 0, 1 depending on whether c1 is <, =, > than c2
    public static int compare(GregorianCalendar c1, GregorianCalendar c2) {
        int cmp;
        cmp = Integer.compare(c1.get(YEAR), c2.get(YEAR));
        if (cmp == 0) {
            cmp = Integer.compare(c1.get(MONTH), c2.get(MONTH));
            if (cmp == 0) {
                cmp = Integer.compare(c1.get(DAY_OF_MONTH), c2.get(DAY_OF_MONTH));
                if (cmp == 0) {
                    cmp = Integer.compare(c1.get(HOUR), c2.get(HOUR));
                    if (cmp == 0) {
                        cmp = Integer.compare(c1.get(MINUTE), c2.get(MINUTE));
                        if (cmp == 0) {
                            cmp = Integer.compare(c1.get(SECOND), c2.get(SECOND));
                            if (cmp == 0) {
                                cmp = Integer.compare(c1.get(MILLISECOND), c2.get(MILLISECOND));
                            }
                        }
                    }
                }
            }
        }
        return cmp;
    }
}
```

编写 test driver 'CalendarTest.java':

```java
import java.util.*;
import static java.util.GregorianCalendar.*;
import static org.junit.Assert.*;
import static org.junit.Assume.*;

import org.junit.runner.RunWith;
import com.pholser.junit.quickcheck.*;
import com.pholser.junit.quickcheck.generator.*;
import edu.berkeley.cs.jqf.fuzz.*;

@RunWith(JQF.class)
public class CalendarTest {

    @Fuzz
    public void testLeapYear(@From(CalendarGenerator.class) GregorianCalendar cal) {
        // Assume that the date is Feb 29
        assumeTrue(cal.get(MONTH) == FEBRUARY);
        assumeTrue(cal.get(DAY_OF_MONTH) == 29);

        // Under this assumption, validate leap year rules
        assertTrue(cal.get(YEAR) + " should be a leap year", CalendarLogic.isLeapYear(cal));
    }

    @Fuzz
    public void testCompare(@Size(max=100) List<@From(CalendarGenerator.class) GregorianCalendar> cals) {
        // Sort list of calendar objects using our custom comparator function
        Collections.sort(cals, CalendarLogic::compare);

        // If they have an ordering, then the sort should succeed
        for (int i = 1; i < cals.size(); i++) {
            Calendar c1 = cals.get(i-1);
            Calendar c2 = cals.get(i);
            assumeFalse(c1.equals(c2)); // Assume that we have distinct dates
            assertTrue(c1 + " should be before " + c2, c1.before(c2));  // Then c1 < c2
        }
    }
}
```

编写 input generator 'CalendarGenerator.java':

```java
import java.util.GregorianCalendar;
import java.util.TimeZone;

import com.pholser.junit.quickcheck.generator.GenerationStatus;
import com.pholser.junit.quickcheck.generator.Generator;
import com.pholser.junit.quickcheck.random.SourceOfRandomness;

import static java.util.GregorianCalendar.*;

public class CalendarGenerator extends Generator<GregorianCalendar> {

    public CalendarGenerator() {
        super(GregorianCalendar.class); // Register the type of objects that we can create
    }

    // This method is invoked to generate a single test case
    @Override
    public GregorianCalendar generate(SourceOfRandomness random, GenerationStatus __ignore__) {
        // Initialize a calendar object
        GregorianCalendar cal = new GregorianCalendar();
        cal.setLenient(true); // This allows invalid dates to silently wrap (e.g. Apr 31 ==> May 1).

        // Randomly pick a day, month, and year
        cal.set(DAY_OF_MONTH, random.nextInt(31) + 1); // a number between 1 and 31 inclusive
        cal.set(MONTH, random.nextInt(12) + 1); // a number between 1 and 12 inclusive
        cal.set(YEAR, random.nextInt(cal.getMinimum(YEAR), cal.getMaximum(YEAR)));

        // Optionally also pick a time
        if (random.nextBoolean()) {
            cal.set(HOUR, random.nextInt(24));
            cal.set(MINUTE, random.nextInt(60));
            cal.set(SECOND, random.nextInt(60));
        }

        // Let's set a timezone
        // First, get supported timezone IDs (e.g. "America/Los_Angeles")
        String[] allTzIds = TimeZone.getAvailableIDs();

        // Next, choose one randomly from the array
        String tzId = random.choose(allTzIds);
        TimeZone tz = TimeZone.getTimeZone(tzId);

        // Assign it to the calendar
        cal.setTimeZone(tz);

	// Return the randomly generated calendar object
        return cal;
    }
}
```

编译 `Calendar.java`, `CalendarTest.java`, `CalendarGenerator.java`:

```shell
javac -cp .:$(../jqf/scripts/classpath.sh) CalendarLogic.java CalendarGenerator.java CalendarTest.java
```

利用 Zest 进行模糊测试:

```shell
../jqf/bin/jqf-zest -c .:$(../jqf/scripts/classpath.sh) CalendarTest testLeapYear
```

输出如下：

```terminal
Semantic Fuzzing with Zest
--------------------------

Test name:            CalendarTest#testLeapYear
Instrumentation:      Janala
Results directory:    /root/test/tutorial/fuzz-results
Elapsed time:         2h 54m 26s (no time limit)
Number of executions: 29,648,562 (no trial limit)
Valid inputs:         2,490,117 (8.40%)
Cycles completed:     11404
Unique failures:      1
Queue size:           5 (5 favored last cycle)
Current parent input: 0 (favored) {301/360 mutations}
Execution speed:      2,713/sec now | 2,832/sec overall
Total coverage:       11 branches (0.02% of map)
Valid coverage:       9 branches (0.01% of map)
```

## 使用 Zest 对 Google Closure Compiler 进行测试

新建 `pom.xml` 和 `CompilerTest.java` 文件，目录结构如下：

```terminal
.
|-- pom.xml
|-- src
|   `-- test
|       `-- java
|           `-- examples
|               `-- CompilerTest.java
`-- target
    |-- classes
    `-- test-classes
        `-- examples
            `-- CompilerTest.class
```

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>examples</groupId>
    <artifactId>zest-tutorial</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Google Closure: we want to test this library -->
        <dependency>
            <groupId>com.google.javascript</groupId>
            <artifactId>closure-compiler</artifactId>
            <version>v20180204</version>
            <scope>test</scope>
        </dependency>

        <!-- JQF: test dependency for @Fuzz annotation -->
        <dependency>
            <groupId>edu.berkeley.cs.jqf</groupId>
            <artifactId>jqf-fuzz</artifactId>
            <!-- confirm the latest version at: https://mvnrepository.com/artifact/edu.berkeley.cs.jqf -->
            <version>1.8</version> 
            <scope>test</scope>
        </dependency>

        <!-- JUnit-QuickCheck: API to write generators -->
        <dependency>
            <groupId>com.pholser</groupId>
            <artifactId>junit-quickcheck-generators</artifactId>
            <version>1.0</version>
            <scope>test</scope>
        </dependency>

    </dependencies>

    <build>
        <plugins>
            <!-- The JQF plugin, for invoking the command `mvn jqf:fuzz` -->
            <plugin>
                <groupId>edu.berkeley.cs.jqf</groupId>
                <artifactId>jqf-maven-plugin</artifactId>
                <!-- confirm the latest version at: https://mvnrepository.com/artifact/edu.berkeley.cs.jqf -->
                <version>1.8</version>
            </plugin>
        </plugins>
    </build>
</project>
```

```java
package examples;

import java.io.ByteArrayOutputStream;
import java.io.PrintStream;

import com.google.javascript.jscomp.CompilationLevel;
import com.google.javascript.jscomp.Compiler;
import com.google.javascript.jscomp.CompilerOptions;
import com.google.javascript.jscomp.Result;
import com.google.javascript.jscomp.SourceFile;
import com.pholser.junit.quickcheck.From;
import edu.berkeley.cs.jqf.fuzz.Fuzz;
import edu.berkeley.cs.jqf.fuzz.JQF;
import org.junit.Before;
import org.junit.runner.RunWith;

import static org.junit.Assume.*;

@RunWith(JQF.class)
public class CompilerTest {

    static {
        // Disable all logging by Closure passes, to speed up fuzzing
        java.util.logging.LogManager.getLogManager().reset();
    }

    // Compiler, options, and predefined JS environment
    private Compiler compiler = new Compiler(new PrintStream(new ByteArrayOutputStream(), false));
    private CompilerOptions options = new CompilerOptions();
    private SourceFile externs = SourceFile.fromCode("externs", "");

    @Before // Runs before tests are executed
    public void initCompiler() {
        // Don't use threads
        compiler.disableThreads();
        // Don't print things
        options.setPrintConfig(false);
        // Enable all safe optimizations
        CompilationLevel.SIMPLE_OPTIMIZATIONS.setOptionsForCompilationLevel(options);
    }

    /** Compiles an input and returns its result */
    private Result compile(SourceFile input) {
        Result result = compiler.compile(externs, input, options);
        assumeTrue(result.success); // Semantic validity check
        return result;
    }

    /** Entry point for fuzzing with default (arbitrary) string generator */
    @Fuzz
    public void testWithString(String code) {
        SourceFile input = SourceFile.fromCode("input", code);
        compile(input); // No assertions; we are looking for unexpected exceptions
    }
}
```

编译：

```terminal
mvn test-compile
```

进行5分钟的模糊测试：

```terminal
mvn jqf:fuzz -Dclass=examples.CompilerTest -Dmethod=testWithString -Dtime=5m
```

输出如下：

```terminal
Semantic Fuzzing with Zest
--------------------------

Test name:            examples.CompilerTest#testWithString
Results directory:    /root/test/fuzz-closure/target/fuzz-results/examples.CompilerTest/testWithString
Elapsed time:         5m 0s (max 5m 0s)
Number of executions: 73,030 (no trial limit)
Valid inputs:         12,201 (16.71%)
Cycles completed:     0
Unique failures:      1
Queue size:           809 (0 favored last cycle)
Current parent input: 343 (favored) {233/240 mutations}
Execution speed:      230/sec now | 243/sec overall
Total coverage:       8,321 branches (12.70% of map)
Valid coverage:       7,215 branches (11.01% of map)
```

让我们创建一个新类，该类继承 Generator<String>，但在调用其 generate() 方法时仅生成语法上有效的 JavaScript 程序。这是一个复杂的生成器，它模拟 JavaScript 的语法，并递归地生成程序。我们将此文件保存为 src/test/java/examples/JavaScriptCodeGenerator.java

```java
package examples;

import java.util.*;
import java.util.function.*;

import com.pholser.junit.quickcheck.generator.GenerationStatus;
import com.pholser.junit.quickcheck.generator.Generator;
import com.pholser.junit.quickcheck.random.SourceOfRandomness;

/* Generates random strings that are syntactically valid JavaScript */
public class JavaScriptCodeGenerator extends Generator<String> {
    public JavaScriptCodeGenerator() {
        super(String.class); // Register type of generated object
    }

    private GenerationStatus status; // saved state object when generating
    private static final int MAX_IDENTIFIERS = 100;
    private static final int MAX_EXPRESSION_DEPTH = 10;
    private static final int MAX_STATEMENT_DEPTH = 6;
    private static Set<String> identifiers; // Stores generated IDs, to promote re-use
    private int statementDepth; // Keeps track of how deep the AST is at any point
    private int expressionDepth; // Keeps track of how nested an expression is at any point

    private static final String[] UNARY_TOKENS = {
            "!", "++", "--", "~",
            "delete", "new", "typeof"
    };

    private static final String[] BINARY_TOKENS = {
            "!=", "!==", "%", "%=", "&", "&&", "&=", "*", "*=", "+", "+=", ",",
            "-", "-=", "/", "/=", "<", "<<", ">>=", "<=", "=", "==", "===",
            ">", ">=", ">>", ">>=", ">>>", ">>>=", "^", "^=", "|", "|=", "||",
            "in", "instanceof"
    };

    /** Main entry point. Called once per test case. Returns a random JS program. */
    @Override
    public String generate(SourceOfRandomness random, GenerationStatus status) {
        this.status = status; // we save this so that we can pass it on to other generators
        this.identifiers = new HashSet<>();
        this.statementDepth = 0;
        this.expressionDepth = 0;
        return generateStatement(random).toString();
    }

    /** Utility method for generating a random list of items (e.g. statements, arguments, attributes) */
    private static List<String> generateItems(Function<SourceOfRandomness, String> genMethod, SourceOfRandomness random,
                                             int mean) {
        int len = random.nextInt(mean*2); // Generate random number in [0, mean*2) 
        List<String> items = new ArrayList<>(len);
        for (int i = 0; i < len; i++) {
            items.add(genMethod.apply(random));
        }
        return items;
    }

    /** Generates a random JavaScript statement */
    private String generateStatement(SourceOfRandomness random) {
        statementDepth++;
        String result;
        // If depth is too high, then generate only simple statements to prevent infinite recursion
        // If not, generate simple statements after the flip of a coin
        if (statementDepth >= MAX_STATEMENT_DEPTH || random.nextBoolean()) {
            // Choose a random private method from this class, and then call it with `random`
            result = random.choose(Arrays.<Function<SourceOfRandomness, String>>asList(
                    this::generateExpressionStatement,
                    this::generateBreakNode,
                    this::generateContinueNode,
                    this::generateReturnNode,
                    this::generateThrowNode,
                    this::generateVarNode,
                    this::generateEmptyNode
            )).apply(random);
        } else {
            // If depth is low and we won the flip, then generate compound statements
            // (that is, statements that contain other statements)
            result = random.choose(Arrays.<Function<SourceOfRandomness, String>>asList(
                    this::generateIfNode,
                    this::generateForNode,
                    this::generateWhileNode,
                    this::generateNamedFunctionNode,
                    this::generateSwitchNode,
                    this::generateTryNode,
                    this::generateBlock
            )).apply(random);
        }
        statementDepth--; // Reset statement depth when going up the recursive tree
        return result;
    }

    /** Generates a random JavaScript expression using recursive calls */
    private String generateExpression(SourceOfRandomness random) {
        expressionDepth++;
        String result;
        // Choose terminal if nesting depth is too high or based on a random flip of a coin
        if (expressionDepth >= MAX_EXPRESSION_DEPTH || random.nextBoolean()) {
            result = random.choose(Arrays.<Function<SourceOfRandomness, String>>asList(
                    this::generateLiteralNode,
                    this::generateIdentNode
            )).apply(random);
        } else {
            // Otherwise, choose a non-terminal generating function
            result = random.choose(Arrays.<Function<SourceOfRandomness, String>>asList(
                    this::generateBinaryNode,
                    this::generateUnaryNode,
                    this::generateTernaryNode,
                    this::generateCallNode,
                    this::generateFunctionNode,
                    this::generatePropertyNode,
                    this::generateIndexNode,
                    this::generateArrowFunctionNode
            )).apply(random);
        }
        expressionDepth--;
        return "(" + result + ")";
    }

    /** Generates a random binary expression (e.g. A op B) */
    private String generateBinaryNode(SourceOfRandomness random) {
        String token = random.choose(BINARY_TOKENS); // Choose a binary operator at random
        String lhs = generateExpression(random);
        String rhs = generateExpression(random);

        return lhs + " " + token + " " + rhs;
    }

    /** Generates a block of statements delimited by ';' and enclosed by '{' '}' */
    private String generateBlock(SourceOfRandomness random) {
        return "{ " + String.join(";", generateItems(this::generateStatement, random, 4)) + " }";
    }

    private String generateBreakNode(SourceOfRandomness random) {
        return "break";
    }

    private String generateCallNode(SourceOfRandomness random) {
        String func = generateExpression(random);
        String args = String.join(",", generateItems(this::generateExpression, random, 3));

        String call = func + "(" + args + ")";
        if (random.nextBoolean()) {
            return call;
        } else {
            return "new " + call;
        }
    }

    private String generateCaseNode(SourceOfRandomness random) {
        return "case " + generateExpression(random) + ": " +  generateBlock(random);
    }

    private String generateCatchNode(SourceOfRandomness random) {
        return "catch (" + generateIdentNode(random) + ") " +
                generateBlock(random);
    }

    private String generateContinueNode(SourceOfRandomness random) {
        return "continue";
    }

    private String generateEmptyNode(SourceOfRandomness random) {
        return "";
    }

    private String generateExpressionStatement(SourceOfRandomness random) {
        return generateExpression(random);
    }

    private String generateForNode(SourceOfRandomness random) {
        String s = "for(";
        if (random.nextBoolean()) {
            s += generateExpression(random);
        }
        s += ";";
        if (random.nextBoolean()) {
            s += generateExpression(random);
        }
        s += ";";
        if (random.nextBoolean()) {
            s += generateExpression(random);
        }
        s += ")";
        s += generateBlock(random);
        return s;
    }

    private String generateFunctionNode(SourceOfRandomness random) {
        return "function(" + String.join(", ", generateItems(this::generateIdentNode, random, 5)) + ")" + generateBlock(random);
    }

    private String generateNamedFunctionNode(SourceOfRandomness random) {
        return "function " + generateIdentNode(random) + "(" + String.join(", ", generateItems(this::generateIdentNode, random, 5)) + ")" + generateBlock(random);
    }

    private String generateArrowFunctionNode(SourceOfRandomness random) {
        String params = "(" + String.join(", ", generateItems(this::generateIdentNode, random, 3)) + ")";
        if (random.nextBoolean()) {
            return params + " => " + generateBlock(random);
        } else {
            return params + " => " + generateExpression(random);
        }

    }

    private String generateIdentNode(SourceOfRandomness random) {
        // Either generate a new identifier or use an existing one
        String identifier;
        if (identifiers.isEmpty() || (identifiers.size() < MAX_IDENTIFIERS && random.nextBoolean())) {
            identifier = random.nextChar('a', 'z') + "_" + identifiers.size();
            identifiers.add(identifier);
        } else {
            identifier = random.choose(identifiers);
        }

        return identifier;
    }

    private String generateIfNode(SourceOfRandomness random) {
        return "if (" +
                generateExpression(random) + ") " +
                generateBlock(random) +
                (random.nextBoolean() ? generateBlock(random) : "");
    }

    private String generateIndexNode(SourceOfRandomness random) {
        return generateExpression(random) + "[" + generateExpression(random) + "]";
    }

    private String generateObjectProperty(SourceOfRandomness random) {
        return generateIdentNode(random) + ": " + generateExpression(random);
    }

    private String generateLiteralNode(SourceOfRandomness random) {
        // If we are not too deeply nested, then it is okay to generate array/object literals
        if (expressionDepth < MAX_EXPRESSION_DEPTH && random.nextBoolean()) {
            if (random.nextBoolean()) {
                // Array literal
                return "[" + String.join(", ", generateItems(this::generateExpression, random, 3)) + "]";
            } else {
                // Object literal
                return "{" + String.join(", ", generateItems(this::generateObjectProperty, random, 3)) + "}";

            }
        } else {
            // Otherwise, generate primitive literals
            return random.choose(Arrays.<Supplier<String>>asList(
                    () -> String.valueOf(random.nextInt(-10, 1000)), // int literal
                    () -> String.valueOf(random.nextBoolean()),      // bool literal
                    () -> generateStringLiteral(random),
                    () -> "undefined",
                    () -> "null",
                    () -> "this"
            )).get();
        }
    }

    private String generateStringLiteral(SourceOfRandomness random) {
        // Generate an arbitrary string using the default string generator, and quote it
        return '"' + gen().type(String.class).generate(random, status) + '"';
    }

    private String generatePropertyNode(SourceOfRandomness random) {
        return generateExpression(random) + "." + generateIdentNode(random);
    }

    private String generateReturnNode(SourceOfRandomness random) {
        return random.nextBoolean() ? "return" : "return " + generateExpression(random);
    }

    private String generateSwitchNode(SourceOfRandomness random) {
        return "switch(" + generateExpression(random) + ") {"
                + String.join(" ", generateItems(this::generateCaseNode, random, 2)) + "}";
    }

    private String generateTernaryNode(SourceOfRandomness random) {
        return generateExpression(random) + " ? " + generateExpression(random) +
                " : " + generateExpression(random);
    }

    private String generateThrowNode(SourceOfRandomness random) {
        return "throw " + generateExpression(random);
    }

    private String generateTryNode(SourceOfRandomness random) {
        return "try " + generateBlock(random) + generateCatchNode(random);
    }

    private String generateUnaryNode(SourceOfRandomness random) {
        String token = random.choose(UNARY_TOKENS);
        return token + " " + generateExpression(random);
    }

    private String generateVarNode(SourceOfRandomness random) {
        return "var " + generateIdentNode(random);
    }

    private String generateWhileNode(SourceOfRandomness random) {
        return "while (" + generateExpression(random) + ")" + generateBlock(random);
    }
}
```

修改测试驱动程序，添加一个新的模糊测试入口点。这个新的测试方法（我们将其命名为 testWithGenerator）与 testWithString 类似，但它指定其输入参数由我们刚刚编写的 JavaScriptGenerator 生成。修改文件 src/test/java/examples/CompilerTest.java，内容如下：

```java
...
import com.pholser.junit.quickcheck.From;
...
   /* In class CompilerTest... */

    @Fuzz
    public void testWithGenerator(@From(JavaScriptCodeGenerator.class) String code) {
        SourceFile input = SourceFile.fromCode("input", code);
        compile(input);
    }
...
```

再次启动模糊测试引擎，但这次指定方法 testWithGenerator：

```terminal
mvn jqf:fuzz -Dclass=examples.CompilerTest -Dmethod=testWithGenerator -Dtime=5m
```
