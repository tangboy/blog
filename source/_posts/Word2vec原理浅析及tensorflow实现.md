---
title: Word2vec原理浅析及tensorflow实现
date: 2017-09-10 14:10:14
tags:
     - Word2vec
     - tensorflow
---

## Word2vec简介
Word2Vec是由Google的Mikolov等人提出的一个词向量计算模型。

* 输入：大量已分词的文本
* 输出：用一个稠密向量来表示每个词

词向量的重要意义在于将自然语言转换成了计算机能够理解的向量。相对于词袋模型、TF-IDF等模型，词向量能抓住词的上下文、语义，衡量词与词的相似性，在文本分类、情感分析等许多自然语言处理领域有重要作用。

## Word2vec详细实现
word2vec的详细实现，简而言之，就是一个三层的神经网络。要理解word2vec的实现，需要的预备知识是神经网络和Logistic Regression。

### 神经网络结构

{% asset_img 1.png %}

上图是Word2vec的简要流程图。首先假设，词库里的词数为10000; 词向量的长度为300（根据[斯坦福CS224d](https://www.youtube.com/playlist?list=PLlJy-eBtNFt4CSVWYqscHDdP58M3zFHIG)的讲解，词向量一般为25-1000维，300维是一个好的选择）。下面以单个训练样本为例，依次介绍每个部分的含义。

1. 输入层：输入为一个词的one-hot向量表示。这个向量长度为10000。假设这个词为ants，ants在词库中的ID为i，则输入向量的第i个分量为1，其余为0。[0, 0, ..., 0, 0, 1, 0, 0, ..., 0, 0]
2. 隐藏层：隐藏层的神经元个数就是词向量的长度。隐藏层的参数是一个[10000 ，300]的矩阵。 **实际上，这个参数矩阵就是词向量**。回忆一下矩阵相乘，一个one-hot行向量和矩阵相乘，结果就是矩阵的第i行。经过隐藏层，实际上就是把10000维的one-hot向量映射成了最终想要得到的300维的词向量。

{% asset_img 2.png %}

3. 输出层: 输出层的神经元个数为总词数10000，参数矩阵尺寸为[300，10000]。词向量经过矩阵计算后再加上softmax归一化，重新变为10000维的向量，每一维对应词库中的一个词与输入的词（在这里是ants）共同出现在上下文中的概率。

{% asset_img 3.png %}

上图中计算了car与ants共现的概率，car所对应的300维列向量就是输出层参数矩阵中的一列。输出层的参数矩阵是[300，10000]，也就是计算了词库中所有词与ants共现的概率。输出层的参数矩阵在训练完毕后没有作用。
4. 训练：训练样本（x, y）有输入也有输出，我们知道哪个词实际上跟ants共现，因此y也是一个10000维的向量。损失函数跟Logistic Regression相似，是神经网络的最终输出向量和y的交叉熵（cross-entropy）。最后用随机梯度下降来求解。

{% asset_img 4.png %}

上述步骤是一个词作为输入和一个上下文中的词作为输出的情况，但实际情况显然更复杂，什么是上下文呢？用一个词去预测周围的其他词，还是用周围的好多词来预测一个词？这里就要引入实际训练时的两个模型skip-gram和CBOW。
<!--more-->
### skip-gram和CBOW
* skip-gram： 核心思想是根据中心词来预测周围的词。假设中心词是cat，窗口长度为2，则根据cat预测左边两个词和右边两个词。这时，cat作为神经网络的input，预测的词作为label。下图为一个例子：

{% asset_img 5.png %}

在这里窗口长度为2，中心词一个一个移动，遍历所有文本。每一次中心词的移动，最多会产生4对训练样本（input，label）。

* CBOW（continuous-bag-of-words）：如果理解了skip-gram，那CBOW模型其实就是倒过来，用周围的所有词来预测中心词。这时候，每一次中心词的移动，只能产生一个训练样本。如果还是用上面的例子，则CBOW模型会产生下列4个训练样本：
    1. ([quick, brown], the)
    2. ([the, brown, fox], quick)
    3. ([the, quick, fox, jumps], brown)
    4. ([quick, brown, jumps, over], fox)
这时候，input很可能是4个词，label只是一个词，怎么办呢？其实很简单，只要求平均就行了。经过隐藏层后，输入的4个词被映射成了4个300维的向量，对这4个向量求平均，然后就可以作为下一层的输入了。

两个模型相比，skip-gram模型能产生更多训练样本，抓住更多词与词之间语义上的细节，在语料足够多足够好的理想条件下，skip-gram模型是优于CBOW模型的。在语料较少的情况下，难以抓住足够多词与词之间的细节，CBOW模型求平均的特性，反而效果可能更好。

### 负采样（Negative Sampling）
实际训练时，还是假设词库有10000个词，词向量300维，那么每一层神经网络的参数是300万个，输出层相当于有一万个可能类的多分类问题。可以想象，这样的计算量非常非常非常大。
作者Mikolov等人提出了许多优化的方法，在这里着重讲一下负采样。
负采样的思想非常简单，简单地令人发指：我们知道最终神经网络经过softmax输出一个向量，只有一个概率最大的对应正确的单词，其余的称为negative sample。现在只选择5个negative sample，所以输出向量就只是一个6维的向量。要考虑的参数不是300万个，而减少到了1800个！ 这样做看上去很偷懒，实际效果却很好，大大提升了运算效率。
我们知道，训练神经网络时，每一次训练会对神经网络的参数进行微小的修改。在word2vec中，每一个训练样本并不会对所有参数进行修改。假设输入的词是cat，我们的隐藏层参数有300万个，但这一步训练只会修改cat相对应的300个参数，因为此时隐藏层的输出只跟这300个参数有关！
负采样是有效的，我们不需要那么多negative sample。Mikolov等人在论文中说：对于小数据集，负采样的个数在5-20个；对于大数据集，负采样的个数在2-5个。

那具体如何选择负采样的词呢？论文给出了如下公式：

{% asset_img 6.png %}

其中f(w)是词频。可以看到，负采样的选择只跟词频有关，词频越大，越有可能选中。

### Tensorflow实现
最后用tensorflow动手实践一下

这里只是训练了128维的词向量，并通过[TSNE](http://scikit-learn.org/stable/modules/generated/sklearn.manifold.TSNE.html)的方法可视化。作为练手和深入理解word2vec不错，实战还是推荐gensim。

```python
# These are all the modules we'll be using later. Make sure you can import them
# before proceeding further.
%matplotlib inline
from __future__ import print_function
import collections
import math
import numpy as np
import os
import random
import tensorflow as tf
import zipfile
from matplotlib import pylab
from six.moves import range
from six.moves.urllib.request import urlretrieve
from sklearn.manifold import TSNE
```

Download the data from the source website if necessary.

```python
url = 'http://mattmahoney.net/dc/'

def maybe_download(filename, expected_bytes):
  """Download a file if not present, and make sure it's the right size."""
  if not os.path.exists(filename):
    filename, _ = urlretrieve(url + filename, filename)
  statinfo = os.stat(filename)
  if statinfo.st_size == expected_bytes:
    print('Found and verified %s' % filename)
  else:
    print(statinfo.st_size)
    raise Exception(
      'Failed to verify ' + filename + '. Can you get to it with a browser?')
  return filename

filename = maybe_download('text8.zip', 31344016)
```

Read the data into a string.

```python
def read_data(filename):
  """Extract the first file enclosed in a zip file as a list of words"""
  with zipfile.ZipFile(filename) as f:
    data = tf.compat.as_str(f.read(f.namelist()[0])).split()
  return data

words = read_data(filename)
print('Data size %d' % len(words))
```

Build the dictionary and replace rare words with UNK token.

```python
vocabulary_size = 50000

def build_dataset(words):
  count = [['UNK', -1]]
  count.extend(collections.Counter(words).most_common(vocabulary_size - 1))
  dictionary = dict()
  for word, _ in count:
    dictionary[word] = len(dictionary)
  data = list()
  unk_count = 0
  for word in words:
    if word in dictionary:
      index = dictionary[word]
    else:
      index = 0  # dictionary['UNK']
      unk_count = unk_count + 1
    data.append(index)
  count[0][1] = unk_count
  reverse_dictionary = dict(zip(dictionary.values(), dictionary.keys()))
  return data, count, dictionary, reverse_dictionary

data, count, dictionary, reverse_dictionary = build_dataset(words)
print('Most common words (+UNK)', count[:5])
print('Sample data', data[:10])
del words  # Hint to reduce memory.
```

Function to generate a training batch for the skip-gram model.

```python
data_index = 0

def generate_batch(batch_size, num_skips, skip_window):
    global data_index
    assert batch_size % num_skips == 0
    assert num_skips <= 2 * skip_window
    batch = np.ndarray(shape=(batch_size), dtype=np.int32)
    labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)
    span = 2 * skip_window + 1 # [ skip_window target skip_window ]
    buffer = collections.deque(maxlen=span)
    for _ in range(span):
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
    for i in range(batch_size // num_skips):
        target = skip_window  # target label at the center of the buffer
        targets_to_avoid = [ skip_window ]
        for j in range(num_skips):
            while target in targets_to_avoid:
                target = random.randint(0, span - 1)
            targets_to_avoid.append(target)
            batch[i * num_skips + j] = buffer[skip_window]
            labels[i * num_skips + j, 0] = buffer[target]
        buffer.append(data[data_index])
        data_index = (data_index + 1) % len(data)
    return batch, labels

print('data:', [reverse_dictionary[di] for di in data[:8]])

for num_skips, skip_window in [(2, 1), (4, 2)]:
    data_index = 0
    batch, labels = generate_batch(batch_size=8, num_skips=num_skips, skip_window=skip_window)
    print('\nwith num_skips = %d and skip_window = %d:' % (num_skips, skip_window))
    print('    batch:', [reverse_dictionary[bi] for bi in batch])
    print('    labels:', [reverse_dictionary[li] for li in labels.reshape(8)])

```

#### Skip\-Gram
Train a skip\-gram model.
```python
batch_size = 128
embedding_size = 128 # Dimension of the embedding vector.
skip_window = 1 # How many words to consider left and right.
num_skips = 2 # How many times to reuse an input to generate a label.
# We pick a random validation set to sample nearest neighbors. here we limit the
# validation samples to the words that have a low numeric ID, which by
# construction are also the most frequent.
valid_size = 16 # Random set of words to evaluate similarity on.
valid_window = 100 # Only pick dev samples in the head of the distribution.
valid_examples = np.array(random.sample(range(valid_window), valid_size))

#######important#########
num_sampled = 64 # Number of negative examples to sample.

graph = tf.Graph()

with graph.as_default(), tf.device('/cpu:0'):

  # Input data.
  train_dataset = tf.placeholder(tf.int32, shape=[batch_size])
  train_labels = tf.placeholder(tf.int32, shape=[batch_size, 1])
  valid_dataset = tf.constant(valid_examples, dtype=tf.int32)

  # Variables.
  embeddings = tf.Variable(
    tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0))
  softmax_weights = tf.Variable(
    tf.truncated_normal([vocabulary_size, embedding_size],
                         stddev=1.0 / math.sqrt(embedding_size)))
  softmax_biases = tf.Variable(tf.zeros([vocabulary_size]))

  # Model.
  # Look up embeddings for inputs.
  embed = tf.nn.embedding_lookup(embeddings, train_dataset)
  # Compute the softmax loss, using a sample of the negative labels each time.
  loss = tf.reduce_mean(
    tf.nn.sampled_softmax_loss(weights=softmax_weights, biases=softmax_biases, inputs=embed,
                               labels=train_labels, num_sampled=num_sampled, num_classes=vocabulary_size))

  # Optimizer.
  # Note: The optimizer will optimize the softmax_weights AND the embeddings.
  # This is because the embeddings are defined as a variable quantity and the
  # optimizer's `minimize` method will by default modify all variable quantities
  # that contribute to the tensor it is passed.
  # See docs on `tf.train.Optimizer.minimize()` for more details.
  optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)

  # Compute the similarity between minibatch examples and all embeddings.
  # We use the cosine distance:
  norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keep_dims=True))
  normalized_embeddings = embeddings / norm
  valid_embeddings = tf.nn.embedding_lookup(
    normalized_embeddings, valid_dataset)
  similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
```

```python
num_steps = 100001

with tf.Session(graph=graph) as session:
  tf.global_variables_initializer().run()
  print('Initialized')
  average_loss = 0
  for step in range(num_steps):
    batch_data, batch_labels = generate_batch(
      batch_size, num_skips, skip_window)
    feed_dict = {train_dataset : batch_data, train_labels : batch_labels}
    _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    average_loss += l
    if step % 2000 == 0:
      if step > 0:
        average_loss = average_loss / 2000
      # The average loss is an estimate of the loss over the last 2000 batches.
      print('Average loss at step %d: %f' % (step, average_loss))
      average_loss = 0
    # note that this is expensive (~20% slowdown if computed every 500 steps)
    if step % 10000 == 0:
      sim = similarity.eval()
      for i in range(valid_size):
        valid_word = reverse_dictionary[valid_examples[i]]
        top_k = 8 # number of nearest neighbors
        nearest = (-sim[i, :]).argsort()[1:top_k+1]
        log = 'Nearest to %s:' % valid_word
        for k in range(top_k):
          close_word = reverse_dictionary[nearest[k]]
          log = '%s %s,' % (log, close_word)
        print(log)
  final_embeddings = normalized_embeddings.eval()
```

```python
num_points = 400

tsne = TSNE(perplexity=30, n_components=2, init='pca', n_iter=5000)
two_d_embeddings = tsne.fit_transform(final_embeddings[1:num_points+1, :])

def plot(embeddings, labels):
  assert embeddings.shape[0] >= len(labels), 'More labels than embeddings'
  pylab.figure(figsize=(15,15))  # in inches
  for i, label in enumerate(labels):
    x, y = embeddings[i,:]
    pylab.scatter(x, y)
    pylab.annotate(label, xy=(x, y), xytext=(5, 2), textcoords='offset points',
                   ha='right', va='bottom')
  pylab.show()

words = [reverse_dictionary[i] for i in range(1, num_points+1)]
plot(two_d_embeddings, words)
```

{% asset_img 7.png %}

#### CBOW
```python
data_index_cbow = 0

def get_cbow_batch(batch_size, num_skips, skip_window):
    global data_index_cbow
    assert batch_size % num_skips == 0
    assert num_skips <= 2 * skip_window
    batch = np.ndarray(shape=(batch_size), dtype=np.int32)
    labels = np.ndarray(shape=(batch_size, 1), dtype=np.int32)
    span = 2 * skip_window + 1 # [ skip_window target skip_window ]
    buffer = collections.deque(maxlen=span)
    for _ in range(span):
        buffer.append(data[data_index_cbow])
        data_index_cbow = (data_index_cbow + 1) % len(data)
    for i in range(batch_size // num_skips):
        target = skip_window  # target label at the center of the buffer
        targets_to_avoid = [ skip_window ]
        for j in range(num_skips):
            while target in targets_to_avoid:
                target = random.randint(0, span - 1)
            targets_to_avoid.append(target)
            batch[i * num_skips + j] = buffer[skip_window]
            labels[i * num_skips + j, 0] = buffer[target]
        buffer.append(data[data_index_cbow])
        data_index_cbow = (data_index_cbow + 1) % len(data)
    cbow_batch = np.ndarray(shape=(batch_size), dtype=np.int32)
    cbow_labels = np.ndarray(shape=(batch_size // (skip_window * 2), 1), dtype=np.int32)
    for i in range(batch_size):
        cbow_batch[i] = labels[i]
    cbow_batch = np.reshape(cbow_batch, [batch_size // (skip_window * 2), skip_window * 2])
    for i in range(batch_size // (skip_window * 2)):
        # center word
        cbow_labels[i] = batch[2 * skip_window * i]
    return cbow_batch, cbow_labels
```

```python
# actual batch_size = batch_size // (2 * skip_window)
batch_size = 128
embedding_size = 128 # Dimension of the embedding vector.
skip_window = 1 # How many words to consider left and right.
num_skips = 2 # How many times to reuse an input to generate a label.
# We pick a random validation set to sample nearest neighbors. here we limit the
# validation samples to the words that have a low numeric ID, which by
# construction are also the most frequent.
valid_size = 16 # Random set of words to evaluate similarity on.
valid_window = 100 # Only pick dev samples in the head of the distribution.
valid_examples = np.array(random.sample(range(valid_window), valid_size))

#######important#########
num_sampled = 64 # Number of negative examples to sample.

graph = tf.Graph()

with graph.as_default(), tf.device('/cpu:0'):

  # Input data.
    train_dataset = tf.placeholder(tf.int32, shape=[batch_size // (skip_window * 2), skip_window * 2])
    train_labels = tf.placeholder(tf.int32, shape=[batch_size // (skip_window * 2), 1])
    valid_dataset = tf.constant(valid_examples, dtype=tf.int32)

  # Variables.
    embeddings = tf.Variable(
      tf.random_uniform([vocabulary_size, embedding_size], -1.0, 1.0))
    softmax_weights = tf.Variable(
      tf.truncated_normal([vocabulary_size, embedding_size],
                         stddev=1.0 / math.sqrt(embedding_size)))
    softmax_biases = tf.Variable(tf.zeros([vocabulary_size]))

  # Model.
  # Look up embeddings for inputs.
    embed = tf.nn.embedding_lookup(embeddings, train_dataset)

    # reshape embed
    embed = tf.reshape(embed, (skip_window * 2, batch_size // (skip_window * 2), embedding_size))
    # average embed
    embed = tf.reduce_mean(embed, 0)

  # Compute the softmax loss, using a sample of the negative labels each time.
    loss = tf.reduce_mean(
      tf.nn.sampled_softmax_loss(weights=softmax_weights, biases=softmax_biases, inputs=embed,
                               labels=train_labels, num_sampled=num_sampled, num_classes=vocabulary_size))

  # Optimizer.
  # Note: The optimizer will optimize the softmax_weights AND the embeddings.
  # This is because the embeddings are defined as a variable quantity and the
  # optimizer's `minimize` method will by default modify all variable quantities
  # that contribute to the tensor it is passed.
  # See docs on `tf.train.Optimizer.minimize()` for more details.
    optimizer = tf.train.AdagradOptimizer(1.0).minimize(loss)

  # Compute the similarity between minibatch examples and all embeddings.
  # We use the cosine distance:
    norm = tf.sqrt(tf.reduce_sum(tf.square(embeddings), 1, keep_dims=True))
    normalized_embeddings = embeddings / norm
    valid_embeddings = tf.nn.embedding_lookup(
      normalized_embeddings, valid_dataset)
    similarity = tf.matmul(valid_embeddings, tf.transpose(normalized_embeddings))
```

```python
num_steps = 100001

with tf.Session(graph=graph) as session:
  tf.global_variables_initializer().run()
  print('Initialized')
  average_loss = 0
  for step in range(num_steps):
    batch_data, batch_labels = get_cbow_batch(
      batch_size, num_skips, skip_window)
    feed_dict = {train_dataset : batch_data, train_labels : batch_labels}
    _, l = session.run([optimizer, loss], feed_dict=feed_dict)
    average_loss += l
    if step % 2000 == 0:
      if step > 0:
        average_loss = average_loss / 2000
      # The average loss is an estimate of the loss over the last 2000 batches.
      print('Average loss at step %d: %f' % (step, average_loss))
      average_loss = 0
    # note that this is expensive (~20% slowdown if computed every 500 steps)
    if step % 10000 == 0:
      sim = similarity.eval()
      for i in range(valid_size):
        valid_word = reverse_dictionary[valid_examples[i]]
        top_k = 8 # number of nearest neighbors
        nearest = (-sim[i, :]).argsort()[1:top_k+1]
        log = 'Nearest to %s:' % valid_word
        for k in range(top_k):
          close_word = reverse_dictionary[nearest[k]]
          log = '%s %s,' % (log, close_word)
        print(log)
  final_embeddings = normalized_embeddings.eval()
```

```python
num_points = 400

tsne = TSNE(perplexity=30, n_components=2, init='pca', n_iter=5000)
two_d_embeddings = tsne.fit_transform(final_embeddings[1:num_points+1, :])
words = [reverse_dictionary[i] for i in range(200, num_points+1)]
plot(two_d_embeddings, words)

```

{% asset_img 8.png %}
