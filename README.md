# Medical Text Classification using Dissimilarity Space

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/Medical_Transcription.jpg?raw=true)

## Goal

This project goal is to develop a classifier that given a medical transcription text classifies its medical specialty.

**Note:** The goal of this project is not necessarily to achieve state-of-the-art results, but to try the idea of dissimilarity space on the task of text classification (see "main idea" section below).

## Data
The original data contains 4966 records, each including three main elements: <br/>

**transcription** - Medical transcription of some patient (text).  <br/>

**description** - Short description of the transcription (text).  <br/>

**medical_specialty** - Medical specialty classification of transcription (category).  <br/>
There are 40 different categories. Figure 1 displays the distribution of the top 20 categories in the dataset.

<br/>

| ![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/categories_dists.png?raw=true)|
| --- |
| **Figure 1**: Top 20 categories|


One can see that the dataset is very unbalanced - most categories represent less than 5% of the total,  each. 

Due to limitations in time and memory, I use descriptions rather than transcriptions (see Figure 3 and Figure 4, which displays their text length histograms).


| ![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/descriptions_length.png?raw=true) | ![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/transcriptions_length.png?raw=true) |
| --- | --- |
| **Figure 2**: Description texts length| **Figure 3**: Transcription texts length|


## Main Idea
The main idea used in this project is to learn a smart embedding to each medical description in the training set, and use the embedded vectors to train classifiers. Then one can perform the same embedding to a new medical description and predict its medical specialty.

The idea is adapted from [1] with some adjustments because text data is used in this project instead of images.

## Scheme
The training procedure consists of several steps which are schematized in Figure 4.

| ![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/Scheme.png?raw=true) |
| --- |
| **Figure 4**: Training Scheme |

**(1) Training Set**  <br/>
The original train set is used to build a customized data loader. It produces pairs of samples with a probability of 0.5 that both samples belong to the same category.

**(2) Siamese Neural Network (SNN) Training** <br/>
The purpose of this phase is to learn a distance measure d(x,y) by maximizing the similarity between couples of samples in the same category, while minimizing the similarity for couples in different categories. <br/>
Our siamese network model consists of several components:

- **Two identical twin subnetworks** <br/>
two identical sub-network that share the same parameters and weights. Each subnetwork gets as input a text and outputs a feature vector which is designed to represent the text. I chose as a subnetwork a pre-trained Bert model (a huggingface model which trained on abstracts from PubMed and on full-text articles from PubMedCentral, see [2]) followed by a FF layer for fine-tuning.
- **Subtract Block** <br/>
Subtracting the output feature vectors of the subnetworks yields a feature vector Y representing the difference between the texts: <br/>
Y = | f1 - f2 | <br/>
- **Fully Connected Layer (FCL)** <br/>
Learn the distance model to calculate the dissimilarity. The output vector of the subtract block is fed to the FCL which returns a dissimilarity value for the pair of texts in the input. Then a sigmoid function is applied  to the dissimilarity value to convert it to a probability value in the range [0, 1].

We use Binary Cross Entropy as a loss function. Note that in the original siamese neural network paper ([3]) the authors used various loss function which can consider more then two samples......................


**(3-4) Prototype Selection** <br/>
In this phase, K prototypes are extracted from the training set. As the autores of [1] stated, it is not practical to take every sample in the training as a prototype. Alternatively, m centroids for each category separately are computed by clustering technique. This reduces the prototype list from the size of the training sample (K=n) to K=m*C (C=number of categories). I chose K-means for the clustering algorithm.

In order to represent the training samples as vectors for the clustering algorithm, the authors in [1] used the pixel vector of each image. In this project, I utilize one of the subnetworks of the trained SNN to retrieve the feature vectors of every training sample (recall that the subnetwork gives us an embedded vector which represent the input text).

**(5) Projection in the Dissimilarity Space** <br/>
In this phase the data is projected into dissimilarity space. In order to obtain the representation of a sample in the dissimilarity space, we calculate the similarity between the sample and the selected set of prototypes P=p1,...pk, which resulting in a dissimilarity vector: <br/>
F(x)=[d(x,pi),d(x,pi+1),...,d(x,pk)] <br/>
The similarity among a sample and a prototype d(x,y) is obtained using the trained SNN.

**(6) SVM Classifiers** <br/>
In this phase an ensemble of SVMs are trained using a One-Against-All approach: For each category an SVM classifier is trained to discriminate between this category and all the other categories put together. A sample is then assigned to the category that gives the highest confidence score. The inputs for the classifiers are the projected train data.

## Why using Dissimilarity Sapce and not Direct classifier

- imbalanced data
- new categories: no need to train new model?
- To try this method, self learing....

## Evaluation

We evaluate the full procedure using the usual metrics (precision, recall, F1-score) on two left-out datasets:

**A) "Regular" test set** - This dataset includes texts that their categories appear in the train categories. We use this dataset in the following way: <br/>
 - Projecti the test text samples into dissimilarity space using the trained SNN model and the prototype list we found during the training phase.
 - Feed the projected test set into the trained SVM classifiers, and examine the results.

**B) "Unseen" test set** - this dataset includes texts that their categories **don't appear** in the train categories (hence the name "unseen"). We use this dataset to check whatever the trained SNN model can be utilized to measure the distance between texts that belong to "unseen" categories (and then, eventually, classify correctly their category). We check this in the following way: <br/>
 - Split the "unseen" test set into train and test sets.
 - Perform steps 3,4,5,6 in the training phase on the train set. Note that we don't train the SNN model agian.
 - Predict the test set categories as we do in A). 

## Results

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/SNN_loss.png?raw=true)

**3d_embedded_train_mat**
![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/3d_embedded_train_mat.png?raw=true)

**test_confusion_mat**

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/test_confusion_mat.png?raw=true)

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/test_report.jpg?raw=true)


**unseen_test_confusion_mat**

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/unseen_test_confusion_mat.png?raw=true)

![pic](https://github.com/OdedMous/Medical-Transcriptions-Classification/blob/main/images/unseen_test_report.png?raw=true)



## TODO
Problem: 
if the training loss is not decreasing, chances are the model is too simple for the data. The other possibility is that our data just doesn’t contain meaningful information that lets it explain the output

- change to "transcription" of size 512 - didn't work (stucked on 0.6)

- Check the bert embedding on the train set (plot it on 3d and see if the categories are seperated)
-
- check if the vocabelry of BERT is similar to our data vocabelry (see maybe if the ids of the texts contain many UNKNOWN symbol)

- Add another layer of finetuning FF (if the SNN doesnt learn we should try yo increase its power. More parametrs = more power)

- reduce number of categories to 2 - it went from 0.6 to 0.3 but them stop there


- -instead of using the description (less accurate than the transcription)  or using the full transcription (too heavy), sample from the transcription a text of 512 characters (kind of augmentation).

- reduce the number of categories to 2

- change the architecture:
  - discard finetuning FF layer - didn't work
  - decrease the dimension of finetuning layer from 128 to 64 - didn't work
  - increase the dimension of finetunning layer from 128 to 512 - didn't work
  - change to rnn instead of bert - 

- train with keras

- train on other dataset (simpler dataset)
  - spam dataset (2 caegories) - 20 epochs stuck around 0.2

- make the dataloader sample equally from all classes ?

- learning rate: use sceduler = scyclic learning rate

- btach size

- loss function


## Libaries
Pytorch, HuggingFace, sklearn,  Numpy, Plotly

## Resources
[1] Spectrogram Classification Using Dissimilarity Space: https://www.mdpi.com/2076-3417/10/12/4176/htm


