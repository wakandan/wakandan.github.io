---
layout: post
title: "Working code for transfer learning tutorial"
---

The original code is from [this link](https://www.analyticsvidhya.com/blog/2017/06/transfer-learning-the-art-of-fine-tuning-a-pre-trained-model/), however the downloading and using of mnist data in its original form is quite tedious and troublesome. I rewrote this code to make it work with Keras mnist dataset

From the article, since our dataset is small, there are 2 approaches I could transfer learning from vgg16 model:

# Use vgg16 as a feature extractor

In this approach, we remove the last layer of vgg16, add on a MLP we built which has 10 output tensors (for 10 classes), and only retrain *this MLP* model. Concretely, we don't take features from MMIST input directly. We first feed MNIST input into our modified version of vgg16 to get a vector of feature of shape (1, 7\*7\*512). These vectors are then used as input to our MLP model, and with the MNIST output we can train this MLP. Similarly during inference time, we will also need to feed the test data into vgg16 first, then use run this input through our MLP to get the final result. The code is shown below.

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

- The operation to turn an (28x28) mnist image/matrix into a (224x224) image/matrix is not a resize. Some method of resize will copy the original data to target matrix and then pad the rest of the target matrix with either 0 or copies of the original matrix. The first case is still fine, but the second case is worst if you forgot

# Retrain vgg16 with modified last layer

Contrary to the previous approach where we only need to train the new MLP, in this approach we will re-train part of vgg16 network with our MNIST dataset. There are a few things to note:

- We will replace the last layer of vgg16 of 1000 classes with a new dense layer of 10 classes

- We do not train the whole vgg16 for obvious performance reason. We will keep the weights of first layers of vgg16 (in this case 8 layers) unchanged. Then we will retrain.

- In the first version, our input shape to MLP is (60000, 7\*7\*512), however in this version sicne we are using vgg16 input, it's going to be (60000, 224, 224, 3)

Below is the new code, some of them will be the same as in the previous version

{% highlight python %}
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
import cv2
from keras.datasets import mnist

(x_train, y_train), (x_test, y_test) = mnist.load_data()
new_x_train = []
for x in x_train:
    new_x_train.append(convert(x))
new_x_train = np.asarray(new_x_train)
new_y_train = pd.get_dummies(y_train)

X_train, X_valid, Y_train, Y_valid=train_test_split(new_x_train, new_y_train,test_size=0.2, random_state=42)

img_rows, img_cols = 224, 224 # Resolution of inputs
channel = 3
num_classes = 10 
batch_size = 16 
nb_epoch = 10

def create_vgg16_model(img_rows, img_cols, channel=1, num_classes=None):
    model = VGG16(weights='imagenet', include_top=True)
    model.layers.pop()
    x=Dense(num_classes, activation='softmax')(model.layers[-1].output) # ***
    model=Model(model.input,x)
    
    #To set the first 15 layers to non-trainable (weights will not be updated)
    for layer in model.layers[:15]:
        layer.trainable = False
    
    sgd = SGD(lr=1e-4, decay=1e-6, momentum=0.9, nesterov=True)
    model.compile(optimizer=sgd, loss='categorical_crossentropy', metrics=['accuracy'])
    return model

model = create_vgg16_model(img_rows, img_cols, channel, num_classes)

# train the model!!!
model.fit(X_train, Y_train,batch_size=batch_size,epochs=nb_epoch,shuffle=True,verbose=1,validation_data=(X_valid, Y_valid))

{% endhighlight %}

Important thing to note is that part of \*\*\*. In the original article, poping a layer was done like this

{% highlight python %}
model.layers.pop()
model.outputs = [model.layers[-1].output]
model.layers[-1].outbound_nodes = []
x=Dense(num_classes, activation='softmax')(model.output)
{% endhighlight %}

This seems like a working approach for old versions of keras, but a working version of the code is like in the code above. The reason was that for a sequential model, the way to pop a layer is through `model.pop()`, however this vgg16 doesn't seem like a sequential model, therefore there is no such available method call. The next best thing to do is to pop the layer manually, however doing so would not trigger necessary clean up, hence the mess after in the above code. In the newer version, this seems to also work:

{% highlight python %}
model.layers.pop()
x=Dense(num_classes, activation='softmax')(model.layers[-1].output) 
model=Model(model.input,x)
{% endhighlight %}
