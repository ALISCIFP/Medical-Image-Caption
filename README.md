# Self-critical Sequence Training for Image Captioning

This is an unofficial implementation for [Self-critical Sequence Training for Image Captioning](https://arxiv.org/abs/1612.00563). The result of FC model can be replicated. (Not able to replicate Att2in result.)

The author helped me a lot when I tried to replicate the result. Great thanks. The latest topdown and att2in2 model can achieve 1.12 Cider score on Karpathy's test split after self-critical training.

This is based on my [neuraltalk2.pytorch](https://github.com/ruotianluo/neuraltalk2.pytorch) repository. The modifications is:
- Add self critical training.

## Requirements
Python 2.7 (because there is no [coco-caption](https://github.com/tylin/coco-caption) version for python 3)
PyTorch 0.2 (along with torchvision)

You need to download pretrained resnet model for both training and evaluation. The models can be downloaded from [here](https://drive.google.com/open?id=0B7fNdx_jAqhtbVYzOURMdDNHSGM), and should be placed in `data/imagenet_weights`.

## Pretrained models
Pretrained models are provided [here](https://drive.google.com/open?id=0B7fNdx_jAqhtdE1JRXpmeGJudTg). And the performances of each model will be maintained in this [issue](https://github.com/ruotianluo/neuraltalk2.pytorch/issues/10).

If you want to do evaluation only, then you can follow [this section](#generate-image-captions) after downloading the pretrained models.

## Train your own network on ImageCLEF dataset
ImageCLEF 2018 caption: http://www.imageclef.org/2018/caption

ImageCLEF 2017 caption review: http://ceur-ws.org/Vol-1866/

ImageCLEF 2017leaderboard: http://www.imageclef.org/2017/caption


## Train your own network on Eric Xing's Medical image Caption paper https://arxiv.org/abs/1711.08195 dataset.

### Download Chest X-ray  dataset and preprocessing.
Download:https://www.dropbox.com/s/uanpmhfvpe7gbhk/chest_xray.zip?dl=0

jason preprocessed: https://www.dropbox.com/sh/zar9xooe0557wqp/AABagbPxDB3SAUBZoR1umguEa?dl=0

## Train your own network on IU X-Ray

### Download dataset and preprocessing

First, download the coco images from [link](http://mscoco.org/dataset/#download). We need 2014 training images and 2014 val. images. You should put the `train2014/` and `val2014/` in the same directory, denoted as `$IMAGE_ROOT`.

Download preprocessed coco captions from [link](http://cs.stanford.edu/people/karpathy/deepimagesent/caption_datasets.zip) from Karpathy's homepage. Extract `dataset_coco.json` from the zip file and copy it in to `data/`. This file provides preprocessed captions and also standard train-val-test splits.

Once we have these, we can now invoke the `prepro_*.py` script, which will read all of this in and create a dataset (two feature folders, a hdf5 label file and a json file).

```bash
$ python scripts/prepro_labels.py --input_json Chest_Xray/dataset_labels.json --output_json Chest_Xray/cocotalk.json --output_h5 Chest_Xray/cocotalk
$ python scripts/prepro_feats.py --input_json Chest_Xray/dataset_labels.json --output_dir Chest_Xray/cocotalk --images_root $IMAGE_ROOT
```

`prepro_labels.py` will map all words that occur <= 5 times to a special `UNK` token, and create a vocabulary for all the remaining words. The image information and vocabulary are dumped into `data/cocotalk.json` and discretized caption data are dumped into `data/cocotalk_label.h5`.

`prepro_feats.py` extract the resnet101 features (both fc feature and last conv feature) of each image. The features are saved in `data/cocotalk_fc` and `data/cocotalk_att`, and resulting files are about 200GB.

(Check the prepro scripts for more options, like other resnet models or other attention sizes.)

**Warning**: the prepro script will fail with the default MSCOCO data because one of their images is corrupted. See [this issue](https://github.com/karpathy/neuraltalk2/issues/4) for the fix, it involves manually replacing one image in the dataset.

### Start training

```bash
$ python train.py --id fc --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-4 --learning_rate_decay_start 0 --scheduled_sampling_start 0 --checkpoint_path log_fc --save_checkpoint_every 6000 --val_images_use 5000 --max_epochs 30
```

The train script will dump checkpoints into the folder specified by `--checkpoint_path` (default = `save/`). We only save the best-performing checkpoint on validation and the latest checkpoint to save disk space.

To resume training, you can specify `--start_from` option to be the path saving `infos.pkl` and `model.pth` (usually you could just set `--start_from` and `--checkpoint_path` to be the same).

If you have tensorflow, the loss histories are automatically dumped into `--checkpoint_path`, and can be visualized using tensorboard.

The current command use scheduled sampling, you can also set scheduled_sampling_start to -1 to turn off scheduled sampling.

If you'd like to evaluate BLEU/METEOR/CIDEr scores during training in addition to validation cross entropy loss, use `--language_eval 1` option, but don't forget to download the [coco-caption code](https://github.com/tylin/coco-caption) into `coco-caption` directory.

For more options, see `opts.py`.

**A few notes on training.** To give you an idea, with the default settings one epoch of MS COCO images is about 11000 iterations. After 1 epoch of training results in validation loss ~2.5 and CIDEr score of ~0.68. By iteration 60,000 CIDEr climbs up to about ~0.84 (validation loss at about 2.4 (under scheduled sampling)).

### Train using self critical

First you should preprocess the dataset and get the cache for calculating cider score:
```
$ python scripts/prepro_ngrams.py --input_json .../dataset_coco.json --dict_json data/cocotalk.json --output_pkl data/coco-train --split train
```

And also you need to clone my forked [cider](https://github.com/ruotianluo/cider) repository.

Then, copy the model from the pretrained model using cross entropy. (It's not mandatory to copy the model, just for back-up)
```
$ bash scripts/copy_model.sh fc fc_rl
```

Then
```bash
$ python train.py --id fc_rl --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-5 --start_from log_fc_rl --checkpoint_path log_fc_rl --save_checkpoint_every 6000 --language_eval 1 --val_images_use 5000 --self_critical_after 30
```

You will see a huge boost on Cider score, : ).

**A few notes on training.** Starting self-critical training after 30 epochs, the CIDEr score goes up to 1.05 after 600k iterations (including the 30 epochs pertraining).

### Caption images after training

## Generate image captions

### Evaluate on raw images
Now place all your images of interest into a folder, e.g. `blah`, and run
the eval script:

```bash
$ python eval.py --model model.pth --infos_path infos.pkl --image_folder blah --num_images 10
```

This tells the `eval` script to run up to 10 images from the given folder. If you have a big GPU you can speed up the evaluation by increasing `batch_size`. Use `--num_images -1` to process all images. The eval script will create an `vis.json` file inside the `vis` folder, which can then be visualized with the provided HTML interface:

```bash
$ cd vis
$ python -m SimpleHTTPServer
```

Now visit `localhost:8000` in your browser and you should see your predicted captions.

### Evaluate on Karpathy's test split

```bash
$ python eval.py --dump_images 0 --num_images 5000 --model model.pth --infos_path infos.pkl --language_eval 1
```

The defualt split to evaluate is test. The default inference method is greedy decoding (`--sample_max 1`), to sample from the posterior, set `--sample_max 0`.

**Beam Search**. Beam search can increase the performance of the search for greedy decoding sequence by ~5%. However, this is a little more expensive. To turn on the beam search, use `--beam_size N`, N should be greater than 1.



## Train your own network on COCO

### Download COCO dataset and preprocessing

First, download the coco images from [link](http://mscoco.org/dataset/#download). We need 2014 training images and 2014 val. images. You should put the `train2014/` and `val2014/` in the same directory, denoted as `$IMAGE_ROOT`.

Download preprocessed coco captions from [link](http://cs.stanford.edu/people/karpathy/deepimagesent/caption_datasets.zip) from Karpathy's homepage. Extract `dataset_coco.json` from the zip file and copy it in to `data/`. This file provides preprocessed captions and also standard train-val-test splits.

Once we have these, we can now invoke the `prepro_*.py` script, which will read all of this in and create a dataset (two feature folders, a hdf5 label file and a json file).

```bash
$ python scripts/prepro_labels.py --input_json data/dataset_coco.json --output_json data/cocotalk.json --output_h5 data/cocotalk
$ python scripts/prepro_feats.py --input_json data/dataset_coco.json --output_dir data/cocotalk --images_root $IMAGE_ROOT
```

`prepro_labels.py` will map all words that occur <= 5 times to a special `UNK` token, and create a vocabulary for all the remaining words. The image information and vocabulary are dumped into `data/cocotalk.json` and discretized caption data are dumped into `data/cocotalk_label.h5`.

`prepro_feats.py` extract the resnet101 features (both fc feature and last conv feature) of each image. The features are saved in `data/cocotalk_fc` and `data/cocotalk_att`, and resulting files are about 200GB.

(Check the prepro scripts for more options, like other resnet models or other attention sizes.)

**Warning**: the prepro script will fail with the default MSCOCO data because one of their images is corrupted. See [this issue](https://github.com/karpathy/neuraltalk2/issues/4) for the fix, it involves manually replacing one image in the dataset.

### Start training

```bash
$ python train.py --id fc --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-4 --learning_rate_decay_start 0 --scheduled_sampling_start 0 --checkpoint_path log_fc --save_checkpoint_every 6000 --val_images_use 5000 --max_epochs 30
```

The train script will dump checkpoints into the folder specified by `--checkpoint_path` (default = `save/`). We only save the best-performing checkpoint on validation and the latest checkpoint to save disk space.

To resume training, you can specify `--start_from` option to be the path saving `infos.pkl` and `model.pth` (usually you could just set `--start_from` and `--checkpoint_path` to be the same).

If you have tensorflow, the loss histories are automatically dumped into `--checkpoint_path`, and can be visualized using tensorboard.

The current command use scheduled sampling, you can also set scheduled_sampling_start to -1 to turn off scheduled sampling.

If you'd like to evaluate BLEU/METEOR/CIDEr scores during training in addition to validation cross entropy loss, use `--language_eval 1` option, but don't forget to download the [coco-caption code](https://github.com/tylin/coco-caption) into `coco-caption` directory.

For more options, see `opts.py`. 

**A few notes on training.** To give you an idea, with the default settings one epoch of MS COCO images is about 11000 iterations. After 1 epoch of training results in validation loss ~2.5 and CIDEr score of ~0.68. By iteration 60,000 CIDEr climbs up to about ~0.84 (validation loss at about 2.4 (under scheduled sampling)).

### Train using self critical

First you should preprocess the dataset and get the cache for calculating cider score:
```
$ python scripts/prepro_ngrams.py --input_json .../dataset_coco.json --dict_json data/cocotalk.json --output_pkl data/coco-train --split train
```

And also you need to clone my forked [cider](https://github.com/ruotianluo/cider) repository.

Then, copy the model from the pretrained model using cross entropy. (It's not mandatory to copy the model, just for back-up)
```
$ bash scripts/copy_model.sh fc fc_rl
```

Then
```bash
$ python train.py --id fc_rl --caption_model fc --input_json data/cocotalk.json --input_fc_dir data/cocotalk_fc --input_att_dir data/cocotalk_att --input_label_h5 data/cocotalk_label.h5 --batch_size 10 --learning_rate 5e-5 --start_from log_fc_rl --checkpoint_path log_fc_rl --save_checkpoint_every 6000 --language_eval 1 --val_images_use 5000 --self_critical_after 30
```

You will see a huge boost on Cider score, : ).

**A few notes on training.** Starting self-critical training after 30 epochs, the CIDEr score goes up to 1.05 after 600k iterations (including the 30 epochs pertraining).

### Caption images after training

## Generate image captions

### Evaluate on raw images
Now place all your images of interest into a folder, e.g. `blah`, and run
the eval script:

```bash
$ python eval.py --model model.pth --infos_path infos.pkl --image_folder blah --num_images 10
```

This tells the `eval` script to run up to 10 images from the given folder. If you have a big GPU you can speed up the evaluation by increasing `batch_size`. Use `--num_images -1` to process all images. The eval script will create an `vis.json` file inside the `vis` folder, which can then be visualized with the provided HTML interface:

```bash
$ cd vis
$ python -m SimpleHTTPServer
```

Now visit `localhost:8000` in your browser and you should see your predicted captions.

### Evaluate on Karpathy's test split

```bash
$ python eval.py --dump_images 0 --num_images 5000 --model model.pth --infos_path infos.pkl --language_eval 1 
```

The defualt split to evaluate is test. The default inference method is greedy decoding (`--sample_max 1`), to sample from the posterior, set `--sample_max 0`.

**Beam Search**. Beam search can increase the performance of the search for greedy decoding sequence by ~5%. However, this is a little more expensive. To turn on the beam search, use `--beam_size N`, N should be greater than 1.

## Miscellanea
**Using cpu**. The code is currently defaultly using gpu; there is even no option for switching. If someone highly needs a cpu model, please open an issue; I can potentially create a cpu checkpoint and modify the eval.py to run the model on cpu. However, there's no point using cpu to train the model.

**Train on other dataset**. It should be trivial to port if you can create a file like `dataset_coco.json` for your own dataset.

**Live demo**. Not supported now. Welcome pull request.

## Acknowledgements

Thanks the original [neuraltalk2](https://github.com/karpathy/neuraltalk2) and awesome PyTorch team.
