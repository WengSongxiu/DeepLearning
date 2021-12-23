# Advanced Deep Learning Practices

## Sequential -> Keras Functional API

### Drawback of Sequential

In Sequential, there is only 1 input and 1 output, all layers are linearly stacked.  

- Some task need multimodal input

- Some task need predict multi attribute of the input
- Many latest nn architecture demand nonlinear topological structure, as a DAG  (Inception, ResNet)

### Keras Functional API

Use layer as a function, receive and return a tensor.  

```python
from keras.models import Sequential, Model
from keras import layers, Input

# Sequential
seq_model = Sequential()
seq_model.add(layers.Dense(32, activation = 'relu', input_shape = (64,)))
seq_model.add(layers.Dense(32, activation = 'relu'))
seq_model.add(layers.Dense(10, activation = 'softmax'))

# Functional API
input_tensor = Input(shape = (64,))
x = layers.Dense(32, activation = 'relu')(input_tensor)
x = layers.Dense(32, activation = 'relu')(x)
output_tensor = layers.Dense(10, activation = 'softmax')(x)

# Model Class transform the i-o tensor to a model
model = Model(input_tensor, output_tensor)
```

### Multi Input Model

```python
from keras.models import Model
from keras import layers
from keras import Input

text_vocabulary_size = 10000
question_vocabulary_size = 10000
answer_vocabulary_size = 500

text_input = Input(shape=(None,), dtype='int32', name='text')
embedded_text = layers.Embedding(text_vocabulary_size, 64)(text_input)
encoded_text = layers.LSTM(32)(embedded_text)

question_input = Input(shape = (None,), dtype='int32', name='question')
embedded_question = layers.Embedding(question_vocabulary_size, 32)(question_input)
encoded_question = layers.LSTM(16)(embedded_question)

concatenated = layers.concatenate(
    [encoded_text, encoded_question],axis=-1)

answer = layers.Dense(answer_vocabulary_size, activation='softmax')(concatenated)

model = Model([text_input, question_input], answer)
model.compile(optimizer='rmsprop',
              loss='categorical_crossentropy',
              metrics=['acc'])
```

```python
import numpy as np

num_samples = 1000
max_length = 100

text = np.random.randint(1, text_vocabulary_size,
                         size=(num_samples, max_length))

question = np.random.randint(1, question_vocabulary_size,
                             size=(num_samples, max_length))

answers = np.random.randint(answer_vocabulary_size, size=(num_samples))
answers = keras.utils.to_categorical(answers, answer_vocabulary_size)

# Use a list of input
model.fit([text, question], answers, epochs=10, batch_size=128)

# Use a dict of input
model.fit({'text': text, 'question': question}, answers, epochs=10, batch_size=128)
```

### Multi Output Model

```python
from keras import layers
from keras import Input
from keras.models import Model

vocabulary_size = 50000
num_income_groups = 10

posts_input = Input(shape=(None,), dtype='int32', name='posts')
embedded_posts = layers.Embedding(256, vocabulary_size)(posts_input)

x = layers.Conv1D(128, 5, activation='relu')(embedded_posts)
x = layers.MaxPooling1D(5)(x)
x = layers.Conv1D(256, 5, activation='relu')(x)
x = layers.Conv1D(256, 5, activation='relu')(x)
x = layers.MaxPooling1D(5)(x)
x = layers.Conv1D(256, 5, activation='relu')(x)
x = layers.Conv1D(256, 5, activation='relu')(x)
x = layers.GlobalMaxPooling1D()(x)
x = layers.Dense(128, activation='relu')(x)

age_prediction = layers.Dense(1, name='age')(x)
income_prediction = layers.Dense(num_income_groups, 
                                 activation='softmax', 
                                 name='income')(x)
gender_prediction = layers.Dense(1, activation='sigmoid', name='gender')(x)

model = Model(posts_input, [age_prediction, income_prediction, gender_prediction])
```

We need specify different loss function for different head.  

For example, age prediction is a scalar regression task. Gender prediction is a 2-class classification task. They need different training process. However, gradient descent demand minimum of a scalar, so as to train the model, we need to mix these losses to one scalar. The simplest way is to add them up.  

```python
model.compile(optimizer='rmsprop',
              loss=['mse', 'categorical_crossentropy', 'binary_crossentropy'])

# Equivalent to the usage above
model.compile(optimizer='rmsprop',
              loss={'age': 'mse',
                    'income': 'categorical_crossentropy',
                    'gender': 'binary_crossentropy'})
```

