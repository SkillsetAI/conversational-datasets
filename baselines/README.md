# Baseline methods for conversational response selection

This directory provides several baseline methods for conversational response selection. These help to benchmark the performance on the various datasets.

Baselines are intended to be relatively fast to run locally, and are not intended to be highly competitive with state of the art methods. As such they are limited to using a small portion of the training set, typically ten thousand randomly sampled examples.

Note that baselines only use the `context` feature to rank the `response`, and do not take into account `extra_context`s.


## Keyword-based

The [keyword-based baselines](keyword_based.py) use keyword similarity metrics to rank responses given a context. The `TF_IDF` method computes inverse document frequency statistics on the training set. Responses are scored using their [tf-idf cosine similarity](https://janav.wordpress.com/2013/10/27/tf-idf-and-cosine-similarity/) to the context.

The `BM25` method builds on top of the tf-idf similarity, applying an adjustment to the term weights. See [Okapi BM25: a non-binary model](http://nlp.stanford.edu/IR-book/html/htmledition/okapi-bm25-a-non-binary-model-1.html) for further discussion of the approach.


## Vector-based

The [vector-based methods](vector_based.py) use publicly available neural network embedding models to embed contexts and responses into a vector space. The models implemented currently are:

* [USE](https://tfhub.dev/google/universal-sentence-encoder/2) - the universal sentence encoder
* [USE_LARGE](https://tfhub.dev/google/universal-sentence-encoder-large/3) - a larger version of the universal sentence encoder
* [ELMO](https://tfhub.dev/google/elmo/1) - the Embeddings from Language Models approach

all of which are loaded from Tensorflow Hub.


There are two vector-based baseline methods, one for each of the above models. The `SIM` method ranks responses according to their cosine similarity with the context vector. This method does not use the training set at all.

The `MAP` method learns a linear mapping on top of the response vector. The final score of a response with vector <img alt="y" src="https://latex.codecogs.com/svg.latex?\mathbf{y}"> given a context with vector <img alt="x" src="https://latex.codecogs.com/svg.latex?\mathbf{x}"> is the cosine similarity of the context vector with the mapped response vector:

<img alt="the cosine similarity of x and y prime" src="https://latex.codecogs.com/svg.latex?%5Cfrac%7B%5Cmathbf%7Bx%7D%5Ccdot%5Cmathbf%7By%7D%27%7D%7B%5Cleft%5C%7C%5Cmathbf%7Bx%7D%5Cright%5C%7C%20%5Cleft%5C%7C%5Cmathbf%7By%7D%27%5Cright%5C%7C%7D">

where

<img alt="y' is y after a linear mapping" src="https://latex.codecogs.com/svg.latex?%5Cmathbf%7By%7D%27%3D(W%2B%5Calpha%20I)%5Ccdot%5Cmathbf%7By%7D" />

and <img alt="W and alpha" src="https://latex.codecogs.com/svg.latex?W,\:\alpha"> are learned parameters. This allows for learning an arbitrary linear mapping on the context side, while making it easy for the model to interpolate with the `SIM` baseline using the residual connection gated by ![alpha](https://latex.codecogs.com/svg.latex?\alpha). Vectors are L2-normalized before being fed to the MAP method, so that the method is invariant to scaling.

The parameters are learned on the training set, using the dot product loss from [Henderson et al 2017](https://arxiv.org/abs/1705.00652). A sweep over learning rate and regularization parameters is performed using a held-out dev set. The final learned parameters are used on the evaluation set.

The combination of the three embedding models with the two vector-based methods gives six baseline methods: `USE_SIM`, `USE_MAP`, `USE_LARGE_SIM`, `USE_LARGE_MAP`, `ELMO_SIM`, and `ELMO_MAP`.

# Running the baselines

## Copy the data locally

Before running the baselines, it may be beneficial to copy the data locally. This will speed up reading the data, but it is not necessary if you run these on a Google cloud machine in the correct region.

If the dataset is stored in `$DATADIR`, a Google cloud storage path such as `gs://your-bucket/reddit/YYYYMMDD`, then copy a shard of the training set and the test set:

```bash
mkdir data
gsutil cp ${DATADIR?}/train-00001-* data/
gsutil cp ${DATADIR?}/test-00001-* data/
```

For Amazon QA data, you will need to copy two shards of the test set to get enough examples.

This provides a random subset of the train and test set to use for the baselines. Recall that conversational datasets are always randomly shuffled and sharded.

## Run the baselines

When running vector-based methods, make use of tensorflow hub's caching to speed up results:

```bash
export TFHUB_CACHE_DIR=~/.tfhub_cache
```

Then run an individual baseline with:

```bash
python baselines/run_baseline.py  \
  --method TF_IDF \
  --train_dataset data/train-* \
  --test_dataset data/test-*
```

Note that the `USE_LARGE` and `ELMO` baselines are slow, and may benefit from faster hardware. For these methods set `--eval_num_batches 100`.

The [`run_baselines.sh`](run_baselines.sh) script runs all the baselines on all the datasets.
