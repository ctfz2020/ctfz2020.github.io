---
title:  Druid向量化查询
tags:
  - Druid
  - SIMD
---

最近在了解clickhouse, 有文档介绍列式存储的数据支持向量查询, 引起了我的兴趣, 因为druid也是列式存储的. 

查看Druid官网, 发现Druid在0.16版本中开始引入**向量化**查询模型(query vectorization), 官方说法是减少方法调用,并潜在的触发SIMD优化 ([issue7093](https://github.com/apache/druid/issues/7093)).

对Druid进行查询测试, 开启向量化查询确实会降低耗时. 以5000w条数据为例, 开启和关闭向量查询下, 耗时随聚合字段个数变化趋势如下图

<!--more-->

![image-20200804161117516](/assets/vector-1.png)

### 分析

根据图表可以看到, 开启了向量化查询的平均耗时减少了一半. 查看向量化查询有关源码, 发现有两个关键地方

`AggregatorAdapters`负责聚合分组查询时数据桶的各个指标字段聚合

```java
 /**
   * Call {@link VectorAggregator#aggregate(ByteBuffer, int, int[], int[], int)} on all of our aggregators.
   *
   * This method is only valid if the underlying aggregators are {@link VectorAggregator}.
   */
  public void aggregateVector(
      final ByteBuffer buf,
      final int numRows,
      final int[] positions,
      @Nullable final int[] rows
  )
  {
    for (int i = 0; i < adapters.size(); i++) {
      final Adapter adapter = adapters.get(i);
      adapter.asVectorAggregator().aggregate(buf, numRows, positions, rows, aggregatorPositions[i]);
    }
  }
```

`LongSumVectorAggregator`负责单个指标数据的计算

```java
/**
 * 当维度为空时调用该方法
 */
@Override
public void aggregate(final ByteBuffer buf, final int position, final int startRow, final int endRow)
{
    final long[] vector = selector.getLongVector();

    long sum = 0;
    for (int i = startRow; i < endRow; i++) {
      sum += vector[i];
    }

    buf.putLong(position, buf.getLong(position) + sum);
}

/**
 * 当维度不为空时调用
 */
public void aggregate(
      final ByteBuffer buf,
      final int numRows,
      final int[] positions,
      @Nullable final int[] rows,
      final int positionOffset
  )
  {
    final long[] vector = selector.getLongVector();

    for (int i = 0; i < numRows; i++) {
      final int position = positions[i] + positionOffset;
      buf.putLong(position, buf.getLong(position) + vector[rows != null ? rows[i] : i]);
    }
  }
```

普通的查询逻辑是遍历每行数据, 将每行数据中所有的指标字段都聚合一次, 改之后的向量化查询逻辑是每个字段获取一批数据, 按照指标字段依次聚合. 代码中的`long[] vector`数组就是取的一批数据.

举个例子: 现在要聚合数据的第2~7行数据总共6条数据.

![img](/assets/vector-2.png)

不开启向量化查询`AggregatorAdapters`会执行6次, 开启后只会执行2次.

那么这段代码真的使用了SMID指令吗? 通过`XX:CompileCommand`虚拟机参数可以看到java生成的汇编代码, 在历史节点虚拟机启动参数中加入`-XX:CompileCommand=print,*LongSumVectorAggregator.aggregate*`开启汇编代码打印功能.

附录是汇编代码. 里面并没有找到`vmovdqu`,`vpaddd`等SMID指令, 说明这个方法并没有优化.

下面看一个微基准测试(之前这段代码没加上, 已经补充!)

```java
import java.util.concurrent.TimeUnit;

import org.openjdk.jmh.annotations.Benchmark;
import org.openjdk.jmh.annotations.BenchmarkMode;
import org.openjdk.jmh.annotations.Fork;
import org.openjdk.jmh.annotations.Measurement;
import org.openjdk.jmh.annotations.Mode;
import org.openjdk.jmh.annotations.OutputTimeUnit;
import org.openjdk.jmh.annotations.Scope;
import org.openjdk.jmh.annotations.Setup;
import org.openjdk.jmh.annotations.State;
import org.openjdk.jmh.annotations.Threads;
import org.openjdk.jmh.annotations.Warmup;
import org.openjdk.jmh.infra.Blackhole;
import org.openjdk.jmh.runner.Runner;
import org.openjdk.jmh.runner.RunnerException;
import org.openjdk.jmh.runner.options.Options;
import org.openjdk.jmh.runner.options.OptionsBuilder;

@BenchmarkMode(Mode.AverageTime)
@OutputTimeUnit(TimeUnit.NANOSECONDS)
@Warmup(iterations = 3)
@Threads(1)
@State(Scope.Thread)
@Fork(value = 1, jvmArgs= {"-XX:+UnlockDiagnosticVMOptions", "-XX:+UseSuperWord", "-XX:CompileCommand=print,*VectorMultipleColumnTest.testVectorSum*"})
@Measurement(iterations = 5, batchSize = -1, time = 10, timeUnit = TimeUnit.SECONDS)
public class VectorMultipleColumnTest {
	final int length = 512;
	long[] columnA;
	long[] columnB;
	
	@Setup
	public void start() {
		columnA = VectorTest.createLongArray(length);
		columnB = VectorTest.createLongArray(length);
	}
	
	/**
	 * 利用变量tmpSum存放中间结果,依次累加A和B数组, 真实情况下每个字段都需要一个临时变量, 这里为了方便使用同一个
	 * @param hole
	 */
	@Benchmark
	public void testTowLoopSum(Blackhole hole) {
		hole.consume(simpleLoopSum(columnA));
		hole.consume(simpleLoopSum(columnB));
	}
	
	//存放中间结果
	long tmpSum = 0l;
	public long simpleLoopSum(long[] columns) {
		for(int i = 0; i < columns.length; i++) {
			tmpSum += columns[i];
		}
		return tmpSum;
	}
	
	/**
	 * 利用数组tmpResult存放中间结果, 依次和数组A和数组B向量化计算,真实情况下应该是每个字段生成一个中间数组, 这里为了方便使用同一个
	 * @param hole
	 */
	@Benchmark
	public void testVectorSum(Blackhole hole) {
		hole.consume(vectorSum(columnA));
		hole.consume(vectorSum(columnB));
	}
	
	//存放中间结果
	long[] tmpResult = new long[512];
	public long[] vectorSum(long[] columns) {
		add(tmpResult, columns, 0, columns.length);
		return tmpResult;
	}
    
	private static void add(long[] tmp, long[] src, int from, int end) {
		int offset = end - from;
		for(int i = 0; i < offset; i++) {
			tmp[i] += src[i];
		}
	}
	
	public static void main(String[] args) {
		Options opt = new OptionsBuilder()
				.include(VectorMultipleColumnTest.class.getSimpleName())
				.build();
		try {
			new Runner(opt).run();
		} catch (RunnerException e) {
			e.printStackTrace();
		}
	}
}


```

这段代码中`columnA`和`columnB`都是固定长度的数组, 模拟druid代码中的`long[] vector`,  测试方法有两个, 第一个`testTowLoopSum`方法模拟druid代码中的方式, 另一个方法`testVectorSum`借助临时数组完成向量化计算, 以触发SIMD优化(详情可看[JAVA向量化计算](vector.html))

执行结果如下:

![image-20200806182509804](/assets/vector-3.png)

```shell
Benchmark                                Mode  Cnt    Score    Error  Units
VectorMultipleColumnTest.testTowLoopSum  avgt    5  339.785 ± 14.927  ns/op
VectorMultipleColumnTest.testVectorSum   avgt    5  157.644 ±  1.520  ns/op
```

微基准测试结果是很振奋人心的, 向量化求和比普通累加处理性能提高一倍以上. 当然, 对于数据库代码, 性能提升不是像测试代码这么简单修改的事情,  但在特定的情况下,还是存在优化的可能性.

### 结论

Druid的向量化查询比不开启向量化查询降低了耗时,优化幅度很大. 因为并没有完整的测试Druid中的代码,所以性能提升的原因可能归咎于减少了方法调用次数.

但说明了没有使用到SMID指令, 存在继续优化的空间.

### 附录:

查询语句,字段名称为`long_metric_0`~`long_metric_9`

```json
{
    "intervals": ["2020-07-29/2020-07-30"],
    "virtualColumns": [],
    "granularity": "all",
    "context": {
        "sortByDimsFirst": true,
        "vectorize": "false",
        "vectorSize": 512
    },
    "dataSource": "foo",
    "aggregations": [{
        "fieldName": "long_metric_0",
        "name": "longSum0",
        "type": "longSum"
    }],
    "postAggregations": [],
    "queryType": "groupBy",
    "dimensions": []
}
```

查询耗时表格

| 聚合字段个数 | 开启向量化查询耗时(ms) | 关闭向量化查询耗时(ms) |
| ------------ | ---------------------- | ---------------------- |
| 1            | 771                    | 1951                   |
| 2            | 1547                   | 3155                   |
| 3            | 2252                   | 4204                   |
| 4            | 2711                   | 5267                   |
| 5            | 3386                   | 6311                   |
| 6            | 3966                   | 7934                   |
| 7            | 4437                   | 8544                   |
| 8            | 4715                   | 10864                  |
| 9            | 5115                   | 11255                  |
| 10           | 5382                   | 12681                  |

汇编代码

```shell
0x00007f0bb5f04655: movabs  $0xffffffffffffffff,%rax
  0x00007f0bb5f0465f: callq   0x7f0bb5045f60    ; OopMap{[88]=Oop [80]=Oop [72]=Oop off=292}
                                                ;*invokeinterface getLongVector
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@4 (line 64)
                                                ;   {virtual_call}
  0x00007f0bb5f04664: movabs  $0x7f0b7ea50ff8,%rsi  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f0466e: incl    0x138(%rsi)
  0x00007f0bb5f04674: mov     0x40(%rsp),%edi
  0x00007f0bb5f04678: mov     0x48(%rsp),%r9
  0x00007f0bb5f0467d: mov     0x50(%rsp),%r8
  0x00007f0bb5f04682: mov     0x44(%rsp),%ecx
  0x00007f0bb5f04686: mov     0x58(%rsp),%rdx
  0x00007f0bb5f0468b: mov     $0x0,%esi         ;*goto
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@14 (line 66)

  0x00007f0bb5f04690: movabs  $0x7f0b7ea50ff8,%rbx  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f0469a: mov     0xe0(%rbx),%r11d
  0x00007f0bb5f046a1: add     $0x8,%r11d
  0x00007f0bb5f046a5: mov     %r11d,0xe0(%rbx)
  0x00007f0bb5f046ac: movabs  $0x7f0b7ea46150,%rbx  ;   {metadata({method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f046b6: and     $0xfff8,%r11d
  0x00007f0bb5f046bd: cmp     $0x0,%r11d
  0x00007f0bb5f046c1: je      0x7f0bb5f04a77    ; OopMap{rax=Oop r9=Oop r8=Oop rdx=Oop off=391}
                                                ;*if_icmplt
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@64 (line 66)

  0x00007f0bb5f046c7: test    %eax,0x3c9b6a33(%rip)  ;   {poll}
  0x00007f0bb5f046cd: cmp     %ecx,%esi
  0x00007f0bb5f046cf: movabs  $0x7f0b7ea50ff8,%rbx  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f046d9: movabs  $0x1e8,%r11
  0x00007f0bb5f046e3: jl      0x7f0bb5f046f3
  0x00007f0bb5f046e9: movabs  $0x1f8,%r11
  0x00007f0bb5f046f3: mov     (%rbx,%r11),%r13
  0x00007f0bb5f046f7: lea     0x1(%r13),%r13
  0x00007f0bb5f046fb: mov     %r13,(%rbx,%r11)
  0x00007f0bb5f046ff: jnl     0x7f0bb5f049b0    ;*if_icmplt
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@64 (line 66)

  0x00007f0bb5f04705: mov     %rax,0x98(%rsp)
  0x00007f0bb5f0470d: mov     %r9,0x90(%rsp)
  0x00007f0bb5f04715: mov     %ecx,0xb8(%rsp)
  0x00007f0bb5f0471c: movsxd  %esi,%rbx
  0x00007f0bb5f0471f: cmp     0xc(%r8),%esi     ; implicit exception: dispatches to 0x00007f0bb5f04a8e
  0x00007f0bb5f04723: jnb     0x7f0bb5f04a98
  0x00007f0bb5f04729: mov     0x10(%r8,%rbx,4),%ebx  ;*iaload
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@20 (line 67)

  0x00007f0bb5f0472e: add     %edi,%ebx
  0x00007f0bb5f04730: cmp     (%rdx),%rax       ; implicit exception: dispatches to 0x00007f0bb5f04aa1
  0x00007f0bb5f04733: mov     %rdx,%r11
  0x00007f0bb5f04736: movabs  $0x7f0b7ea50ff8,%r13  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f04740: mov     0x8(%r11),%r11d
  0x00007f0bb5f04744: shl     $0x3,%r11
  0x00007f0bb5f04748: cmp     0x158(%r13),%r11
  0x00007f0bb5f0474f: jne     0x7f0bb5f0475e
  0x00007f0bb5f04751: addq    $0x1,0x160(%r13)
  0x00007f0bb5f04759: jmpq    0x7f0bb5f047c4
  0x00007f0bb5f0475e: cmp     0x168(%r13),%r11
  0x00007f0bb5f04765: jne     0x7f0bb5f04774
  0x00007f0bb5f04767: addq    $0x1,0x170(%r13)
  0x00007f0bb5f0476f: jmpq    0x7f0bb5f047c4
  0x00007f0bb5f04774: cmpq    $0x0,0x158(%r13)
  0x00007f0bb5f0477f: jne     0x7f0bb5f04798
  0x00007f0bb5f04781: mov     %r11,0x158(%r13)
  0x00007f0bb5f04788: movq    $0x1,0x160(%r13)
  0x00007f0bb5f04793: jmpq    0x7f0bb5f047c4
  0x00007f0bb5f04798: cmpq    $0x0,0x168(%r13)
  0x00007f0bb5f047a3: jne     0x7f0bb5f047bc
  0x00007f0bb5f047a5: mov     %r11,0x168(%r13)
  0x00007f0bb5f047ac: movq    $0x1,0x170(%r13)
  0x00007f0bb5f047b7: jmpq    0x7f0bb5f047c4
  0x00007f0bb5f047bc: addq    $0x1,0x150(%r13)
  0x00007f0bb5f047c4: mov     %rdx,%r11
  0x00007f0bb5f047c7: mov     %rbx,%rdx
  0x00007f0bb5f047ca: mov     %esi,0x84(%rsp)
  0x00007f0bb5f047d1: mov     %r11,%rsi         ;*invokevirtual getLong
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@32 (line 68)

  0x00007f0bb5f047d4: mov     %r8,0xb0(%rsp)
  0x00007f0bb5f047dc: mov     %edi,0xac(%rsp)
  0x00007f0bb5f047e3: mov     %ebx,0xa8(%rsp)
  0x00007f0bb5f047ea: mov     %r11,0xa0(%rsp)
  0x00007f0bb5f047f2: nop
  0x00007f0bb5f047f3: nop
  0x00007f0bb5f047f4: nop
  0x00007f0bb5f047f5: movabs  $0xffffffffffffffff,%rax
  0x00007f0bb5f047ff: callq   0x7f0bb5045f60    ; OopMap{[160]=Oop [152]=Oop [144]=Oop [176]=Oop off=708}
                                                ;*invokevirtual getLong
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@32 (line 68)
                                                ;   {virtual_call}
  0x00007f0bb5f04804: mov     0x90(%rsp),%r9
  0x00007f0bb5f0480c: cmp     $0x0,%r9
  0x00007f0bb5f04810: movabs  $0x7f0b7ea50ff8,%rdx  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f0481a: movabs  $0x180,%rcx
  0x00007f0bb5f04824: je      0x7f0bb5f04834
  0x00007f0bb5f0482a: movabs  $0x190,%rcx
  0x00007f0bb5f04834: mov     (%rdx,%rcx),%rsi
  0x00007f0bb5f04838: lea     0x1(%rsi),%rsi
  0x00007f0bb5f0483c: mov     %rsi,(%rdx,%rcx)
  0x00007f0bb5f04840: mov     0x84(%rsp),%esi
  0x00007f0bb5f04847: je      0x7f0bb5f04874    ;*ifnull
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@39 (line 68)

  0x00007f0bb5f0484d: movsxd  %esi,%rdx
  0x00007f0bb5f04850: cmp     0xc(%r9),%esi     ; implicit exception: dispatches to 0x00007f0bb5f04aa6
  0x00007f0bb5f04854: jnb     0x7f0bb5f04ab0
  0x00007f0bb5f0485a: mov     0x10(%r9,%rdx,4),%edx  ;*iaload
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@46 (line 68)

  0x00007f0bb5f0485f: movabs  $0x7f0b7ea50ff8,%rcx  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f04869: incl    0x1a0(%rcx)
  0x00007f0bb5f0486f: jmpq    0x7f0bb5f04877    ;*goto
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@47 (line 68)

  0x00007f0bb5f04874: mov     %rsi,%rdx
  0x00007f0bb5f04877: mov     0xa8(%rsp),%ecx
  0x00007f0bb5f0487e: mov     %esi,0x84(%rsp)
  0x00007f0bb5f04885: mov     0x98(%rsp),%rdi
  0x00007f0bb5f0488d: mov     %r9,0x90(%rsp)
  0x00007f0bb5f04895: mov     0xa0(%rsp),%rbx
  0x00007f0bb5f0489d: movsxd  %edx,%r14
  0x00007f0bb5f048a0: cmp     0xc(%rdi),%edx    ; implicit exception: dispatches to 0x00007f0bb5f04ab9
  0x00007f0bb5f048a3: jnb     0x7f0bb5f04ac3
  0x00007f0bb5f048a9: mov     0x10(%rdi,%r14,8),%rdx  ;*laload
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@52 (line 68)

  0x00007f0bb5f048ae: mov     %rbx,%r14
  0x00007f0bb5f048b1: mov     %rdi,0x98(%rsp)
  0x00007f0bb5f048b9: movabs  $0x7f0b7ea50ff8,%rdi  ;   {metadata(method data for {method} {0x00007f0b7ea46150} 'aggregate' '(Ljava/nio/ByteBuffer;I[I[II)V' in 'org/apache/druid/query/aggregation/LongSumVectorAggregator')}
  0x00007f0bb5f048c3: mov     0x8(%r14),%r14d
  0x00007f0bb5f048c7: shl     $0x3,%r14
  0x00007f0bb5f048cb: cmp     0x1c0(%rdi),%r14
  0x00007f0bb5f048d2: jne     0x7f0bb5f048e1
  0x00007f0bb5f048d4: addq    $0x1,0x1c8(%rdi)
  0x00007f0bb5f048dc: jmpq    0x7f0bb5f04947
  0x00007f0bb5f048e1: cmp     0x1d0(%rdi),%r14
  0x00007f0bb5f048e8: jne     0x7f0bb5f048f7
  0x00007f0bb5f048ea: addq    $0x1,0x1d8(%rdi)
  0x00007f0bb5f048f2: jmpq    0x7f0bb5f04947
  0x00007f0bb5f048f7: cmpq    $0x0,0x1c0(%rdi)
  0x00007f0bb5f04902: jne     0x7f0bb5f0491b
  0x00007f0bb5f04904: mov     %r14,0x1c0(%rdi)
  0x00007f0bb5f0490b: movq    $0x1,0x1c8(%rdi)
  0x00007f0bb5f04916: jmpq    0x7f0bb5f04947
  0x00007f0bb5f0491b: cmpq    $0x0,0x1d0(%rdi)
  0x00007f0bb5f04926: jne     0x7f0bb5f0493f
  0x00007f0bb5f04928: mov     %r14,0x1d0(%rdi)
  0x00007f0bb5f0492f: movq    $0x1,0x1d8(%rdi)
  0x00007f0bb5f0493a: jmpq    0x7f0bb5f04947
  0x00007f0bb5f0493f: addq    $0x1,0x1b8(%rdi)
  0x00007f0bb5f04947: mov     %rdx,%rdi
  0x00007f0bb5f0494a: add     %rax,%rdi
  0x00007f0bb5f0494d: mov     %rcx,%rdx
  0x00007f0bb5f04950: mov     %rdi,%rcx
  0x00007f0bb5f04953: mov     %rbx,%rsi         ;*invokevirtual putLong
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@54 (line 68)

  0x00007f0bb5f04956: mov     %rbx,0xa0(%rsp)
  0x00007f0bb5f0495e: nop
  0x00007f0bb5f0495f: nop
  0x00007f0bb5f04960: nop
  0x00007f0bb5f04961: nop
  0x00007f0bb5f04962: nop
  0x00007f0bb5f04963: nop
  0x00007f0bb5f04964: nop
  0x00007f0bb5f04965: movabs  $0xffffffffffffffff,%rax
  0x00007f0bb5f0496f: callq   0x7f0bb5045f60    ; OopMap{[176]=Oop [144]=Oop [160]=Oop [152]=Oop off=1076}
                                                ;*invokevirtual putLong
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@54 (line 68)
                                                ;   {virtual_call}
  0x00007f0bb5f04974: mov     0x84(%rsp),%esi
  0x00007f0bb5f0497b: incl    %esi
  0x00007f0bb5f0497d: mov     0x98(%rsp),%rax
  0x00007f0bb5f04985: mov     0xac(%rsp),%edi
  0x00007f0bb5f0498c: mov     0x90(%rsp),%r9
  0x00007f0bb5f04994: mov     0xb0(%rsp),%r8
  0x00007f0bb5f0499c: mov     0xb8(%rsp),%ecx
  0x00007f0bb5f049a3: mov     0xa0(%rsp),%rdx
  0x00007f0bb5f049ab: jmpq    0x7f0bb5f04690    ;*iload
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@61 (line 66)

  0x00007f0bb5f049b0: add     $0xd0,%rsp
  0x00007f0bb5f049b7: pop     %rbp
  0x00007f0bb5f049b8: test    %eax,0x3c9b6742(%rip)  ;   {poll_return}
  0x00007f0bb5f049be: retq                      ;*return
                                                ; - org.apache.druid.query.aggregation.LongSumVectorAggregator::aggregate@67 (line 70)

  0x00007f0bb5f049bf: mov     %eax,0xfffffffffffec000(%rsp)
  0x00007f0bb5f049c6: push    %rbp
  0x00007f0bb5f049c7: sub     $0xd0,%rsp
```

