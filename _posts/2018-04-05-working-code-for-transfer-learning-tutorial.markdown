---
layout: post
title: "Working code for transfer learning tutorial"
---

The original code is from [this link](https://www.analyticsvidhya.com/blog/2017/06/transfer-learning-the-art-of-fine-tuning-a-pre-trained-model/), however the downloading and using of mnist data in its original form is quite tedious and troublesome. I rewrote this code to make it work with Keras mnist dataset

Some key take aways:

- Since our dataset is small, to do transfer learning we need to keep the bottom layers of VGG16, use them as a feature extractor. The result of these layers are used to feed into our MLP for learning, and we train this MLP alone without touching the bottom layers. However when one wants to do inference, of course he needs to feed the test data into the bottom layers first, then feeding the results into the new MLP for the final prediction

- The operation to turn an (28x28) mnist image/matrix into a (224x224) image/matrix is not a resize. Some method of resize will copy the original data to target matrix and then pad the rest of the target matrix with either 0 or copies of the original matrix. The first case is still fine, but the second case is worst if you forgot


{% highlight python %}
# importing required libraries
%matplotlib inline
from keras.models import Sequential
from scipy.misc import imread
get_ipython().magic('matplotlib inline')
import matplotlib.pyplot as plt
import numpy as np
import keras
from keras.layers import Dense
import pandas as pd
from keras.applications.vgg16 import VGG16
from keras.preprocessing import image
from keras.applications.vgg16 import preprocess_input
import numpy as np
from keras.applications.vgg16 import decode_predictions
from keras.layers import Dense, Activation
from sklearn.model_selection import train_test_split
import cv2

# loading mnist data
from keras.datasets import mnist

(x_train, y_train), (x_test, y_test) = mnist.load_data()

plt.imshow(x_train[0])

def convert_to_3channel(a):
    """
    Upscale mnist 28x28 image to 224x224 image and duplicate gray scale
    channel to fake it to RGB image
    @param a: np array of shape (28, 28)
    @return: np array of shape (224, 224, 3)
    """
    resized_a = cv2.resize(a, dsize=(224, 224), interpolation=cv2.INTER_CUBIC)
    stacked_a = np.stack((resized_a,)*3, -1)
    return stacked_a

#convert training data from grayscale 28x28 to 3 channel 224x224 by duplicating 1 channel 3 times
new_x_train = []
for x in x_train:
    new_x_train.append(convert_to_3channel(x))
new_x_train = np.asarray(new_x_train) #new_x_train has shape (60000, 224, 224, 3)

#this will show a similar intepolated image to the one above
plt.imshow(new_x_train[0])

#loading the vgg16 model
vgg16_model = VGG16(weights='imagenet', include_top=False)

#use the existing bottom layers of vgg16 for feature extraction
features_train = vgg16_model.predict(new_x_train) #shape = (60000, 7, 7, 512)

# flatten into 1-d array for feeding into a MLP network we are going to build later
train_x = features_train.reshape(60000, 7*7*512) # shape = (60000, 25088)

# our y train is still in shape (60000,), we need to turn it into 1-hot vectors 
# pandas has this convenient function for doing just that
train_y = pd.get_dummies(y_train) # shape = (60000, 10)

#split into train-test split
X_train, X_valid, Y_train, Y_valid=train_test_split(train_x,train_y,test_size=0.3, random_state=42)

#building the model
model = Sequential()
model.add(Dense(1000, input_dim=25088, activation='relu', kernel_initializer='uniform'))
model.add(keras.layers.core.Dropout(0.3, noise_shape=None, seed=None))
model.add(Dense(500,input_dim=1000,activation='sigmoid'))
model.add(keras.layers.core.Dropout(0.4, noise_shape=None, seed=None))
model.add(Dense(150,input_dim=500,activation='sigmoid'))
model.add(keras.layers.core.Dropout(0.2, noise_shape=None, seed=None))
model.add(Dense(units=10))
model.add(Activation('softmax'))
model.compile(loss='categorical_crossentropy', optimizer='adam', metrics=['accuracy'])

#start training!!!
model.fit(X_train, Y_train, epochs=20, batch_size=128, validation_data=(X_valid, Y_valid))

#to infer, remember that you will need to pass the input into the original vgg16 base layers first
test_input = convert_to_3channel(x_test[0]) # convert to 3-channel RGB
test_input = vgg16_model.predict(test_input) # run the input through base layers
test_input = test_input.reshape(1, 7*7*512) # flatten the output
model.predict(features_test[0].reshape(1, 7*7*512)) # feed i into our MLP

{% endhighlight %}
