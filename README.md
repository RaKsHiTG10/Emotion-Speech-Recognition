
# Install kaggle
!pip install -q kaggle

from google.colab import drive
drive.mount('/content/drive')

# Create a kaggle folder
! mkdir ~/.kaggle/

! kaggle datasets download -d ejlok1/toronto-emotional-speech-set-tess

!unzip /content/toronto-emotional-speech-set-tess.zip

# Copy the kaggle.json folder to folder created
! cp /content/drive/MyDrive/Kaggle/kaggle.json ~/.kaggle/

! chmod 600 ~/.kaggle/kaggle.json

# Check if kaggle.json exists
!ls /content/drive/MyDrive/Kaggle/kaggle.json

import os
os.environ['KAGGLE_CONFIG_DIR'] = "/content"

import pandas as pd #data manipulation
import numpy as np # multi dimensional array and matrices (numerical calculation)
import os #provides way of using operating system dependent functionality such as reading and writing to the file system or reading a file from file
import matplotlib.pyplot as plt #for generating plots like hist
import seaborn as sns #data visualization
import librosa #its is a python pacakage for music and audio analysis ..it provides building blocks necessary to create music info system.. it is often used to extract features from the audio
import librosa.display #module within lib to display to visualize the data
import IPython.display as ipd #to display the audio voices in the notebook allowing you to play audio directly within the notebook by this we can play the audio
import warnings #to ignore the warnings especially when working with a lot of data with complex operations
warnings.filterwarnings('ignore')
from keras import utils #deep learning library deep learning API written in python keras means its on the top of tensor its used to create and train neural networks the utils..
#utils is a module which include utility function and classes such as converting levels to categorical forms

# Loading the Dataset

paths = [] #list of all the file path
labels = [] #list of all the label corresponding to each of file
for dirname, _, filenames in os.walk('/content/tess toronto emotional speech set data'): # os.walk means function to traverse through the directory i.e dataset
#directory name is te current directory path i.e current working directory
    for filename in filenames: #filename is list of file name in current directory
        paths.append(os.path.join(dirname, filename)) #it constructs the full file path and append it to the path list
        label = filename.split('_')[-1] #slits the file name by underscore and take in the last part which is expected to be in the format
        label = label.split('.')[0] #splits the particular part the dot to remove the file exchange
        labels.append(label.lower()) #appends(add) the label to lower case
    if len(paths) == 2800: #if the path reaches to 2800 the loop will automatically break.. indicates dataset is large the dataset carry seven different types of emotion
    #that again that 2800 files are stored inside data
      break
print ('Dataset is loaded')

len(paths)

paths[:5] #first five paths of dataset

labels[:5]

# Create a dataframe
df = pd.DataFrame()
df['speech'] = paths
df['label'] = labels
df.head()

df['label'].value_counts()

df.info()

sns.countplot(x = 'label', data = df)

def waveplot(data, sr, emotion): #func create a wave form plot to the audio data ; data means the audio signal;
#sr means integer representing the sampling rate refers to the number of samples or data points collected per unit time in Hz;
#emotion: representing the emotion of label associated with the audio data
  plt.figure(figsize=(10,4))
  plt.title(emotion, size=20)
  librosa.display.waveshow(data, sr=sr) #shows the waveform of the audio signal using librosa
  plt.show()

def spectogram(data, sr, emotion): #function that creates a spectogram plot for the audio data
  x = librosa.stft(data) # stft is short time fourier transform of the audio signal data which is the way to represent data that will frequently content of the signal changes over time to time
  xdb = librosa.amplitude_to_db(abs(x)) #xdb is the librosa amplitude to db (std unit to measure sound)
  plt.figure(figsize=(11,4))
  plt.title(emotion, size=20)
  librosa.display.specshow(xdb, sr=sr, x_axis='time', y_axis='hz') #shows the spectogram using librosa, visualize the amplitude of the frequency over time
  plt.colorbar() #add color bar to the plot showing the scale

df.info()

print(df.head())
print(df['label'].unique())

df.head()

df['speech'].unique()# unique: the method returns an array containing values from the speech coulumn
#unique is useful for task such as verifying data uniqueness checking for specific data entries or preparing data for further analysis

