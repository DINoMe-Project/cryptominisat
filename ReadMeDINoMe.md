Welcome to the DINoMeCounter wiki!
# Preparing
Follow [README.markdown](https://github.com/ziqiaozhou/DINoMeCounter/blob/master/README.markdown) to install dependencies

# Compiling
Run 
```
mkdir build;
cd build;
sh ../cmake.sh;
make;
```
It might take several minutes. You could use make -j$numthreads to speed up.

# Usage
Now you have several tools to use: 
1. cryptominisat5: similar to the msoos's cryptominisat5.
2. cmsat5-count/count: a tool to count the union and intersection set used to compute jaccard distance. 
3. cmsat5-count/sampler: a tool to generate samples of solutions for interpretable model. 
4. cmsat5-compose/compose: a tool to composite multi-cycle / multi-copy CNF

## input CNF file
We customized the CNF parser to read attacker-controlled (C), secret(S), other inputs(I),  observable outputs(O).
An example CNF inputs (input.cnf):
```
p cnf 8 3
c control --> [ -1 2 ]
c secret --> [ 3 4 ]
c observe --> [ 5 6 ]
c other --> [ 7 8 ]
1 3 5 0
4 6 0
-4 -6 0
```

## compose
We first compose a CNF with S,S',I,I',O,O' by running 
`./cmsat5-compose/compose --compose_mode=copy --keepsymbol=1  --cycles=2 \
--memoutmult=5000 --composedfile=count.cnf  --simplify_interval=1  \
 --out=./ --preschedule="renumber" input.cnf tmp.cnf`

It would outputs count.cnf
```
p cnf 14 6
c control --> [ -1 2]
c observe_0 --> [ 5 6]
c observe_1 --> [ 11 12]
c other_0 --> [ 7 8]
c other_1 --> [ 13 14]
c secret_0 --> [ 3 -6]
c secret_1 --> [ 9 -12]
c --------- unit clauses
c ------------ vars appearing inverted in cls
c -------- irred bin cls
4 6 0
-4 -6 0
10 12 0
-10 -12 0
c -------- irred long cls
1 9 11 0
1 3 5 0
c ------------ equivalent literals
```

## Counting
An example cmd to estimate the sizes of union and intersection set is 
```
./cmsat5-count/count count.cnf --num_xor_cls=${xor} --max_count_times=${ntimes} --xor_ratio=0.5 \
--record_solution=1 --count_out=${xor}.count --out=$outdir \
--nsample=${nsample} --max_sol=40 --min_log_size=0  --max_xor_per_var=10 \
--debug=0  --total_call_timeout=200 --one_call_timeout=50
```
* num_xor_cls: the size of hash function used to sampling secret set S and S'. 
* record_solution: whether to record solution into a file
* count_out: storing counting size into ${xor}.count
* total_call_timeout: timeout threshold for all solver calls in one iteration
* one_call_timeout: timeout threshold for one solver call

For the given example, we can directly output the exact count value as the set size is small. However, when set size is large, we should tune several parameters:
* max_sol: maximum number of solution to search for in one iteration.
* min_log_size: minimum hash size used to estimate counting set size, by default: 0
* max_xor_per_var: maximum hash size used to estimate counting set size, by default: -1 (maximum)

## Sampling
1. Sampling for noninterference set
```
cmsat5-count/sampler count.cnf --num_xor_cls ${xor} --xor_ratio=0.4 --max_sol=500 --record_solution=1 --count_out=${xor}_relax --out=$dir --nsample=100 --noninter=-1
cmsat5-count/sampler count.cnf  --num_xor_cls ${xor} --xor_ratio=0.4 --max_sol=500 --record_solution=1 --count_out=${xor}_strict --out=$dir  --nsample=${nsample} --noninter=1
```
2. Sampling for interference set
```
cmsat5-count/sampler count.cnf  --num_xor_cls ${xor} --xor_ratio=0.4 --max_sol=500 --record_solution=1 --count_out=${xor} --out=$dir --nsample=${nsample} --noninter=0 
```
