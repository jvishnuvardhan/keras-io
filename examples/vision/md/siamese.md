# Siamese network with a contrastive loss

**Author:** Mehdi<br>
**Date created:** 2021/05/06<br>
**Last modified:** 2021/05/06<br>
**Description:** Similarity learning using siamese network with contrastive loss.


<img class="k-inline-icon" src="https://colab.research.google.com/img/colab_favicon.ico"/> [**View in Colab**](https://colab.research.google.com/github/keras-team/keras-io/blob/master/examples/vision/ipynb/siamese.ipynb)  <span class="k-dot">•</span><img class="k-inline-icon" src="https://github.com/favicon.ico"/> [**GitHub source**](https://github.com/keras-team/keras-io/blob/master/examples/vision/siamese.py)



---
## Introduction

[Siamese Network](https://en.wikipedia.org/wiki/Siamese_neural_network)
is any Neural Network which share weights between two or more sister networks,
each producing embedding vector of its respective input and these embeddings
are then passed through some
[distance heuristic](https://developers.google.com/machine-learning/clustering/similarity/measuring-similarity)
to find the distance between them. This distance is later used to increase the
contrast between embeddings of inputs of different classes and decrease it with
that of similar class by employing some loss function, with the main objective
of contrasting [vector spaces](https://en.wikipedia.org/wiki/Vector_space)
from which these sample inputs were taken.

---
## Setup


```python
import random
import numpy as np
import tensorflow as tf
from tensorflow import keras
from tensorflow.keras import layers
import matplotlib.pyplot as plt
```

Define some hyperparameters


```python
epochs = 10
batch_size = 16
# margin for constrastive loss
margin = 1
```

---
## Load the MNIST dataset


```python
(x_train_val, y_train_val), (x_test, y_test) = keras.datasets.mnist.load_data()

# Change the data type to a floating point format
x_train_val = x_train_val.astype("float32")
x_test = x_test.astype("float32")

```

Use list slicing to split train_val data into `train` and `val`


```python
# Keep 50% of train_val  in validation set
x_train, x_val = x_train_val[:30000], x_train_val[30000:]
y_train, y_val = y_train_val[:30000], y_train_val[30000:]
del x_train_val, y_train_val

```

---
## Create pairs of images

We will train the model to differentiate each digit from one another. For
example, digit `0` needs to be differentiated from the rest of the
digits (`1` through `9`), digit `1` - from `0` and `2` through `9`, and so on.
To carry this out, we will select N random images from class A (for example,
for digit `0`) and pair it with N random images from another class B
(for example, for digit `1`). Then, we can repeat this process for all classes
of digits (until digit `9`). Once we have paired digit `0` with other digits,
we can repeat this process for the remaining classes for the rest of the digits
(from `1` until `9`).


```python

def make_pairs(x, y):
    """Creates a tuple containing image pairs with corresponding label.

    Arguments:
        x: List containing images, each index in this list corresponds to one image.
        y: List containing labels, each label with datatype of `int`.

    Returns:
        Tuple containing two numpy arrays as (pairs_of_samples, labels),
        where pairs_of_samples' shape is (2len(x), 2,n_features_dims) and
        labels are a binary array of shape (2len(x)).
    """

    num_classes = max(y) + 1
    digit_indices = [np.where(y == i)[0] for i in range(num_classes)]

    pairs = []
    labels = []

    for idx1 in range(len(x)):
        # add a matching example
        x1 = x[idx1]
        label1 = y[idx1]
        idx2 = random.choice(digit_indices[label1])
        x2 = x[idx2]

        pairs += [[x1, x2]]
        labels += [1]

        # add a non-matching example
        label2 = random.randint(0, num_classes - 1)
        while label2 == label1:
            label2 = random.randint(0, num_classes - 1)

        idx2 = random.choice(digit_indices[label2])
        x2 = x[idx2]

        pairs += [[x1, x2]]
        labels += [0]

    return np.array(pairs), np.array(labels).astype("float32")


# make train pairs
pairs_train, labels_train = make_pairs(x_train, y_train)

# make validation pairs
pairs_val, labels_val = make_pairs(x_val, y_val)

# make test pairs
pairs_test, labels_test = make_pairs(x_test, y_test)
```

**pairs_train.shape = (60000, 2, 28, 28)**

Imagine it as:

**pairs_train.shape = (60000, pair.shape)**


`pairs_train` contains 60K `pairs` in `axis 0`, shape of each pair
is (2,28,28) hence `each pair` of `pairs_train` contains one image in its
`axis 0` (do not confuse it with the `axis 0` of `pairs_train`) and the
other one in the `axis 1`. We will slice `pairs_train` on its `axix 0`
followed by desired axis of pair to obtain all images (60K) which belong
either to the `axis 0` or the `axis 1` of all the pairs of `pairs_train`.


**Note:** Do not confuse axes of `pairs_train` with those of
`pair within pairs_train`, `pairs_train` have only one axis `axis 0` which
contain 60K pairs, whereas each `pair within pairs_train` have two axis,
each for one image of a pair.

Separate train pairs


```python
x_train_1 = pairs_train[:, 0]
x_train_2 = pairs_train[:, 1]
# x_train_1.shape = (60000, 28, 28)
```

Separate validation pairs


```python
x_val_1 = pairs_val[:, 0]
x_val_2 = pairs_val[:, 1]
# x_val_1.shape = (60000, 28, 28)
```

Separate test pairs


```python
x_test_1 = pairs_test[:, 0]
x_test_2 = pairs_test[:, 1]
# x_test_1.shape = (20000, 28, 28)

```

---
## Visualize


```python

def visualize(pairs, labels, to_show=6, num_col=3, predictions=None, test=False):
    """Creates a plot of pairs and labels, and prediction if it's test dataset.

    Arguments:
        pairs: Numpy Array, of pairs to visualize, having shape
               (Number of pairs, 2, 28, 28).
        to_show: Int, number of examples to visualize (default is 6)
                `to_show` must be an integral multiple of `num_col`.
                 Otherwise it will be trimmed if it is greater than num_col,
                 and incremented if if it is less then num_col.
        num_col: Int, number of images in one row - (default is 3)
                 For test and train respectively, it should not exceed 3 and 7.
        predictions: Numpy Array of predictions with shape (to_show, 1) -
                     (default is None)
                     Must be passed when test=True.
        test: Boolean telling whether the dataset being visualized is
              train dataset or test dataset - (default False).

    Returns:
        None.
    """

    # Define num_row
    # If to_show % num_col != 0
    #    trim to_show,
    #       to trim to_show limit num_row to the point where
    #       to_show % num_col == 0
    #
    # If to_show//num_col == 0
    #    then it means num_col is greater then to_show
    #    increment to_show
    #       to increment to_show set num_row to 1
    num_row = to_show // num_col if to_show // num_col != 0 else 1

    # `to_show` must be an integral multiple of `num_col`
    #  we found num_row and we have num_col
    #  to increment or decrement to_show
    #  to make it integral multiple of `num_col`
    #  simply set it equal to num_row * num_col
    to_show = num_row * num_col

    # Plot the images
    fig, axes = plt.subplots(num_row, num_col, figsize=(5, 5))
    for i in range(to_show):

        # If the number of rows is 1, the axes array is one-dimensional
        if num_row == 1:
            ax = axes[i % num_col]
        else:
            ax = axes[i // num_col, i % num_col]

        ax.imshow(tf.concat([pairs[i][0], pairs[i][1]], axis=1), cmap="gray")
        ax.set_axis_off()
        if test:
            ax.set_title("True: {} | Pred: {:.5f}".format(labels[i], predictions[i][0]))
        else:
            ax.set_title("Label: {}".format(labels[i]))
    if test:
        plt.tight_layout(rect=(0, 0, 1.9, 1.9), w_pad=0.0)
    else:
        plt.tight_layout(rect=(0, 0, 1.5, 1.5))
    plt.show()

```

Inspect train pairs


```python
visualize(pairs_train[:-1], labels_train[:-1], to_show=4, num_col=4)
```


    
![png](/img/examples/vision/siamese/siamese_22_0.png)
    


Inspect validation pairs


```python
visualize(pairs_val[:-1], labels_val[:-1], to_show=4, num_col=4)
```


    
![png](/img/examples/vision/siamese/siamese_24_0.png)
    


Inspect test pairs


```python
visualize(pairs_test[:-1], labels_test[:-1], to_show=4, num_col=4)
```


    
![png](/img/examples/vision/siamese/siamese_26_0.png)
    


---
## Define the model

There will be two input layers, each leading to its own network, which
produces embeddings. Lambda layer will merge them using
[Euclidean distance](https://en.wikipedia.org/wiki/Euclidean_distance) and the
merged layer will be fed to final network.


```python
# Provided two tensors t1 and t2
# Euclidean distance = sqrt(sum(square(t1-t2)))
def euclidean_distance(vects):
    """Find the Euclidean distance between two vectors.

    Arguments:
        vects: List containing two tensors of same length.

    Returns:
        Tensor containing euclidean distance
        (as floating point value) between vectors.
    """

    x, y = vects
    sum_square = tf.math.reduce_sum(tf.math.square(x - y), axis=1, keepdims=True)
    return tf.math.sqrt(tf.math.maximum(sum_square, tf.keras.backend.epsilon()))


input = layers.Input((28, 28, 1))
x = tf.keras.layers.BatchNormalization()(input)
x = layers.Conv2D(4, (5, 5), activation="tanh")(x)
x = layers.AveragePooling2D(pool_size=(2, 2))(x)
x = layers.Conv2D(16, (5, 5), activation="tanh")(x)
x = layers.AveragePooling2D(pool_size=(2, 2))(x)
x = layers.Flatten()(x)

x = tf.keras.layers.BatchNormalization()(x)
x = layers.Dense(10, activation="tanh")(x)
embedding_network = keras.Model(input, x)


input_1 = layers.Input((28, 28, 1))
input_2 = layers.Input((28, 28, 1))

# As mentioned above, Siamese Network share weights between
# tower networks (sister networks). To allow this, we will use
# same embedding network for both tower networks.
tower_1 = embedding_network(input_1)
tower_2 = embedding_network(input_2)

merge_layer = layers.Lambda(euclidean_distance)([tower_1, tower_2])
normal_layer = tf.keras.layers.BatchNormalization()(merge_layer)
output_layer = layers.Dense(1, activation="sigmoid")(normal_layer)
siamese = keras.Model(inputs=[input_1, input_2], outputs=output_layer)

```

Define Constrastive Loss


```python

def loss(margin=1):
    """Provides 'constrastive_loss' an enclosing scope with variable 'margin'.

  Arguments:
      margin: Integer, defines the baseline for distance for which pairs
              should be classified as dissimilar. - (default is 1).

  Returns:
      'constrastive_loss' function with data ('margin') attached.
  """

    # Contrastive loss = mean( (1-true_value) * square(prediction) +
    #                         true_value * square( max(margin-prediction, 0) ))
    def contrastive_loss(y_true, y_pred):
        """Calculates the constrastive loss.

      Arguments:
          y_true: List of labels, each label is of type float32.
          y_pred: List of predictions of same length as of y_true,
                  each label is of type float32.

      Returns:
          A tensor containing constrastive loss as floating point value.
      """

        square_pred = tf.math.square(y_pred)
        margin_square = tf.math.square(tf.math.maximum(margin - (y_pred), 0))
        return tf.math.reduce_mean(
            (1 - y_true) * square_pred + (y_true) * margin_square
        )

    return contrastive_loss

```

Compile the model with constrastive loss


```python
siamese.compile(loss=loss(margin=margin), optimizer="RMSprop", metrics=["accuracy"])
siamese.summary()

```

<div class="k-default-codeblock">
```
Model: "model_1"
__________________________________________________________________________________________________
Layer (type)                    Output Shape         Param #     Connected to                     
==================================================================================================
input_2 (InputLayer)            [(None, 28, 28, 1)]  0                                            
__________________________________________________________________________________________________
input_3 (InputLayer)            [(None, 28, 28, 1)]  0                                            
__________________________________________________________________________________________________
model (Functional)              (None, 10)           5318        input_2[0][0]                    
                                                                 input_3[0][0]                    
__________________________________________________________________________________________________
lambda (Lambda)                 (None, 1)            0           model[0][0]                      
                                                                 model[1][0]                      
__________________________________________________________________________________________________
batch_normalization_2 (BatchNor (None, 1)            4           lambda[0][0]                     
__________________________________________________________________________________________________
dense_1 (Dense)                 (None, 1)            2           batch_normalization_2[0][0]      
==================================================================================================
Total params: 5,324
Trainable params: 4,808
Non-trainable params: 516
__________________________________________________________________________________________________

```
</div>
Train the model


```python
history = siamese.fit(
    [x_train_1, x_train_2],
    labels_train,
    validation_data=([x_val_1, x_val_2], labels_val),
    batch_size=batch_size,
    epochs=epochs,
)
```

<div class="k-default-codeblock">
```
Epoch 1/10
3750/3750 [==============================] - 42s 11ms/step - loss: 0.2559 - accuracy: 0.5061 - val_loss: 0.2341 - val_accuracy: 0.7966
Epoch 2/10
3750/3750 [==============================] - 44s 12ms/step - loss: 0.1015 - accuracy: 0.8817 - val_loss: 0.0368 - val_accuracy: 0.9526
Epoch 3/10
3750/3750 [==============================] - 44s 12ms/step - loss: 0.0538 - accuracy: 0.9293 - val_loss: 0.0250 - val_accuracy: 0.9686
Epoch 4/10
3750/3750 [==============================] - 43s 12ms/step - loss: 0.0436 - accuracy: 0.9436 - val_loss: 0.0214 - val_accuracy: 0.9726
Epoch 5/10
3750/3750 [==============================] - 43s 12ms/step - loss: 0.0381 - accuracy: 0.9507 - val_loss: 0.0183 - val_accuracy: 0.9761
Epoch 6/10
3750/3750 [==============================] - 44s 12ms/step - loss: 0.0371 - accuracy: 0.9518 - val_loss: 0.0182 - val_accuracy: 0.9766
Epoch 7/10
3750/3750 [==============================] - 46s 12ms/step - loss: 0.0342 - accuracy: 0.9557 - val_loss: 0.0167 - val_accuracy: 0.9787
Epoch 8/10
3750/3750 [==============================] - 54s 14ms/step - loss: 0.0328 - accuracy: 0.9577 - val_loss: 0.0166 - val_accuracy: 0.9786
Epoch 9/10
3750/3750 [==============================] - 44s 12ms/step - loss: 0.0316 - accuracy: 0.9593 - val_loss: 0.0157 - val_accuracy: 0.9800
Epoch 10/10
3750/3750 [==============================] - 43s 12ms/step - loss: 0.0302 - accuracy: 0.9614 - val_loss: 0.0149 - val_accuracy: 0.9813

```
</div>
---
## Visualize results


```python

def plt_metric(history, metric, title, has_valid=True):
    """Plots the given 'metric' from 'history'.

    Arguments:
        history: history attribute of History object returned from Model.fit.
        metric: Metric to plot, a string value present as key in 'history'.
        title: A string to be used as title of plot.
        has_valid: Boolean, true if valid data was passed to Model.fit
        	    else false.

    Returns:
        None.
    """
    plt.plot(history[metric])
    if has_valid:
        plt.plot(history["val_" + metric])
        plt.legend(["train", "validation"], loc="upper left")
    plt.title(title)
    plt.ylabel(metric)
    plt.xlabel("epoch")
    plt.show()


# Plot the accuracy
plt_metric(history=history.history, metric="accuracy", title="Model accuracy")

# Plot the constrastive loss
plt_metric(history=history.history, metric="loss", title="Constrastive Loss")
```


    
![png](/img/examples/vision/siamese/siamese_36_0.png)
    



    
![png](/img/examples/vision/siamese/siamese_36_1.png)
    


Evaluate the model


```python
results = siamese.evaluate([x_test_1, x_test_2], labels_test)
print("test loss, test acc:", results)
```

<div class="k-default-codeblock">
```
625/625 [==============================] - 4s 6ms/step - loss: 0.0141 - accuracy: 0.9818
test loss, test acc: [0.014080599881708622, 0.9817500114440918]

```
</div>
Visualize the predictions


```python
predictions = siamese.predict([x_test_1, x_test_2])
visualize(pairs_test, labels_test, to_show=3, predictions=predictions, test=True)
```


    
![png](/img/examples/vision/siamese/siamese_40_0.png)
    