Attention : serious imbalance of loss contribution will lead to the direction of optimization point to the task with biggest loss. To solve this, we need to allocate different quota to different loss.

```python
model.compile(optimizer='rmsprop',
              loss=['mse', 'categorical_crossentropy', 'binary_crossentropy'],
              loss_weights=[0.25, 1., 10.])

model.compile(optimizer='rmsprop',
             loss={'age': 'mse',
                   'income': 'categorical_crossentropy',
                   'gender': 'binary_crossentropy'},
             loss_weights={'age': 0.25,
                           'income': 1.,
                           'gender': 10.})
```

```python
model.fit(posts, [age_targets, income_targets, gender_targets], epochs=10, batch_size=64)

model.fit(posts, {'age': age_targets,
                  'income': income_targets,
                  'gender': gender_targets},
          epochs=10, batch_size=64)
```

### Directed Acyclic Graph

#### Inception

Inception is a CNN based architecture, it is a stacking of modules, these modules look like a tiny independent network, divided into multi branches.  

The basic inception have 3-4 branches, first 1 x 1 conv, then 3 x 3 conv, finally add all features together.  

This kind of design help network to learn spatial and channel-wise feature separately.  

##### 1 x 1 Convolution ( Pointwise Convolution )

- Mix different channel in the input tensor up
- Would not mix up trans-space information (cause it look into one pixel once)
- Contribute to separating channel feature learning and spatial feature learning

```python
from keras import layers

branch_a = layers.Conv2D(128, 1, activation='relu', strides=2)(x)

branch_b = layers.Conv2D(128, 1, activation='relu')(x)
branch_b = layers.Conv2D(128, 3, activation='relu', strides=2)(branch_b)

branch_c = layers.AveragePooling2D(3, strides=2)(x)
branch_c = layers.Conv2D(128, 3, activation='relu')(branch_c)

branch_d = layers.Conv2D(128, 1, activation='relu')(x)
branch_d = layers.Conv2D(128, 3, activation='relu')(branch_d)
branch_d = layers.Conv2D(128, 3, activation='relu', strides=2)(branch_d)

output = layers.concatenate([branch_a, branch_b, branch_c, branch_d], axis=-1)
```

Keras has a built-in implementation of InceptionV3

```
keras.application.inception_v3.InceptionV3
```

#### Xception

Xception = Extreme Inception  

Xception push learn channel feature and spatial feature separately to its extreme. Replace Inception module with deep separable convolution, the spatial feature and channel feature are completely separated.      

Xception's parameters' num roughly equals to Inception V3. Because of its more efficient use of parameters, it have better performance on ImageNet and other large scale datasets.

#### Residual Connection

Residual connection solved two common issue in large scale deep learning model

-  Gradient Vanishing
- Representation Bottleneck

The output of former layer add to later layer's activation. If they are of different shape, it use a linear transform to change former output's shape equal to the target shape.

- Identity Residual Connection

  ```python
  from keras import layers
  x = ...
  y = layers.Conv2D(128, 3, activation='relu', padding='same')(x)
  y = layers.Conv2D(128, 3, activation='relu', padding='same')(y)
  y = layers.Conv2D(128, 3, activation='relu', padding='same')(y)
  y = layers.add([y, x])
  ```

- Linear Residual Connection

  ```python
  from keras import layers
  x = ...
  y = layers.Conv2D(128, 3, activation='relu', padding='same')(x)
  y = layers.Conv2D(128, 3, activation='relu', padding='same')(y)
  y = layers.MaxPooling2D(2, strides=2)(y)
  # Use 1 x 1 conv, subsampling x to the shape of y
  residual = layers.Conv2D(128, 1, strides=2, padding='same')(x)
  y = layers.add([y, residual])
  ```

##### Representation Bottleneck

In Sequential model, every continuous representation layer is constructed on the former layer, that is, it can only get the information activated in the former layer. If one layer is too small, the model is limited to how much information can be held in this layer.  

Any loss of information is permanent, residual connection can resend earlier information to later layer.  

##### Gradient Vanishing

Backpropagation propagates loss in the output to the layer in the bottom. If the feedback signal need to go through many layers, it may be very weak, even lost, result to untrainable network.  

Both deep network and recurrent network for sequence processing have this problem.   

We already know how LSTM solve this problem: LSTM introduce a carry track, which propagate information in the track parallel to the main track.  