from IPython.display import Audio
emotion = 'fear'
path = np.array(df['speech'][df['label']== emotion])[0] # converts rhe results into numpy array and types the first element
#assuming there is only one path corresponding to emotion
data, sr = librosa.load(path) # loads the file specified by the path
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path) # allows to directly play audio in notebook or any intetractive environment
#overally it is a sequence of operations crucial for analyzing audio datain such projects providing both visual insights and auditory info of data
#means how it visually looks like and how the audio is playing

emotion = 'neutral'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

emotion = 'angry'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

emotion = 'happy'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

emotion = 'sad'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

emotion = 'ps'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

emotion = 'disgust'
path = np.array(df['speech'][df['label']== emotion])[0]
data, sr = librosa.load(path)
waveplot(data, sr, emotion)
spectogram(data, sr, emotion)
Audio(path)

# Feature Extraction

# MFCC Calculation Steps:

#Pre-emphasis: A high-pass filter is applied to the signal to emphasize higher frequencies.
#Framing: The audio signal is divided into small overlapping frames to capture short-term spectral features.
#Windowing: Each frame is windowed (e.g., using a Hamming window) to reduce spectral leakage.
#Fast Fourier Transform (FFT): The Fourier transform is applied to convert each frame from the time domain to the frequency domain.
#Power Spectrum: The magnitude of the FFT is squared to obtain the power spectrum.
#Mel Filter Bank: The power spectrum is passed through a series of Mel-spaced filters to model human hearing. Each filter sums the power in its frequency range.
#Logarithm: The logarithm of the Mel-filtered energies is taken to mimic human loudness perception.
#Discrete Cosine Transform (DCT): The DCT is applied to the log Mel spectrum to obtain the cepstral coefficients, which are the MFCCs.

def extract_mfcc(file_name): # extract mfcc means is a single parameter or filename wrt the path of an audio file
  y, sr = librosa.load(file_name, duration=3, offset=0.5) #y uses library to load the audio file specified by the filename
  #s is the dimensional array containing the audio signal; y and sr is is the integer that represent the sampling rate of the audio file ;
  #duration 3 is only the first 3 seconds of audio ; offset 0.5 means it skips the first 0.5 sec of audio because there is some gap after which audio starts
  mfcc = np.mean(librosa.feature.mfcc(y=y, sr=sr, n_mfcc=40).T, axis=0) #extract the mfcc from the audio signal y using the sampling rate i.e sr
  #np.mean calculate the mean along with the time of the audio; mfcc=40 is specified typically mfcc matrix as same
  #code is extract mfcc function loads a specified audio file and extract 40 mfcc from three second segment starting from 0.5 sec into the audio and avg of mfcc over time and
  #it returns resulting feature vector
  return mfcc
  #mfcc is widely used speech and audio processing to represent the power spectrum of audio signal that makes useful for task such as speech emotion
  #emotion detection or audio classification problem

extract_mfcc(df['speech'][0]) # means select the column from df column 0 means it access the first element in the column
#extract_mfcc recalls the function passing the path to the first audio file

x_mfcc = df['speech'].apply(lambda x: extract_mfcc(x)) # means it applies the extract mfcc function to the audio file path in the speech column of the data frame and
#store the resulting mfcc feature vector in x_mfcc variable ; apply lambda it extract mfcc ,applies the lambda function it is anonymous function to speech column
#lambda function takes an argument x and passes to the extract mfcc function

x_mfcc # displays the series that contains mfcc feature vector from each audio file path in the speech column of the dataframe df

X = [x for x in x_mfcc] # x means it is iterating over each element in x_mfcc
X = np.array(X)
X.shape

#input split
X = np.expand_dims(X, -1) # function that add an extra dimensions to the numpy array ; -1 means it adds new axis at the end(last position)
X.shape # dataset is packed so we add one axis so tat we can edit features in it

from sklearn.preprocessing import OneHotEncoder
enc = OneHotEncoder() #create instance for one hot encoder class then transform the category level into one hot encoder
y = enc.fit_transform(df['label'].values.reshape(-1,1)).toarray() # to array method is used to convert into sparse matrix dense array(dense are easy to manipulate especially works for tasks such as debugging and data visualization)

y.shape

from sklearn.model_selection import train_test_split # splitting random rs or matrices into random train and validation
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42) # 20% for test rest 80% for train; random state 43 means it sets the seed for random number generation
#ensures that the split is reproducible i.e it is a set a seed for random number generation x train--input feature, x test -input validation, y  train- target set of target level and y test is target validation

