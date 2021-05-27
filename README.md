# 3D-Tracking-MVS
Course project for 3DV 2021 Spring @ ETH Zurich

## Preprocessing 
- Split video into image frames
```
python src/utils/v2img.py --pathIn=data/0125-0135/CAM1/CAM1.mp4 --pathOut=data/0125-0135/CAM1/img --splitnum=1
```
- Estimate football pitch homography (size 120m * 90m [ref](https://www.quora.com/What-are-the-official-dimensions-of-a-soccer-field-in-the-FIFA-World-Cup))
```
python src/utils/computeHomo.py --img=data/0125-0135/RIGHT/img/image0000.jpg --out_dir=data/0125-0135/RIGHT/
```
[FIFA official document](https://img.fifa.com/image/upload/datdz0pms85gbnqy4j3k.pdf)
- Handle moving cameras
```
python src/utils/mov2static.py --calib_file=data/calibration_results/0125-0135/CAM1/calib.txt --img_dir=data/0125-0135/CAM1/img --output_dir=data/0125-0135/CAM1/img_static
```
- Convert ground truth/annotation json to text file
```
python src/utils/json2txt.py --jsonfile=data/0125-0135/0125-0135.json
```
- Backproject to ground plane

Equation to find the intersection of a line with a plane ([ref](https://math.stackexchange.com/questions/2041296/algorithm-for-line-in-plane-intersection-in-3d))
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
- Train Faster RCNN
```
Jiao Ying ???
```
- Run Tracktor
```
Jiao Ying ???
```
- Run ReID(team id) model
```
Jiao Ying ???
```
- Convert tracking results to coordinates on the pitch
```
python src/calib.py --calib_path=PATH_TO_CALIB --res_path=PATH_TO_TRACKING_RESULT --xymode
```
## Cross-camera link

- Run multi-cam tracker
```
python src/runMCTRacker.py 
```

- Run multi-cam tracker with team id constraint
```
python src/runMCTRacker.py --doreid
```

## Evaluation

- Produce quatitative results (visualize results)
> visualize 2d bounding box

```
# if format <x, y, w, h>
python src/utils/visualize.py --img_dir=data/0125-0135/RIGHT/img --result_file=output/tracktor/16m_right_prediction.txt 
# if format <x1, y1, x2, y2>
python src/utils/visualize.py --img_dir=data/0125-0135/RIGHT/img --result_file=output/iou/16m_right.txt --xymode
# if 3d mode
python src/utils/visualize.py --img_dir=data/0125-0135/RIGHT/img --result_file=output/tracktor/RIGHT.txt --calib_file=data/calibration_results/0125-0135/RIGHT/calib.txt  --pitchmode
```
> visualize 3d position on the pitch
- visualize tracking result with ground truth and voronoi (with footyviz)
```
python src/visualize_on_pitch.py --result_file=PATH_TO_TRACKING_RESULT --ground_truth=PATH_TO_GROUND_TRUTH
```
- visualize ground truth on camera frames
```
python src/visualize_tracab --img_path=PATH_TO_IMAGES --calib_path=PATH_TO_CALIB --gt_path=PATH_TO_TRACAB_GT --output_path=PATH_TO_OUTPUT_VIDEO
```
- Produce quantitative result

```
# <fid, objid, x, y, w, h, .., ...>
python src/motmetrics/apps/eval_motchallenge.py data/0125-0135/ output/tracktor_filtered

```

## Acknowledgement
We would like to thank the following Github repos or softwares:
[Supervisely]()
[Tracktor++]()



## Useful literature

- Learning to Track and Identify Players from Broadcast Sports Videos 2012 :rainbow:[[paper](https://www.cs.ubc.ca/~murphyk/Papers/weilwun-pami12.pdf)]
- Multicamera people tracking with probabilistic occupancy map 2013 [[paper](https://infoscience.epfl.ch/record/145991)][[project](https://www.epfl.ch/labs/cvlab/research/research-surv/research-body-surv-index-php/)]
- Multi-camera multi-player tracking with deep player identification in sports video 2020 [[paper](https://www.sciencedirect.com/science/article/abs/pii/S0031320320300650)]

## Useful Github repo
[Full pipeline: POM+DeepOcculusion+pyKSP](https://www.epfl.ch/labs/cvlab/research/research-surv/research-body-surv-index-php/) <br/>
[Soccer player tracking system](https://github.com/AndresGalaviz/Football-Player-Tracking) <br/>
[CVPR2020: How To Train Your Deep Multi-Object Tracker](https://github.com/yihongXU/deepMOT)