Residual connection's principle similar to this, and more simple: It introduce a linear track to carry information, parallel with the direction of main stacking layers, which helps propagate gradient through deep layers.  

### Share Weight

We can reuse a layer instance in functional API. If you call a layer instance twice, rather than instantiation a new layer every time, you can reuse the same weight. Thus, construct a model with shared branch.  

For example, to evaluate semantic similarity of two sentence, the model have two inputs.  

It has no sense to learn two separate model to process two input sentence, we only use one LSTM layer to handle the two sentence. We call it Siamese LSTM, or shared LSTM model.

```python
from keras import layers
from keras import Input
from keras.models import Model

lstm = layers.LSTM(32)

left_input = Input(shape=(None, 128))
left_output = lstm(left_input)

right_input = Input(shape=(None, 128))
right_output = lstm(right_input)

merged = layers.concatenate([left_output, right_output], axis=-1)
predictions = layers.Dense(1, activation='sigmoid')(merged)

model = Model([left_input, right_input], predictions)
model.fit([left_data, right_data], targets)
```

### Model As Layer

Call a model on an input tensor, it will return a output tensor.  

Call an instance, whether layer or model instance, it will reuse the representation the instance has learned.  

For example: two camera for depth perception

```python
from keras import layers
from keras import applications
from keras import Input

xception_base = applications.Xception(weights=None, include_top=False)

left_input = Input(shape=(250, 250, 3))
right_input = Input(shape=(250, 250, 3))

left_features = xception_base(left_input)
right_input = xception_base(right_input)

merged_features = layers.concatenate([left_features, right_input], axis=-1)
```

## Callback Function & TensorBoard

### Callback Function

Callback is an object sent to model when calling fit function, it will be called by the model at different time in the training process. It can access all available data about the state and performance of the model, and do something like: interrupt training, save model, load different weights or change the state of the model.  

Some Usage :

- Model checkpoint : Save current weights on different timing
- Early stopping 
- Tuning some parameter during training
- Record training and validating result, or visualize the representation in real time

```python
keras.callbacks.ModelCheckpoint
keras.callbacks.EarlyStopping
keras.callbacks.LearningRateScheduler
keras.callbacks.ReduceLROnPlateau
keras.callbacks.CSVLogger
```

#### EarlyStopping & ModelCheckpoint

```python
import keras

callbacks_list = [
	keras.callbacks.EarlyStopping(
		monitor='acc',
		patience=1,
    ),
	keras.callbacks.ModelCheckpoint(
		filepath='my_model.h5',
		monitor='val_loss',
		save_best_only=True,
	)
]

model.compile(
    optimizer='rmsprop',
	loss='binary_crossentropy',
	metrics=['acc'])

model.fit(x, y,
          epochs=10,
          batch_size=32,
          callbacks=callbacks_list,
          validation_data=(x_val, y_val))
```

#### ReduceLROnPlateau

If val loss do not reduce any more, you can use this callback function to reduce learning rate.  

If loss plateau appears, magnify or reduce lr is an efficient strategy of jumping out local minimum.  

```python
callbacks_list = [
	keras.callbacks.ReduceLROnPlateau(
		monitor='val_loss'
		factor=0.1,
		patience=10,
	)
]

model.fit(x, y,
          epochs=10,
          batch_size=32,
          callbacks=callbacks_list,
          validation_data=(x_val, y_val))
```

#### Write Customized Callback Function

Setup a subclass of class ```keras.callbacks.Callback```, then you can implement such methods :

- on_epoch_begin
- on_epoch_end
- on_batch_begin
- on_batch_end
- on_train_begin
- on_train_end

Callback Function and access such attributes :

- self.model
- self.validation_data

Below is an example of customized callback function, it save every layer's activation to hard disk after every epoch, the activation is calculated on the first sample of validation data.

```python
import keras
import numpy as np

class ActivationLogger(keras.callbacks.Callback):
    def set_model(self, model):
        self.model = model
        layer_outputs = [layer.output for layer in model.layers]
        self.activations_model = keras.models.Model(model.input,
                                                    layer_outputs)
        
    def on_epoch_end(self, epoch, logs=None):
        if self.validation_data is None:
        	raise RuntimeError('Requires validation_data.')
            
        validation_sample = self.validation_data[0][0:1]
        activations = self.activations_model.predict(validation_sample)
        
        f = open('activations_at_epoch_' + str(epoch) + '.npz', 'w')
        np.savez(f, activations)
        f.close()
```

### TensorBoard

