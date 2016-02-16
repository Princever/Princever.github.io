---  
layout: post
title: Supporting Data Uncertainty in Array Databases
category: "Algorithm and Data Stucture"
--- 

This is about my summation about 「Supporting Data Uncertainty in Array Databases」

### Issue And Problem ###
Uncertain data management has been recently studied in **Large-scale scientific** applications which have noisy and uncertain, and its key is to capturing uncertainty. In this paper, we focus on addressing position uncertainty lies in finding a storage and evaluation strategy that minimizes both I/O and CPU costs while returning all tuples that satisfy the query with a required probability. 




In multi-dimensional array(chunk-based storage scheme obj close likely to be stored in the same chunk), a tuple some of whose **value attribute** is uncertain(even some time dimension is uncertain). This means **uncertainty of position** lead to a tuple belong to multiple cells(Position uncertainty).

1.Show subarray and structure-join are two most important array operations.

2.For subarray, store-multiple: bounds the overhead of querying by **placing a few replicas(making the query region very large)** for the tuples with large variances.

3.For **structure-join**: new evaluation strategy — subarray-based join(SBJ) without a prebuilt index and employs tight conditions for running repeated subarray queries on inner array of join.

4.Using synthetic workloads and the Sloan Digital Sky Survey. 


### Array Data ###

Background: A database contains a collection of arrays. Each arrays represent as **A(D<sup>d</sup>;V<sup>m</sup>)**, sometimes **A<sup>d</sup>**. e.g In SDSS, A<sup>2</sup>(x-loc, y-loc; luminosity, color).

For dimension attributes: 

1.discrete — requires a linear order.

2.continuous — user-defined mapping function to discretizing the domain in to an ordered set of sets. To used as **index**.

multiple values of a dimension value lead to mapping into the **same cell**. Multiply tuples in a cell.

Define the tuple’s possible range Rt as a hyper- rectangle within which the tuple existence probability is (approximately) 1. 
Construct Rt by taking t’s marginal distribution, fi of each uncertain dimension: 

1.fi~U(a, b) —— [a, b], p = 1; 

2.fi~N(μ, σ), possible range (μ−3σ, μ+3σ) p = 0.997. 

3.fi~**an arbitrary distribution** with mean μ and standard deviation σ, define the possible range to be (μ − kσ, μ + kσ) with a sufficiently large k chosen based on **Chebyshev’s inequality**(possible region Rt to the boundary of cells).

### Array Algebra ###

+Value-based: operate **only on the value attributes** of tuples.

+Structure-based: operators operate on **dimension attributes** and optionally on **value attributes**:

1.Subarray takes an array A and a condition θ on the dimension attributes, and **returns a new array with the tuples that satisfy the condition θ**. The output array always has the same dimensions as the input, but usually fewer cells and tuples. 

2.Structure-Join (SJoin) takes as input an array A<sup>d</sup>, a second array B<sup>d</sup> of the same dimensionality, and a join condition θ. SJoin(A, B, θ) **returns an array C<sup>2d</sup>**, where the cell C[i<sub>1</sub>, · · · , i<sub>d</sub>, i<sub>d+1</sub>, · · · , i<sub>2d</sub>] contains the result of θ-join between the tuples in A[i<sub>1</sub>,··· ,i<sub>d</sub>] and the tuples in B[i<sub>d+1</sub>,··· ,i<sub>2d</sub>]. The equivalent expression is R<sub>A</sub> ⋈<sub>θ</sub> R<sub>B</sub>.

