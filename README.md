# HeadPoseEstimator

This document is intended to provide a detailed description of the component and how it works for future development. The component is a C++ implementation of the publication in [Head Pose Estimation in Sign Language Video](http://link.springer.com/chapter/10.1007%2F978-3-642-38886-6_34).

The SLMotion component `HeadPoseEstimator` computes three head angles (yaw, pitch, roll) from an image where a face can be found and facial landmarks and a skin mask have been previously estimated. The component uses the *flandmark* package for facial landmark detection, but any other landmark estimator can be used. The facial landmarks are obtained using the `FacialLandmarkDetector` component from SLMotion. The skin mask component used by default is the `RuleSkinDetector` but any other similar skin mask component is also compatible for head pose estimation.

The headpose estimator employs SVM regression to obtain the angles from the extracted feature vector. The SVM model in this implementation was computed using the `libSVM` 3.17 package.

There is a difference in performance between what was reported in the paper and the SLMotion implementation. This is because for use in SLMotion, the SVM model was rebuilt and the parameters were also updated accordingly. There is also a subtle difference in regression performance when using either windows or linux as platforms even if the same SVM model file is used. The correlation of the head pose estimator in the 'danny' video experiment is as follows:

Model | Yaw | Pitch | Roll
:--- | ---: | ---: | ---:
Reported | 0.92 | 0.74 | 0.95 
SLMotion | 0.91 | 0.79 | 0.94
SLMotion (no filtering) | 0.88 | 0.75 | 0.90

## Usage

The 

```sh
./slmotion --components FaceDetector,RuleSkinDetector ../../Downloads/Panasonic_TWdata1.avi -o ../../hpe/data/mocapmask-%f.png --feature-out ../../hpe/data/command.dat -v --complex-visualisation showMatrix skinmask;cropToRect facedetection --disable-annotation-bar
```

### Output

kk

## Feature Vector

If the face landmarks are estimated with a different package, the HeadPose component expect them to be in the same format as computed by `SLMotion`. The landmark data is a vector of 8 coordinate points `std::vector<cv::Point2f>` with the following format:

Index (x,y)   | Point
:---------:   | :---
(0, 1)        | face center
(2, 3)        | right eye, inner corner
(4, 5)        | left eye, inner corner
(6, 7)        | mouth, right corner
(8, 9)        | mouth,left corner
(10, 11)      | right eye, outter corner
(12, 13)      | left eye, outter corner
(14, 15)      | nose tip

The face bounding box is also employed, the default format implemented is from `cv::Rect`. Finally, any skin detector can be used while it returns a mask for the whole image. To increase performance, only the face region can be estimated but the code needs to support this change.

The quantified steps for the head pose range with this setup

 . | 45 | 30 | 15 | 0 | -15 | -30 | -45
:---: | :---: | :---: | :---: | :---: | :---: | :---: | :---:
30 | | | | 2 | | | |
15 | | | | 1 | | | | 
0 | 3 | 2 | 1 | 0 | -1 | -2 | -3
-15 | | | | -1        	
-30 | | | | -2

## Training a new Regressor

To train a new regressor the `libSVM` tools are necessary. The default feature vector for yaw and pitch are as introduced in the paper, but the code also supports different combinations making small changes according to the feature vector size and SVR model file to be used.

For **yaw**, the used vector for training is of size 14. The vector consists of landmarks 2 to 15 normalized and displaced by skin area. For **pitch** the vector is of size 20, including all landmarks and the 4 skin mask areas.

First a file with the sparse representation of yaw feature vectors is needed. The file can be created from matlab using the provided libraries by `libSVM´.
```cpp
features_sparse = sparse(dataset);
libsvmwrite('data/features.train', labels, features_sparse);
```

Then the training data is scaled between -1 and +1. First the ranges are computed, then a new training data file is created with scaled elements.
```
./svm-scale -s ranges/features.train.range data/features.train
./svm-scale -r ranges/features.train.range data/features.train > data/features.train.scale
```

A parameter search tool for regression ([gridregression.py](http://www.csie.ntu.edu.tw/~cjlin/libsvmtools/gridsvr/gridregression.py)) is used to obtain the best parameter combination for the current training data.
```
python gridregression.py data/features.train.scale
```

The final model is created from the set of parameters `-c`, `-g`, and `-p` obtained from the last step to create a model such as
```
./svm-train -s 3 -c 2 -g 1 -p 0.25 -b 1 features.train
```

All tested models in the reported performances were trained this way. The exception is that this procedure was done automatically for all possible options.

The performance is obtained such that
```
./svm-predict -b 1 features.test features.train.model
```

### Filter

A FIR-II filter is used to remove noise from the final 


## Possible extensions

### Adding scaling to the estimator



```cpp
const std::string DEFAULT_YAW_RANGE = "../share/slmotion/hpe_yaw.range";
const std::string DEFAULT_PIT_RANGE = "../share/slmotion/hpe_pitch.range";

std::vector<std::vector<double>> getRangeVector(std::string m);
  
yawRange(DEFAULT_YAW_RANGE),
pitRange(DEFAULT_PIT_RANGE),
yawRangeVector(getRangeVector(yawRange)),
pitRangeVector(getRangeVector(pitRange)),

inline void setYawRange(std::string x) {
  yawRange = x;
}

inline void setPitRange(std::string x) {
  pitRange = x;
}

std::string yawRange;  // file name of the svm yaw range
std::string pitRange;  // file name of the svm pit range
std::vector<std::vector<double>> yawRangeVector;
std::vector<std::vector<double>> pitRangeVector;
```

To read the range file in the `libSVM` format the following code can be used:

```cpp
/**
* Reads libSVM range file
* Variation from libSVM original code for svm-scale.c
*/
std::vector<std::vector<double>> getRangeVector(std::string restore_filename) {
    std::vector<std::vector<double>> v(2);
       FILE *fp_restore = NULL;

       fp_restore = fopen(restore_filename.c_str(),"r");
   	if(fp_restore==NULL) {
    	std::cerr << "Can not open range file" << std::endl;
    	exit(1);
   	}

    int idx, c;
    double lower, upper, fmin, fmax;

    c = fgetc(fp_restore);
    if(c == 'x') {
    	fscanf(fp_restore, "%lf %lf\n", &lower, &upper);
    	while(fscanf(fp_restore,"%d %lf %lf\n",&idx,&fmin,&fmax)==3) {
        	v[0].push_back(fmin);
        	v[1].push_back(fmax);
    	}
    }
    fclose(fp_restore);
    return v;
}
```
