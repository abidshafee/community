# Keras categorical inputs

| Status        | Accepted                                             |
:-------------- |:---------------------------------------------------- |
| **Author(s)** | Zhenyu Tan (tanzheny@google.com), Francois Chollet (fchollet@google.com)|
| **Sponsor**   | Karmel Allison (karmel@google.com), Martin Wicke (wicke@google.com) |
| **Updated**   | 2019-02-22                                           |

## Objective

This document proposes 4 new preprocessing Keras layers (`IndexLookup`, `CategoryCrossing`, `CategoryEncoding`, `Hashing`), and an extension to existing op (`tf.sparse.from_dense`) to allow users to:
* Perform feature engineering for categorical inputs
* Replace feature columns and `tf.keras.layers.DenseFeatures` with proposed layers
* Introduce sparse inputs that work with Keras linear models and other layers that support sparsity

Other proposed layers for replacement of feature columns such as `tf.feature_column.bucketized_column` and `tf.feature_column.numeric_column` has been discussed [here](https://github.com/keras-team/governance/blob/master/rfcs/20190502-preprocessing-layers.md).

The proposed layers should support ragged tensors.

## Motivation

Specifically, by introducing the 4 layers, we aim to address these pain points:
* Users have to define both feature columns and Keras Inputs for the model, resulting in code duplication and deviation from DRY (Do not repeat yourself) principle. See this [Github issue](https://github.com/tensorflow/tensorflow/issues/27416).
* Users with large dimension categorical inputs will incur large memory footprint and computation cost, if wrapped with indicator column through `tf.keras.layers.DenseFeatures`.
* Currently there is no way to correctly feed Keras linear model or dense layer with multivalent categorical inputs or weighted categorical inputs.

## User Benefit

We expect to get rid of the user painpoints once migrating off feature columns.

## Example Workflows

Two example workflows are presented below. These workflows can be found at this [colab](https://colab.sandbox.google.com/drive/1cEJhSYLcc2MKH7itwcDvue4PfvrLN-OR#scrollTo=22sa0D19kxXY).

### Workflow 1

The first example gives an equivalent code snippet to canned `LinearEstimator` [tutorial](https://www.tensorflow.org/tutorials/estimator/linear) on the Titanic dataset:

```python
dftrain = pd.read_csv('https://storage.googleapis.com/tf-datasets/titanic/train.csv')
y_train = dftrain.pop('survived')

CATEGORICAL_COLUMNS = ['sex', 'n_siblings_spouses', 'parch', 'class', 'deck', 'embark_town', 'alone']
NUMERICAL_COLUMNS = ['age', 'fare']
# input list to create functional model.
model_inputs = []
# input list to feed linear model.
linear_inputs = []
for feature_name in CATEGORICAL_COLUMNS:
  feature_input = tf.keras.Input(shape=(1,), dtype=tf.string, name=feature_name, sparse=True)
  vocab_list = sorted(dftrain[feature_name].unique())
  # Map string values to indices
  x = tf.keras.layers.IndexLookup(vocabulary=vocab_list, name=feature_name)(feature_input)
  x = tf.keras.layers.CategoryEncoding(num_categories=len(vocab_list))(x)
  linear_inputs.append(x)
  model_inputs.append(feature_input)

for feature_name in NUMERICAL_COLUMNS:
  feature_input = tf.keras.Input(shape=(1,), name=feature_name)
  linear_inputs.append(feature_input)
  model_inputs.append(feature_input)

linear_model = tf.keras.experimental.LinearModel(units=1)
linear_logits = linear_model(linear_inputs)
model = tf.keras.Model(model_inputs, linear_logits)

model.compile('sgd', loss=tf.keras.losses.BinaryCrossEntropy(from_logits=True), metrics=['accuracy'])

dataset = tf.data.Dataset.from_tensor_slices((
  (tf.sparse.from_dense(dftrain.sex, "Unknown"), tf.sparse.from_dense(dftrain.n_siblings_spouses, -1),
  tf.sparse.from_dense(dftrain.parch, -1), tf.sparse.from_dense(dftrain['class'], "Unknown"), tf.sparse.from_dense(dftrain.deck, "Unknown"),
  tf.expand_dims(dftrain.age, axis=1), tf.expand_dims(dftrain.fare, axis=1)),
  y_train)).batch(bach_size).repeat(n_epochs)

model.fit(dataset)
```

### Workflow 2

The second example gives an instruction on how to transition from categorical feature columns to the proposed layers. Note that one difference for vocab categorical column is that, instead of providing a pair of mutually exclusive `default_value` and `num_oov_buckets` where `default_value` represents the value to map input to given out-of-vocab value, and `num_oov_buckets` represents value range of [len(vocab), len(vocab)+num_oov_buckets) to map input to from a hashing function given out-of-vocab value. In practice, we believe out-of-vocab values should be mapped to the head, i.e., [0, num_oov_tokens), and in-vocab values should be mapped to [num_oov_tokens, num_oov_tokens+len(vocab)).

1. Categorical vocab list column

Original:
```python
fc = tf.feature_column.categorical_feature_column_with_vocabulary_list(
     key, vocabulary_list, dtype, default_value, num_oov_buckets)
```
Proposed:
```python
x = tf.keras.Input(shape=(1,), name=key, dtype=dtype)
layer = tf.keras.layers.IndexLookup(
            vocabulary=vocabulary_list, num_oov_tokens=num_oov_buckets)
out = layer(x)
```

2. categorical vocab file column

Original:
```python
fc = tf.feature_column.categorical_column_with_vocab_file(
       key, vocabulary_file, vocabulary_size, dtype,
       default_value, num_oov_buckets)
```
Proposed:
```python
x = tf.keras.Input(shape=(1,), name=key, dtype=dtype)
layer = tf.keras.layers.IndexLookup(
            vocabulary=vocabulary_file, num_oov_tokens=num_oov_buckets)
out = layer(x)
```
Note: `vocabulary_size` is only valid if `adapt` is called. Otherwise if user desires to lookup for the first K vocabularies in vocab file, then shrink the vocab file by only having the first K lines.

3. categorical hash column

Original:
```python
fc = tf.feature_column.categorical_column_with_hash_bucket(
       key, hash_bucket_size, dtype)
```
Proposed:
```python
x = tf.keras.Input(shape=(1,), name=key, dtype=dtype)
layer = tf.keras.layers.Hashing(num_bins=hash_bucket_size)
out = layer(x)
```

4. categorical identity column

Original:
```python
fc = tf.feature_column.categorical_column_with_identity(
       key, num_buckets, default_value)
```
Proposed:
```python
x = tf.keras.Input(shape=(1,), name=key, dtype=dtype)
layer = tf.keras.layers.Lambda(lambda x: tf.where(tf.logical_or(x < 0, x > num_buckets), tf.fill(dims=tf.shape(x), value=default_value), x))
out = layer(x)
```

5. cross column

Original:
```python
fc_1 = tf.feature_column.categorical_column_with_vocabulary_list(key_1, vocabulary_list, 
         dtype, default_value, num_oov_buckets)
fc_2 = tf.feature_column.categorical_column_with_hash_bucket(key_2, hash_bucket_size,
         dtype)
fc = tf.feature_column.crossed_column([fc_1, fc_2], hash_bucket_size, hash_key)
```
Proposed:
```python
x1 = tf.keras.Input(shape=(1,), name=key_1, dtype=dtype)
x2 = tf.keras.Input(shape=(1,), name=key_2, dtype=dtype)
layer1 = tf.keras.layers.IndexLookup(
           vocabulary=vocabulary_list,  
           num_oov_tokens=num_oov_buckets)
x1 = layer1(x1)
layer2 = tf.keras.layers.Hashing(
           num_bins=hash_bucket_size)
x2 = layer2(x2)
layer = tf.keras.layers.CategoryCrossing(num_bins=hash_bucket_size)
out = layer([x1, x2])
```

6. weighted categorical column

Original:
```python
fc = tf.feature_column.categorical_column_with_vocab_list(key, vocabulary_list,
         dtype, default_value, num_oov_buckets)
weight_fc = tf.feature_column.weighted_categorical_column(fc, weight_feature_key, 
         dtype=weight_dtype)
linear_model = tf.estimator.LinearClassifier(units, feature_columns=[weight_fc])
```
Proposed:
```python
x1 = tf.keras.Input(shape=(1,), name=key, dtype=dtype)
x2 = tf.keras.Input(shape=(1,), name=weight_feature_key, dtype=weight_dtype)
layer = tf.keras.layers.IndexLookup(
           vocabulary=vocabulary_list,   
           num_oov_tokens=num_oov_buckets)
x1 = layer(x1)
x = tf.keras.layers.CategoryEncoding(num_categories=len(vocabulary_list)+num_oov_buckets)([x1, x2])
linear_model = tf.keras.premade.LinearModel(units)
linear_logits = linear_model(x)
```

## Design Proposal
We propose a `IndexLookup` layer to replace `tf.feature_column.categorical_column_with_vocabulary_list` and `tf.feature_column.categorical_column_with_vocabulary_file`, a `Hashing` layer to replace `tf.feature_column.categorical_column_with_hash_bucket`, a `CategoryCrossing` layer to replace `tf.feature_column.crossed_column`, and another `CategoryEncoding` layer to convert the sparse input to the format required by linear models.

```python
`tf.keras.layers.IndexLookup`
IndexLookup(PreprocessingLayer):
"""This layer transforms categorical inputs to index space.
   If input is dense/sparse, then output is dense/sparse."""

  def __init__(self, max_tokens=None, num_oov_tokens=1, vocabulary=None,
               name=None, **kwargs):
    """Constructs a IndexLookup layer.

    Args:
      max_tokens: The maximum size of the vocabulary for this layer. If None,
              there is no cap on the size of the vocabulary. This is used when `adapt`
              is called.
      num_oov_tokens: Non-negative integer. The number of out-of-vocab tokens. 
              All out-of-vocab inputs will be assigned IDs in the range of 
              [0, num_oov_tokens) based on a hash. When
              `vocabulary` is None, it will convert inputs in [0, num_oov_tokens)
      vocabulary: the vocabulary to Lookup the input. If it is a file, specify the file
              path to represent the  source vocab file, example 'tmp/vocab_file.txt';
              If it is a list/tuple, it represents the source vocab list, example '[A, B, C]';
              If it is None, the vocabulary can later be set.
      name: Name to give to the layer.
     **kwargs: Keyword arguments to construct a layer.

    Input: a string or int tensor of shape `[batch_size, d1, ..., dm]`
    Output: an int tensor of shape `[batch_size, d1, ..., dm]`

    Example:
      If one input sample is `["a", "c", "d", "a", "x"]` and the vocabulary is ["a", "b", "c", "d"],
      and a single OOV token is used (`num_oov_tokens=1`), then the corresponding output sample is
      `[1, 3, 4, 1, 0]`. 0 stands for an OOV token.
    """
    pass

`tf.keras.layers.CategoryCrossing`
CategoryCrossing(PreprocessingLayer):
"""This layer transforms multiple categorical inputs to categorical outputs
   by Cartesian product, and hash the output if necessary.
   If any of the inputs is sparse, then all outputs will be sparse. Otherwise, all outputs will be dense."""

  def __init__(self, depth=None, num_bins=None, name=None, **kwargs):
    """Constructs a CategoryCrossing layer.
    Args:
      depth: depth of input crossing. By default None, all inputs are crossed
             into one output. It can be an int or tuple/list of ints, where inputs are
             combined into all combinations of output with degree of `depth`. For example,
             with inputs `a`, `b` and `c`, `depth=2` means the output will be [ab;ac;bc]
      num_bins: Number of hash bins. By default None, no hashing is performed.
      name: Name to give to the layer.
      **kwargs: Keyword arguments to construct a layer.

    Input: a list of int tensors of shape `[batch_size, d1, ..., dm]`
    Output: a single int tensor of shape `[batch_size, d1, ..., dm]`

    Example:
      If the layer receives two inputs, `a=[[1, 2]]` and `b=[[1, 3]]`, and
      if depth is 2, then the output will be a string tensor `[[b'1_X_1',
      b'1_X_3', b'2_X_1', b'2_X_3']]` if not hashed, or integer tensor
      `[[hash(b'1_X_1'), hash(b'1_X_3'), hash(b'2_X_1'), hash(b'2_X_3')]]` if hashed.
    """
    pass

`tf.keras.layers.CategoryEncoding`
CategoryEncoding(PreprocessingLayer):
"""This layer transforms categorical inputs from index space to category space.
   If input is dense/sparse, then output is dense/sparse."""

  def __init__(self, num_categories, mode="count", axis=-1, sparse_out=True, name=None, **kwargs):
    """Constructs a CategoryEncoding layer.
    Args:
      num_categories: Number of elements in the vocabulary.
      mode: how to reduce a categorical input if multivalent, can be one of "count",  
          "avg_count", "binary", "tfidf". It can also be None if this is not a multivalent input,
          and simply needs to convert input from index space to category space. "tfidf" is only
          valid when adapt is called on this layer.
      axis: the axis to reduce, by default will be the last axis, specially true 
          for sequential feature columns.
      sparse_out: boolean to indicate whether the output should be dense or sparse tensor.
      name: Name to give to the layer.
     **kwargs: Keyword arguments to construct a layer.

    Input: a int tensor of shape `[batch_size, d1, ..., dm-1, dm]`
    Output: a float tensor of shape `[batch_size, d1, ..., dm-1, num_categories]`

    Example:
      If the input is 2 by 2 dense integer tensor '[[0, 2], [2, 2]]' with `num_categories=3`, then
      output is 2 by 3 dense integer tensor '[[1, 0, 1], [0, 0, 2]]' with a `count` encoding, or
      dense float tensor '[[.5, 0, .5], [0, 0, 1.]]' with a `avg_count` encoding, or dense integer tensor
      '[[1, 0, 1], [0, 0, 1]]' with a `binary` encoding.
    """
    pass

`tf.keras.layers.Hashing`
Hashing(PreprocessingLayer):
"""This layer transforms categorical inputs to hashed output.
   If input is dense/sparse, then output is dense/sparse."""
  def __init__(self, num_bins, name=None, **kwargs):
    """Constructs a Hashing layer.

    Args:
      num_bins: Number of hash bins.
      name: Name to give to the layer.
      **kwargs: Keyword arguments to construct a layer.

    Input: a int tensor of shape `[batch_size, d1, ..., dm]`
    Output: a int tensor of shape `[batch_size, d1, ..., dm]`

    Example:
      If the input is a 5 by 1 string tensor '[['A'], ['B'], ['C'], ['D'], ['E']]' with `num_bins=2`,
      then output is 5 by 1 integer tensor '[[0], [0], [1], [1], [0]]', i.e.,
      [[hash('A')], [hash('B')], [hash('C')], [hash('D')], [hash('E')]].
    """
    pass

```

We also propose to extend the current `tf.sparse.from_dense` op with a `ignore_value` to convert dense tensors to sparse tensors given user specified ignore values. This op can be used in both `tf.data` or [TF Transform](https://www.tensorflow.org/tfx/transform/get_started). In previous feature column world, "" is ignored for dense string input and -1 is ignored for dense int input.

```python
`tf.sparse.from_dense`
def from_dense(tensor, name=None, ignore_value=0):
  """Convert dense/sparse tensor to sparse while dropping user specified values.

  Args:
    tensor: A dense `Tensor` to be converted to a `SparseTensor`.
    name: Optional name for the op.
    ignore_value: The value to be dropped from input. Default to 0 for backward compatibility.
  """
  pass
```

### Alternatives Considered
An alternative is to provide solutions on top of feature columns. This will make user code to be slightly cleaner but far less flexible.

### Performance Implications
End to End benchmark should be same as other preprocessing layers.

### Dependencies
This proposal does not add any new dependencies.

### Engineering Impact
These changes will include more layers and thus binary size and build time. It will not impact startup time.
This code can be tested in its own and maintained in its own buildable unit.

### Platforms and Environments
This proposal should work in all platforms and environments.

### Best Practices, Tutorials and Examples
This proposal does not change the best engineering practices.

### Compatibility
No backward compatibility issues.

### User Impact
User facing changes to migrate feature column based Keras modeling to preprocessing layer based Keras modeling, as the example workflow suggests.


## Code Snippets

Below is a more detailed illustration of how each layer works. If there is a vocabulary list of countries:
```python
vocabulary_list = ["Italy", "France", "England", "Austria", "Germany"]
inp = np.asarray([["Italy", "Italy"], ["Germany", ""]])
sp_inp = tf.sparse.from_dense(inp, ignore_value="")
cat_layer = tf.keras.layers.IndexLookup(vocabulary=vocabulary_list)
sp_out = cat_layer(sp_inp)
```

The categorical layer will first convert the input to:
```python
sp_out.indices = <tf.Tensor: id=8, shape=(3, 2), dtype=int64, numpy=
                 array([[0, 0], [0, 1] [1, 0]])>
sp_out.values = <tf.Tensor: id=28, shape=(3,), dtype=int64,        
                 numpy=array([0, 0, 4])>
```

The `CategoryEncoding` layer will then convert the input from index space to category space, e.g., from a sparse tensor with indices shape as [batch_size, n_columns] and values in the range of [0, n_categories) to a sparse tensor with indices shape as [batch_size, n_categories] and values as the frequency of each value that occured in the example:
```python
encoding_layer = CategoryEncoding(num_categories=len(vocabulary_list))
sp_encoded_out = encoding_layer(sp_out)
sp_encoded_out.indices = <tf.Tensor: id=8, shape=(2, 2), dtype=int64, numpy=
                              array([[0, 0], [1, 4]])>
sp_encoded_out.values = <tf.Tensor: id=28, shape=(3,), dtype=int64,        
                             numpy=array([2., 1.])>
```
A weight input can also be passed into the layer if different categories/examples should be treated differently.

If this input needs to be crossed with another categorical input, say a vocabulary list of days, then use `CategoryCrossing` which works in the same way as `tf.feature_column.crossed_column` without setting `depth`:
```python
days = ["Monday", "Tuesday", "Wednesday", "Thursday", "Friday", "Saturday", "Sunday"]
inp_days = tf.sparse.from_dense(np.asarray([["Sunday"], [""]]), ignore_value="")
layer_days = IndexLookup(vocabulary=days)
sp_out_2 = layer_days(inp_days)

sp_out_2.indices = <tf.Tensor: id=161, shape=(1, 2), dtype=int64, numpy=array([[0, 0]])>
sp_out_2.values = <tf.Tensor: id=181, shape=(1,), dtype=int64, numpy=array([6])>

cross_layer = CategoryCrossing(num_bins=5)
# Use the output from IndexLookup (sp_out), not CategoryEncoding (sp_combined_out)
crossed_out = cross_layer([sp_out, sp_out_2])

cross_out.indices = <tf.Tensor: id=186, shape=(2, 2), dtype=int64, numpy=
                        array([[0, 0], [0, 1]])>
cross_out.values = <tf.Tensor: id=187, shape=(2,), dtype=int64, numpy=array([3, 3])>
```

## Questions and Meeting Notes
We'd like to gather feedbacks on `IndexLookup`, specifically we propose migrating off from mutually exclusive `num_oov_buckets` and `default_value` and replace with `num_oov_tokens`.
1. Naming for encoding v.s. vectorize: encoding can mean many things, vectorize seems to general. We will go with "CategoryEncoding"
2. "mode" should be "count" or "avg_count", instead of "sum" and "mean".
3. Rename "sparse_combiner" to "mode", which aligns with scikit-learn.
4. Have a 'sparse_out' flag for "CategoryEncoding" layer.
5. Hashing -- we refer to hashing when we mean fingerprinting. Keep using "Hashing" for layer name, but document how it relies on tf.fingerprint, and also provides option for salt.
5. Rename "CategoryLookup" to "IndexLookup"