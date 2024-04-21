# PISA Example

This is an example of indexing and running retrieval experiments using the PISA engine (https://github.com/pisa-engine/pisa).

This example follows the pipeline depicted in the official documentation: https://pisa-engine.github.io/pisa/book/guide/indexing-pipeline.html

___
#### Data
**Collection:** Wikilarge (http://dg3rtljvitrle.cloudfront.net/wiki-large.tar.gz). For convenience we put all documents in a single file in TREC format.

**Queries:** MQT (Million Query Track) 1K sample (https://trec.nist.gov/data/million.query.html).

___

Directory structure for the example (the names are self-explanatory of their contents). The commands bellow refer to file locations inside this structure.

```.
./collection/
./index/
./index/1_forward/
./index/2_inverted/
./index/3_compressed/
./queries/
./runs/
```



#### Building forward index
```
cat ./collection/wikilarge.trec | ~/pisa/build/bin/parse_collection -j 12 -f trectext -F porter2 --html -o ./index/1_forward/wikilarge.fwd
```
#### Inverting forward index
```
~/pisa/build/bin/invert -j 12 -i ./index/1_forward/wikilarge.fwd -o ./index/2_inverted/wikilarge.inv --term-count `wc -w < ./index/1_forward/wikilarge.fwd.terms`
```

#### Creating wand data file
```
../pisa/build/bin/create_wand_data -s bm25 --bm25-b 0.4 --bm25-k1 0.9 --block-size 128 -c  ./index/2_inverted/wikilarge.inv --compress --quantize 8 -o ./index/3_compressed/wikilarge.inv.wand
```

#### Compressing the index 
(We select the SIMD-BP codec. See: https://pisa-engine.github.io/pisa/book/guide/compressing.html)
```
../pisa/build/bin/compress_inverted_index -c ./index/2_inverted/wikilarge.inv -e block_simdbp -o ./index/3_compressed/wikilarge.inv.simdbp --check
```

**Note:** There is a version of the index inside ```./index/3_compressed/``` diretcory which is (besides) compressed using ´´´gzip´´´ to avoid surpassing 50 Mb (Github's max file size suggestion). ´´´Gunzip´´´ it if you want to run tests without reindexing the collection.


#### Reordering docids using graph bisection algorithm 
(We use the recursive graph bisection algorithm. See: https://pisa-engine.github.io/pisa/book/guide/reordering.html)
```
../pisa/build/bin/reorder-docids --bp --collection ./index/2_inverted/wikilarge.inv --output ./index/2_inverted/wikilarge.inv.bp 
```

#### Compressing the reordered index (~7% size reduction for this example)
```
../pisa/build/bin/compress_inverted_index -c ./index/2_inverted/wikilarge.inv.bp -e block_simdbp -o ./index/3_compressed/wikilarge.inv.bp.simdbp --check
```

#### Prepare queries (stem queries using porter2 algorithm and map to term-ids from the index. 
(Terms not in lexicon are discarded and some queries may become empty)
```
~/pisa/build/bin/map_queries -q queries/MQT_1Kqueries.sample.stemmed -F porter2 --terms ./index/1_forward/wikilarge.fwd.termlex --query-id > queries/MQT_1Kqueries.sample.stemmed.mapped
```

#### Timing queries (note that boolean queries do not require the use of the wand file). 
For global times exclude the ```--extract``` option
```
../pisa/build/bin/queries -s bm25 -e block_simdbp -k 10 -a and -i ./index/3_compressed/wikilarge.inv.simdbp -q queries/MQT_1Kqueries.sample.stemmed.mapped --extract > runs/MQT_on_wikilarge.AND.k10.times
```
```
../pisa/build/bin/queries -s bm25 -e block_simdbp -k 10 -a maxscore -i ./index/3_compressed/wikilarge.inv.simdbp --wand ./index/3_compressed/wikilarge.inv.wand --compressed-wand -q queries/MQT_1Kqueries.sample.stemmed.mapped --extract > runs/MQT_on_wikilarge.MS.k10.times
```

#### Retrieving results
```
../pisa/build/bin/evaluate_queries -s bm25 -e block_simdbp -k 10 -a maxscore -i ./index/3_compressed/wikilarge.inv.simdbp --wand ./index/3_compressed/wikilarge.inv.wand --compressed-wand --documents ./index/1_forward/wikilarge.fwd.doclex -q queries/MQT_1Kqueries.sample.stemmed.mapped > runs/MQT_on_wikilarge.MS.k10.results
```
___

More information, check the project page: https://github.com/pisa-engine/pisa




