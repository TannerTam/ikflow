# IKFlow
Normalizing flows for Inverse Kinematics. Open source implementation to the paper ["IKFlow: Generating Diverse Inverse Kinematics Solutions"](https://ieeexplore.ieee.org/abstract/document/9793576)

[![arxiv.org](https://img.shields.io/badge/cs.RO-%09arXiv%3A2111.08933-red)](https://arxiv.org/abs/2111.08933)


Runtime curve for getting *exact* IK solutions for the Franka Panda (maximum positional/rotational error: 1mm, .572 deg) (generated with `
python scripts/benchmark_generate_exact_solutions.py --model_name=panda__full__lp191_5.25m`):

![alt text](picture/exact_ik_runtime__model_panda__full__lp191_5.25m.png)


## Setup

The following section outlines the setup procedures required to run the visualizer that this project uses. The only supported OS is Ubuntu. Visualization may work on Mac and Windows, I haven't tried it though. For Ubuntu, there are different system wide dependencies for `Ubuntu > 21` and `Ubuntu < 21`. For example, `qt5-default` is not in the apt repository for Ubuntu 21.0+ so can't be installed. See https://askubuntu.com/questions/1335184/qt5-default-not-in-ubuntu-21-04.

<ins>Ubuntu >= 21.04</ins>
```
sudo apt-get install -y qtbase5-dev qtchooser qt5-qmake qtbase5-dev-tools libosmesa6 build-essential qtcreator
export PYOPENGL_PLATFORM=osmesa # this needs to be run every time you run a visualization script in a new terminal - annoying, I know
```
<ins>Ubuntu <= 20.x.y</ins>

(This includes 20.04 LTS, 18.04 LTS, ...)
```
sudo apt-get install -y qt5-default build-essential qtcreator
```

Lastly, install with uv:
``` bash
git clone https://github.com/jstmn/ikflow.git && cd ikflow
uv sync
uv pip install -e .
```


## Getting started

**> Example 1: Use IKFlow to generate approximate IK solutions for the Franka Panda**

Evaluate a pretrained IKFlow model for the Franka Panda arm. Note that the value for `model_name` - in this case `panda__full__lp191_5.25m` should match an entry in `model_descriptions.yaml` 
```
uv run python scripts/evaluate.py --testset_size=500 --model_name=panda__full__lp191_5.25m
```

**> Example 2: Use IKFlow to generate exact IK solutions for the Franka Panda**

Additional examples are provided in examples/example.py. This file includes examples of collision checking and pose error calculation, among other utilities.

```
ik_solver, _ = get_ik_solver("panda__full__lp191_5.25m")
target_poses = torch.tensor(
    [
        [0.25, 0, 0.5, 1, 0, 0, 0],
        [0.35, 0, 0.5, 1, 0, 0, 0],
        [0.45, 0, 0.5, 1, 0, 0, 0],
    ],
    device=device,
)
solutions, _ = ik_solver.generate_exact_ik_solutions(target_poses)
```


**> Example 3: Visualize the solutions returned by the `fetch_arm__large__mh186_9.25m` model**

Run the following:
```
uv run python scripts/visualize.py --model_name=fetch_arm__large__mh186_9.25m --demo_name=oscillate_target
```
![ikflow solutions for oscillating target pose](picture/ikflow__fetcharm__oscillating-target.gif?raw=true)

Run an interactive notebook: `jupyter notebook notebooks/robot_visualizations.ipynb`


## Notes
This project uses the `w,x,y,z` format for quaternions. That is all.


## Training new models

The training code uses [Pytorch Lightning](https://www.pytorchlightning.ai/) to setup and perform the training and [Weights and Biases](https://wandb.ai/) ('wandb') to track training runs and experiments. WandB isn't required for training but it's what this project is designed around. Changing the code to use Tensorboard should be straightforward (so feel free to put in a pull request for this if you want it :)).

First, create a dataset for the robot:
```
uv run python scripts/build_dataset.py --robot_name=panda --training_set_size=25000000 --only_non_self_colliding
```

Then start a training run:
```
# Login to wandb account - Only needs to be run once
uv run wandb login

# Set wandb project name and entity
export WANDB_PROJECT=ikflow 
export WANDB_ENTITY=<your wandb entity name>

uv run python scripts/train.py \
    --robot_name=panda \
    --nb_nodes=12 \
    --batch_size=128 \
    --learning_rate=0.0005
```

## Common errors

1. GLUT font retrieval function when running a visualizer. Run `export PYOPENGL_PLATFORM=osmesa` and then try again. See https://bytemeta.vip/repo/MPI-IS/mesh/issues/66

```
Traceback (most recent call last):
  File "visualize.py", line 4, in <module>
    from ikflow.visualizations import _3dDemo
  File "/home/jstm/Projects/ikflow/utils/visualizations.py", line 10, in <module>
    from klampt import vis
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/klampt/vis/__init__.py", line 3, in <module>
    from .glprogram import *
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/klampt/vis/glprogram.py", line 11, in <module>
    from .glviewport import GLViewport
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/klampt/vis/glviewport.py", line 8, in <module>
    from . import gldraw
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/klampt/vis/gldraw.py", line 10, in <module>
    from OpenGL import GLUT
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/OpenGL/GLUT/__init__.py", line 5, in <module>
    from OpenGL.GLUT.fonts import *
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/OpenGL/GLUT/fonts.py", line 20, in <module>
    p = platform.getGLUTFontPointer( name )
  File "/home/jstm/Projects/ikflow/venv/lib/python3.8/site-packages/OpenGL/platform/baseplatform.py", line 350, in getGLUTFontPointer
    raise NotImplementedError( 
NotImplementedError: Platform does not define a GLUT font retrieval function
```

2. If you get this error: `tkinter.TclError: no display name and no $DISPLAY environment variable`, add the lines below to the top of `ik_solvers.py` (anywhere before `import matplotlib.pyplot as plt` should work).
``` python
import matplotlib
matplotlib.use("Agg")
```


## Citation
```
@ARTICLE{9793576,
  author={Ames, Barrett and Morgan, Jeremy and Konidaris, George},
  journal={IEEE Robotics and Automation Letters}, 
  title={IKFlow: Generating Diverse Inverse Kinematics Solutions}, 
  year={2022},
  volume={7},
  number={3},
  pages={7177-7184},
  doi={10.1109/LRA.2022.3181374}
}
```