3.If the dimension attributes are continuous-valued, θ takes a form of proximity join. A common form is linear proximity (a.k.a. l<sub>1</sub>-distance) join, **\|A.d<sub>i</sub> −B.d<sub>i</sub>\| < δ** for each dimension attribute d<sub>i</sub>. While the tuples may belong to multiple cells with non-zero probabilities, so the join between A and B must return **all pairs** of tuples that satisfies. first, leverage the semantics of cross-product in the above **SJoin definition** involves **pairing each cell in A with each cell in B** and then pairing the tuples within those cells. Define probabilistic Structure-Join as follows: Given A<sup>d</sup> and B<sup>d</sup>, a join condition θ, and a probability threshold λ, SJoin(A,B, θ, λ) returns an array C<sup>2d</sup> where C[i<sub>1</sub>, · · · , i<sub>d</sub>, i<sub>d+1</sub>, · · · , i<sub>2d</sub>] contains the result of probabilistic θ-join, **A[i<sub>1</sub>,··· ,i<sub>d</sub>] ⋈<sub>θ,λ</sub> B[i<sub>d+1</sub>,··· ,i<sub>2d</sub>] = {(t<sub>1</sub>,t<sub>2</sub>)\|t<sub>1</sub> ∈ A[i<sub>1</sub>,··· ,i<sub>d</sub>], t2 ∈ B[i<sub>d+1</sub>,··· ,i<sub>2d</sub>], ∫∫<sub>θ</sub> f<sub>t1</sub>(x)·f<sub>t2</sub>(y)dxdy ≥ λ}**. Many other structure operators can be implemented using Subarray and Structure-Join.


### Native support for subarray ###

**Store-All**: One solution is to store a copy of the tuple in **each cell** of the tuple’s possible range.

**Store-Mean**: To reduce storage overheads, we next consider storing a tuple **only once based on the mean values** of its dimension attributes. Given a hyper-rectangle query region Q, its expanded query region Q ̃ is a super hyper- rectangle Q ̃ (⊇ Q) such that any tuple whose possible range overlaps with Q has at least one copy stored in C(Q ̃) 

**Store-Multiple**: A scheme that employs **limited replication** of tuples and guarantees that from any cell in a tuple’s possible range, walking at most **k cells(the step size)** along each dimension is able to find a copy of the tuple. 

### Support for structure join ###

Subarray-based Join (SBJ)

The basic idea is that for each outer **cell C<sub>A</sub>**, form a **subarray condition θ<sub>CA</sub>** on the inner array B, run the Subarray query **on B to retrieve relevant tuples**, and finally **pair the A tuples and B tuples** for exact evaluation using integration.
Given a tuple t<sub>A</sub> , let (l , u) denote the lower and upper bounds.
For each tuple t<sub>A</sub> from A, **expand its possible range by δ** on each dimension, denoted by I<sub>t<sub>A</sub></sub> , then pair t<sub>A</sub> with all tuples t<sub>B</sub> from B whose possible ranges **overlap with I<sub>t<sub>A</sub></sub>**.
When A is stored using **store-mean**: relaxing the condition using the minimum lower bound and maximum upper bound of possible ranges of all tuples in C<sub>A</sub>.
When A is stored using **store-multiply**: do not need to relax the join condition.

### Algorithm for Subarray-based Join (SBJ) ###

	for each read block RA in A do 
		toRead.clear(); 
		map.clear(); 
		for each cell CA in RA do 
			loadToMemory(CA ); 
			Q ← formQueryRegion(CA ); 
			S ← Subarray(B, Q);
			for each cell CB in S do 
				toRead.add(CB ); 
				map.get(CB ).add(CA ); 
		for each cell CB in toRead do 
			loadToMemory(CB ); 		
			for each cell CA in map.get(CB) do 
				for each tuple tA in CA do 
					for each tuple tB in CB do 
						filter(tA , tB ); 
						validate(tA , tB ); 
	removeDuplicates(); 

### Novelty of The Proposed Approach ###

1.Proposing a new way of storing tuples and optimized CPU running time. Making native support for **subarray** and **structure join**.

2.The new storage and evaluation schemes find a **balance** between **space** and **time** with the step size k in **Store-Multiple**.

3.Proposed an algorithm about **Subarray-based Join** and define the operation of SBJ.



### Summary of Experiments ###

Datasets: SDSS(dimension attributes: rowc, cols). Describes each dimension attribute using a Gaussian distribution.


### Evaluation of Subarray Techniques ###

#### Expt 1: Cost Breakdown ####

I/O cost first decreasesand then increases. So the optimal I/O cost appears in the middle of the spectrum of k(step size)

The CPU cost does not change with the step size when the probability threshold λ is high

the optimal step size shifts right when the average possible range increases; it shifts left when λ is very small or the per integration cost is high, and it increases with the query region size.

#### Expt 2: Model Accuracy ####

Pick 3 x 3 query region for 2D datasets and 2 x 2 x 2 for 3D datasets that evenly scattered over the array.

