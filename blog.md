## **`parallelStream` 源码分析及技术解读**

`parallelStream` 是 Java 8 引入的一个非常重要的特性，它允许通过并行处理流中的元素来提升性能。`parallelStream` 是 `Stream` 接口的一个方法，底层实现通过并行处理来提高数据处理的速度，特别是在处理大量数据时。通过 `parallelStream`，我们可以让 `Stream` 在多核处理器上并行执行任务，从而提高计算效率。

在这篇文章中，我们将深入分析 `parallelStream` 的源码实现原理，讨论其工作机制，以及如何正确使用它来提升性能。同时，我们也会通过示例来演示如何使用并行流进行处理。

## **1. `parallelStream` 工作原理概述**

`parallelStream` 是 `Stream` 接口的一个方法，它会将流的操作转化为并行操作。对于一个串行流，操作的顺序是固定的，而并行流则可以在多个线程中并行处理流中的元素。通过 `parallelStream`，Java 底层会把任务分割成多个小任务，并利用线程池来并行执行这些小任务。

### **工作机制**

`parallelStream` 背后依赖于 `ForkJoinPool`，它是 Java 7 引入的一个线程池，用于支持大规模的任务分解和并行执行。具体来说，`parallelStream` 会在以下几个方面做一些优化：

- **数据分割**：`parallelStream` 会将流的数据分割成若干个子任务，每个子任务在一个单独的线程上执行。如何分割数据，取决于流的源头（如集合、数组等）。
- **任务并行化**：流中的每个元素都会在不同的线程中处理，最终的结果会进行合并。
- **合并结果**：每个子任务的结果会被合并成最终结果，通常会在一个统一的“归约”操作中进行。

### **`parallelStream` 与 `ForkJoinPool`**

`ForkJoinPool` 是 `parallelStream` 使用的线程池，它能够自动地把大的任务分解成小的子任务，并将它们并行执行。`ForkJoinPool` 的工作机制是基于工作窃取算法的：当某个线程执行任务较少时，它可以从其他线程“窃取”任务来执行，从而平衡线程之间的工作量。

## **2. `parallelStream` 源码分析**

### **`Stream.parallel()` 源码**

在 `Stream` 接口中，`parallelStream` 实际上是调用了 `Stream.parallel()` 方法来实现并行流的功能。

```java
public interface Stream<T> {

    Stream<T> parallel();
    
    // Other methods...
}
```

`parallel()` 方法的源码实现如下：

```java
public Stream<T> parallel() {
    // 如果流已经是并行的，就直接返回当前流
    if (isParallel())
        return this;
    // 如果流是串行的，则将流的状态设置为并行，并返回新的并行流
    return new ParallelStream<>(this);
}
```

这个方法的作用非常简单：它会将当前的串行流切换为并行流。`parallel()` 方法并不直接实现并行处理，它会返回一个新的并行流，底层实现则会通过 `ForkJoinPool` 来进行并行操作。

### **并行流的实现**

在 `Stream` 接口中，`parallelStream` 是通过返回一个并行流对象来实现的，核心部分依赖于 `AbstractPipeline` 类。

```java
abstract class AbstractPipeline<E, S extends BaseStream<E, S>, S> implements BaseStream<E, S> {
    
    // 定义流的并行性
    private final boolean parallel;
    
    AbstractPipeline(S source, boolean parallel) {
        this.source = source;
        this.parallel = parallel;
    }
    
    public boolean isParallel() {
        return parallel;
    }
    
    // 创建并行流的具体实现
    @Override
    public S parallel() {
        return newParallelStream();
    }
    
    protected abstract S newParallelStream();
}
```

可以看到，`AbstractPipeline` 类提供了并行流的基本框架和方法。在构造函数中，`parallel` 字段定义了流是否为并行流。`newParallelStream()` 是一个抽象方法，具体的实现由不同的流类型来提供。

在流的处理过程中，`Stream` 的操作会根据 `parallel` 的标识来决定是串行执行还是并行执行。

