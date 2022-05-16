---
layout:       post title:        "disruptor源码分析"
date:         2022-05-11 11:40:00 author:       "Brade"
header-style: text header-mask:  0.3 catalog:      true tags:

- disruptor
- Java
- 源码

---

# disruptor 启动流程

> 本文中 `disruptor` 使用的版本为 `3.4`. 具体可以查看 [Github](https://github.com/LMAX-Exchange/disruptor) .

## 1 关于 `disruptor`

- 英国外汇交易公司LMAX开发的一个高性能的异步处理队列, 研发的初衷是解决内存队列的延迟问题(在性能测试中发现竟然与I/O操作处于同样的数量级).
- 单线程能支撑每秒600万订单, 2010年在QCon演讲后,获得了业界关注. 2011年,企业应用软件专家Martin Fowler专门撰写长文介绍.同年它还获得了Oracle官方的Duke大奖.
- Apache Storm, Camel, Log4j2 在内的很多知名项目都应用了 `Disruptor` .

## 2 `disruptor` 的优雅设计,解决队列速度慢的问题

- `ArrayBlockingQueue`: 基于数组形式的队列，通过加锁的方式，来保证多线程情况下数据的安全;
- `LinkedBlockingQueue`: 基于链表形式的队列，也通过加锁的方式，来保证多线程情况下数据的安全;
- `ConcurrentLinkedQueue`: 基于链表形式的队列，通过CAS的方式;
- `Disruptor`:
    - 环形数组结构: 为了避免垃圾回收,采用数组而非链表. 同时,数组对处理器的缓存机制更加友好.
    - 元素位置定位: 数组长度2^n,通过位运算,加快定位的速度. 下标采取递增的形式,不用担心index溢出的问题. index是long类型,即使100万QPS的处理速度,也需要30万年才能用完。
    - 无锁设计: 每个生产者或者消费者线程,会先申请可以操作的元素在数组中的位置,申请到之后,直接在该位置写入或者读取数据.

## 3 测试用例

```java

public class DisruptorTest {

    private final static AtomicLong eventCount = new AtomicLong();

    /**
     * 1个生产者生产10条数据,5个消费者每个独立消费10次,然后再5个消费者共同消费10次
     * @param args
     * @throws InterruptedException
     */
    public static void main(String[] args) throws InterruptedException {
        EventFactory<StringEvent> eventFactory = StringEvent::new;

        Disruptor<StringEvent> disruptor = new Disruptor<>(eventFactory, 16, Executors.defaultThreadFactory(), ProducerType.SINGLE, new BlockingWaitStrategy());

        // 调用事件
        disruptor.handleEventsWith(buildEventHandle(5)).thenHandleEventsWithWorkerPool(buildWorkEventHandle(5));
        disruptor.start();

        // 获取缓存队列环
        RingBuffer<StringEvent> ringBuffer = disruptor.getRingBuffer();

        // 生产者发布数据
        String data[] = new String[10];
        for (int i = 1; i <= 10; i++) {
            data[i - 1] = "data: " + i + "";
        }

        disruptor.publishEvents((event, sequence, d) -> event.setValue(d), data);

        // disruptor.shutdown();
    }

    /**
     * 消费者消费数据(独立消费)
     *
     * @param count
     * @return
     */
    private static EventHandler<StringEvent>[] buildEventHandle(int count) {

        EventHandler<StringEvent>[] handlers = new EventHandler[count];
        // 消费者
        Consumer<StringEvent> consumer = s -> printData(s, "general");

        for (int i = 0; i < count; i++) {
            // 事件处理器
            EventHandler<StringEvent> eventHandler = (event, sequence, endOfBatch) -> {
                // 这里延时100ms，模拟消费事件的逻辑的耗时
                Thread.sleep(100);
                consumer.accept(event);
            };
            handlers[i] = eventHandler;
        }

        return handlers;
    }

    /**
     * 消费者消费数据(共享消费)
     *
     * @param count
     * @return
     */
    private static WorkHandler<StringEvent>[] buildWorkEventHandle(int count) {

        WorkHandler<StringEvent>[] handlers = new WorkHandler[count];
        // 消费者
        Consumer<StringEvent> consumer = s -> printData(s, "workders");

        for (int i = 0; i < count; i++) {
            // 事件处理器
            WorkHandler<StringEvent> eventHandler = (event) -> {
                // 这里延时100ms，模拟消费事件的逻辑的耗时
                Thread.sleep(100);
                consumer.accept(event);
            };
            handlers[i] = eventHandler;
        }

        return handlers;
    }

    /**
     * 输出数据
     * @param s
     * @param consumerType
     */
    private static void printData(StringEvent s, String consumerType) {
        System.out.println(Thread.currentThread().getName() + ": data is consumed by " + consumerType + " [" + eventCount.incrementAndGet() + "] -->" + s.getValue());
    }

    /**
     * 事件对象
     */
    public static class StringEvent {

        private String value;

        public String getValue() {
            return value;
        }

        public StringEvent setValue(String value) {
            this.value = value;
            return this;
        }
    }
}

```

## 4 启动流程解析

1. 创建事件
    - 一般指数据对象,比如上面用例中的`StringEvent`,或者订单,通知等
2. 创建 `Disruptor` 实例
    - 构造方法:

    ```java
    
   /**
     * Create a new Disruptor.
     *
     * @param eventFactory   the factory to create events in the ring buffer.
     * @param ringBufferSize the size of the ring buffer, must be power of 2.
     * @param threadFactory  a {@link ThreadFactory} to create threads for processors.
     * @param producerType   the claim strategy to use for the ring buffer.
     * @param waitStrategy   the wait strategy to use for the ring buffer.
     */
    public Disruptor(
            final EventFactory<T> eventFactory,
            final int ringBufferSize,
            final ThreadFactory threadFactory,
            final ProducerType producerType,
            final WaitStrategy waitStrategy)
    {
        this(
            RingBuffer.create(producerType, eventFactory, ringBufferSize, waitStrategy),
            new BasicExecutor(threadFactory));
    }
    
    ```

    - 环形队列:

    ```java
    
    /**
     * Create a new Ring Buffer with the specified producer type (SINGLE or MULTI)
     *
     * @param <E> Class of the event stored in the ring buffer.
     * @param producerType producer type to use {@link ProducerType}.
     * @param factory      used to create events within the ring buffer.
     * @param bufferSize   number of elements to create within the ring buffer.
     * @param waitStrategy used to determine how to wait for new elements to become available.
     * @return a constructed ring buffer.
     * @throws IllegalArgumentException if bufferSize is less than 1 or not a power of 2
     */
    public static <E> RingBuffer<E> create(
        ProducerType producerType,
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
    {
        switch (producerType)
        {
            case SINGLE:
                return createSingleProducer(factory, bufferSize, waitStrategy);
            case MULTI:
                return createMultiProducer(factory, bufferSize, waitStrategy);
            default:
                throw new IllegalStateException(producerType.toString());
        }
    }
   
    /**
     * 创建单生产者的环形队列
     * Create a new single producer RingBuffer with the specified wait strategy.
     *
     * @param <E> Class of the event stored in the ring buffer.
     * @param factory      used to create the events within the ring buffer.
     * @param bufferSize   number of elements to create within the ring buffer.
     * @param waitStrategy used to determine how to wait for new elements to become available.
     * @return a constructed ring buffer.
     * @throws IllegalArgumentException if bufferSize is less than 1 or not a power of 2
     * @see SingleProducerSequencer
     */
    public static <E> RingBuffer<E> createSingleProducer(
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
    {
        SingleProducerSequencer sequencer = new SingleProducerSequencer(bufferSize, waitStrategy);

        return new RingBuffer<E>(factory, sequencer);
    }
   
   /**
     * 创建多生产者的环形队列
     * Create a new multiple producer RingBuffer with the specified wait strategy.
     *
     * @param <E> Class of the event stored in the ring buffer.
     * @param factory      used to create the events within the ring buffer.
     * @param bufferSize   number of elements to create within the ring buffer.
     * @param waitStrategy used to determine how to wait for new elements to become available.
     * @return a constructed ring buffer.
     * @throws IllegalArgumentException if bufferSize is less than 1 or not a power of 2
     * @see MultiProducerSequencer
     */
    public static <E> RingBuffer<E> createMultiProducer(
        EventFactory<E> factory,
        int bufferSize,
        WaitStrategy waitStrategy)
    {
        MultiProducerSequencer sequencer = new MultiProducerSequencer(bufferSize, waitStrategy);

        return new RingBuffer<E>(factory, sequencer);
    }

    ```

    ```java
       abstract class RingBufferFields<E> extends RingBufferPad {
       private static final int BUFFER_PAD;
       private static final long REF_ARRAY_BASE;
       private static final int REF_ELEMENT_SHIFT;
       private static final Unsafe UNSAFE = Util.getUnsafe();
   
       static {
           // 返回数组中一个元素占用的大小
           final int scale = UNSAFE.arrayIndexScale(Object[].class);
           if (4 == scale) {
               REF_ELEMENT_SHIFT = 2;
           } else if (8 == scale) {
               REF_ELEMENT_SHIFT = 3;
           } else {
               throw new IllegalStateException("Unknown pointer size");
           }
           // 缓存填充,填充前后
           BUFFER_PAD = 128 / scale;
           // Including the buffer pad in the array base offset
           // UNSAFE.arrayBaseOffset(Object[].class): 获取数组第一个元素的偏移地址,即 16
           // 64 位的 JVM 数组类型的基础偏移都是 16,原始类型的基础偏移都是 12 (测试结果在不同 JVM 下可能会有所区别)
           // REF_ARRAY_BASE = 128 + 16 = 144
           // 定位数组中每个元素在内存中的位置:增量为4,向左左移2位相当于 * 4,即 32 * 4 = 128 位运算提高速度
           REF_ARRAY_BASE = UNSAFE.arrayBaseOffset(Object[].class) + (BUFFER_PAD << REF_ELEMENT_SHIFT);
       }
   
       private final long indexMask;
       private final Object[] entries;
       protected final int bufferSize;
       protected final Sequencer sequencer;
   
       RingBufferFields(
               EventFactory<E> eventFactory,
               Sequencer sequencer) {
           this.sequencer = sequencer;
           this.bufferSize = sequencer.getBufferSize();
   
           if (bufferSize < 1) {
               throw new IllegalArgumentException("bufferSize must not be less than 1");
           }
           // bufferSize 必须是 2 的幂
           if (Integer.bitCount(bufferSize) != 1) {
               throw new IllegalArgumentException("bufferSize must be a power of 2");
           }
   
           this.indexMask = bufferSize - 1;
           // 初始化数组并指定长度: BUFFER_PAD(32) * 2 + 16 = 80
           // BUFFER_PAD为数组填充大小,避免数组的有效元素出现伪共享
           // 数组的前X个元素会出现伪共享和后X个元素可能会出现伪共享,可能和无关数据加载到同一个缓存行
           // 额外创建2个填充空间的大小,首尾填充,避免数组的有效载荷和其它成员加载到同一缓存行
           this.entries = new Object[sequencer.getBufferSize() + 2 * BUFFER_PAD];
           fill(eventFactory);
       }
   
       /**
        * 填充数组元素:从下标为32开始
        *
        * @param eventFactory
        */
       private void fill(EventFactory<E> eventFactory) {
           for (int i = 0; i < bufferSize; i++) {
               // BUFFER_PAD+i为真正的数组索引:32+i
               entries[BUFFER_PAD + i] = eventFactory.newInstance();
           }
       }
   
       /**
        * 从数组 entries 中获取 序列 sequence 对应的元素值
        *
        * @param sequence
        * @return
        */
       @SuppressWarnings("unchecked")
       protected final E elementAt(long sequence) {
           // REF_ARRAY_BASE(获取数组第一个元素的偏移地址) + (sequence & indexMask) << REF_ELEMENT_SHIFT)(序列对应在数组中的偏移量)
           // REF_ARRAY_BASE: 144
           // 144 + (0/1/2 左移 2 位,相当于*4),即144 +(0/4/8/12)
           // bufferSize 必须2的幂
           return (E) UNSAFE.getObject(entries, REF_ARRAY_BASE + ((sequence & indexMask) << REF_ELEMENT_SHIFT));
       }
   }
    ```


3. 指定事件消费者
    - 独立事件消费者

    ```java
    
    /**
     * <p>Set up event handlers to handle events from the ring buffer. These handlers will process events
     * as soon as they become available, in parallel.</p>
     *
     * <p>This method can be used as the start of a chain. For example if the handler <code>A</code> must
     * process events before handler <code>B</code>:</p>
     * <pre><code>dw.handleEventsWith(A).then(B);</code></pre>
     *
     * <p>This call is additive, but generally should only be called once when setting up the Disruptor instance</p>
     *
     * @param handlers the event handlers that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    @SuppressWarnings("varargs")
    @SafeVarargs
    public final EventHandlerGroup<T> handleEventsWith(final EventHandler<? super T>... handlers)
    {
        return createEventProcessors(new Sequence[0], handlers);
    }
    
    EventHandlerGroup<T> createEventProcessors(
        final Sequence[] barrierSequences,
        final EventHandler<? super T>[] eventHandlers)
    {
        checkNotStarted();

        final Sequence[] processorSequences = new Sequence[eventHandlers.length];
        final SequenceBarrier barrier = ringBuffer.newBarrier(barrierSequences);

        for (int i = 0, eventHandlersLength = eventHandlers.length; i < eventHandlersLength; i++)
        {
            final EventHandler<? super T> eventHandler = eventHandlers[i];

            final BatchEventProcessor<T> batchEventProcessor =
                new BatchEventProcessor<>(ringBuffer, barrier, eventHandler);

            if (exceptionHandler != null)
            {
                batchEventProcessor.setExceptionHandler(exceptionHandler);
            }
            // 将 batchEventProcessor (独立消费者批量处理器)存储
            consumerRepository.add(batchEventProcessor, eventHandler, barrier);
            processorSequences[i] = batchEventProcessor.getSequence();
        }

        updateGatingSequencesForNextInChain(barrierSequences, processorSequences);

        return new EventHandlerGroup<>(this, consumerRepository, processorSequences);
    }

    ```

    - 共享事件消费者

    ```java

    /**
     * Set up a {@link WorkerPool} to distribute an event to one of a pool of work handler threads.
     * Each event will only be processed by one of the work handlers.
     * The Disruptor will automatically start this processors when {@link #start()} is called.
     *
     * @param workHandlers the work handlers that will process events.
     * @return a {@link EventHandlerGroup} that can be used to chain dependencies.
     */
    @SafeVarargs
    @SuppressWarnings("varargs")
    public final EventHandlerGroup<T> handleEventsWithWorkerPool(final WorkHandler<T>... workHandlers)
    {
        return createWorkerPool(new Sequence[0], workHandlers);
    }
   
    EventHandlerGroup<T> createWorkerPool(
        final Sequence[] barrierSequences, final WorkHandler<? super T>[] workHandlers)
    {
        final SequenceBarrier sequenceBarrier = ringBuffer.newBarrier(barrierSequences);
        final WorkerPool<T> workerPool = new WorkerPool<>(ringBuffer, sequenceBarrier, exceptionHandler, workHandlers);

        // 将 workerPool (共享消费者池)存储
        consumerRepository.add(workerPool, sequenceBarrier);

        final Sequence[] workerSequences = workerPool.getWorkerSequences();

        updateGatingSequencesForNextInChain(barrierSequences, workerSequences);

        return new EventHandlerGroup<>(this, consumerRepository, workerSequences);
    }
    
    ```

    ```java
     /**
     * 更新链中下一个的门控序列
     * @param barrierSequences
     * @param processorSequences
     */
    private void updateGatingSequencesForNextInChain(final Sequence[] barrierSequences, final Sequence[] processorSequences)
    {
        if (processorSequences.length > 0)
        {
            // 将消费者的 sequences 传给 ringBuffer
            ringBuffer.addGatingSequences(processorSequences);
            for (final Sequence barrierSequence : barrierSequences)
            {
                ringBuffer.removeGatingSequence(barrierSequence);
            }
            consumerRepository.unMarkEventProcessorsAsEndOfChain(barrierSequences);
        }
    }

    ```

4. 启动消费者

    - 启动方法:

    ```java

    /**
     * <p>Starts the event processors and returns the fully configured ring buffer.</p>
     *
     * <p>The ring buffer is set up to prevent overwriting any entry that is yet to
     * be processed by the slowest event processor.</p>
     *
     * <p>This method must only be called once after all event processors have been added.</p>
     *
     * @return the configured ring buffer.
     */
    public RingBuffer<T> start()
    {   
        // 检查是否只启动了一次
        checkOnlyStartedOnce();
        for (final ConsumerInfo consumerInfo : consumerRepository)
        { 
            // 根据 consumerRepository.add() 方法中存储的类型 batchEventProcessor/workerPool 进行执行
            // 使用独立线程执行
            consumerInfo.start(executor);
        }

        return ringBuffer;
    }

    ```

    - BatchEventProcessor

    ```java

    public void run()
    {
        if (running.compareAndSet(IDLE, RUNNING))
        {
            sequenceBarrier.clearAlert();

            notifyStart();
            try
            {
                if (running.get() == RUNNING)
                {
                    processEvents();
                }
            }
            finally
            {
                notifyShutdown();
                running.set(IDLE);
            }
        }
        else
        {
            // This is a little bit of guess work.  The running state could of changed to HALTED by
            // this point.  However, Java does not have compareAndExchange which is the only way
            // to get it exactly correct.
            if (running.get() == RUNNING)
            {
                throw new IllegalStateException("Thread is already running");
            }
            else
            {
                earlyExit();
            }
        }
    }
   
    private void processEvents()
    {
        T event = null;
        long nextSequence = sequence.get() + 1L;

        while (true)
        {
            try
            {
                final long availableSequence = sequenceBarrier.waitFor(nextSequence);
                if (batchStartAware != null)
                {
                    batchStartAware.onBatchStart(availableSequence - nextSequence + 1);
                }

                while (nextSequence <= availableSequence)
                {
                    event = dataProvider.get(nextSequence);
                    eventHandler.onEvent(event, nextSequence, nextSequence == availableSequence);
                    nextSequence++;
                }

                sequence.set(availableSequence);
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (running.get() != RUNNING)
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                handleEventException(ex, nextSequence, event);
                sequence.set(nextSequence);
                nextSequence++;
            }
        }
    }

    ```

    - WorkProcessor

    ```java

    public void run()
    {
        if (!running.compareAndSet(false, true))
        {
            throw new IllegalStateException("Thread is already running");
        }
        sequenceBarrier.clearAlert();

        notifyStart();

        boolean processedSequence = true;
        long cachedAvailableSequence = Long.MIN_VALUE;
        long nextSequence = sequence.get();
        T event = null;
        while (true)
        {
            try
            {
                // if previous sequence was processed - fetch the next sequence and set
                // that we have successfully processed the previous sequence
                // typically, this will be true
                // this prevents the sequence getting too far forward if an exception
                // is thrown from the WorkHandler
                if (processedSequence)
                {
                    processedSequence = false;
                    do
                    {
                        nextSequence = workSequence.get() + 1L;
                        sequence.set(nextSequence - 1L);
                    }
                    while (!workSequence.compareAndSet(nextSequence - 1L, nextSequence));
                }

                if (cachedAvailableSequence >= nextSequence)
                {
                    event = ringBuffer.get(nextSequence);
                    workHandler.onEvent(event);
                    processedSequence = true;
                }
                else
                {
                    cachedAvailableSequence = sequenceBarrier.waitFor(nextSequence);
                }
            }
            catch (final TimeoutException e)
            {
                notifyTimeout(sequence.get());
            }
            catch (final AlertException ex)
            {
                if (!running.get())
                {
                    break;
                }
            }
            catch (final Throwable ex)
            {
                // handle, mark as processed, unless the exception handler threw an exception
                exceptionHandler.handleEventException(ex, nextSequence, event);
                processedSequence = true;
            }
        }

        notifyShutdown();

        running.set(false);
    }

    ```


