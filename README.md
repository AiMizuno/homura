# Homura [![CircleCI](https://circleci.com/gh/moskomule/homura/tree/master.svg?style=svg)](https://circleci.com/gh/moskomule/homura/tree/master)

[document](https://moskomule.github.io/homura)

*Homura* is a support tool for research experiments.


## Requirements

### minimal requirements

```
Python>=3.7
PyTorch>=1.0
torchvision>=0.2.1
tqdm # automatically installed
```

### optional

```
tensorboardX
miniargs
colorlog
optuna
```

To enable distributed training using Synced BN and FP 16, install apex.

```
git clone https://github.com/NVIDIA/apex.git
cd apex
python setup.py install --cuda_ext --cpp_ext
```

### test

```
pytest .
```

## install

```console
pip install git+https://github.com/AiMizuno/homura
```

or

```console
git clone https://github.com/AiMizuno/homura
cd homura; pip install -e .
```


# APIs

## basics

* Device Agnostic
* Useful features

```python
from homura import optim, lr_scheduler
from homura import trainers, callbacks, reporters
from torchvision.models import resnet50
from torch.nn import functional as F

# model will be registered in the trainer
resnet = resnet50()
# optimizer and scheduler will be registered in the trainer, too
optimizer = optim.SGD(lr=0.1, momentum=0.9)
scheduler = lr_scheduler.MultiStepLR(milestones=[30,80], gamma=0.1)

# list of callbacks or reporters can be registered in the trainer
with reporters.TensorboardReporter([callbacks.AccuracyCallback(), 
                                    callbacks.LossCallback()]) as reporter:
    trainer = trainers.SupervisedTrainer(resnet, optimizer, loss_f=F.cross_entropy, 
                                         callbacks=reporter, scheduler=scheduler)
```

Now `iteration` of trainer can be updated as follows,

```python
from homura.utils.containers import Map

def iteration(trainer: Trainer, data: Tuple[torch.Tensor]) -> Mapping[torch.Tensor]:
    input, target = data
    output = trainer.model(input)
    loss = trainer.loss_f(output, target)
    results = Map(loss=loss, output=output)
    if trainer.is_train:
        trainer.optimizer.zero_grad()
        loss.backward()
        trainer.optimizer.step()
    # registered values can be called in callbacks
    results.user_value = user_value
    return results

SupervisedTrainer.iteration = iteration
# or   
trainer.update_iteration(iteration) 
```

Also, `dict` of models, optimizers, loss functions are supported.

```python
trainer = CustomTrainer({"generator": generator, "discriminator": discriminator},
                        {"generator": gen_opt, "discriminator": dis_opt},
                        {"reconstruction": recon_loss, "generator": gen_loss},
                        **kwargs)
```

## reproductivity


```python
from homura.reproductivity import set_deterministic
set_deterministic(1)
```

## debugger

```python
>>> debug.module_debugger(nn.Sequential(nn.Linear(10, 5), 
                                        nn.Linear(5, 1)), 
                          torch.randn(4, 10))
[homura.debug|2019-02-25 17:57:06|DEBUG] Start forward calculation
[homura.debug|2019-02-25 17:57:06|DEBUG] forward> name=Sequential(1)
[homura.debug|2019-02-25 17:57:06|DEBUG] forward>   name=Linear(2)
[homura.debug|2019-02-25 17:57:06|DEBUG] forward>   name=Linear(3)
[homura.debug|2019-02-25 17:57:06|DEBUG] Start backward calculation
[homura.debug|2019-02-25 17:57:06|DEBUG] backward>   name=Linear(3)
[homura.debug|2019-02-25 17:57:06|DEBUG] backward> name=Sequential(1)
[homura.debug|2019-02-25 17:57:06|DEBUG] backward>   name=Linear(2)
[homura.debug|2019-02-25 17:57:06|INFO] Finish debugging mode
```

# Examples

See [examples](examples).

* [cifar10.py](examples/cifar10.py): training ResNet-20 or WideResNet-28-10 with random crop on CIFAR10
* [imagenet.py](examples/imagenet.py): training a CNN on ImageNet on multi GPUs (single and     multi process)
* [gap.py](examples/gap.py): better implementation of generative adversarial perturbation