### **`ForkJoinPool` 的使用**

`ForkJoinPool` 是 `parallelStream` 并行处理的核心。它负责将任务拆分成多个小任务并将其分配给工作线程来并行执行。`ForkJoinPool` 提供了 `invoke()` 方法来执行任务，而任务的拆分和合并则依赖于 `RecursiveTask` 或 `RecursiveAction` 类。

下面是 `Stream` 中并行流操作的一个示例实现：

```java
class ParallelStream<T> extends AbstractPipeline<T, ParallelStream<T>, Stream<T>> {
    
    ParallelStream(Stream<T> source) {
        super(source, true);
    }
    
    @Override
    protected Stream<T> newParallelStream() {
        // 返回一个并行流
        return new ParallelStream<>(this.source);
    }
    
    @Override
    public void forEach(Consumer<? super T> action) {
        // 使用 ForkJoinPool 执行并行任务
        ForkJoinPool.commonPool().submit(() -> {
            source.forEach(action);
        });
    }
}
```

### **并行流的操作**

在并行流中，常见的操作包括 `map`、`filter`、`reduce` 等。这些操作会被分解成多个任务并行执行，最终合并结果。例如，对于 `map` 操作，它会将流中的每个元素在多个线程中处理，并通过合并操作将结果汇总。

### **任务分割与合并**

`parallelStream` 会将任务分割成多个小任务，并通过 `ForkJoinPool` 来并行执行。在任务执行完成后，它会通过合并器将结果合并成最终结果。这是通过使用 `Collector` 接口来实现的，`Collector` 定义了如何将并行任务的结果合并。

例如，在执行 `reduce` 操作时，它会使用如下的策略：

- **分割任务**：将流分割成多个部分，分别执行。
- **合并任务**：每个任务完成后，结果会通过合并操作合并成最终结果。

## **3. 使用并行流的注意事项**

尽管 `parallelStream` 提供了并行处理的便利，但在实际使用中，我们仍然需要谨慎对待并行流的使用。以下是一些常见的注意事项：

- **性能问题**：对于小数据量，使用并行流可能不会带来性能提升，反而可能因为线程创建和上下文切换的开销而降低性能。
- **无序操作**：并行流可能改变元素的顺序，因此对于有顺序要求的操作需要格外小心。
- **线程安全问题**：并行流会在多个线程中执行，因此操作中的共享资源需要确保线程安全，避免竞态条件。

## **4. 示例与性能对比**

### **示例 1：简单的并行流操作**

以下是一个简单的并行流操作示例：

```java
import java.util.List;
import java.util.Arrays;

public class ParallelStreamDemo {
    public static void main(String[] args) {
        List<Integer> numbers = Arrays.asList(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);
        
        // 串行流处理
        long startTime = System.currentTimeMillis();
        numbers.stream().map(n -> n * n).forEach(System.out::println);
        long endTime = System.currentTimeMillis();
        System.out.println("串行流处理时间: " + (endTime - startTime) + "ms");
        
        // 并行流处理
        startTime = System.currentTimeMillis();
        numbers.parallelStream().map(n -> n * n).forEach(System.out::println);
        endTime = System.currentTimeMillis();
        System.out.println("并行流处理时间: " + (endTime - startTime) + "ms");
    }
}
```

### **示例 2：性能对比**

我们可以通过一个简单的性能测试来对比串行流和并行流的性能：

```java
import java.util.List;
import java.util.ArrayList;

public class ParallelStreamPerformanceTest {
    public static void main(String[] args) {
        List<Long> list = new ArrayList<>();
        for (long i = 0; i < 10_000_000; i++) {
            list.add(i);
        }

        // 串行流
        long start = System.nanoTime();
        list.stream().mapToLong(Long::longValue).sum();
        long end = System.nanoTime();
        System.out.println("串行流总时间: " + (end - start));

        // 并行流
        start = System.nanoTime();
        list.parallelStream().mapToLong(Long
```



