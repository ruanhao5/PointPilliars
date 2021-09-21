# PointPillars使用手册

源代码参考：[traveller59/second.pytorch: SECOND for KITTI/NuScenes object detection (github.com)](https://github.com/traveller59/second.pytorch)，这里进行了简单修改，使用docker可以直接运行代码。

## 下载代码

相关代码已经放在GitHub网站[ruanhao5/PointPilliars (github.com)](https://github.com/ruanhao5/PointPilliars)。

下载代码：

```bash
git clone https://github.com/ruanhao5/PointPilliars.git
```

## 准备数据

新建存放KITTI数据的文件夹：

```bash
cd PointPilliars
mkdir data-KITTI
```

然后开始下载KITTI数据集，有两种途径可以下载，分别是KITTI官网和格物钛官网。

**官网下载**：

可以去官网[The KITTI Vision Benchmark Suite (cvlibs.net)](http://www.cvlibs.net/datasets/kitti/eval_object.php?obj_benchmark=3d)下载官方数据集，主要有4个文件需要下载：

- Camera calibration matrices of object data set (16 MB):
  https://s3.eu-central-1.amazonaws.com/avg-kitti/data_object_calib.zip

- left color images of object data set (12GB):
  https://s3.eu-central-1.amazonaws.com/avg-kitti/data_object_image_2.zip
- Training labels of object data set (5 MB):
  https://s3.eu-central-1.amazonaws.com/avg-kitti/data_object_label_2.zip

- Volodyne points clouds(29GB):
  https://s3.eu-central-1.amazonaws.com/avg-kitti/data_object_velodyne.zip

**格物钛网站下载**：

如果官网下载失败，可以去Graviti 格物钛官网[Download - Data Decorators/KITTI-object - Graviti](https://gas.graviti.cn/dataset/data-decorators/KITTIObject/download)下载。注册账号，然后就可以下载了。同样还是上面提到的4个文件。

下载完成后，可以得到如下4个文件：

```bash
总用量 39G
-rwxrwxrwx 1 ruanhao ruanhao  27G 6月  18 10:22 data_object_velodyne.zip
-rwxrwxrwx 1 ruanhao ruanhao 5.4M 6月  18 10:18 data_object_label_2.zip
-rwxrwxrwx 1 ruanhao ruanhao  12G 6月  18 10:16 data_object_image_2.zip
-rwxrwxrwx 1 ruanhao ruanhao  26M 6月  18 10:14 data_object_calib.zip
```

将下载的4个文件夹存放到`data-KITTI`中，然后解压：

```bash
cd data-KITTI/

# 解压当前目录下的4个zip文件
unzip data_object_velodyne.zip
unzip data_object_label_2.zip
unzip data_object_image_2.zip
unzip data_object_calib.zip

# 删除zip文件
rm -rf ./*.zip
```

解压完成后，按照下面的形式组织(需要自己添加空的文件夹`velodyne_reduced`)

```
└── data-KITTI
       ├── training    <-- 7481 train data
       |   ├── image_2 <-- for visualization
       |   ├── calib
       |   ├── label_2
       |   ├── velodyne
       |   └── velodyne_reduced <-- empty directory
       └── testing     <-- 7580 test data
           ├── image_2 <-- for visualization
           ├── calib
           ├── velodyne
           └── velodyne_reduced <-- empty directory
```

## docker

> Note: docker image is available on CUDA 11.0

不想浪费时间在环境调试上，可以使用我做好的docker。使用如下命令，拉取镜像：

```bash
docker pull ruanhao5/pointpillars:v5.0
```

之后的docker命令，需要使用nvidia下的docker操作。

## 训练数据

因为运行时间较长，建议使用tmux：

```bash
tmux new -s pointpilliars		# create a new PointPillars session
```

运行docker镜像：

```bash
cd PointPilliars
docker run -it --rm --gpus all -v $PWD:/workspace/ -w /workspace/ --shm-size 256G ruanhao5/pointpillars:v5.0 /bin/bash
```

产生KITTI infos（这可能会花上几分钟）：

```bash
cd /workspace/
python second/create_data.py kitti_data_prep --root_path=/workspace/data-KITTI/
```

顺利的话，可以得到1个文件夹`gt_database`和5个文件：

- kitti_infos_train.pkl
- kitti_infos_val.pkl
- kitti_infos_trainval.pkl
- kitti_infos_test.pkl
- kitti_dbinfos_train.pkl

原先的空文件夹`velodyne_reduced`也会有生成的数据。

配置文件我默认使用2个GPU，如果修改GPU个数，那么需要修改配置文件：

> 如果使用多个GPU，需要修改配置文件里面的`steps` and `steps_per_eval` 参数，除以GPU个数即可。
>
> 比如：`second/configs/pointpillars/car/xyres_16.config`文件中的`steps` and `steps_per_eval`，原始数字是296960和9280，因为我使用2个GPU，所以除以2，变为148480和4640。

训练数据并预测结果：

```bash
# train 使用2块GPU
# 汽车
cd /workspace/second
CUDA_VISIBLE_DEVICES=0,1 python ./pytorch/train.py train --config_path=./configs/pointpillars/car/xyres_16.config --model_dir=../car_xyres16_2GPU/ --multi_gpu=True --resume=True
# (Note:文件夹car_xyres16_2GPU不存在，才能正常运行)


# 行人和骑自行车的人
cd /workspace/second
CUDA_VISIBLE_DEVICES=0,1 python ./pytorch/train.py train --config_path=./configs/pointpillars/ped_cycle/xyres_16.config --model_dir=../ped_cycle_xyres16_2GPU/ --multi_gpu=True --resume=True
# (Note:文件夹ped_cycle_xyres16_2GPU不存在，才能正常运行)

结果保存在${model_dir}/log.txt中
```

得到如下的结果：

```bash
Car AP(Average Precision)@0.70, 0.70, 0.70:
bbox AP:90.50, 88.48, 86.89
bev  AP:89.85, 86.94, 84.35
3d   AP:84.20, 75.02, 68.71
aos  AP:0.67, 1.77, 2.74
Car AP(Average Precision)@0.70, 0.50, 0.50:
bbox AP:90.50, 88.48, 86.89
bev  AP:90.76, 89.85, 89.26
3d   AP:90.75, 89.74, 89.07
aos  AP:0.67, 1.77, 2.74
---------------------------------------------
# 25 epoch

Cyclist AP(Average Precision)@0.50, 0.50, 0.50:
bbox AP:83.27, 67.14, 63.49
bev  AP:83.39, 64.32, 59.52
3d   AP:82.86, 61.61, 57.14
aos  AP:0.80, 4.24, 4.23
Cyclist AP(Average Precision)@0.50, 0.25, 0.25:
bbox AP:83.27, 67.14, 63.49
bev  AP:84.37, 66.96, 63.35
3d   AP:84.32, 66.26, 62.66
aos  AP:0.80, 4.24, 4.23
Pedestrian AP(Average Precision)@0.50, 0.50, 0.50:
bbox AP:54.68, 51.64, 48.75
bev  AP:60.04, 54.13, 50.63
3d   AP:52.40, 47.15, 42.72
aos  AP:31.90, 29.47, 27.59
Pedestrian AP(Average Precision)@0.50, 0.25, 0.25:
bbox AP:54.68, 51.64, 48.75
bev  AP:69.62, 65.49, 61.69
3d   AP:69.52, 65.12, 61.33
aos  AP:31.90, 29.47, 27.59
```