from keras.models import Sequential #function from keras sequential for intializing the model dense for fully connected and lstm for long term memory and dropout for regularization
from keras.layers import Dense, LSTM, Dropout # lstm- long short term memory .. it construct the model in keras and sequential layers tracking layers we used here are
#lstm dropout dense layer dropout and lstm are used for sequence processing and class
#lstm working-- previous input +previous output + previous memory
model= Sequential([LSTM(256, return_sequences=False, input_shape= (40, 1)),
    Dropout(0.5),
    Dense(128, activation='relu'), # relu is rectified linear unit adds fully connected dense layer with 128 and 64 unit ; equation for relu function - fx = max(0,1)
    Dropout(0.5), # Add dropout after dense layer
    Dense(64, activation='relu'),
    Dropout(0.5),
    Dense(7, activation='softmax') # soft max function is for multi classification purpose.. it converts logic means raw application into probablity used when our accuracy is not that great
    #and if u want accuracy in logics or in discrete func like yes no
    ])
model.compile(optimizer='adam', loss='categorical_crossentropy', metrics=['accuracy']) # adam for optimizing the models weight; matrix accuracy means specify evaluation of matix to be monitor during training
model.summary()
Model: "sequential"
#configure the model with loss fun optimizer and evaluation matrix and it displays a comprehensive overview of the model architecture its aim aim to understand its structure and parameter

# Train the model
history = model.fit(X_train, y_train, validation_data=(X_test, y_test), batch_size=64, epochs=30) # epoch - iteration over entire training data i.e no of iteration we want to train our model
#batch size 64 means no of samples that will be propogated through the network before operating model parameter.. larger batch size results in faster training

"""Plot the results"""

epoch = list(range(30)) # list of epoch from 0 to 29
acc = history.history['accuracy'] # list extracted from history.history which contains the training accuracy for each and accuracy is the key used to access the info
val_acc = history.history['val_accuracy']
plt.plot(epoch, acc, label='Training Accuracy')
plt.plot(epoch, val_acc, label='Validation Accuracy')
plt.xlabel('Epochs')
plt.ylabel('Accuracy')
plt.legend()
plt.show()

loss = history.history['loss'] #contains training loss for each epoch
val_loss = history.history['val_loss'] # list that contain validation loss of each
plt.plot(epoch, loss, label='Training Loss')
plt.plot(epoch, val_loss, label='Validation Loss')
plt.xlabel('Epochs')
plt.ylabel('Loss')
plt.legend() # labels the each line ie training loss and testing
plt.show()
#helps us to identify how well your model is minimizing the loss function indicating the improvement both for training and valdiation data
# Loss function

y_pred = model.predict(X_test)
y_pred_classes  = np.argmax(y_pred, axis=1)
y_val_classes = np.argmax(y_test, axis=1)

# Compute confusion matrix
from sklearn.metrics import confusion_matrix, classification_report # table that summarizes performance of the classification model
cm = confusion_matrix(y_val_classes, y_pred_classes)
# print the confusion matrix
print("Confusion matrix")
print(cm)
#for computing the confusion matrix based on true class level and predicted class level

# Print the classification report
target_name = ['angry', 'disgust', 'fear', 'happy', 'neutral', 'ps', 'sad']
print("Classification report")
print(classification_report(y_val_classes, y_pred_classes, target_names=target_name))

plt.figure(figsize=(8, 6))
sns.heatmap(cm, annot=True, vmin=0, fmt='d', cmap='Blues', xticklabels = target_name, yticklabels = target_name)
plt.xlabel('Predicted labels')
plt.ylabel('True labels')
plt.title('Confusion Matrix')
plt.show
#classification report that generates a text report that includes precision(true positive/true positive+ false positive), recall (true positive/true positive+ false negative)
#f1 score 2× Precision+Recall/Precision×Recall and support(number of actual occurrences of the class in the dataset. It is simply the count of true instances for each label)
#Specificity =True Negatives (TN)/True Negatives (TN)+False Positives (FP)
#fmt =d means sets format of annotation to all the integers

prediction = model.predict(X_test[0].reshape(1, 40, 1))
prediction

df['label'].unique()
label_prediction_map = {label: value for label, value in zip(labels, prediction[0])}
label_prediction_map

