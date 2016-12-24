## 记忆搜索
在递归调用中，可能会有一些数据会被重复计算。这时候可以考虑使用记忆化搜索或者动态规划将其中的一些值保存起来，避免后续的调用重复计算。例如在计算斐波那契和的时候，任何一个值在拓展后都可能被重复计算多次，并且每次的值都是一样的。因此采用这种方法可以进行优化。  
````java
//用于存储已经计算过的斐波那契数列
private static int[] mem;

//一般版本
public static int normalfib( int n) {
        if(n <= 1) return n;
        return normalfib(n-1) + normalfib (n-2);
}
//记忆搜索
public static int memfib (int n) {
        if(n <= 1) return n;
        //命中直接返回
        if( mem[n] != 0) return mem[n];
        mem[n] = memfib(n-1) + memfib(n-2);
        return mem[n];
}
````