### ForkJoinPool 是什么？

`ForkJoinPool` 是 Java 7 引入的一个特殊的线程池实现，旨在支持并行任务的分解与合并。它是通过**分治法**（Divide and Conquer）来处理计算密集型任务的。`ForkJoinPool` 对于任务的拆分与合并有着优化的机制，能够高效地管理大量的任务，尤其适用于递归任务、并行流等需要任务拆分和合并的场景。

`ForkJoinPool` 在 `parallelStream` 等并行流的实现中扮演着重要角色。`parallelStream` 会利用 `ForkJoinPool` 来分配和调度并行任务。

### ForkJoinPool 的工作机制

`ForkJoinPool` 的主要特性包括：

1. **工作窃取算法**：
   - `ForkJoinPool` 使用**工作窃取算法**（Work Stealing Algorithm），即每个工作线程都有一个任务队列，当一个线程的任务队列为空时，它会“偷”其他线程的任务。这种机制有助于提高 CPU 的利用率，避免某些线程处于空闲状态。
   
2. **任务的拆分和合并**：
   - `ForkJoinPool` 的任务通常是一个递归任务，通常会继承 `RecursiveTask` 或 `RecursiveAction` 类。任务会被拆分成小的子任务，直到足够小可以直接执行，执行完成后再将结果合并。
   - 任务的拆分和合并是 `ForkJoinPool` 最为核心的特性。

3. **高效的线程利用**：
   - `ForkJoinPool` 会根据实际的任务量和核心数来动态调整线程池的大小。它采用了轻量级的线程创建和销毁机制，避免了传统线程池中线程的频繁创建和销毁带来的开销。

### ForkJoinPool 的构造方法及参数

`ForkJoinPool` 提供了几个不同的构造方法，可以用来定制线程池的行为。

#### 1. **`ForkJoinPool()`** - 默认构造方法

```java
public ForkJoinPool();
```

默认构造方法会创建一个拥有默认线程数的 `ForkJoinPool`。它的默认线程数等于当前系统的处理器核心数。

#### 2. **`ForkJoinPool(int parallelism)`** - 指定并行度

```java
public ForkJoinPool(int parallelism);
```

- `parallelism`：指定并行度，即最多可以有多少个工作线程同时运行。这个参数一般会传递一个比 CPU 核心数多的值来提高并行度，或者根据任务的需求来设置。

`parallelism` 会决定线程池的最大工作线程数。一般来说，这个值设为系统的 CPU 核心数就足够了，因为它保证了每个核心有一个工作线程。

#### 3. **`ForkJoinPool(int parallelism, ForkJoinWorkerThreadFactory factory, Thread.UncaughtExceptionHandler handler, boolean asyncMode)`** - 完全自定义构造方法

```java
public ForkJoinPool(int parallelism, 
                    ForkJoinWorkerThreadFactory factory, 
                    Thread.UncaughtExceptionHandler handler, 
                    boolean asyncMode);
```

这个构造方法允许开发者完全自定义 `ForkJoinPool` 的行为。它接受四个参数：

- **`parallelism`**：线程池的并行度，指定线程池的最大线程数。
- **`factory`**：一个 `ForkJoinWorkerThreadFactory` 工厂，用于创建工作线程。如果不指定，系统会使用默认的工厂。
- **`handler`**：未捕获异常处理器，用于处理工作线程中出现的未捕获异常。
- **`asyncMode`**：如果为 `true`，`ForkJoinPool` 将以异步模式运行。在这种模式下，任务完成后不会立即执行结果的合并，而是等待所有任务都完成后再进行合并。

#### 4. **`ForkJoinPool.commonPool()`** - 公共池

```java
public static ForkJoinPool commonPool();
```

`commonPool()` 返回一个共享的 `ForkJoinPool` 实例，通常用于没有显式指定池的情况，例如 `parallelStream`。`commonPool` 采用系统默认的核心数作为并行度，适合一般的并行任务。

