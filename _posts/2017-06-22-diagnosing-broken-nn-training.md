---
layout: post
title: "Diagnosing Broken Neural Network Training: A Small Case Study"
date: 2017-06-22
comments: true
---

## Background

Recently, I've been catching up on my practical neural network knowledge, first
by going through Michael Nielson's [Neural Networks and Deep Learning](
http://neuralnetworksanddeeplearning.com/) online book and then the [fast.ai
course](http://course.fast.ai/).

After implementing a toy version of a
classic neural network consisting only of fully-connected (i.e. dense) layers
for making classifications on the [MNIST database of handwritten digits](
http://yann.lecun.com/exdb/mnist/), I figured I'd take a stab at implementing
a new neural network for the same task using [Keras](https://keras.io/) and
enter the results into the [Kaggle Digit Recognizer competition](https://www.
kaggle.com/c/digit-recognizer).

There are quite a few guides to creating an MNIST handwritten digit classifier
using Keras, but the one I primarily followed was an [article by Jason
Brownlee](http://machinelearningmastery.com/handwritten-digit-recognition-using-
convolutional-neural-networks-python-keras/).

## The Model
For reference, the original convolutional model I attempted to play with was
written with Keras as follows:

```
model = Sequential()
model.add(Conv2D(32, (5, 5), input_shape=(1, 28, 28), activation='relu'))
model.add(MaxPooling2D(pool_size=(2, 2)))
model.add(Dropout(0.2))
model.add(Flatten())
model.add(Dense(128, activation='relu'))
model.add(Dense(num_classes, activation='softmax'))
model.compile(loss='categorical_crossentropy', optimizer='adam', metrics=['accuracy'])
```

## The Initial Problem

After finishing my first draft of the implementation, I ran into an error when
attempting to train the model. Executing the statement
```
model.fit(X_train, Y_train, validation_split=0.1, epochs=10, batch_size=200,
verbose=2)
```
yielded the error
```
ValueError: Error when checking target: expected dense_2 to have shape (None,
10) but got array with shape (60000, 1)
```

Because the error was related to the second layer of the model, which was a
dense layer meant for forming classifications, I figured the problem lay
somewhere in the label data. I began looking into the label data with some
cowboy diagnostics at the point right before training.

```
Y_train = np_utils.to_categorical(Y_train)
num_classes = Y_test.shape[1]

# Display some diagnostic data.
print(Y_train.shape)
print(num_classes)

# Train model.
model.fit(X_train, Y_train, validation_split=0.1, epochs=10, batch_size=200,
          verbose=2)
```

Not surprisingly, `Y_train.shape` yielded `(60000, 1)`, which meant that it was
not being split into a dedicated dimension per class as expected for the output
from [`np_utils.to_categorical()`](https://keras.io/utils/
#to_categorical); the proper shape should have been `(60000, 10)`. However, the
test label array `Y_test` was returning the expected shape of `(10000, 10)`,
which was perplexing.

Upon further digging, after passing through `np_utils.to_categorical()`, the
value of `Y_train` was just an array of zeroes. I eventually attempted to force
`np_utils.to_categorical()` to return a 60000 x 10 array by explicitly passing
in the number of classes through `np_utils.to_categorical(Y_train, 10)`, which
allowed training to proceed, but yielded another problem.

## The Network Isn't Training

While the network did not seem to be learning after the first training epoch,
the network's accuracy was not only 100% after each training epoch, but the
overall loss and accuracy were the exact same between every training session,
even with many different tweaks to the model. This seemed very wrong!

Because the probability of the model being the problem was unlikely, I began to
look at the underlying data entering into the model. The first step was
to investigate whether the original data checked out. Because I had been running
into problems with converting the label data into the proper dimensions, I first
looked into the label data (i.e. `Y_train`).

```
# Load MNIST data.
(X_train, Y_train), (X_test, Y_test) = mnist.load_data()

# Display label data array shape.
print(Y_train.shape)

# Count the number of occurrences of each class label.
data_labels = {
    0: 0,
    1: 0,
    2: 0,
    3: 0,
    4: 0,
    5: 0,
    6: 0,
    7: 0,
    8: 0,
    9: 0,
}
for label in Y_train:
    data_labels[label] += 1

# Display some diagnostic data.
print(data_labels)
print(sum(data_labels.values())
```

I used a dictionary of "buckets" to count the occurrence of each class in the
labels. When displaying the counts of each class, they seemed reasonably
distributed, and the total number of labels amounted to the proper size: 60,000.
No surprises there - the source data itself looked okay.

I then proceeded to look at the label data again at the point right before
calling `model.fit()`. Although the shape was correct, `Y_train` was now an
array where all the labels were set to 0 (i.e. `[1., 0., 0., 0., 0., 0., 0., 0.,
0., 0.]`). Not seeing where the source of the issue was, I figured it might be
the way the data was transformed by `np_utils.to_categorical()`, so I wrote a
patch to get around it:

```
    tmp = np.zeros((Y_train.shape[0], 10))
    for i, y in enumerate(Y_train):
        for j in range(10):
            if j == y:
                tmp[i][j] = 1.0
    Y_train = tmp
```

Checking `Y_train` afterward yielded an array that looked exactly like I needed
it to, but it still did not change training results! Nor did it change the
apparent values held in `Y_train` right before the call to `model.fit()`.
Scratching my head, I checked on anything that might have modified `Y_train`
**before** passing it through `np_utils.to_categorical()`. Lo-and-behold! There
was a line I mindlessly fudged by accidentally normalizing the training labels
instead of the testing input:

```
X_train = X_train / 255
Y_train = Y_train / 255
```
instead of
```
X_train = X_train / 255
X_test = X_test / 255
```

After fixing that, the network began training normally without any issues!

## Conclusion

This wasn't a particularly profound problem, and the error was quite stupid,
but hopefully this might help give others insight into diagnosing bugs
with their own neural networks. It might also serve to make you feel less bad
about the your own frustrations and mistakes.