Model returns the optimal step size in 89.6% of workloads when the tuples’ μ values are uniformly distributed and in 83.3% of workloads when the tuples’ μ values are normally distributed. In those cases when our model selects a suboptimal step size, the average performance loss is 2.72%.

#### Expt 3: Comparing Schemes ####

Store-mutiple works the best. when all tuples have small possible ranges, the three storage schemes do not differ much. However, for datasets when Sσ = 100, store-all often incurs tremendous storage overheads and I/O costs in querying. When query grows larger, their difference is reduced because the optimal step size of store-multiple tends to be larger. This means that infrequent replication of tuples works fine if q is large, and most tuples have only one copy under store-multiple, similar to store-mean. 


### Evaluation of Structure-Join ###

#### Expt 4: Subarray-Based Join(SBJ) ####

Demonstrate that SBJ’s performance is sensitive to the
storage scheme.

For a fixed kout, the optimal inner step size k∗ is in the middle in
of its spectrum.

Once kin is fixed, the optimal k∗ also occurs in the middle 

The model returns the optimal step sizes in most cases and the overall performance loss is within 6% (if any).

#### Expt 5: Comparison of Join Algorithms ####

For all datasets tested, SBJ outperforms BNLJ, 46.3% better when λ=0.9 and 91.3% better when λ=0.01. SBJ saves both I/Os on the inner array and the CPU cost by reducing the number of tuples to be filtered. 

SBJ’s performance is not sensitive to large variance tuples when λ=0.9, but shows an increase in cost when λ=0.01, because many more tuples will be returned as join results.


### Case Study using SDSS Datasets ###

#### Expt 6: Storage ####

Store- multiple configured with step size <1,1> by our cost model incurs much less storage cost than the index schemes, and it approximates store-mean but with much better performance. Specifically, over 79% tuples have only 1 copy and over 92% tuples have at most 3 copies.

#### Expt 7: Subarray ####

Comparison to results of synthetic data: Store-mean with fences is orders of magnitude slower than other schemes when q ≤ 1%, and is 7 times slower than store-multiple when q =10%. 

Comparison to index-based methods: store-multiple is 1.7 - 4.3 times faster than U-index and 8.3 - 18.3 times faster than G-index. In contrast, probing on-disk indexes incurs tremendous leaf I/Os due to the nature of multi-path search in R-tree based indexes. Based on profiling numbers, when we vary q from 0.01% to 10%, the accessed child nodes averaged over all non-leaf nodes are in the range of (10.33%, 52.93%) for U-index and (22.13%, 66.29%) for G-index.

#### Expt 8: Structure-join ####

Comparison to Index Join: IBJ with U-index works poorly, 1 to 2 orders of magnitude worse than SBJ. Based on profiling results for 1.89M tuples, the index I/O dominates. With store-mean, the large-variance tuples lead to large range queries on the inner and most (even all) of the leaf nodes are accessed. With the (rare) existence of such tuples, the average number of U-index leaf nodes accessed per outer tuple is 38.9. Further, such tuples destroy the locality of caching for the same reason. For the 76830 outer tuples, with a 91.2% cache hit rate, the amount of leaf I/Os is already 263498, more than 10 times worse than SBJ. In contrast, store- multiple addresses large variance tuples with replication and finds a good tradeoff between tuple replication and query expansion. In addition, SBJ puts a tight predicate on the inner array to read relevant inner cells, and utilizes the memory to form blocks of outer tuples so that many of them can share the inner I/Os. As such, SBJ largely preserves the data locality that the array database provides for the access to dimension attributes. To verity this, we compared SBJ with an ideal case where each relevant inner chunk (i.e., con-taining the join candidates for some outer tuple) is visited exactly once. SBJ approximates the ideal case with 1.6x-2x I/Os.

Comparison to BNLJ: The difference between SBJ and BNLJ is magnified as BNLJ scans the whole inner array for each outer block, which is much bigger in SDSS than the synthetic datasets.

Scalability: SBJ scales the best among the three. For the same outer block and the same δ, the subarray region formed us- ing Proposition 4.3 is exactly the same. However, bigger datasets have more tuples with large possible ranges and increased chance of having tuple copies in formed subarray regions, which results in a modest increase of the cost.

Thanks for viewing! Don't forget following me on <a href="https://github.com/Princever">GitHub</a>!