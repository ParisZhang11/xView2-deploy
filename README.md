# System setup
xView2 inference requires a tremendous amount of computing power. Currently, CPU inference is wildly 
impractical. To that end, unless you have a dedicated workstation with ample GPU power such as an Nvidia DGX station,
we recommend a cloud based solution such as AWS or Google Cloud Compute utilizing a GPU optimized instance. Prices vary
on instance type and area to be inferred. Example instances:
1. AWS EC2
    1. P4d.24xlarge
    2. P3.16xlarge
2. G Cloud
    1. Todo!

# Installation
Todo!

# Usage
|Argument|Required|Default|Help
|---|---|---|---|
|--pre-directory|Yes|None|Directory containing pre-disaster imagery. This is searched recursively.|
|--post-directory|Yes|None|Directory containing post-disaster imagery. This is searched recursively.|
|--is_use_gpu|Sort of (CPU inference is wildly impractical)|False|If True, use GPUs|
|--num_procs|Yes|4|Number of processors for multiprocessing|
|--batch_size|Yes|16|Number of chips to run inference on at once|
|--num_workers|Yes|8|Number of workers loading data into RAM. Recommend 4 * num_gpu|
|--pre_crs|No|None|The Coordinate Reference System (CRS) for the pre-disaster imagery. This will only be utilized if images lack CRS data.|
|--post_crs|No|None|The Coordinate Reference System (CRS) for the post-disaster imagery. This will only be utilized if images lack CRS data.|
|--destination_crs|No|EPSG:4326|The Coordinate Reference System (CRS) for the output overlays.|
|--dp_mode|No|False|Run models serially, but using DataParallel|
|--save_intermediates|No|False|Store intermediate runfiles|
|--agol_user|No|None|ArcGIS online username|
|--agol_password|No|None|ArcGIS online password|
|--agol_feature_service|No|None|ArcGIS online feature service to append damage polygons.|

# Example invocation for damage assessment
On 2 GPUs:
`CUDA_VISIBLE_DEVICES=0,1 python handler.py --pre_directory <pre dir> --post_directory <post dir> --output_directory <output dir> --staging_directory <staging dir>  --destination_crs EPSG:4326 --post_crs EPSG:26915 --model_weight_path weights/weight.pth --model_config_path configs/model.yaml --n_procs <n_proc> --batch_size 2 --num_workers 6`

# Notes:
   - CRS may not be mixed within each type of imagery (pre/post). However, pre and post imagery are not required to share the same CRS.


# xView2 1st place solution
1st place solution for "xView2: Assess Building Damage" challenge. https://www.xview2.org

# Introduction to Solution

Solution developed using this environment:
 - Python 3 (based on Anaconda installation)
 - Pytorch 1.1.0+ and torchvision 0.3.0+ 
 - Nvidia apex https://github.com/NVIDIA/apex
 - https://github.com/skvark/opencv-python
 - https://github.com/aleju/imgaug


Hardware:
Current training batch size requires at least 2 GPUs with 12GB each. (Initially trained on Titan V GPUs). For 1 GPU batch size and learning rate should be found in practice and changed accordingly.

"train", "tier3" and "test" folders from competition dataset should be placed to the current folder.

Use "train.sh" script to train all the models. (~7 days on 2 GPUs).
To generate predictions/submission file use "predict.sh".
"evalution-docker-container" folder contains code for docker container used for final evalution on hold out set (CPU version).

# Trained models
Trained model weights available here: https://vdurnov.s3.amazonaws.com/xview2_1st_weights.zip

(Please Note: the code was developed during the competition and designed to perform separate experiments on different models. So, published as is without additional refactoring to provide fully training reproducibility).


# Data Cleaning Techniques

Dataset for this competition well prepared and I have not found any problems with it.
Training masks generated using json files, "un-classified" type treated as "no-damage" (create_masks.py). "masks" folders will be created in "train" and "tier3" folders.

The problem with different nadirs and small shifts between "pre" and "post" images solved on models level:
 - Frist, localization models trained using only "pre" images to ignore this additional noise from "post" images. Simple UNet-like segmentation Encoder-Decoder Neural Network architectures used here.
 - Then, already pretrained localization models converted to classification Siamese Neural Network. So, "pre" and "post" images shared common weights from localization model and the features from the last Decoder layer concatenated to predict damage level for each pixel. This allowed Neural Network to look at "pre" and "post" separately in the same way and helped to ignore these shifts and different nadirs as well.
 - Morphological dilation with 5*5 kernel applied to classification masks. Dilated masks made predictions more "bold" - this improved accuracy on borders and also helped with shifts and nadirs.


# Data Processing Techniques

Models trained on different crops sizes from (448, 448) for heavy encoder to (736, 736) for light encoder.
Augmentations used for training:
 - Flip (often)
 - Rotation (often)
 - Scale (often)
 - Color shifts (rare)
 - Clahe / Blur / Noise (rare)
 - Saturation / Brightness / Contrast (rare)
 - ElasticTransformation (rare)

Inference goes on full image size (1024, 1024) with 4 simple test-time augmentations (original, filp left-right, flip up-down, rotation to 180).


# Details on Modeling Tools and Techniques

All models trained with Train/Validation random split 90%/10% with fixed seeds (3 folds). Only checkpoints from epoches with best validation score used.

For localization models 4 different pretrained encoders used:
from torchvision.models:
 - ResNet34
from https://github.com/Cadene/pretrained-models.pytorch:
 - se_resnext50_32x4d
 - SeNet154
 - Dpn92

Localization models trained on "pre" images, "post" images used in very rare cases as additional augmentation.

Localization training parameters:
Loss: Dice + Focal
Validation metric: Dice
Optimizer: AdamW

Classification models initilized using weights from corresponding localization model and fold number. They are Siamese Neural Networks with whole localization model shared between "pre" and "post" input images. Features from last Decoder layer combined together for classification. Pretrained weights are not frozen.
Using pretrained weights from localization models allowed to train classification models much faster and to have better accuracy. Features from "pre" and "post" images connected at the very end of the Decoder in bottleneck part, this helping not to overfit and get better generalizing model.

Classification training parameters:
Loss: Dice + Focal + CrossEntropyLoss. Larger coefficient for CrossEntropyLoss and 2-4 damage classes.
Validation metric: competition metric
Optimizer: AdamW
Sampling: classes 2-4 sampled 2 times to give them more attention.

Almost all checkpoints finally finetuned on full train data for few epoches using low learning rate and less augmentations.

Predictions averaged with equal coefficients for both localization and classification models separately.

Different thresholds for localization used for damaged and undamaged classes (lower for damaged).


# Conclusion and Acknowledgments

Thank you to xView2 team for creating and releasing this amazing dataset and opportunity to invent a solution that can help to response to the global natural disasters faster. I really hope it will be usefull and the idea will be improved further.

# References
 - Competition and Dataset: https://www.xview2.org
 - UNet: https://arxiv.org/pdf/1505.04597.pdf
 - Pretrained models for Pytorch: https://github.com/Cadene/pretrained-models.pytorch
 - My 1st place solution from "SpaceNet 4: Off-Nadir Building Footprint Detection Challenge" (some ideas came from here): https://github.com/SpaceNetChallenge/SpaceNet_Off_Nadir_Solutions/tree/master/cannab
