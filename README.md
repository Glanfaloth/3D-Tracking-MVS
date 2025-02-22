# 3D Player Tracking with Multi-View Stream
Project for 3DV 2021 Spring @ ETH Zurich [[Report Link](./document/3dtracking_report_2021.pdf)]

This repo contains a full pipeline to support 3D position tracking of soccer players, with multi-view calibrated moving/fixed video sequences as inputs.
- In single-camera tracking stage, Tracktor++ is used to get 2D positions.
- In multi-camera tracking stage, 2D positions are projected into 3D positions. Then across-camera association is achieved as an optimization problem with spatial, temporal and visual constraints.
- In the end, visualization in 2D, 3D and a voronoi visualization for sports coaching purpose are provided.

|3D Tracking|Sports Coaching|
|:-------------------------:|:-------------------------:|
|![demo](https://github.com/Glanfaloth/3D-Tracking-MVS/blob/main/misc/cam1_right_team.gif)|![demo](https://github.com/Glanfaloth/3D-Tracking-MVS/blob/main/misc/cam1_right_team_gt_voronoi.gif)|

## Demo
Check demo scripts as examples
> Currently, processed data is under protection due to legal issues. 

- Run the demo visualization on the moving cameras
```shell
bash script/demo_moving.sh
```
- Run the demo visualization on the fixed cameras
```shell
bash script/demo_fix.sh
```

## Preprocessing 
- Split video into image frames
```python
python src/utils/v2img.py
       --pathIn=data/0125-0135/CAM1/CAM1.mp4
       --pathOut=data/0125-0135/CAM1/img
       --splitnum=1
```
- Estimate football pitch homography (size 120m * 90m [ref](https://www.quora.com/What-are-the-official-dimensions-of-a-soccer-field-in-the-FIFA-World-Cup))
> [FIFA official document](https://img.fifa.com/image/upload/datdz0pms85gbnqy4j3k.pdf)

```python
python src/utils/computeHomo.py
       --img=data/0125-0135/RIGHT/img/image0000.jpg
       --out_dir=data/0125-0135/RIGHT/
```

- Handle moving cameras
```python
python src/utils/mov2static.py
       --calib_file=data/calibration_results/0125-0135/CAM1/calib.txt
       --img_dir=data/0125-0135/CAM1/img
       --output_dir=data/0125-0135/CAM1/img_static
```
- Convert ground truth/annotation json to text file
```python
python src/utils/json2txt.py --jsonfile=data/0125-0135/0125-0135.json
```


- After processing, data folder structure should be like:
```
data
├── 0125-0135
│   ├── CAM1
│   │   ├── img
│   │   ├── img_static
│   │   └── homo.npy
│   ├── RIGHT
│   │   
│   ├── proj_config.txt
│   ├── 16m_left.txt
│   ├── 16m_right.txt
│   └── id_mapping.txt
│       
└── calibration_results
    └── 0125-0135
        ├── CAM1
        └── RIGHT
```

- [Download preprocessed data](https://polybox.ethz.ch/index.php/s/CvcT5pxOY90bpIF)
> only include homography and config files, large image folder not included

## Single-camera tracking
- Object Detector: frcnn_fpn
Train object detector and generate detection results with [this](https://colab.research.google.com/drive/18CI160namP1-sF82H6sgrDycvHZ1PbPm?usp=sharing) Google Colab notebook. [[pretrained model](https://polybox.ethz.ch/index.php/s/SrBn2DtKEJQaWFg?path=%2Ftrained_frcnn_fpn)]
- Run Tracktor++
Put trainded object detector ```model_epoch_50.model``` into  ```src/tracking_wo_bnw/output/faster_rcnn_fpn_training_soccer/```.
Put data and calibration results into ```src/tracking_wo_bnw/```.

```python
cd src/tracking_wo_bnw
python experiments/scripts/test_tracktor.py
```
- Run ReID(team id) model
```python
python src/team_classification/team_svm.py PATH_TO_TRACKING_RESULT PATH_TO_IMAGES
```
- Convert tracking results to coordinates on the pitch
> Equation to find the intersection of a line with a plane ([ref](https://math.stackexchange.com/questions/2041296/algorithm-for-line-in-plane-intersection-in-3d))

```python
python src/calib.py --calib_path=PATH_TO_CALIB --res_path=PATH_TO_TRACKING_RESULT --xymode --reid

# also plot the camera positions for fixed cameras
python src/calib.py --calib_path=PATH_TO_CALIB --res_path=PATH_TO_TRACKING_RESULT --viz
```
## Across-camera association

- Run two-cam tracker
```python
python src/runMCTRacker.py 

# add team id constraint
python src/runMCTRacker.py --doreid
```

- Run multi-cam tracker (e.g. 8 cams)
```python
python src/runTreeMCTracker.py --doreid
```

## Evaluation

- Produce quatitative results (visualize results)
> visualize 2d bounding box

```python
# if format <x, y, w, h>
python src/utils/visualize.py
       --img_dir=data/0125-0135/RIGHT/img
       --result_file=output/tracktor/16m_right_prediction.txt 
# if format <x1, y1, x2, y2>
python src/utils/visualize.py
       --img_dir=data/0125-0135/RIGHT/img
       --result_file=output/iou/16m_right.txt
       --xymode
# if with team id
python src/utils/visualize.py
       --img_dir=data/0125-0135/RIGHT/img
       --result_file=output/tracktor/16m_right_prediction.txt
       --reid
# if 3d mode
python src/utils/visualize.py
       --img_dir=data/0125-0135/RIGHT/img
       --result_file=output/tracktor/RIGHT.txt
       --calib_file=data/calibration_results/0125-0135/RIGHT/calib.txt
       --pitchmode
```
> visualize 3d tracking result with ground truth and voronoi diagram

```python
python src/utils/visualize_on_pitch.py
       --result_file=PATH_TO_TRACKING_RESULT
       --ground_truth=PATH_TO_GROUND_TRUTH
```
> visualize 3d ground truth on camera frames (reprojection)

```python
python src/utils/visualize_tracab
       --img_path=PATH_TO_IMAGES
       --calib_path=PATH_TO_CALIB
       --gt_path=PATH_TO_TRACAB_GT
       --output_path=PATH_TO_OUTPUT_VIDEO
```
- Produce quantitative result

```python
# 2d <frame id, objid, x, y, w, h, .., ...>
python src/motmetrics/apps/eval_motchallenge.py data/0125-0135/ output/tracktor_filtered

# 3d
python src/utils/eval3d.py
       --pred=output/pitch/EPTS_3_pitch.txt_EPTS_4_pitch.txt.txt
       --fixcam
       --gt=data/fixedcam/gt_pitch_550.txt
python src/utils/eval3d.py
       --fixcam
       --boxplot
```


## Result

> 0125-0135 right moving camera
```text
SORT
         IDF1   IDP   IDR  Rcll  Prcn GT MT PT ML   FP   FN IDs   FM   MOTA  MOTP IDt IDa IDm
RIGHT   58.3% 49.4% 71.3% 83.4% 57.8% 25 15 10  0 3674 1001  27  181  22.0% 0.340   5  23   1
CAM1    43.8% 35.3% 57.5% 67.9% 41.7% 22  6 15  1 2786  942  36  112 -28.2% 0.360   7  27   0
OVERALL 53.3% 44.4% 66.8% 78.3% 52.1% 47 21 25  1 6460 1943  63  293   5.5% 0.346  12  50   1

IOU
         IDF1   IDP   IDR  Rcll  Prcn GT MT PT ML   FP   FN IDs   FM   MOTA  MOTP IDt IDa IDm
RIGHT   61.8% 53.3% 73.6% 82.9% 60.0% 25 17  8  0 3330 1029  46  279  26.9% 0.336  15  23   1
CAM1    42.6% 33.2% 59.2% 67.7% 38.0% 22  6 15  1 3238  948  53  144 -44.4% 0.355   7  31   0
OVERALL 54.9% 45.6% 68.9% 77.9% 51.5% 47 23 23  1 6568 1977  99  423   3.6% 0.341  22  54   1

Tracktor
         IDF1   IDP   IDR  Rcll  Prcn GT MT PT ML   FP   FN IDs   FM   MOTA  MOTP IDt IDa IDm
RIGHT   72.6% 69.7% 75.8% 85.9% 79.0% 25 19  6  0 1376  848  21  213  62.8% 0.322   7  15   1
CAM1    49.3% 39.4% 65.9% 69.4% 41.5% 22  6 15  1 2871  899  17  146 -29.0% 0.344  10   7   0
OVERALL 63.7% 56.7% 72.6% 80.5% 63.0% 47 25 21  1 4247 1747  38  359  32.7% 0.328  17  22   1

```

## Acknowledgement
We would like to thank the following Github repos or softwares:
- [Supervisely](https://app.supervise.ly/init)
- [Tracktor++](https://github.com/phil-bergmann/tracking_wo_bnw)
- [IoU tracker](https://github.com/GBJim/iou_tracker)
- [SORT](https://github.com/abewley/sort)
- [Football crunching (footyviz)](https://medium.com/football-crunching)
- [Pymotmetrics](https://github.com/cheind/py-motmetrics)

## Authors
Yuchang Jiang, Tianyu Wu, Ying Jiao, Yelan Tao
