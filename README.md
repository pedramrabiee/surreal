# Surreal initiative

- **SURREAL** stands for "**S**tanford **U**niversity **R**epository for **Re**inforcement-learning **Al**gorithms". Note that this is just a temporary acronym. Surreal will become something much more impactful in the years to come. Stay tuned. 

- _Surreal_ is an **open-source, cloud-integrated platform that scales up state-of-the-art RL algorithms and enables a new wave of distributed RL research**. _Surreal_ is both a collection of codebases and the name of a vibrant research team at Prof. Fei-Fei Li's lab. 

- Potential research opportunities on top of _Surreal_:

    * RL in robotics, particularly hard manipulation tasks.
    * Improve on fundamental RL problems: exploration, stability, sample efficiency, etc.
    * Multi-agent RL topics: cooperative, adversarial, communication, etc.
    * Self-play: bimanual manipulation, OpenAI sumo-bot, AlphaZero extensions.
    * Human in the loop learning. Augmented intelligence.
    * Evolution strategies and neuro-evolution.
    * RL applied to a wide range of domains, like dialogue systems, protein folding, and healthcare.
   
- We focus on **depth** instead of breadth. Unlike [RLlab](https://github.com/rll/rllab) and [Nervana coach](https://github.com/NervanaSystems/coach), we do not attempt to implement a huge variety of barebone RL algorithms. The first iteration of _Surreal_ will focus solely on continuous control: DDPG for off-policy and PPO for on-policy case. We will implement distributed DDPG and PPO with all of the bells and whistles known to us. They will be state-of-the-art in both learning performance and wallclock speed. 

- Starting from _Surreal_, we will establish a number of battle-tested libraries and utilities to be shared among Stanford AI lab members. 

    - Motivated by DeepMind's way of fast idea iteration: all it takes to produce a new paper is ~100 lines of new code + a strong collection of supporting codebases. For instance, please compare the [Dueling DQN](http://arxiv.org/abs/1511.06581) with the [vanilla DQN](https://www.dropbox.com/s/8fvbr86a0v5ejco/Human-level%20control%20through%20deep%20reinforcement%20learning%20-%20Mnih%20et%20al.pdf?dl=0). I think it's fewer than 20 lines of code delta.
    - Reduce potential bug dimensions. If you modify the algorithm and then training diverges, you can safely assume that your modification is the _only_ source of problem.
    - Reproducible benchmarking.
    - Maintain readable code for all future publications.


# Community guidelines

## Checklist before you start

- We have weekly Surreal all-hands meeting. It typically happens at 7 pm and provides free dinner! Wednesday is the most likely day, although it shifts from week to week to accommodate special situations like conference deadlines and midterm exams. Please check with the admins (@jimfan and @yukez) about your schedule.
 
- You should have write access to both `Surreal` and `Tensorplex` github repos. 

- Join our Slack channel: https://stanfordvl.slack.com. You will be added as a multi-channel guest with at least one access, the `#surreal-dev` channel. Slack is our _primary means of communication_. Make sure you regularly check the messages.

- Ask Yuke for access to our Google Cloud cluster.

- Ask Jim for access to our Google Drive shared folder. It contains some more onboarding docs and reading materials. 

## Coding requirements

1. Please **periodically check your Github notifications** and address any comments in the issues that `@` you. 

2. Please try your best to adhere to [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html). It's very hard to beautify big batches of nasty code, so please do it incrementally! Some of the most important rules:

    * Do not exceed 80 chars per line unless it's a URL in the comment or a long error message.
    * Two blank lines between class def and *top-level* function definitions.
    * One blank line between *member methods inside* a class definition. 
    * I strongly recommend you skim through the style guide. It's well-written. 

3. Docs! Docs! Docs! Make sure you write docstrings as you write the code. We *will not have time* to go through every source file and write docs ad-hoc. Please conform to [Google docstring style](http://sphinxcontrib-napoleon.readthedocs.io/en/latest/example_google.html). 

4. **Report any breaking changes on Slack**. Please also make records on the wiki pages. The date on the wiki page should be Monday of the week.

5. Report any bugs or feature requests on Slack **AND open Github issues**. Add appropriate labels to the issue, such as "urgent", "wishlist", etc. Make sure you assign either yourself or at least one other person.

6. You should always create a pull request instead of committing directly to master, unless:
    1. Your commit is a typo fix, documentation update, or other trivial fixes.
    2. You created a new file/script that does not interfere with the rest of the codebase.
    3. Feel free to commit to `surreal/scratch/` folder. It is meant for demos that you want to share with the team.

## Wiki page

Please check the [wiki page](https://github.com/StanfordVL/Surreal/wiki) regularly. 

The wiki will record all performance progress, API changes and important breaking updates.


# Infrastructure design

## Fully-integrated cloud solution for RL

**Cloud** is first-class citizen. Infrastructure should be **codified**. 

- Most other competitor platforms (e.g. [Ray](https://rise.cs.berkeley.edu/projects/ray/) and [TF-agent](https://github.com/tensorflow/agents)) only provide a distributed runtime, but do not offer an easy way to *orchestrate the distributed runtime on the cloud*. For example, it is highly non-trivial to configure a Google Cloud account to scale resources, specify node pools, set up networking, and manage experiments from multiple team members sharing the account. It typically takes huge amount of manual effort, and once the team migrates to a different account or service provider, they have to start all over again. For admins who have lived through this, they know first-hand that it is tedious, annoying, and error-prone.
    
- _Surreal_'s philosophy is different: machine learning infrastructure should be codified, version-controlled, and easily replicated with minimal manual effort or ungodly shell scripts. _Surreal_ shields the users from the messy cloud and bare-metal configurations that are otherwise unavoidable for scaling up. _Surreal_ is designed for the dirty world.

- Thanks to the cross-platform toolchains,  _Surreal_ supports all major cloud providers: Google Cloud, Amazon Web Services, and Microsoft Azure. With some additional effort, it may also support any custom-built cloud infrastructure.

- We provide docker images for both CPU and GPU training and deployment. Libraries like [dm_control](https://github.com/deepmind/dm_control) and `glfw` might need difficult workarounds to install correctly. Containerization avoids all the dependency hell and helps maintain reproducibility of the training environment.


## Layers of Infrastructure

### Kubernetes: logical grouping

Each distributed RL experiment is a logical set of processes. 

For example, an [Ape-X](https://arxiv.org/abs/1803.00933) experiment with 8 actors would require 8 processes plus learner, parameter server, replay server, tensorboard, and logging server. They should be scheduled on the cloud as a logical set that can communicate internally. Different sets of processes denote different experiments that should not interfere with each other. 

To achieve this, _Surreal_ has tight integrations with [Kubernetes](https://kubernetes.io/), an open-source system for automating deployment, scaling, and management of containerized applications. Kubernetes can auto-scale the number of VMs as experiments are launched or terminated, which is economically essential to low-budget teams (like ours).

### Terraform: automate cloud setup

**TODO**: Kubernetes still requires some amount of manual administration, such as creating cluster pools of different CPU and GPU types, spinning up the network file server, and adding [node taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/). The procedure needs to be repeated when we migrate to a different Google Cloud account or switch to a new cloud provider altogether. 

Fortunately, [Terraform](https://www.terraform.io/intro/index.html) comes to our rescue. Configuration files describe to Terraform the components needed to run a single application or an entire datacenter. Terraform generates an execution plan describing what it will do to reach the desired state, and then executes it to build the described infrastructure. 

Armed with Terraform, _Surreal_ can automate the infrastructure setup from scratch. This removes the last drop of manual labor and frees up researchers' time for actually useful work.


## Distributed protocol

### ZeroMQ: thin wrapper around TCP socket

**TODO**

### PyArrow: fast memory transfer

**TODO**


# Installation

## Overview

We currently support two ways to run Surreal experiments. 

1. Locally on your laptop with `tmux`. No containerization needed, but you will have to install all the dependencies manually. The `tmux` mode can be launched by the interactive notebook `surreal/main/cluster_dashboard_new.ipynb`. 

2. Remotely on Google Cloud with `kurreal` (`= "Kubernetes" + "Surreal"`) command line tool. `kurreal` is installed to system path when you install the `surreal` package (instruction in the next section). You can run `kurreal --help` anywhere to read the manual.

## Surreal and supporting libs

### Anaconda 3

[Download and install](https://www.anaconda.com/download).

We use python3 for all development. Note that numpy has officially announced that it would [drop support for python 2 in 2019](https://python3statement.org/), so it's time to move on! 

**Warning**: Conda 4.4.something may have some problem working with tmux. 4.3.29 runs fine. Fix: Run `conda install conda=4.3.29`. \[Need more investigation\]

### Surreal package

`-e` means "editable installation". If you change any code in the `Surreal/` repo, it will be reflected system-wide for all `import surreal`. 

```
git clone https://github.com/StanfordVL/Surreal.git
pip install -e Surreal/
```

### Tensorplex package

Read [Tensorplex API on Github](https://github.com/StanfordVL/Tensorplex).

```
git clone https://github.com/StanfordVL/Tensorplex.git
pip install -e Tensorplex/
```

### MujocoManipulation
Follow the instructions in [MujocoManipulation](https://github.com/StanfordVL/MujocoManipulation)


## Cloud toolchain 

### Docker

[Download and install](https://docs.docker.com/install/).

We have one docker image for CPU-only and another image for GPU-only. All agents and eval processes use the CPU-only image, while the learner process uses the GPU-only image if the training is done on GPU. It is a workaround for [`dm_control` issue](https://github.com/deepmind/dm_control/issues/21).

Unless you are adding new dependencies, you **should not rebuild or push the image yourself!** Please contact the admins (@jimfan, @jirenz) if you have special needs.

The CPU/GPU `Dockerfile`s are in `installation/` folder.

- Docker hub: `SurrealAI/surreal` (will migrate to `surrealai/surreal-cpu` soon)

TODO: more docs here

### Minikube

**Warning**: minikube mode is broken now. Will be fixed after NIPS.

[Download and install](https://kubernetes.io/docs/tasks/tools/install-minikube/).

1. Install Virtualbox driver.

```bash
sudo apt-get update
sudo apt-get install virtualbox
```

2. Install [Minikube](https://github.com/kubernetes/minikube/releases) executable.

```bash
curl -Lo minikube https://storage.googleapis.com/minikube/releases/v0.25.0/minikube-linux-amd64 && chmod +x minikube && sudo mv minikube /usr/local/bin/
```

- `minikube-setenv.sh` to communicate with local docker.
- `minikube-mount.sh` to mount hostDir


## One-time setup

**TODO**

Copy the system config file from `surreal/kube/sample.surreal.yml` to your home dir `~/.surreal.yml` and fill in the blanks (e.g. `<username>`). 

git generate access token

decide which branch is the temp branch to snapshot your code


## Miscellaneous

- Connect to tensorboard by `autossh`. This is unnecessary if you launch experiments with `kurreal`.

```bash
autossh -N -L localhost:9006:10.138.0.33:6006 gke-mycluster-nonagent-pool-0b0a9484-l3kg.us-west1-b.<gcloud-url>
```

- [Setup `sshfs`](docs/sshfs-setup.md) to mount remote NFS on your laptop.


