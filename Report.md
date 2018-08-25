
# 1. Learning Algorithm

### 1.1 Deep Q Network
As the state space is continuous, an action-value function is approximated by a deep neural networks with 3 hidden layers having 128, 64, and 32 units, respectively, and an output layers with dimension equal to the dimension of action space. Output signals from each hidden unit are passed through a rectified linear unit (ReLU) to model the nonlinearity of the underlying action-value function. The network can be represented in a Pytorch sequential form as
```
nn.Sequential(nn.Linear(state_space_size, 128),
              nn.ReLU(),
              nn.Linear(128, 64),
              nn.ReLU(),
              nn.Linear(64, 32),
              nn.ReLU(),
              nn.Linear(32,4))
```

### 1.2 Double Deep Q Learning
The DQN is trained using a [Q-learning](!http://incompleteideas.net/book/bookdraft2017nov5.pdf) control algorithm which is an **off-policy** method where actions performed at each time step are determined from a _behavior policy_ distinct from a _target policy_ being optimized. The objective is to find an **optimal policy** $\pi_{*}$ that maximizing **expected accumulative future rewards**. The optimal policy $\pi_{*}$ can easily derive from an **optimal action-value function** $q_{*}$ as 

$$\pi_{*}(a|s) = \underset{a\in A}{\operatorname{argmax}} q_{*}(s,a).    \qquad(1)$$

The original [Q-learning](!https://link.springer.com/content/pdf/10.1007/BF00992698.pdf) algorithm for a _finite Markov decision preocess_ estimates an optimal policy by iteratively improving the estimated $q$ based on recieved rewards as
$$q(s,a)=q(s,a)+\alpha[r+\gamma\underset{a\in A}{\operatorname{max}}_{a}q(s',a) -q(s,a)].\qquad(2) $$
At convergence, the optimal policy $\pi_{*}$ can easily obtain using eq. (1) 


Instead of estimated $q$ as above, for a continuous state space, it is more efficient to approximate $q_{*}$ using a deep neural network (see Section 1.1).  As described in [Mnih et.al's work](!https://www.nature.com/articles/nature14236), the objective can be framed as minimizing the discrepancy between the action-value of each state and the target return of the state computed based on a reward and the action-value of the next state as
$$ \hat{\theta} = \underset{\theta}{\operatorname{argmin}}\mathbb{E}_{(s,a,r,s')\sim P(D)}\psi(r+\gamma\underset{a\in A(s')}{\operatorname{max}} q(s',a;\theta^{-}) -q(s,a;\theta)), \qquad(3)$$
where $\theta$ are the parameters of a behavior DQN, $\theta^{-}$ are stationary parameters of a target DQN, and $\psi$ is a distance function such as an L-p norm and a Huber distance. The term inside the distance function is called _temporal difference (TD) error_, i.e., minimization of TD error. At convergence, TD error, in principle, approaches 0. 

To reduce overoptimism (overestimation of action values) of Q-learning, [double DQN](!https://arxiv.org/abs/1509.06461) modifies the loss function eq. (3) as
$$ \hat{\theta} = \underset{\theta}{\operatorname{argmin}}\mathbb{E}_{(s,a,r,s')\sim P(D)}\psi(r+\gamma q(s',\underset{a\in A(s')}{\operatorname{argmax}}q(s,a;\theta);\theta^{-}) -q(s,a;\theta)), \qquad(4)$$
According to eq. (4), $\theta$ are updated using
$$ \theta^{t+1} = \theta^{t} -\alpha\nabla_{\theta}L(\theta^{t}) $$

### 1.3 Prioritized Experience Replay
The [DQN](!https://www.nature.com/articles/nature14236) described above was learned from uniform random samples of experiences (described by e.g., states, actions, rewards, next states) regardless of scarcity and/or importance of each experience. Schaul et al. proposed to impose priority of experiences in random selection of experiences. The priority $p_i$ of each experience $i$ is determined based on TD error as
$$ p_i = |r_i+\gamma q(s'_{i},a_i)- q(s_i,a_i)|. \qquad(5)$$
Since each experience should be sampled at least once, some small $\varepsilon$ is added to eq. (5). For [proportional prioritization](!https://www.nature.com/articles/nature14236), the priority for each experience in DDQN is thus 
<br>
$$ p_i = |r_i+\gamma q(s'_{i},\underset{a_i \in A(s'_{i})}{\operatorname{argmax}}q(s'_{i},a_i;\theta);\theta^{-})- q(s_i,a_i; \theta)|+\varepsilon. $$
The probability of sampling an experience $i$ is therefore
$$ P(i)=\frac{p^{\alpha}_i}{\sum_{k}p^{\alpha}_k} $$
where $\alpha$ controls the degree of uniform randomness of sampling with $\alpha=0$ for complete uniform random sampling. <br>
The prioritized sampling modifies the distribution of experiences learned by the behavior DQN. Important sampling (IS) weights are used to correct the experience distribution with the strength of correction gradually increased toward 1. at the convergence of action values, defined by
$$ w_i = \bigg(\frac{1}{N}\cdot \frac{1}{P(i)}\bigg)^{\beta}$$
where $N$ is the size of the replay memory and $\beta$ controls the strength of IS correction. 


### 1.4 Hyper-parameters
The hyper-parameters for DDQN with prioritized experience replay includes
    
* **DDQN**
    * `batch_size`: number of experiences (minibatch size) used to train in each epoch, default `64`
    * $\gamma$: discount rate of rewards, default `.99`
    * `update_every`: number of step before which parameters of the target DQN get updated, default `4`
    * $\tau$: strength of the influence of the parameters of the behavior DQN to the parameters of the target DQN, default `1e-3`
    * `learning_rate`: factor of the gradient of eq. (4) used to modify the values of the behavior-DQN parameters, default `5e-4`
    
* **Replay Memory**
    * `Memory size`: size of replay memory, default `1e5`
    * $\alpha$: degree of uniform randomness of sampling, default `0.7`
    * $\beta$: IS strength for correction of the underlying experience distribution, default `0.5` increased to `1.`
    * $\varepsilon$: minimum value of priority, default `1e-3`
    

# 2. Agent Performance

### 2.1 Rewards
The DDQN agent was able to solve the banana collecting problem with increasing expected accumulated rewards as shown in the plot below.

![score](prioritized_ddqn/images/score_plot.png)


### 2.2 First 50 episodes
![firstepisodes](prioritized_ddqn/images/ezgif.com-video-to-gif.gif)

### 2.3 Solved 
The values of the parameters of the DDQN that successfully solve this problem can be found in `prioritized_ddqn/checkpoint.pth`
![solved](prioritized_ddqn/images/fin_ezgif.com-video-to-gif.gif)


# 3. Future Work

* Implement Dueling Network Architecture for advantage saliency
* Apply DDQN with prioritized expereince replay and dueling network for problems whose environment states are defined using 2D images
* Explore [multi-step learning and noisy nets](!https://arxiv.org/abs/1710.02298)