### ForkJoinPool 的重要参数

`ForkJoinPool` 的核心参数可以总结为以下几项：

1. **`parallelism`**：并行度，决定线程池最多可以有多少个并行工作线程。
   - 默认值：CPU 核心数。
   - 自定义时，根据应用的实际需求进行调整，通常与 CPU 核心数一致，或者根据任务的粒度调整。

2. **`ForkJoinWorkerThreadFactory`**：自定义线程工厂，用来创建工作线程。
   - 默认工厂：`ForkJoinPool.defaultForkJoinWorkerThreadFactory`，创建一个默认的工作线程。
   - 可以实现自定义工厂，控制线程的创建过程，甚至可以为每个工作线程设置不同的属性。

3. **`Thread.UncaughtExceptionHandler`**：未捕获异常的处理器。每个线程在运行时，如果发生未捕获的异常，可以通过这个处理器来捕获和处理。
   - 默认处理器：`Thread.getDefaultUncaughtExceptionHandler()`。

4. **`asyncMode`**：是否使用异步模式执行任务。这个参数允许调整任务完成后的合并策略，异步模式不会立即合并子任务的结果，而是会等所有任务都完成之后再进行合并。

### ForkJoinPool 的使用示例

下面通过代码示例，展示如何使用 `ForkJoinPool` 来并行处理任务。

#### 示例 1：使用默认构造方法

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;

public class ForkJoinPoolExample {
    
    static class SumTask extends RecursiveTask<Long> {
        private final long[] array;
        private final int start, end;
        
        SumTask(long[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected Long compute() {
            if (end - start <= 2) {
                long sum = 0;
                for (int i = start; i <= end; i++) {
                    sum += array[i];
                }
                return sum;
            } else {
                int mid = (start + end) / 2;
                SumTask left = new SumTask(array, start, mid);
                SumTask right = new SumTask(array, mid + 1, end);
                
                left.fork();  // 异步执行左边的任务
                long rightResult = right.compute();  // 同步执行右边的任务
                long leftResult = left.join();  // 获取左边任务的结果
                
                return leftResult + rightResult;  // 合并结果
            }
        }
    }

    public static void main(String[] args) {
        long[] array = new long[100];
        for (int i = 0; i < 100; i++) {
            array[i] = i + 1;
        }
        
        ForkJoinPool pool = new ForkJoinPool();
        SumTask task = new SumTask(array, 0, array.length - 1);
        
        long result = pool.invoke(task);
        System.out.println("Total sum: " + result);
    }
}
```

在这个示例中，`SumTask` 类继承了 `RecursiveTask`，它负责计算数组的总和。任务会被分割成两部分，直到每个部分包含足够少的元素，能够直接进行计算。

- `fork()` 方法将任务分配给工作线程异步执行。
- `join()` 方法等待任务执行完成，并返回结果。
- `compute()` 方法是任务的实际执行方法。

#### 示例 2：使用自定义线程工厂

```java
import java.util.concurrent.RecursiveTask;
import java.util.concurrent.ForkJoinPool;
import java.util.concurrent.ForkJoinWorkerThreadFactory;
import java.util.concurrent.ThreadFactory;

public class CustomForkJoinPoolExample {
    
    static class SumTask extends RecursiveTask<Long> {
        private final long[] array;
        private final int start, end;
        
        SumTask(long[] array, int start, int end) {
            this.array = array;
            this.start = start;
            this.end = end;
        }
        
        @Override
        protected Long compute() {
            if (end - start <= 2) {
                long sum = 0;
                for (int i = start; i <= end; i++) {
                    sum += array[i];
                }
                return sum;
            } else {
                int mid = (start + end) / 2;
                SumTask left = new SumTask(array, start, mid);
                SumTask right = new SumTask(array, mid + 1, end);
                
                left.fork();
                long rightResult = right.compute();
                long leftResult = left.join();
                
                return leftResult + rightResult;
            }
        }
    }

