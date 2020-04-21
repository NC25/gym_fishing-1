
## Introduction

This example shows how to train a [DQN (Deep Q
Networks)](https://storage.googleapis.com/deepmind-media/dqn/DQNNaturePaper.pdf)
agent on the Cartpole environment using the TF-Agents library.

![Cartpole
environment](https://raw.githubusercontent.com/tensorflow/agents/master/docs/tutorials/images/cartpole.png)

It will walk you through all the components in a Reinforcement Learning
(RL) pipeline for training, evaluation and data collection.

## Setup

``` r
library(reticulate)
reticulate::use_virtualenv("~/.virtualenvs/tf2.0")
#reticulate::py_discover_config()
```

``` python


from __future__ import absolute_import, division, print_function

import gym_fishing

import gym
import base64
import numpy as np
import tensorflow as tf

from tf_agents.agents.dqn import dqn_agent
from tf_agents.drivers import dynamic_step_driver
from tf_agents.environments import suite_gym
from tf_agents.environments import tf_py_environment
from tf_agents.eval import metric_utils
from tf_agents.metrics import tf_metrics
from tf_agents.networks import q_network
from tf_agents.policies import random_tf_policy
from tf_agents.replay_buffers import tf_uniform_replay_buffer
from tf_agents.trajectories import trajectory
from tf_agents.utils import common

#### Acitvate compatibility settings 
##tf.compat.v1.enable_v2_behavior()
### change default float
## tf.keras.backend.set_floatx('float64')

## Disable GPU, optional
#import os
#os.environ["CUDA_VISIBLE_DEVICES"] = "-1"

tf.version.VERSION
```

    ## '2.1.0'

``` python
## Hyperparameters
num_iterations = 20000             # @param {type:"integer"}
initial_collect_steps = 1000       # @param {type:"integer"} 
collect_steps_per_iteration = 1    # @param {type:"integer"}
replay_buffer_max_length = 100000  # @param {type:"integer"}
batch_size = 64                    # @param {type:"integer"}
learning_rate = 1e-3               # @param {type:"number"}
log_interval = 200                 # @param {type:"integer"}
num_eval_episodes = 10             # @param {type:"integer"}
eval_interval = 1000               # @param {type:"integer"}

## Environment
env_name = 'fishing-v0'
env = suite_gym.load(env_name)
env.reset()
```

    ## TimeStep(step_type=array(0, dtype=int32), reward=array(0., dtype=float32), discount=array(1., dtype=float32), observation=array([0.75]))

The `environment.step` method takes an `action` in the environment and
returns a `TimeStep` tuple containing the next observation of the
environment and the reward for the action.

The `time_step_spec()` method returns the specification for the
`TimeStep` tuple. Its `observation` attribute shows the shape of
observations, the data types, and the ranges of allowed values. The
`reward` attribute shows the same details for the
    reward.

``` python
print('Observation Spec:')
```

    ## Observation Spec:

``` python
print(env.time_step_spec().observation)
```

    ## BoundedArraySpec(shape=(1,), dtype=dtype('float64'), name='observation', minimum=0.0, maximum=2.0)

``` python
print('Reward Spec:')
```

    ## Reward Spec:

``` python
print(env.time_step_spec().reward)
```

    ## ArraySpec(shape=(), dtype=dtype('float32'), name='reward')

The `action_spec()` method returns the shape, data types, and allowed
values of valid
    actions.

``` python
print('Action Spec:')
```

    ## Action Spec:

``` python
print(env.action_spec())
```

    ## BoundedArraySpec(shape=(), dtype=dtype('int64'), name='action', minimum=0, maximum=2)

In the Cartpole environment:

  - `observation` is an array of 1 floating point number: current fish
    density
  - `reward` is a scalar float value
  - `action` is a scalar integer with only three possible values:
  - `0` - “maintain previous harvest”
  - `1` - “increase harvest by 20%”
  - `2` - “decrease harvest by 20%”

<!-- end list -->

``` python
time_step = env.reset()
print('Time step:')
```

    ## Time step:

``` python
print(time_step)
```

    ## TimeStep(step_type=array(0, dtype=int32), reward=array(0., dtype=float32), discount=array(1., dtype=float32), observation=array([0.75]))

``` python
action = np.array(1, dtype=np.int32)

next_time_step = env.step(action)
print('Next time step:')
```

    ## Next time step:

``` python
print(next_time_step)
```

    ## TimeStep(step_type=array(1, dtype=int32), reward=array(0.015, dtype=float32), discount=array(1., dtype=float32), observation=array([0.70273185]))

Usually two environments are instantiated: one for training and one for
evaluation.

``` python
train_py_env = suite_gym.load(env_name)
eval_py_env = suite_gym.load(env_name)
```

The Cartpole environment, like most environments, is written in pure
Python. This is converted to TensorFlow using the `TFPyEnvironment`
wrapper.

The original environment’s API uses Numpy arrays. The `TFPyEnvironment`
converts these to `Tensors` to make it compatible with Tensorflow agents
and policies.

``` python
train_env = tf_py_environment.TFPyEnvironment(train_py_env)
eval_env = tf_py_environment.TFPyEnvironment(eval_py_env)
```

## Agent

The algorithm used to solve an RL problem is represented by an `Agent`.
TF-Agents provides standard implementations of a variety of `Agents`,
including:

  - [DQN](https://storage.googleapis.com/deepmind-media/dqn/DQNNaturePaper.pdf)
    (used in this
    tutorial)
  - [REINFORCE](http://www-anw.cs.umass.edu/~barto/courses/cs687/williams92simple.pdf)
  - [DDPG](https://arxiv.org/pdf/1509.02971.pdf)
  - [TD3](https://arxiv.org/pdf/1802.09477.pdf)
  - [PPO](https://arxiv.org/abs/1707.06347)
  - [SAC](https://arxiv.org/abs/1801.01290).

The DQN agent can be used in any environment which has a discrete action
space. At the heart of a DQN Agent is a `QNetwork`, a neural network
model that can learn to predict `QValues` (expected returns) for all
actions, given an observation from the environment.

Use `tf_agents.networks.q_network` to create a `QNetwork`, passing in
the `observation_spec`, `action_spec`, and a tuple describing the number
and size of the model’s hidden layers.

``` python
fc_layer_params = (100,)
q_net = q_network.QNetwork(
    train_env.observation_spec(),
    train_env.action_spec(),
    fc_layer_params=fc_layer_params)
```

Now use `tf_agents.agents.dqn.dqn_agent` to instantiate a `DqnAgent`. In
addition to the `time_step_spec`, `action_spec` and the QNetwork, the
agent constructor also requires an optimizer (in this case,
`AdamOptimizer`), a loss function, and an integer step
counter.

``` python
optimizer = tf.compat.v1.train.AdamOptimizer(learning_rate=learning_rate)
train_step_counter = tf.Variable(0)

agent = dqn_agent.DqnAgent(
    train_env.time_step_spec(),
    train_env.action_spec(),
    q_network=q_net,
    optimizer=optimizer,
    td_errors_loss_fn=common.element_wise_squared_loss,
    train_step_counter=train_step_counter)
```

    ## WARNING:tensorflow:Layer QNetwork is casting an input tensor from dtype float64 to the layer's dtype of float32, which is new behavior in TensorFlow 2.  The layer has dtype float32 because it's dtype defaults to floatx.
    ## 
    ## If you intended to run this layer in float32, you can safely ignore this warning. If in doubt, this warning is likely only an issue if you are porting a TensorFlow 1.X model to TensorFlow 2.
    ## 
    ## To change all layers to have dtype float64 by default, call `tf.keras.backend.set_floatx('float64')`. To change just this layer, pass dtype='float64' to the layer constructor. If you are the author of this layer, you can disable autocasting by passing autocast=False to the base Layer constructor.
    ## 
    ## WARNING:tensorflow:Layer TargetQNetwork is casting an input tensor from dtype float64 to the layer's dtype of float32, which is new behavior in TensorFlow 2.  The layer has dtype float32 because it's dtype defaults to floatx.
    ## 
    ## If you intended to run this layer in float32, you can safely ignore this warning. If in doubt, this warning is likely only an issue if you are porting a TensorFlow 1.X model to TensorFlow 2.
    ## 
    ## To change all layers to have dtype float64 by default, call `tf.keras.backend.set_floatx('float64')`. To change just this layer, pass dtype='float64' to the layer constructor. If you are the author of this layer, you can disable autocasting by passing autocast=False to the base Layer constructor.

``` python
agent.initialize()
```

## Policies

A policy defines the way an agent acts in an environment. Typically, the
goal of reinforcement learning is to train the underlying model until
the policy produces the desired outcome.

In this tutorial:

  - The desired outcome is keeping the pole balanced upright over the
    cart.
  - The policy returns an action (left or right) for each `time_step`
    observation.

Agents contain two policies:

  - `agent.policy` — The main policy that is used for evaluation and
    deployment.
  - `agent.collect_policy` — A second policy that is used for data
    collection.

<!-- end list -->

``` python
eval_policy = agent.policy
collect_policy = agent.collect_policy
```

# Policies can be created independently of agents. For example,

# use `tf_agents.policies.random_tf_policy` to create a policy

# which will randomly select an action for each `time_step`.

``` python
random_policy = random_tf_policy.RandomTFPolicy(train_env.time_step_spec(),
                                                train_env.action_spec())
```

To get an action from a policy, call the `policy.action(time_step)`
method. The `time_step` contains the observation from the environment.
This method returns a `PolicyStep`, which is a named tuple with three
components:

  - `action` — the action to be taken (in this case, `0`, `1` or `2`)
  - `state` — used for stateful (that is, RNN-based) policies
  - `info` — auxiliary data, such as log probabilities of actions

<!-- end list -->

``` python
example_environment = tf_py_environment.TFPyEnvironment(
    suite_gym.load(env_name))

time_step = example_environment.reset()
random_policy.action(time_step)
```

    ## PolicyStep(action=<tf.Tensor: shape=(1,), dtype=int64, numpy=array([0])>, state=(), info=())

## Metrics and Evaluation

The most common metric used to evaluate a policy is the average return.
The return is the sum of rewards obtained while running a policy in an
environment for an episode. Several episodes are run, creating an
average return.

The following function computes the average return of a policy, given
the policy, environment, and a number of episodes.

``` python
#@test {"skip": true}
def compute_avg_return(environment, policy, num_episodes=10):

  total_return = 0.0
  for _ in range(num_episodes):

    time_step = environment.reset()
    episode_return = 0.0

    while not time_step.is_last():
      action_step = policy.action(time_step)
      time_step = environment.step(action_step.action)
      episode_return += time_step.reward
    total_return += episode_return

  avg_return = total_return / num_episodes
  return avg_return.numpy()[0]
```

See also the metrics module for standard implementations of different
metrics:
<https://github.com/tensorflow/agents/tree/master/tf_agents/metrics>

Running this computation on the `random_policy` shows a baseline
performance in the environment.

``` python
compute_avg_return(eval_env, random_policy, num_eval_episodes)
```

    ## 1.4563386

## Replay Buffer

The replay buffer keeps track of data collected from the environment.
This tutorial uses
`tf_agents.replay_buffers.tf_uniform_replay_buffer.TFUniformReplayBuffer`,
as it is the most common.

The constructor requires the specs for the data it will be collecting.
This is available from the agent using the `collect_data_spec` method.
The batch size and maximum buffer length are also required.

``` python
replay_buffer = tf_uniform_replay_buffer.TFUniformReplayBuffer(
    data_spec = agent.collect_data_spec,
    batch_size = train_env.batch_size,
    max_length = replay_buffer_max_length)

# For most agents, `collect_data_spec` is a named tuple called 
# `Trajectory`, containing the specs for observations, actions,
# rewards, and other items.
agent.collect_data_spec
```

    ## Trajectory(step_type=TensorSpec(shape=(), dtype=tf.int32, name='step_type'), observation=BoundedTensorSpec(shape=(1,), dtype=tf.float64, name='observation', minimum=array(0.), maximum=array(2.)), action=BoundedTensorSpec(shape=(), dtype=tf.int64, name='action', minimum=array(0), maximum=array(2)), policy_info=(), next_step_type=TensorSpec(shape=(), dtype=tf.int32, name='step_type'), reward=TensorSpec(shape=(), dtype=tf.float32, name='reward'), discount=BoundedTensorSpec(shape=(), dtype=tf.float32, name='discount', minimum=array(0., dtype=float32), maximum=array(1., dtype=float32)))

``` python
agent.collect_data_spec._fields
```

    ## ('step_type', 'observation', 'action', 'policy_info', 'next_step_type', 'reward', 'discount')

## Data Collection

Now execute the random policy in the environment for a few steps,
recording the data in the replay buffer.

``` python
#@test {"skip": true}
def collect_step(environment, policy, buffer):
  time_step = environment.current_time_step()
  action_step = policy.action(time_step)
  next_time_step = environment.step(action_step.action)
  traj = trajectory.from_transition(time_step, action_step, next_time_step)

  # Add trajectory to the replay buffer
  buffer.add_batch(traj)

def collect_data(env, policy, buffer, steps):
  for _ in range(steps):
    collect_step(env, policy, buffer)

collect_data(train_env, random_policy, replay_buffer, steps=100)
```

This loop is so common in RL, that we provide standard implementations.
For more details see the drivers module.
<https://github.com/tensorflow/agents/blob/master/tf_agents/docs/python/tf_agents/drivers.md>
The replay buffer is now a collection of Trajectories.

For the curious: Uncomment to peel one of these off and inspect it.

``` python
# iter(replay_buffer.as_dataset()).next()
```

The agent needs access to the replay buffer. This is provided by
creating an iterable `tf.data.Dataset` pipeline which will feed data to
the agent.

Each row of the replay buffer only stores a single observation step. But
since the DQN Agent needs both the current and next observation to
compute the loss, the dataset pipeline will sample two adjacent rows for
each item in the batch (`num_steps=2`).

This dataset is also optimized by running parallel calls and prefetching
data.

``` python
# Dataset generates trajectories with shape [Bx2x...]
dataset = replay_buffer.as_dataset(
    num_parallel_calls=3, 
    sample_batch_size=batch_size, 
    num_steps=2).prefetch(3)


dataset
```

    ## <PrefetchDataset shapes: (Trajectory(step_type=(64, 2), observation=(64, 2, 1), action=(64, 2), policy_info=(), next_step_type=(64, 2), reward=(64, 2), discount=(64, 2)), BufferInfo(ids=(64, 2), probabilities=(64,))), types: (Trajectory(step_type=tf.int32, observation=tf.float64, action=tf.int64, policy_info=(), next_step_type=tf.int32, reward=tf.float32, discount=tf.float32), BufferInfo(ids=tf.int64, probabilities=tf.float32))>

``` python
iterator = iter(dataset)

print(iterator)
```

    ## <tensorflow.python.data.ops.iterator_ops.OwnedIterator object at 0x7f895473e940>

For the curious:

Uncomment to see what the dataset iterator is feeding to the agent.
Compare this representation of replay data to the collection of
individual trajectories shown earlier.

``` python
# iterator.next()
```

## Training the agent

Two things must happen during the training loop:

  - collect data from the environment
  - use that data to train the agent’s neural network(s)

This example also periodicially evaluates the policy and prints the
current score. The following will take ~5 minutes to
run.

``` python
# (Optional) Optimize by wrapping some of the code in a graph using TF function.
agent.train = common.function(agent.train)

# Reset the train step
agent.train_step_counter.assign(0)

# Evaluate the agent's policy once before training.
```

    ## <tf.Variable 'UnreadVariable' shape=() dtype=int32, numpy=0>

``` python
avg_return = compute_avg_return(eval_env, agent.policy, num_eval_episodes)
returns = [avg_return]

for _ in range(num_iterations):

  # Collect a few steps using collect_policy and save to the replay buffer.
  for _ in range(collect_steps_per_iteration):
    collect_step(train_env, agent.collect_policy, replay_buffer)

  # Sample a batch of data from the buffer and update the agent's network.
  experience, unused_info = next(iterator)
  train_loss = agent.train(experience).loss

  step = agent.train_step_counter.numpy()

  if step % log_interval == 0:
    print('step = {0}: loss = {1}'.format(step, train_loss))

  if step % eval_interval == 0:
    avg_return = compute_avg_return(eval_env, agent.policy, num_eval_episodes)
    print('step = {0}: Average Return = {1}'.format(step, avg_return))
    returns.append(avg_return)
```

    ## step = 200: loss = 0.11605465412139893
    ## step = 400: loss = 0.15190616250038147
    ## step = 600: loss = 0.14457088708877563
    ## step = 800: loss = 0.5178611874580383
    ## step = 1000: loss = 1.0558955669403076
    ## step = 1000: Average Return = 18.66695785522461
    ## step = 1200: loss = 1.9564869403839111
    ## step = 1400: loss = 3.9584717750549316
    ## step = 1600: loss = 4.898134231567383
    ## step = 1800: loss = 7.679220199584961
    ## step = 2000: loss = 8.510913848876953
    ## step = 2000: Average Return = 5.146400451660156
    ## step = 2200: loss = 11.627363204956055
    ## step = 2400: loss = 141.7628936767578
    ## step = 2600: loss = 15.438224792480469
    ## step = 2800: loss = 15.43415355682373
    ## step = 3000: loss = 9.88300895690918
    ## step = 3000: Average Return = 0.05002465099096298
    ## step = 3200: loss = 20.038455963134766
    ## step = 3400: loss = 17.854103088378906
    ## step = 3600: loss = 16.82758140563965
    ## step = 3800: loss = 28.694719314575195
    ## step = 4000: loss = 26.907867431640625
    ## step = 4000: Average Return = 0.049999989569187164
    ## step = 4200: loss = 19.416719436645508
    ## step = 4400: loss = 20.233592987060547
    ## step = 4600: loss = 24.67291259765625
    ## step = 4800: loss = 22.06404685974121
    ## step = 5000: loss = 26.064830780029297
    ## step = 5000: Average Return = 12.298906326293945
    ## step = 5200: loss = 26.758907318115234
    ## step = 5400: loss = 33.10454559326172
    ## step = 5600: loss = 27.019775390625
    ## step = 5800: loss = 34.37115478515625
    ## step = 6000: loss = 34.27666091918945
    ## step = 6000: Average Return = 0.049999989569187164
    ## step = 6200: loss = 19.487308502197266
    ## step = 6400: loss = 47.31824493408203
    ## step = 6600: loss = 29.84111213684082
    ## step = 6800: loss = 40.066810607910156
    ## step = 7000: loss = 28.960214614868164
    ## step = 7000: Average Return = 0.050206154584884644
    ## step = 7200: loss = 30.31696891784668
    ## step = 7400: loss = 42.36946487426758
    ## step = 7600: loss = 44.4133415222168
    ## step = 7800: loss = 37.08155059814453
    ## step = 8000: loss = 39.9971923828125
    ## step = 8000: Average Return = 0.05286868289113045
    ## step = 8200: loss = 27.222286224365234
    ## step = 8400: loss = 49.07503890991211
    ## step = 8600: loss = 31.345123291015625
    ## step = 8800: loss = 37.31706237792969
    ## step = 9000: loss = 42.145145416259766
    ## step = 9000: Average Return = 1.67754328250885
    ## step = 9200: loss = 46.64209747314453
    ## step = 9400: loss = 119.00503540039062
    ## step = 9600: loss = 41.97471618652344
    ## step = 9800: loss = 32.619136810302734
    ## step = 10000: loss = 43.567291259765625
    ## step = 10000: Average Return = 0.04999999701976776
    ## step = 10200: loss = 39.88463592529297
    ## step = 10400: loss = 39.38738250732422
    ## step = 10600: loss = 51.961036682128906
    ## step = 10800: loss = 40.06568145751953
    ## step = 11000: loss = 50.80973815917969
    ## step = 11000: Average Return = 1.432436227798462
    ## step = 11200: loss = 42.54496765136719
    ## step = 11400: loss = 32.430816650390625
    ## step = 11600: loss = 802.4616088867188
    ## step = 11800: loss = 39.44080352783203
    ## step = 12000: loss = 51.047996520996094
    ## step = 12000: Average Return = 0.05000000447034836
    ## step = 12200: loss = 51.86787033081055
    ## step = 12400: loss = 55.650821685791016
    ## step = 12600: loss = 58.60679244995117
    ## step = 12800: loss = 42.811614990234375
    ## step = 13000: loss = 63.44834518432617
    ## step = 13000: Average Return = 4.845945358276367
    ## step = 13200: loss = 48.980743408203125
    ## step = 13400: loss = 51.2372932434082
    ## step = 13600: loss = 58.69709014892578
    ## step = 13800: loss = 53.944602966308594
    ## step = 14000: loss = 1486.1343994140625
    ## step = 14000: Average Return = 1.6138265132904053
    ## step = 14200: loss = 47.452293395996094
    ## step = 14400: loss = 42.223297119140625
    ## step = 14600: loss = 59.82592010498047
    ## step = 14800: loss = 62.15824890136719
    ## step = 15000: loss = 87.45833587646484
    ## step = 15000: Average Return = 0.049999989569187164
    ## step = 15200: loss = 35.690914154052734
    ## step = 15400: loss = 51.707435607910156
    ## step = 15600: loss = 73.21919250488281
    ## step = 15800: loss = 78.62881469726562
    ## step = 16000: loss = 54.728187561035156
    ## step = 16000: Average Return = 1.8588874340057373
    ## step = 16200: loss = 67.74588012695312
    ## step = 16400: loss = 74.81892395019531
    ## step = 16600: loss = 66.0600357055664
    ## step = 16800: loss = 56.63567352294922
    ## step = 17000: loss = 84.92991638183594
    ## step = 17000: Average Return = 0.049999989569187164
    ## step = 17200: loss = 84.96891784667969
    ## step = 17400: loss = 57.70912170410156
    ## step = 17600: loss = 61.29108428955078
    ## step = 17800: loss = 74.41133880615234
    ## step = 18000: loss = 1352.251953125
    ## step = 18000: Average Return = 0.05711386352777481
    ## step = 18200: loss = 68.48721313476562
    ## step = 18400: loss = 63.81706619262695
    ## step = 18600: loss = 54.7495231628418
    ## step = 18800: loss = 67.11639404296875
    ## step = 19000: loss = 517.0784301757812
    ## step = 19000: Average Return = 0.049999989569187164
    ## step = 19200: loss = 58.78782272338867
    ## step = 19400: loss = 76.49166870117188
    ## step = 19600: loss = 57.772865295410156
    ## step = 19800: loss = 70.38288116455078
    ## step = 20000: loss = 61.835819244384766
    ## step = 20000: Average Return = 0.05294724553823471

## Visualization

``` python
# ### Plots
# 
# Use `matplotlib.pyplot` to chart how the policy improved during training.
# 
# One iteration of `Cartpole-v0` consists of 200 time steps. The environment gives a reward of `+1` for each step the pole stays up, so the maximum return for one episode is 200. The charts shows the return increasing towards that maximum each time it is evaluated during training. (It may be a little unstable and not increase monotonically each time.)


import matplotlib
import matplotlib.pyplot as plt
#@test {"skip": true}

iterations = range(0, num_iterations + 1, eval_interval)
plt.plot(iterations, returns)
```

    ## [<matplotlib.lines.Line2D object at 0x7f894472e710>]

``` python
plt.ylabel('Average Return')
```

    ## Text(0, 0.5, 'Average Return')

``` python
plt.xlabel('Iterations')
```

    ## Text(0.5, 0, 'Iterations')

``` python
plt.ylim(top=250)
```

    ## (-0.880847903713584, 250.0)

``` python
plt.show()
#fig.savefig('foo.png', bbox_inches='tight')
```

<img src="dqn_fishing_files/figure-gfm/unnamed-chunk-23-1.png" width="672" />