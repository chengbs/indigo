---
title: "各种Java序列化性能比较"
layout: post
date: 2017-08-10 22:44
image: /assets/images/markdown.jpg
headerImage: false
tag:
- java
star: true
category: blog
author: sun
description: Markdown summary with different options
---

## 各种Java序列化性能比较

这里比较Java对象序列化 XML JSON Kryo POF等序列化性能比较。

很多人以为JDK的Java序列化肯定是将Java对象转换成二进制序列化最快的方式，JDK7出来以后，我们发现实际上每次新的JDK比旧版本快。

我们通常以为将Java对象序列化成二进制比序列化成XML或Json更快，其实是错误的，如果你关心性能，建议避免Java序列化。

Java序列化有很多的要求，最主要的一个是包含能够序列化任何东西（或至少任何实现Serializable接口）。这样才能进入其他JVM之中，这很重要，所以有时性能不是主要的要求，标准的格式才最重要。

我们经常看到CPU花费很多时间内进行Java序列化，下面我们研究一下，假设一定Order，虽然只有几个字节，但是序列化以后不是几十个字节，而是600多个字节：

Ordr代码：

{% highlight java %}
public class Order implements Serializable {
    private long id;
    private String description;
    private BigDecimal totalCost = BigDecimal.valueOf(0);
    private List orderLines = new ArrayList();
    private Customer customer;
}
{% endhighlight %}

序列化输出：

----sr--model.Order----h#-----J--idL--customert--Lmodel/Customer;L--descriptiont--Ljava/lang/String;L--orderLinest--Ljava/util/List;L--totalCostt--Ljava/math/BigDecimal;xp--------ppsr--java.util.ArrayListx-----a----I--sizexp----w-----sr--model.OrderLine--&-1-S----I--lineNumberL--costq-~--L--descriptionq-~--L--ordert--Lmodel/Order;xp----sr--java.math.BigDecimalT--W--(O---I--scaleL--intValt--Ljava/math/BigInteger;xr--java.lang.Number-----------xp----sr--java.math.BigInteger-----;-----I--bitCountI--bitLengthI--firstNonzeroByteNumI--lowestSetBitI--signum[--magnitudet--[Bxq-~----------------------ur--[B------T----xp----xxpq-~--xq-~--

正如你可能已经注意到，Java序列化写入不仅是完整的类名，也包含整个类的定义，包含所有被引用的类。类定义可以是相当大的，也许构成了性能和效率的问题，当然这是编写一个单一的对象。如果您正在编写了大量相同的类的对象，这时类定义的开销通常不是一个大问题。另一件事情是，如果你的对象有一类的引用（如元数据对象），那么Java序列化将写入整个类的定义，不只是类的名称，因此，使用Java序列化写出元数据（meta-data）是非常昂贵的。
 

## Externalizable

通过实现Externalizable接口，这是可能优化Java序列化的。实现此接口，避免写出整个类定义，只是类名被写入。它需要你实施readExternal和writeExternal方法方法的，所以需要做一些工作，但相比仅仅是实现Serializable更快，更高效。

Externalizable对小数目对象有效的多。但是对大量对象，或者重复对象，则效率低。

{% highlight java %}
public class Order implements Externalizable {
    private long id;
    private String description;
    private BigDecimal totalCost = BigDecimal.valueOf(0);
    private List orderLines = new ArrayList();
    private Customer customer;

    public Order() {
    }

    public void readExternal(ObjectInput stream) throws IOException, ClassNotFoundException {
        this.id = stream.readLong();
        this.description = (String)stream.readObject();
        this.totalCost = (BigDecimal)stream.readObject();
        this.customer = (Customer)stream.readObject();
        this.orderLines = (List)stream.readObject();
    }

    public void writeExternal(ObjectOutput stream) throws IOException {
        stream.writeLong(this.id);
        stream.writeObject(this.description);
        stream.writeObject(this.totalCost);
        stream.writeObject(this.customer);
        stream.writeObject(this.orderLines);
    }
}
{% endhighlight %}

