---
{"dg-publish":true,"permalink":"/DB/DuckDB Sort优化/","dgPassFrontmatter":true}
---

#Database

# Intro
该论文针对如何在数据库进行高效排序的问题进行了探索。
### 多列排序
数据库排序相比传统排序，通常需要针对不同列采用不同的排序方法，比如文中举的例子中，ORDER BY指定了需要对`c_birth_country`和`c_birth_year`采用不同的排序顺序，这类多列排序通常有两种比较方法：**逐列比较**和**逐行比较**。同时，数据库的内存数据通常有两种表示形式：**行存**和**列存**。因此，多列排序的比较操作有四种情况，论文分别针对这四种情况进行性能测试以及结果的分析。
```
SELECT * 
FROM customer 
ORDER BY c_birth_country DESC NULLS LAST, 
        c_birth_year ASC NULLS FIRST;
```
### 执行方式
数据库中的执行引擎可以分为两种流派：**向量化执行**和**代码生成**。该文对排序在这两种执行方式下的性能表现有何不同进行了探究。
# 实验设置
为了避免不同数据库实现在解析或者其他位置产生对最终结果的影响，该论文实现了一个[microbench](https://github.com/lnkuiper/experiments/tree/master/sorting_simulation)，其模拟了不同数据库的数据布局以比较不同设计对排序结果的影响。
### 实验负载
- 数据分布：
	1. Random
	2. Unique128：结果集范围有128种数据，均匀分布
	3. PowerLaw：结果集范围有128种数据，但分布有倾斜，即少数几个数值大量出现
- 排序列：$(1,4)$
- 数据量：$(2^{10} , 2^{24})$
# 行存与列存排序
### 列存
#### 理论分析
- **行排序**：列存行排序需要维护一组行索引，每次比较通过两个行索引分别逐列比较。
	1. 若数据分布存在大量重复，比较操作通常需要访问多列，由于列存，逐列访问通常会触发cache miss
	2. 比较操作有大量分支，即是否比较下一列，除非数据分布呈现规律性（比如没有重复元素），否则硬件很难预测，会导致大量分支判断失败
- **列排序**：列排序每次比较一列数据，单列排序中比较操作不需要访问多列也不会有大量分支
#### 实验结果分析
在数据量为 $2^{24}$ ，3列比较的测试结果中可以看到：
1. 行排序(C/T)的Cache Miss和Branch Miss相比于列排序（C/S）都要高。
2. 在Random数据集中，行排序通常仅仅需要比较第一列，这可以被硬件很好的预测到，因此行排序与列排序的指标接近。
![Pasted image 20240831124633.png](/img/user/DB/img/Pasted%20image%2020240831124633.png)
从相对性能测试结果（列排序:行排序，即若对应方块为2，表示列排序性能为行排序的两倍）中：
1. 大部分情况列排序的性能都比行排序高。
2. 在数据分布呈现随机，或者单列比较时，行排序不会有过多cache miss和branch miss，两者性能相近。
![Pasted image 20240831141729.png](/img/user/DB/img/Pasted%20image%2020240831141729.png)
### 行存
#### 理论分析
- cache miss
	对于行存储，无论使用行排序还是列排序，都需要将整行数据加载至缓存，因此cache miss应该相近
- branch miss
	与列存储的结果类似，使用行排序的比较函数由于需要同时比较多列，在某些场景下，branch miss会更高。
#### 实验分析
在数据量为 $2^{24}$ ，3列比较的测试结果中可以看到：
1. cache miss和branch misses都比列存排序底一个量级
	猜测：
	在该数据量下，数据无法完全容纳入CPU缓存，对于行存，其可以容纳局部行，在对该局部进行排序时，不会造成cache miss。
	- 对于**列存行排序**，每次比较需要隔列访问，因此每次比较都有可能造成cache miss。
	- 对于**列存列排序**，排序分为多次排序，每次排序完需要首先进行扫描，扫描后需要通过下一列进一步排序，该过程同样会造成的cache miss。
1. 与理论分析一致，行排列和列排序的cache miss相近。列排序略高，文中解释是因为每次排序完需要扫描相同key做下一列排序。
2. 列排序branch miss更低

![Pasted image 20240831153059.png](/img/user/DB/img/Pasted%20image%2020240831153059.png)

研究将行存的两种排序方式与列存列排序（列存最优）进行了比较，从结果上看：
1. 行存两种排序方式的性能都要优于列存，**行存行排序更优**
2. 数据量较小时所有数据可以容纳于缓存中，行存列存两者性能接近，

![Pasted image 20240831154035.png](/img/user/DB/img/Pasted%20image%2020240831154035.png)
### 总结
从实验结果上看，行存行排序的性能在数据量大时总是优于列存。因此后文duckdb采用了将列存暂时转换为行存进行排序，再转换回来的方法。



# 编译执行与向量化执行
### 编译执行
理论上，编译执行可以完全消除解释执行所需的开销。其问题在于实现较为困难。
### 向量化
duckdb采用向量化执行，因此对排序在向量化场景的缺陷进行了更为详细的讨论。相比于编译执行，向量化执行属于解释执行，即需要动态解析数据，并通过向量化的方式降低解析开销。
如果使用逐列排序，可以做到全列只做一次解析，降低解释开销。然而，在某些场景下，比如merge sort，不可避免需要做全行比较，此时每次比较操作都需要进行动态解析。
为了体现动态解析下的开销，该文手写了一个静态函数版本和动态函数版本的排序性能对比测试。从数据中可以观察到动态解析的开销是比较明显的。

![Pasted image 20240901020832.png](/img/user/DB/img/Pasted%20image%2020240901020832.png)
# 优化
经过以上的分析，得出了优化的方向：
duckdb属于列存向量化执行，虽然使用列排序在cache miss，branch miss，解释开销上都能够得到缓解，然而有些场景仍然需要使用行排序。**行存行排序**在排序上的性能表现是最优。
### normalized keys
Normalized keys将单行多列数据按照某种编码方式将其拼接为一个可比较的二进制格式，经过编码后的行可以直接通过memcmp进行比较，memcpy进行移动。
比如对于降序排序，可以通过将所有二进制位进行反转的方式来实现。[该优化也在arrow中得到了使用](https://arrow.apache.org/blog/2022/11/07/multi-column-sorts-in-arrow-rust-part-1/)。
通过该方法：1. 将列存转换为行存 2. 消除了行排序比较需要的动态解释开销
### static *mem* 
memcpy函数的签名为：`memcpy(void *dest, void *src, size_t size)`
文章提到在编译期提供size通常能够使编译器提供更好的实现。对于向量化执行，无法针对查询进行编译，因此该参数只能以变量的方式提供。该文提出使用如下静态分发的方式，其本质上是手写生成不同长度版本的memcpy，由于向量化执行中单个排序内size是固定的，这点可以被硬件很好的预测到，因此不存在分支预测带来的开销。
```
void s_memcpy(void *dest, void *src, size_t size) {
	switch (size) {
	case 1:
		return memcpy(dest, src, 1);
	case 2:
		return memcpy(dest, src, 2);
	...
}
```
从实验结果可以看到，该方法并不一定可以带来提升，主要取决于架构以及长度。（文章并没有更加深入地探究原因，比如分析生成的汇编
![Pasted image 20240901120357.png](/img/user/DB/img/Pasted%20image%2020240901120357.png)
### Radix Sort
### 并发多路排序

# Ref
1. [These Rows Are Made for Sorting and That’s Just What We’ll Do](https://hannes.muehleisen.org/publications/ICDE2023-sorting.pdf)
2. [怎样把数据库排序做到全球第一？](https://zhuanlan.zhihu.com/p/664312966?utm_psn=1813033991550427137)
3. An Encoding Method for Multifield Sorting and Indexing


