# Reviewing the CLAM model
This repository is a training and an evaluation of the [CLAM](https://github.com/mahmoodlab/CLAM) model by doing a binary, slide-level classification on a subset of Camelyon16 (with limited resources) for a tutored university project.

# The task
In the context of oncology, genetic biomarkers can be genetic alterations that are used by the pathologists to characterize a type of cancer to be able to provide a personalized treatment to a patient. The pathologist must then study a surgically resected tissue on glass under a microscope to detect these genetic biomarkers. To adapt this process to deep learning, WSI images are used (Whole Slide Images). They are generated by using a high resolution scanner on the tissue section. The result is an image that can be composed of billions of pixels. Since this is impossible to load at once in memory, these images are generally divided in the literature into patches of uniform size in order to be processed.

<p align="center">
  <img src="/report_images/wsi_example.png" alt="Example of a WSI"/>
  <br>
  <em>Example of a WSI on the left, with a patch extracted on the right.</em>
</p>

In this work, the proposed task is to use the [Camelyon16](https://camelyon16.grand-challenge.org/) dataset, composed of 400 WSI of sentinel lymph node tissue sections, to train an attention-based model to do binary classifications on tumors to detect breast cancer.
The chosen model is [CLAM](https://github.com/mahmoodlab/CLAM), from the paper "Lu, M.Y., Williamson, D.F.K., Chen, T.Y. et al. Data-efficient and weakly supervised computational pathology on whole-slide images. Nat Biomed Eng 5, 555–570 (2021)". This model applies the principles of another paper : Attention-based Deep Multiple Instance Learning” (Maximilian Ilse, Jakub M. Tomczak, Max Welling, ICML 2018). We chose this model because it had a great impact on the field, because there are a lot of open source material available for it, and because it uses weakly supervised learning, which simplifies the workflow for this task.

# The method :

<p align="center">
  <img src="/report_images/seg_patch.png" alt="Step 1, segmenting and extracting patches from the WSI" width="50%"/>
  <br>
  <em>Step 1 - segmenting and extracting patches from the WSI.</em>
</p>

This first step is common to most works on WSI in the literature. The goal is to obtain the tissue part of the image, because most of the WSI is composed of empty glass. We must also divide the slide into uniform patches to be able to process it. Here, we use the pipeline provided by the authors directly (binary thresholding with otsu mixed with opencv functions for the segmentation, and the openslide library to extract patches at given coordinates).

<p align="center">
  <img src="/report_images/feature_extractions_eng.png" alt="Step2, feature extraction with a resnet-50" width="50%"/>
  <br>
  <em>Step 2 - Feature extraction with a resnet-50 (the modifications are not represented).</em>
</p>

The second step is a feature extraction from the patches of each slide using a resnet-50 model simplified (only the first 3 layers are kept). the resnet-50 is pre-trained on imagenet without further fine-tuning : this is weakly supervised, so we only use the slide-level labels, not the patch-level labels (we only have to know if a WSI is cancerous or not, we do not need to precisely localize the tumor on the image). The extracted features are stored in files (pytorch compressed format) to be reused at will. This step is the most time-consuming one, a single slide's feature extraction can take a few minutes to a few dozens depending on the amount of tissue in the slide.

The third and final step is feeding the features to the CLAM model. The goal is, for each slide, to extract attention scores from the features by passing them through an attention layer. Once this is done, there are 2 operations done with simple classifiers :

<p align="center">
  <img src="/report_images/clam_losses.png" alt="The CLAM model operations" width="50%"/>
  <br>
  <em>Step 3 - After the attention scores computation, we compute both losses.</em>
</p>

The idea behind the attention pooling is to do an intelligent max pooling - at the beginning of the state of the art on this task, a simple max pooling was done with the logic being "if a patch is cancerous, the whole slide should be treated as cancerous". This logic is valid, but too sensitive to false positives, which explain the interest behind an intelligent pooling. The idea behind the instance loss computation is to train the model to distinguish between interesting instances and their opposite (not to recognize tumors). At the end, both losses are added in a weighted sum to calculate the final loss.
The instance losses serves only for training. We backprop the final loss and only monitor and evaluate the bag loss.

# Our results

Our datasets are composed of 100 slides for training (80% training, 20% validation) and 36 slides for testing. The classes distribution is balanced for each set. We had to limit ourselves because our resources are limited and we couldn't compute the features for more slides.
The principal metric used is AUROC. Our training is done on 50 epochs, using early stopping with a patience of 5 epochs. We keep the model from the epoch with the lowest validation loss.

<p align="center">
  <img src="/report_images/final_loss.png" alt="Our final bag losses" width = "40%"/>
  <br>
  <em>Our final loss</em>
</p>

After an iterating process, we end up with a learning rate of 0.0002, dropouts of 0.25 and weight decay of 1e-5 (back to the author's parameters). We tried several other options, like momentum or other regularization techniques like class weights (at the end of the day, we still classify instances, so even in tumor slides, most of the tissue is still not cancerous) without significant results. The reason is that the model is highly dependant of the amount of data it receives :

<p align="center">
  <img src="/report_images/data_comparison1.png" alt="The model's performances depending on the number of WSIs - example 1" width = 80%/>
  <br>
  <em>On top, the performances on the test set. At the bottom, the number of WSIs it received at training</em>
</p>
You can see that when we were limited to 30 slides, the loss was constant and the model wasn't learning anything. The performances were worse than a random. But when it is fed more data, the model starts little by little to become better than a random : the ROC curves gets further and further away from the linear curve at the center. We end up with an AUROC of 0.72 for our best model with 100 WSIs, and 0.9 when we use the author's weights from a model trained on 899 WSIs.
<p align="center">
  <img src="/report_images/authors_weights.png" alt="The model's performances depending on the number of WSIs 2 - example 2" width = 60%/>
  <br>
  <em>On the left, our best model's performances, on the right, our performances with the author's weights</em>
</p>

After a final training with a 5-folds cross validation, our mean AUROC validation is 0.89. When we keep the model with the lowest loss, its AUROC on the test set is 0.69, which is our final performance.

# Improvements

Several improvements are possible for our work, the most important being the extraction of more features, which will largely improve the performances. 
It can also be interesting to implement the ideas proposed in the paper MS-CLAM: Mixed Supervision for the classification and localization of tumors in Whole Slide Images (Paul Tourniaire, Marius Ilie, Paul Hofman, Nicholas Ayache, Hervé Delingette.), which suggest, amongst other things, a mixed supervision by fine-tuning the feature extractor. They manage to obtain fair improvements on CLAM's performances. 
Finally, it would be good to do a qualitative evaluation by aligning the attention scores with a given WSI's thumbnail to be able to see on what the model is concentrating (lack of time).

# Conclusion

The CLAM model is a very strong option if you have a lot of data and few labels. It can reach a very good performance. However, if its not the case (i.e your number of slides is low or average, but you have a sufficient amount of labels), you should explore other options, probably supervised learning techniques with, for example, a random forest aggregator.

# References
* https://camelyon16.grand-challenge.org/
* Lu, M.Y., Williamson, D.F.K., Chen, T.Y. et al. Data-efficient and weakly supervised computational pathology on whole-slide images. Nat Biomed Eng 5, 555–570 (2021)
* Maximilian Ilse, Jakub M. Tomczak, Max Welling. Attention-based Deep Multiple Instance Learning (ICML 2018)
* Paul Tourniaire, Marius Ilie, Paul Hofman, Nicholas Ayache, Hervé Delingette. MS-CLAM: Mixed Supervision for the classification and localization of tumors in Whole Slide Images. Medical Image Analysis, In press, 85, pp.102763. ff10.1016/j.media.2023.102763ff. ffhal-03972289f
* https://github.com/mahmoodlab/CLAM/tree/master

# Special thanks
A big thanks to Paul Tourniaire from the MS-CLAM research team for helping me further my understanding of the CLAM method and providing me with more pre-computed features when i didn't have enough, which enabled me to produce results.