序列化输出：

----sr--model.Order---*3--^---xpw---------psr--java.math.BigDecimalT--W--(O---I--scaleL--intValt--Ljava/math/BigInteger;xr--java.lang.Number-----------xp----sr--java.math.BigInteger-----;-----I--bitCountI--bitLengthI--firstNonzeroByteNumI--lowestSetBitI--signum[--magnitudet--[Bxq-~----------------------ur--[B------T----xp----xxpsr--java.util.ArrayListx-----a----I--sizexp----w-----sr--model.OrderLine-!!|---S---xpw-----pq-~--q-~--xxx
 

## EclipseLink MOXy - XML 和 JSON

序列化成XML或JSON可以允许其他语言访问，可以实现REST服务等。缺点是文本格式的效率比优化的二进制格式低一些，使用JAXB,你需要使用JAXB注释类，或提供一个XML配置文件。使用@XmlIDREF处理循环。

{% highlight java %}
@XmlRootElement
public class Order {
    @XmlID
    @XmlAttribute
    private long id;
    @XmlAttribute
    private String description;
    @XmlAttribute
    private BigDecimal totalCost = BigDecimal.valueOf(0);
    private List orderLines = new ArrayList();
    private Customer customer;
}

public class OrderLine {
    @XmlIDREF
    private Order order;
    @XmlAttribute
    private int lineNumber;
    @XmlAttribute
    private String description;
    @XmlAttribute
    private BigDecimal cost = BigDecimal.valueOf(0);
}
{% endhighlight %}

XML输出：

<order id="0" totalCost="0">
<orderLines lineNumber="1" cost="0">
<order>0</order
></orderLines
></order>

JSOn输出：

{"order":{"id":0,"totalCost":0,"orderLines":[{"lineNumber":1,"cost":0,"order":0}]}}

 

## Kryo

Kryo 是一种快速，高效的序列化的Java框架。 KRYO是新的BSD许可下一个开源项目提供。这是一个很小的项目，只有3名成员，它首先在2009年出品。

工作原理类似于Java序列化KRYO，尊重瞬态字段，但不要求一类是可序列化的。KRYO有一定的局限性，比如需要有一个默认的构造函数的类，在序列化将java.sql.Time java.sql.Date java.sql.Timestamp类会遇到一些问题。

order序列化结果：

------java-util-ArrayLis-----model-OrderLin----java-math-BigDecima---------model-Orde-----

## Oracle Coherence POF

Oracle Coherence 产品提供其自己优化的二进制格式，称为POF （可移植对象格式） 。 Oracle Coherence的是一个内存中的数据网格解决方案（分布式缓存） 。是一个商业产品，并需要许可证。

POF提供了一个序列化框架，并可以独立使用。 POF要求类实现一个PortableObject接口和读/写方法。您还可以实现一个单独的序列化类，或使用最新版本的序列化的注解。 POF要求每个类都被提前分配一个固定ID，所以你需要通过某种方式确定这个ID 。 POF格式是二进制格式，非常紧凑，高效，快速的，但确实需要你付出一些工作。

POF的总字节数为一个单一的订单/订单行对象为32个字节， 1593字节100 OrderLines的。我不会放弃的结果， POF是一个商业许可产品的一部分，但是是非常快的。

{% highlight java %}
public class Order implements PortableObject {
    private long id;
    private String description;
    private BigDecimal totalCost = BigDecimal.valueOf(0);
    private List orderLines = new ArrayList();
    private Customer customer;

    public Order() {
    }

    public void readExternal(PofReader in) throws IOException {
        this.id = in.readLong(0);
        this.description = in.readString(1);
        this.totalCost = in.readBigDecimal(2);
        this.customer = (Customer)in.readObject(3);
        this.orderLines = (List)in.readCollection(4, new ArrayList());
    }

    public void writeExternal(PofWriter out) throws IOException {
        out.writeLong(0, this.id);
        out.writeString(1, this.description);
        out.writeBigDecimal(2, this.totalCost);
        out.writeObject(3, this.customer);
        out.writeCollection(4, this.orderLines);
    }
}
{% endhighlight raw %}

序列化结果：

-----B--G---d-U------A--G-------


## 性能比较

一个订单包含一个Oderline
Serializer	Size (bytes)	Serialize (operations/second)	Deserialize (operations/second)	% Difference (from Java serialize)	% Difference (deserialize)
Java Serializable	636	128,634	19,180	0%	0%
Java Externalizable	435	160,549	26,678	24%	39%
EclipseLink MOXy XML	101	348,056	47,334	170%	146%
Kryo	90	359,368	346,984	179%	1709%
一个订单100个oderlines:


Serializer	Size (bytes)	Serialize (operations/second)	Deserialize (operations/second)	% Difference (from Java serialize)	% Difference (deserialize)
Java Serializable	2,715	16,470	10,215	0%	0%
Java Externalizable	2,811	16,206	11,483	-1%	12%
EclipseLink MOXy XML	6,628	7,304	2,731	-55%	-73%
Kryo	1216	22,862	31,499	38%	208%
 

要获得象C那样的序列化性能，直接自己编写。

## Serialization ByteBuffer Unsafe三者性能比较:

三者性能测试代码：

{% highlight raw %}
import sun.misc.Unsafe;
import java.io.ByteArrayInputStream;
import java.io.ByteArrayOutputStream;
import java.io.ObjectInputStream;
import java.io.ObjectOutputStream;
import java.io.Serializable;
import java.lang.reflect.Field;
import java.nio.ByteBuffer;
import java.util.Arrays;

public final class TestSerialisationPerf {
	public static final int REPETITIONS = 1 * 1000 * 1000;

	private static ObjectToBeSerialised ITEM = new ObjectToBeSerialised(1010L, true, 777, 99,
			new double[] { 0.1, 0.2, 0.3, 0.4, 0.5, 0.6, 0.7, 0.8, 0.9, 1.0 },
			new long[] { 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 });

	public static void main(final String[] arg) throws Exception {
		for (final PerformanceTestCase testCase : testCases) {
			for (int i = 0; i < 5; i++) {
				testCase.performTest();

				System.out.format("%d %s\twrite=%,dns read=%,dns total=%,dns\n", i, testCase.getName(),
						testCase.getWriteTimeNanos(), testCase.getReadTimeNanos(),
						testCase.getWriteTimeNanos() + testCase.getReadTimeNanos());

				if (!ITEM.equals(testCase.getTestOutput())) {
					throw new IllegalStateException("Objects do not match");
				}

				System.gc();
				Thread.sleep(3000);
			}
		}
	}

	private static final PerformanceTestCase[] testCases = {
			new PerformanceTestCase("Serialisation", REPETITIONS, ITEM) {
				ByteArrayOutputStream baos = new ByteArrayOutputStream();

				public void testWrite(ObjectToBeSerialised item) throws Exception {
					for (int i = 0; i < REPETITIONS; i++) {
						baos.reset();

						ObjectOutputStream oos = new ObjectOutputStream(baos);
						oos.writeObject(item);
						oos.close();
					}
				}

				public ObjectToBeSerialised testRead() throws Exception {
					ObjectToBeSerialised object = null;
					for (int i = 0; i < REPETITIONS; i++) {
						ByteArrayInputStream bais = new ByteArrayInputStream(baos.toByteArray());
						ObjectInputStream ois = new ObjectInputStream(bais);
						object = (ObjectToBeSerialised) ois.readObject();
					}

					return object;
				}
			},

			new PerformanceTestCase("ByteBuffer", REPETITIONS, ITEM) {
				ByteBuffer byteBuffer = ByteBuffer.allocate(1024);

				public void testWrite(ObjectToBeSerialised item) throws Exception {
					for (int i = 0; i < REPETITIONS; i++) {
						byteBuffer.clear();
						item.write(byteBuffer);
					}
				}

				public ObjectToBeSerialised testRead() throws Exception {
					ObjectToBeSerialised object = null;
					for (int i = 0; i < REPETITIONS; i++) {
						byteBuffer.flip();
						object = ObjectToBeSerialised.read(byteBuffer);
					}

					return object;
				}
			},

			new PerformanceTestCase("UnsafeMemory", REPETITIONS, ITEM) {
				UnsafeMemory buffer = new UnsafeMemory(new byte[1024]);

				public void testWrite(ObjectToBeSerialised item) throws Exception {
					for (int i = 0; i < REPETITIONS; i++) {
						buffer.reset();
						item.write(buffer);
					}
				}

				public ObjectToBeSerialised testRead() throws Exception {
					ObjectToBeSerialised object = null;
					for (int i = 0; i < REPETITIONS; i++) {
						buffer.reset();
						object = ObjectToBeSerialised.read(buffer);
					}

					return object;
				}
			}, };
}

abstract class PerformanceTestCase {
	private final String name;
	private final int repetitions;
	private final ObjectToBeSerialised testInput;
	private ObjectToBeSerialised testOutput;
	private long writeTimeNanos;
	private long readTimeNanos;

	public PerformanceTestCase(final String name, final int repetitions, final ObjectToBeSerialised testInput) {
		this.name = name;
		this.repetitions = repetitions;
		this.testInput = testInput;
	}

	public String getName() {
		return name;
	}

	public ObjectToBeSerialised getTestOutput() {
		return testOutput;
	}

	public long getWriteTimeNanos() {
		return writeTimeNanos;
	}

	public long getReadTimeNanos() {
		return readTimeNanos;
	}

	public void performTest() throws Exception {
		final long startWriteNanos = System.nanoTime();
		testWrite(testInput);
		writeTimeNanos = (System.nanoTime() - startWriteNanos) / repetitions;

		final long startReadNanos = System.nanoTime();
		testOutput = testRead();
		readTimeNanos = (System.nanoTime() - startReadNanos) / repetitions;
	}

	public abstract void testWrite(ObjectToBeSerialised item) throws Exception;

	public abstract ObjectToBeSerialised testRead() throws Exception;
}

class ObjectToBeSerialised implements Serializable {
	private static final long serialVersionUID = 10275539472837495L;

	private final long sourceId;
	private final boolean special;
	private final int orderCode;
	private final int priority;
	private final double[] prices;
	private final long[] quantities;

	public ObjectToBeSerialised(final long sourceId, final boolean special, final int orderCode, final int priority,
			final double[] prices, final long[] quantities) {
		this.sourceId = sourceId;
		this.special = special;
		this.orderCode = orderCode;
		this.priority = priority;
		this.prices = prices;
		this.quantities = quantities;
	}

	public void write(final ByteBuffer byteBuffer) {
		byteBuffer.putLong(sourceId);
		byteBuffer.put((byte) (special ? 1 : 0));
		byteBuffer.putInt(orderCode);
		byteBuffer.putInt(priority);

		byteBuffer.putInt(prices.length);
		for (final double price : prices) {
			byteBuffer.putDouble(price);
		}

		byteBuffer.putInt(quantities.length);
		for (final long quantity : quantities) {
			byteBuffer.putLong(quantity);
		}
	}

	public static ObjectToBeSerialised read(final ByteBuffer byteBuffer) {
		final long sourceId = byteBuffer.getLong();
		final boolean special = 0 != byteBuffer.get();
		final int orderCode = byteBuffer.getInt();
		final int priority = byteBuffer.getInt();

		final int pricesSize = byteBuffer.getInt();
		final double[] prices = new double[pricesSize];
		for (int i = 0; i < pricesSize; i++) {
			prices[i] = byteBuffer.getDouble();
		}

		final int quantitiesSize = byteBuffer.getInt();
		final long[] quantities = new long[quantitiesSize];
		for (int i = 0; i < quantitiesSize; i++) {
			quantities[i] = byteBuffer.getLong();
		}

		return new ObjectToBeSerialised(sourceId, special, orderCode, priority, prices, quantities);
	}

	public void write(final UnsafeMemory buffer) {
		buffer.putLong(sourceId);
		buffer.putBoolean(special);
		buffer.putInt(orderCode);
		buffer.putInt(priority);
		buffer.putDoubleArray(prices);
		buffer.putLongArray(quantities);
	}

	public static ObjectToBeSerialised read(final UnsafeMemory buffer) {
		final long sourceId = buffer.getLong();
		final boolean special = buffer.getBoolean();
		final int orderCode = buffer.getInt();
		final int priority = buffer.getInt();
		final double[] prices = buffer.getDoubleArray();
		final long[] quantities = buffer.getLongArray();

		return new ObjectToBeSerialised(sourceId, special, orderCode, priority, prices, quantities);
	}

	@Override
	public boolean equals(final Object o) {
		if (this == o) {
			return true;
		}
		if (o == null || getClass() != o.getClass()) {
			return false;
		}

		final ObjectToBeSerialised that = (ObjectToBeSerialised) o;

		if (orderCode != that.orderCode) {
			return false;
		}
		if (priority != that.priority) {
			return false;
		}
		if (sourceId != that.sourceId) {
			return false;
		}
		if (special != that.special) {
			return false;
		}
		if (!Arrays.equals(prices, that.prices)) {
			return false;
		}
		if (!Arrays.equals(quantities, that.quantities)) {
			return false;
		}

		return true;
	}
}

class UnsafeMemory {
	private static final Unsafe unsafe;
	static {
		try {
			Field field = Unsafe.class.getDeclaredField("theUnsafe");
			field.setAccessible(true);
			unsafe = (Unsafe) field.get(null);
		} catch (Exception e) {
			throw new RuntimeException(e);
		}
	}

	private static final long byteArrayOffset = unsafe.arrayBaseOffset(byte[].class);
	private static final long longArrayOffset = unsafe.arrayBaseOffset(long[].class);
	private static final long doubleArrayOffset = unsafe.arrayBaseOffset(double[].class);

	private static final int SIZE_OF_BOOLEAN = 1;
	private static final int SIZE_OF_INT = 4;
	private static final int SIZE_OF_LONG = 8;

	private int pos = 0;
	private final byte[] buffer;

	public UnsafeMemory(final byte[] buffer) {
		if (null == buffer) {
			throw new NullPointerException("buffer cannot be null");
		}

		this.buffer = buffer;
	}

	public void reset() {
		this.pos = 0;
	}

	public void putBoolean(final boolean value) {
		unsafe.putBoolean(buffer, byteArrayOffset + pos, value);
		pos += SIZE_OF_BOOLEAN;
	}

	public boolean getBoolean() {
		boolean value = unsafe.getBoolean(buffer, byteArrayOffset + pos);
		pos += SIZE_OF_BOOLEAN;

		return value;
	}

	public void putInt(final int value) {
		unsafe.putInt(buffer, byteArrayOffset + pos, value);
		pos += SIZE_OF_INT;
	}

	public int getInt() {
		int value = unsafe.getInt(buffer, byteArrayOffset + pos);
		pos += SIZE_OF_INT;

		return value;
	}

	public void putLong(final long value) {
		unsafe.putLong(buffer, byteArrayOffset + pos, value);
		pos += SIZE_OF_LONG;
	}

	public long getLong() {
		long value = unsafe.getLong(buffer, byteArrayOffset + pos);
		pos += SIZE_OF_LONG;

		return value;
	}

	public void putLongArray(final long[] values) {
		putInt(values.length);

		long bytesToCopy = values.length << 3;
		unsafe.copyMemory(values, longArrayOffset, buffer, byteArrayOffset + pos, bytesToCopy);
		pos += bytesToCopy;
	}

	public long[] getLongArray() {
		int arraySize = getInt();
		long[] values = new long[arraySize];

		long bytesToCopy = values.length << 3;
		unsafe.copyMemory(buffer, byteArrayOffset + pos, values, longArrayOffset, bytesToCopy);
		pos += bytesToCopy;

		return values;
	}

	public void putDoubleArray(final double[] values) {
		putInt(values.length);

		long bytesToCopy = values.length << 3;
		unsafe.copyMemory(values, doubleArrayOffset, buffer, byteArrayOffset + pos, bytesToCopy);
		pos += bytesToCopy;
	}

	public double[] getDoubleArray() {
		int arraySize = getInt();
		double[] values = new double[arraySize];

		long bytesToCopy = values.length << 3;
		unsafe.copyMemory(buffer, byteArrayOffset + pos, values, doubleArrayOffset, bytesToCopy);
		pos += bytesToCopy;

		return values;
	}
}
{% endhighlight %}

测试结果

2.8GHz Nehalem - Java 1.7.0_04
==============================
0 Serialisation write=2,517ns read=11,570ns total=14,087ns
1 Serialisation write=2,198ns read=11,122ns total=13,320ns
2 Serialisation write=2,190ns read=11,011ns total=13,201ns
3 Serialisation write=2,221ns read=10,972ns total=13,193ns
4 Serialisation write=2,187ns read=10,817ns total=13,004ns
0 ByteBuffer write=264ns read=273ns total=537ns
1 ByteBuffer write=248ns read=243ns total=491ns
2 ByteBuffer write=262ns read=243ns total=505ns
3 ByteBuffer write=300ns read=240ns total=540ns
4 ByteBuffer write=247ns read=243ns total=490ns
0 UnsafeMemory write=99ns read=84ns total=183ns
1 UnsafeMemory write=53ns read=82ns total=135ns
2 UnsafeMemory write=63ns read=66ns total=129ns
3 UnsafeMemory write=46ns read=63ns total=109ns
4 UnsafeMemory write=48ns read=58ns total=106ns

2.4GHz Sandy Bridge - Java 1.7.0_04
===================================
0 Serialisation write=1,940ns read=9,006ns total=10,946ns
1 Serialisation write=1,674ns read=8,567ns total=10,241ns
2 Serialisation write=1,666ns read=8,680ns total=10,346ns
3 Serialisation write=1,666ns read=8,623ns total=10,289ns
4 Serialisation write=1,715ns read=8,586ns total=10,301ns
0 ByteBuffer write=199ns read=198ns total=397ns
1 ByteBuffer write=176ns read=178ns total=354ns
2 ByteBuffer write=174ns read=174ns total=348ns
3 ByteBuffer write=172ns read=183ns total=355ns
4 ByteBuffer write=174ns read=180ns total=354ns
0 UnsafeMemory write=38ns read=75ns total=113ns
1 UnsafeMemory write=26ns read=52ns total=78ns
2 UnsafeMemory write=26ns read=51ns total=77ns
3 UnsafeMemory write=25ns read=51ns total=76ns
4 UnsafeMemory write=27ns read=50ns total=77ns

i5 1.7GHz Sandy Bridge - Java 1.8.0_121 mac air
===================================
0 Serialisation	write=3,638ns read=17,277ns total=20,915ns
1 Serialisation	write=2,437ns read=17,104ns total=19,541ns
2 Serialisation	write=2,507ns read=17,309ns total=19,816ns
3 Serialisation	write=2,436ns read=16,045ns total=18,481ns
4 Serialisation	write=2,374ns read=16,030ns total=18,404ns
0 ByteBuffer	write=223ns read=273ns total=496ns
1 ByteBuffer	write=187ns read=238ns total=425ns
2 ByteBuffer	write=195ns read=250ns total=445ns
3 ByteBuffer	write=191ns read=248ns total=439ns
4 ByteBuffer	write=209ns read=238ns total=447ns
0 UnsafeMemory	write=41ns read=85ns total=126ns
1 UnsafeMemory	write=36ns read=77ns total=113ns
2 UnsafeMemory	write=28ns read=83ns total=111ns
3 UnsafeMemory	write=32ns read=93ns total=125ns
4 UnsafeMemory	write=33ns read=70ns total=103ns


很显然允许自己内存操作的 Unsafe性能是最快的。
 
 