    public static void main(String[] args) {
        long[] array = new long[100];
        for (int i = 0; i < 100; i++)


```
`parallelStream` 默认使用的是 **`ForkJoinPool.commonPool()`** 线程池。

### ForkJoinPool.commonPool() 详解

`ForkJoinPool.commonPool()` 是 Java 7 引入的一个全局共享的 `ForkJoinPool` 实例，它是 Java 默认用于 `parallelStream` 和一些其他并行任务（如 `CompletableFuture`）的线程池。

#### 关键点：
- **共享池**：`commonPool()` 是一个全局共享的线程池，它的生命周期是由 JVM 管理的，所有使用并行流的任务都会共享这个池。
- **线程数**：`ForkJoinPool.commonPool()` 的默认并行度通常是 **`Runtime.getRuntime().availableProcessors()`**，即当前机器的 CPU 核心数。比如，如果你的系统有 4 个 CPU 核心，那么 `commonPool()` 会默认有 4 个工作线程。
- **工作窃取**：这个池使用工作窃取算法（work-stealing algorithm），即当一个线程的任务队列为空时，它会尝试从其他线程的队列中“偷”任务，这有助于提高 CPU 利用率，避免线程的空闲。

### 默认线程池的使用方式

当你调用 `stream()` 或 `parallelStream()` 时，如果没有显式指定使用哪个线程池，它默认会使用 `ForkJoinPool.commonPool()` 来执行任务。这个池提供了一个统一的线程池管理方案，不需要用户手动管理线程池的生命周期。

#### 示例：

```java
import java.util.Arrays;
import java.util.List;

public class ParallelStreamDefaultPool {
    public static void main(String[] args) {
        List<Integer> list = Arrays.asList(1, 2, 3, 4, 5);

        // 使用 parallelStream，默认使用 ForkJoinPool.commonPool()
        list.parallelStream().forEach(i -> {
            System.out.println(Thread.currentThread().getName() + " processing " + i);
        });
    }
}
```

在上述示例中，`parallelStream` 会默认使用 `ForkJoinPool.commonPool()` 来执行并行任务，并且打印出每个任务的执行线程名，可以观察到并行任务是由共享池中的线程来执行的。

### 如何调整默认线程池

1. **修改并行度**：通过设置 `ForkJoinPool.commonPool()` 的最大并行度，你可以通过 **JVM 参数** 来影响 `commonPool` 的并行度。比如，设置 `-Djava.util.concurrent.ForkJoinPool.common.parallelism=8` 可以将池的最大并行度设置为 8。

2. **使用自定义线程池**：
   如果你需要更精确的控制并行度或使用自定义的线程池（例如针对特定任务需要更多或更少的线程），你可以通过显式创建 `ForkJoinPool` 来替代 `commonPool()`。

   ```java
   ForkJoinPool customPool = new ForkJoinPool(8); // 创建一个包含 8 个工作线程的 ForkJoinPool
   customPool.submit(() -> {
       list.parallelStream().forEach(i -> {
           System.out.println(Thread.currentThread().getName() + " processing " + i);
       });
   }).join();
   ```

在上面的示例中，`parallelStream` 将会在自定义的线程池 `customPool` 中执行，而不是使用默认的 `commonPool`。

### 总结

- **默认线程池**：`parallelStream` 使用的是 `ForkJoinPool.commonPool()`。
- **并行度**：`commonPool` 默认的并行度等于系统的 CPU 核心数（即 `Runtime.getRuntime().availableProcessors()`）。
- **控制并行度**：可以通过 JVM 参数或自定义 `ForkJoinPool` 来调整默认线程池的行为。

这种设计使得并行流能够高效地利用多核处理器，同时避免了创建和销毁线程带来的开销。但对于一些特定的应用场景，可能需要自定义线程池来满足特殊的需求。



# 深入解析 `ForkJoinPool`：为何向死锁池提交任务仍能完成？

`ForkJoinPool` 是 Java 并发编程中的一项重要工具，广泛应用于大规模任务并行处理场景。与传统线程池不同，`ForkJoinPool` 使用了**工作窃取**（work-stealing）算法，能够高效地分配和执行任务。然而，很多开发者可能会有疑问：“为什么当向 `ForkJoinPool` 提交任务时，即使池中某些线程死锁，其他任务依然能够正常完成？”本文将结合 `ForkJoinPool` 的源码分析，探讨其在死锁情况下仍能正常完成任务的原因。

## 1. `ForkJoinPool` 的基本工作原理

### 1.1 线程池的工作模式

`ForkJoinPool` 是 Java 7 引入的线程池实现，专门用于分治任务的执行。在 `ForkJoinPool` 中，任务被递归地分解为子任务，这些子任务由池中的线程并行处理。`ForkJoinPool` 使用了 **工作窃取（work-stealing）算法**，它允许线程在自己的任务队列为空时，从其他线程的任务队列中窃取任务。

`ForkJoinPool` 内部维护着多个线程，并且每个线程都有自己的任务队列。当任务递归分解后，子任务被推送到这些任务队列中，线程会在自己的队列中执行任务。如果某个线程的任务队列为空，它将主动从其他线程的队列中窃取任务。

### 1.2 工作窃取与死锁的关系

`ForkJoinPool` 的工作窃取模型使得它在面对死锁时仍能保持高效的执行。当一个线程因等待某些资源（如锁或外部任务的完成）而阻塞时，其他线程仍然能够窃取并执行其他任务。虽然某些任务可能会处于“死锁”状态，`ForkJoinPool` 的工作窃取机制仍能保证任务池中的其他线程执行其他任务。

## 2. `ForkJoinPool` 如何应对死锁问题

### 2.1 线程池的资源分配与任务调度

死锁通常指的是多个线程互相等待对方释放资源，形成一个环状等待关系，从而导致系统无法继续执行。然而，在 `ForkJoinPool` 中，线程的调度并不完全依赖于任务的顺序和锁的获取。因此，死锁不会导致整个池的崩溃或无法执行。

#### **任务的并行度**

`ForkJoinPool` 默认的并行度通常等于 CPU 核心数（通过 `Runtime.getRuntime().availableProcessors()` 获取）。当一个线程被阻塞（例如在等待一个锁或 I/O 操作），池中的其他线程仍然可以继续执行任务。例如，线程 A 可能因等待任务 B 完成而处于阻塞状态，但这并不会影响线程 B、C、D 等其他线程继续执行其他任务。

#### **工作窃取机制**

假设某个任务进入了等待状态，`ForkJoinPool` 中的其他线程会尝试从等待线程的任务队列中窃取任务。这种工作窃取机制有效地避免了死锁导致整个池停止的情形。

### 2.2 死锁的局部性

尽管死锁可能发生，但它通常是局部的，即仅限于某些线程之间。如果线程 A 和线程 B 因为锁的竞争发生死锁，其他线程（如线程 C 和线程 D）仍然能够继续执行。`ForkJoinPool` 的设计保证了即使某些线程死锁，其他线程依然能够继续执行，并且池中任务的执行并不会因为局部死锁而停滞。

#### **线程阻塞与空闲线程的任务窃取**

`ForkJoinPool` 中的线程会在没有任务时主动窃取其他线程的任务，而非等待自己队列中的任务执行完后再继续工作。这种设计使得即使某些线程处于阻塞状态，其他线程仍然可以继续执行任务，防止了死锁的全局扩展。

## 3. 源码分析：`ForkJoinPool` 如何避免死锁

我们来深入分析 `ForkJoinPool` 的一些关键源码，看看它如何避免死锁，并且保证任务的并行执行。

### 3.1 `ForkJoinPool` 的线程池结构

`ForkJoinPool` 内部有一个数组，保存着所有线程。每个线程会有自己的任务队列。线程在执行任务时，首先会从自己队列中获取任务。如果任务队列为空，它会尝试从其他线程的任务队列中窃取任务。

```java
public class ForkJoinPool extends AbstractExecutorService {
    final WorkQueue[] workQueues;
    final ForkJoinWorkerThreadFactory factory;
    final ThreadFactory defaultForkJoinWorkerThreadFactory;

    public ForkJoinPool(int parallelism) {
        this.workQueues = new WorkQueue[parallelism];
        // 初始化工作队列和工厂等
    }
}
```

### 3.2 任务窃取逻辑

当一个线程的队列为空，它会去尝试从其他线程的任务队列中窃取任务。具体实现如下：

```java
final class WorkQueue {
    final ForkJoinTask<?>[] stack;
    volatile int top;  // 栈顶指针
    volatile long stealCount;  // 窃取计数

    public ForkJoinTask<?> pop() {
        ForkJoinTask<?> task = stack[top];
        if (task != null) {
            stack[top] = null;
            top--;
        }
        return task;
    }

    public ForkJoinTask<?> steal() {
        // 从其他线程的队列中窃取任务
        // 例如，尝试从队列尾部窃取任务
        if (stealCount > 0) {
            return pop();
        }
        return null;
    }
}
```

这段代码展示了 `ForkJoinPool` 内部的 **`WorkQueue`** 如何通过 `pop()` 和 `steal()` 方法进行任务的获取和窃取。即使某个线程因为任务等待或死锁而处于阻塞状态，其他线程依然可以继续从空闲线程队列中窃取任务。

### 3.3 线程池调度

`ForkJoinPool` 还提供了 `submit()` 和 `invokeAll()` 等方法来调度任务。任务的调度是通过将任务提交到池中的队列来完成的，而不是直接执行。

```java
public <T> ForkJoinTask<T> submit(ForkJoinTask<T> task) {
    addTask(task);  // 将任务添加到任务队列
    return task;
}
```

### 3.4 死锁与线程阻塞

虽然在 `ForkJoinPool` 中死锁的发生不可避免，但它通常是局部的。线程池中的其他线程仍然可以继续执行任务。只有当所有线程都因任务等待或死锁而被阻塞时，才会导致整体停滞。因此，`ForkJoinPool` 的工作窃取机制在大多数情况下能够有效防止死锁导致的全局性停滞。

## 4. 示例：死锁情况下的任务执行

考虑以下代码，其中 `task2` 与 `task1` 存在资源依赖关系，并且可能导致死锁：

```java
ForkJoinPool pool = new ForkJoinPool();

pool.submit(() -> {
    synchronized (lockA) {
        synchronized (lockB) {
            System.out.println("Task 1");
        }
    }
});

pool.submit(() -> {
    synchronized (lockB) {
        synchronized (lockA) {
            System.out.println("Task 2");
        }
    }
});
```

在上述代码中，`task1` 和 `task2` 存在循环依赖关系，会形成死锁。尽管如此，池中的其他线程（如 `task3`）仍然可以执行：

```java
pool.submit(() -> System.out.println("Task 3"));
```

这段代码可以确保即使存在死锁，池中其他线程也能继续执行任务，保持并发执行的能力。

## 5. 总结

- **工作窃取机制**：`ForkJoinPool` 使用工作窃取算法，即使某些线程被阻塞，其他线程依然可以继续执行任务。即使发生了死锁，池中的其他线程也不会受到影响。
- **死锁的局部性**：死锁通常是局部的，仅影响部分线程。`ForkJoinPool` 设计保证了即使部分任务死锁，其他线程依然能继续工作。
- **源码分析**：通过分析 `ForkJoinPool` 的源码，我们可以看到工作窃取机制如何有效地避免死锁问题，并确保高效的任务执行。

因此，`ForkJoinPool` 的工作窃取机制和高并发任务调度策略使得即使在死锁的情况下，任务池仍能保持高效的运行。通过合理设计和使用 `ForkJoinPool`，我们可以更好地处理并发任务，避免死锁带来的影响。