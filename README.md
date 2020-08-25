# SnowFlake
SnowFlake 雪花算法的简单分析和代码样例
# SnowFlake(雪花算法)

### 结构组成

SnowFlake算法生成ID的结果是一个64bit大小的整数，它的结构如下：

![image-20200715144235256](C:\Users\liche\AppData\Roaming\Typora\typora-user-images\image-20200715144235256.png)

1. **1bit**，不用，因为二进制中最高位是符号位，1表示负数，0表示正数。生成的id一般都是用整数，所以最高位固定为0。
2. **41bit-时间戳**，用来记录时间戳，毫秒级。
    \- 41位可以表示![2^{41}-1](https://math.jianshu.com/math?formula=2%5E%7B41%7D-1)个数字，
    \- 如果只用来表示正整数（计算机中正数包含0），可以表示的数值范围是：0 至 ![2^{41}-1](https://math.jianshu.com/math?formula=2%5E%7B41%7D-1)，减1是因为可表示的数值范围是从0开始算的，而不是1。
    \- 也就是说41位可以表示![2^{41}-1](https://math.jianshu.com/math?formula=2%5E%7B41%7D-1)个毫秒的值，转化成单位年则是![(2^{41}-1) / (1000 * 60 * 60 * 24 *365) = 69](https://math.jianshu.com/math?formula=(2%5E%7B41%7D-1)%20%2F%20(1000%20*%2060%20*%2060%20*%2024%20*365)%20%3D%2069)年
3. **10bit-工作机器id**，用来记录工作机器id。
    \- 可以部署在![2^{10} = 1024](https://math.jianshu.com/math?formula=2%5E%7B10%7D%20%3D%201024)个节点，包括5位datacenterId和5位workerId
    \- 5位（bit）可以表示的最大正整数是![2^{5}-1 = 31](https://math.jianshu.com/math?formula=2%5E%7B5%7D-1%20%3D%2031)，即可以用0、1、2、3、....31这32个数字，来表示不同的datecenterId或workerId
4. **12bit-序列号**，序列号，用来记录同毫秒内产生的不同id。
    \-  12位（bit）可以表示的最大正整数是![2^{12}-1 = 4095](https://math.jianshu.com/math?formula=2%5E%7B12%7D-1%20%3D%204095)，即可以用0、1、2、3、....4094这4095个数字，来表示同一机器同一时间截（毫秒)内产生的4095个ID序号。

由于在Java中64bit的整数是long类型，所以在Java中SnowFlake算法生成的id就是long来存储的。



作者：SmartBin
链接：https://www.jianshu.com/p/2a27fbd9e71a
来源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。

### 各部分位移运算再做合并运算

```java
public static void main(String[] args) {
        long a = 5L;
        System.out.println("a=" + toBinary(a));
        a = a<<20;
        System.out.println("a=" + toBinary(a) + "\r\n");
        long b = 8L;
        System.out.println("b=" + toBinary(b));
        b = b <<10;
        System.out.println("b=" + toBinary(b) + "\r\n");
        long c = 10L;
        System.out.println("c=" + toBinary(c) + "\r\n");
        System.out.println("三个数字位移后合并的结果：");
        long d = a | b | c;
        System.out.println("d=" + toBinary(d));
        /**
         * a=0000000000000000000000000000000000000000000000000000000000000101
         * a=0000000000000000000000000000000000000000010100000000000000000000
         *
         * b=0000000000000000000000000000000000000000000000000000000000001000
         * b=0000000000000000000000000000000000000000000000000010000000000000
         *
         * c=0000000000000000000000000000000000000000000000000000000000001010
         *
         * 三个数字位移后合并的结果：
         * d=0000000000000000000000000000000000000000010100000010000000001010
         */
    }
 public static String toBinary(long num){
        String binary = Long.toBinaryString(num);
        while (binary.length()< 64){
            binary = "0"+ binary;
        }
        return  binary;
    }
```





### 用mask防止溢出

关注这行代码：

```java
sequence = (sequence +1) & sequenceMask;
```

用不同的数值与seqMask进行位与运算，进行测试：

```java
@Test
public void testMask(){
    /**
     * seqMask表示计算12位二进制表示的最大正整数，相当于2^12-1=4095
     */
    long seqMask = -1L ^ (-1L << 12L);
    System.out.println("seqMask: " + seqMask);
    System.out.println(1L & seqMask);
    System.out.println(2L & seqMask);
    System.out.println(3L & seqMask);
    System.out.println(4L & seqMask);
    System.out.println(4095L & seqMask);
    System.out.println(4096L & seqMask);
    System.out.println(4097L & seqMask);
    System.out.println(4098L & seqMask);
    /**
     * seqMask: 4095
     * 1
     * 2
     * 3
     * 4
     * 4095
     * 0
     * 1
     * 2
     */
}
```

**表明通过位与运算保证计算的结果范围始终是在0-4095**

### 用位运算汇总结果

```java
 // ID偏移组合生成最终的ID，并返回ID
        long nextId = ((timestamp - twepoch) << timestampLeftShift) | (processId << datacenterIdShift) | (workerId << workerIdShift) | sequence;
        return nextId;
```

写个测试脚本，把参数都附在里面，运行并打印结果

```java
 @Test
    public void testResult(){
        /**
         *System.out.println(System.currentTimeMillis());
         *1594793149501
         */
        long timestamp = 1594793149501L;
        long twepoch = 1288834974657L;
        long datacenterId = 17L;
        long workerId = 1L;
        long sequence = 0L;

        System.out.printf("\ntimestamp: %d \n",timestamp);
        System.out.printf("twepoch: %d \n",twepoch);
        System.out.printf("datacenterId: %d \n",datacenterId);
        System.out.printf("workerId: %d \n",workerId);
        System.out.printf("sequence: %d \n",sequence);
        System.out.println("==================================");
        System.out.printf("(timestamp - twepoch): %d \n",(timestamp - twepoch));
        System.out.printf("((timestamp - twepoch) << 12L): %d \n",((timestamp-twepoch)<<22L));
        System.out.printf("(datacenterId << 17L): %d \n",(datacenterId << 17L));
        System.out.printf("(workerId << 12L)： %d \n",(workerId << 12L));
        System.out.printf("sequence: %d \n",sequence);
        long result = ((timestamp-twepoch) << 22L) |
                (datacenterId << 17L) |
                (workerId << 12L)|
                sequence;
        System.out.println("合并后的结果为： " + result);
        /**
         * timestamp: 1594793149501
         * twepoch: 1288834974657
         * datacenterId: 17
         * workerId: 1
         * sequence: 0
         * ==================================
         * (timestamp - twepoch): 305958174844
         * ((timestamp - twepoch) << 12L): 1283281596580888576
         * (datacenterId << 17L): 2228224
         * (workerId << 12L)： 4096
         * sequence: 0
         * 合并后的结果为： 1283281596583120896
         */
    }
```

由于在java中64bit的整数是long类型，所以在java中SnowFlake算法生成的id就是long 来存储的；

SnowFlake可以保证：

> 1.所有生成的id按时间趋势递增；
>
> 2.整个分布式系统内不会产生重复的id(因为有datacenterId和workerId来区分)

雪花算法最终版本：

https://gitee.com/blueses/snowflake-demo/blob/master/07-snowflake/src/main/java/snowflake07/SnowflakeUtils.java

```java
public class SnowflakeUtils {

    /** 时间部分所占长度 */
    private static final int TIME_LEN = 41;
    /** 数据中心id所占长度 */
    private static final int DATA_LEN = 5;
    /** 机器id所占长度 */
    private static final int WORK_LEN = 5;
    /** 毫秒内序列所占长度 */
    private static final int SEQ_LEN = 12;

    /** 定义起始时间 2015-01-01 00:00:00 */
    private static final long START_TIME = 1420041600000L;
    /** 上次生成ID的时间截 */
    private static long LAST_TIME_STAMP = -1L;
    /** 时间部分向左移动的位数 22 */
    private static final int TIME_LEFT_BIT = 64 - 1 - TIME_LEN;

    /** 自动获取数据中心id（可以手动定义 0-31之间的数） */
    private static final long DATA_ID = getDataId();
    /** 自动机器id（可以手动定义 0-31之间的数） */
    private static final long WORK_ID = getWorkId();
    /** 数据中心id最大值 31 */
    private static final int DATA_MAX_NUM = ~(-1 << DATA_LEN);
    /** 机器id最大值 31 */
    private static final int WORK_MAX_NUM = ~(-1 << WORK_LEN);
    /** 随机获取数据中心id的参数 32 */
    private static final int DATA_RANDOM = DATA_MAX_NUM + 1;
    /** 随机获取机器id的参数 32 */
    private static final int WORK_RANDOM = WORK_MAX_NUM + 1;
    /** 数据中心id左移位数 17 */
    private static final int DATA_LEFT_BIT = TIME_LEFT_BIT - DATA_LEN;
    /** 机器id左移位数 12 */
    private static final int WORK_LEFT_BIT = DATA_LEFT_BIT - WORK_LEN;

    /** 上一次的毫秒内序列值 */
    private static long LAST_SEQ = 0L;
    /** 毫秒内序列的最大值 4095 */
    private static final long SEQ_MAX_NUM = ~(-1 << SEQ_LEN);


    public synchronized static long genId(){
        long now = System.currentTimeMillis();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (now < LAST_TIME_STAMP) {
            throw new RuntimeException(String.format("系统时间错误！ %d 毫秒内拒绝生成雪花ID！", START_TIME - now));
        }

        if (now == LAST_TIME_STAMP) {
            LAST_SEQ = (LAST_SEQ + 1) & SEQ_MAX_NUM;
            if (LAST_SEQ == 0){
                now = nextMillis(LAST_TIME_STAMP);
            }
        } else {
            LAST_SEQ = 0;
        }

        //上次生成ID的时间截
        LAST_TIME_STAMP = now;

        return ((now - START_TIME) << TIME_LEFT_BIT) | (DATA_ID << DATA_LEFT_BIT) | (WORK_ID << WORK_LEFT_BIT) | LAST_SEQ;
    }


    /**
     * 获取下一不同毫秒的时间戳，不能与最后的时间戳一样
     */
    public static long nextMillis(long lastMillis) {
        long now = System.currentTimeMillis();
        while (now <= lastMillis) {
            now = System.currentTimeMillis();
        }
        return now;
    }

    /**
     * 获取字符串s的字节数组，然后将数组的元素相加，对（max+1）取余
     */
    private static int getHostId(String s, int max){
        byte[] bytes = s.getBytes();
        int sums = 0;
        for(int b : bytes){
            sums += b;
        }
        return sums % (max+1);
    }

    /**
     * 根据 host address 取余，发生异常就获取 0到31之间的随机数
     */
    public static int getWorkId(){
        try {
            return getHostId(Inet4Address.getLocalHost().getHostAddress(), WORK_MAX_NUM);
        } catch (UnknownHostException e) {
            return new Random().nextInt(WORK_RANDOM);
        }
    }

    /**
     * 根据 host name 取余，发生异常就获取 0到31之间的随机数
     */
    public static int getDataId() {
        try {
            return getHostId(Inet4Address.getLocalHost().getHostName(), DATA_MAX_NUM);
        } catch (UnknownHostException e) {
            return new Random().nextInt(DATA_RANDOM);
        }
    }


    public static void main(String[] args) {
        Set ids = new HashSet();
        long start = System.currentTimeMillis();
        for (int i = 0; i < 3000000; i++) {
            ids.add(genId());
        }
        long end = System.currentTimeMillis();
        System.out.println("共生成id[" + ids.size() + "]个，花费时间[" + (end - start) + "]毫秒");
    }
}

```