Loop of Improvement : ``` Idea -> Experiment -> Result -> Idea```  

TensorBoard is a visualization tool based on browser built in TensorFlow.  

Here are some function of TensorBoard :

- Monitoring on indicators with visualization during training
- Visualize architecture of model
- Visualize histogram of activation and gradient
- Research embedding with 3-dim visualization

```python
callbacks = [
	keras.callbacks.TensorBoard(
		log_dir='my_log_dir',
		histogram_freq=1,
		embeddings_freq=1,
	)
]

history = model.fit(x_train,
                    y_train,
                    epochs=20,
                    batch_size=128,
                    validation_split=0.2,
                    callbacks=callbacks)
```

``` $ tensorboard --logdir=my_log_dir```  

``` http://localhost:6006```  

- SCALARS
  - acc
  - loss

- HISTOGRAM

- EMBEDDING
  - PCA
  - t-SNE

- GRAPHS

  - Visualization of Bottom TensorFlow Operation Graph

  - A more concise way  ```keras.utils.plot_model``` 

    - Layer Graph rather than Operation Graph

    - ``` pip install pydot, pydot-ng, graphviz```
    - ```plot_model(model, show_shapes = True, to_file = 'model.png')```

## To Model's Extreme

### Advanced Architecture

#### Batch Normalization

Normalization makes different sample similar to each other, it helps better learning and generalization.  

``` normalized_data = (data - np.mean(data, axis = ...)) / np.std(data, axis = ...)```  

We need consider normalization after every transformation in the network. Though data sent to dense or conv2d with mean 0 and std 1, it is not sure the output of network also conform to this.  

Principle of Batch Normalization : It save the exponential shift average value of mean and variance of every batch of data during training inside the network.  

Effect of Batch Normalization : It helps gradient propagation, avoid of gradient vanishing, allow for deeper network.  

Batch Normalization often use after conv layer or dense layer.  

BN receive a axis parameter, specify an axis to normalize, default is -1, means the last axis of input tensor.  

##### Batch Renormalization

Compared with BN, it has obvious advantage with cost increased slightly.

##### Self-normalizing Neural Network

It use special activation function (selu) and special initializer (lecun_normal), with the ability to keep data normalized after passing through any dense layer.

#### Depthwise Separable Convolution

- Lighter - Less trainable weights
- Faster - Less float calculation
- Better - More performance

Execute spatial convolution on every channel, and mix them up by 1 x 1 convolution.  

If your data's spatial location is highly correlated, and channels are relatively independent, it is better to do separable convolution.  

``` layers.SeparableConv2D()```  

For larger model, depthwise separable convolution is the basis of Xception.

### Hyper-parameter Optimization

Steps :

- Select a set of hyper-parameters automatically
- Construct corresponding model
- Fit model on training data, measure its final performance on validating data
- Select next group of hyper-parameters automatically
- Repeat the process above
- Finally, measure the performance on testing data

Methods :

- Bayesian Optimization
- Genetic Algorithm
- Simple Random Search

We cannot do gradient descent in hyper-parameter space.  

Usually random search is a good solution. There is a tool actually better than random search, it is a python lib named Hyperopt, use a tree of Parzen estimator to predict which set of hyper-parameter will get good result.  

Hyperas : Hyperopt + Keras  

Attention : You are tuning your hyper-parameter based on validation data. So your model will over-fitting on validation set at some point.  

### Ensemble Learning

Ensembling based on such a hypothesis : For different good model trained independently, their good performance may from different reason, every model make decision from slightly different perspective, get a part of 'truth', not all the truth. Gather their decision together, you can get a more precise description.  

Ensemble Learning do weighted average on different model. The weight is calculated based on the performance on validating data. Usually, a more accurate model gets a bigger weight.  

To guarantee a good ensembled model, the key is the diversity of a set of models. If all the models' bias on the same direction, ensemble learning also has this bias. However, if all the models' bias on different direction, the ensembled model will have more precise and stable performance.  

All the models should both good and different. Usually, this means use different architecture, even different machine learning methods.  

A effective way : tree-based method (random forest, gbdt ... ) + deep learning.

Author's empirical experience : during his Kaggle competition, the model with lowest score derived from a method which different from all other models in ensemble learning (regularized greedy forest) was given a very small weight. However, the result was unexpected, because of its difference with other models, it provided information that other models can not get, improved the result tremendously.  

Recently, a successful way to ensemble is "wide and deep", which means a collection of deep learning and shallow learning.  