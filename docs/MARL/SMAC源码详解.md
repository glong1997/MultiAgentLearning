> SMAC是一个星际争霸开源的通过操作单位来训练多智能体的环境

首先先了解一下星际争霸3个种族的概念：

```
Protoss：神族
Terran：人族
Zerg：虫族
```



## main.py

```python
from runner import Runner
from smac.env import StarCraft2Env
from common.arguments import get_common_args, get_coma_args, get_mixer_args, get_centralv_args, get_reinforce_args, \
    get_commnet_args, get_g2anet_args

'''
程序入口
'''
if __name__ == '__main__':
    for i in range(8):  # 总共跑八个回合
        args = get_common_args()  # 拿到训练的基本参数
        if args.alg.find('coma') > -1:
            args = get_coma_args(args)
        elif args.alg.find('central_v') > -1:
            args = get_centralv_args(args)
        elif args.alg.find('reinforce') > -1:
            args = get_reinforce_args(args)
        else:  # 否则就是价值函数分解流派的参数
            args = get_mixer_args(args)
        if args.alg.find('commnet') > -1:
            args = get_commnet_args(args)
        if args.alg.find('g2anet') > -1:
            args = get_g2anet_args(args)

        # 1、（1）创建环境对象
        '''
        1、地图
        2、多少steps（步）执行一次actions
        3、难度大小
        4、游戏版本：建议latest
        5、回放录像，自己设置路径
        '''
        env = StarCraft2Env(map_name=args.map,
                            step_mul=args.step_mul,
                            difficulty=args.difficulty,
                            game_version=args.game_version,
                            replay_dir=args.replay_dir)
        # 1、（1）拿到环境信息，get_env_info()方法以字典的形式返回值。
        '''
        n_actions：一个智能体可以执行的所有动作，self.n_actions = self.n_actions_no_attack（6个非攻击动作，北、南、东、西、停止、阵亡 + self.n_enemies（4个动作，锁定敌人）
        n_agents：智能体的数量
        state_shape：全局状态的size，如果设置为局部观测，那么返回的get_obs_size() * n_agents
        obs_shape：每个智能体自身局部观测的size，根据get_obs_size()得到
        episode_limit：
        '''
        env_info = env.get_env_info()
        args.n_actions = env_info["n_actions"]
        args.n_agents = env_info["n_agents"]
        args.state_shape = env_info["state_shape"]
        args.obs_shape = env_info["obs_shape"]
        args.episode_limit = env_info["episode_limit"]

        # 2、创建Runner类型的对象
        runner = Runner(env, args)
        if not args.evaluate:   # 如果evalute为False，那么就开始训练
            runner.run(i)
        else:   # 否则对模型进行评估，评估返回胜率
            win_rate, _ = runner.evaluate()
            print('The win rate of {} is  {}'.format(args.alg, win_rate))   # ？？？算法的胜率为？？？
            break
        env.close()     # 关闭环境

```



注：下面四个参数很重要，这些参数在开始的时候被传到专门的arguments.py里面，其他函数会调用

    args.n_actions = env_info["n_actions"]
    args.n_agents = env_info["n_agents"]
    args.state_shape = env_info["state_shape"]
    args.obs_shape = env_info["obs_shape"]
obs_shape

```python
def get_obs_size(self):
    """Returns the size of the observation."""
    nf_al = 4 + self.unit_type_bits
    nf_en = 4 + self.unit_type_bits

    if self.obs_all_health:
        nf_al += 1 + self.shield_bits_ally
        nf_en += 1 + self.shield_bits_enemy

    own_feats = self.unit_type_bits
    if self.obs_own_health:	# 关注自己的血量
        own_feats += 1 + self.shield_bits_ally
    if self.obs_timestep_number:
        own_feats += 1

    if self.obs_last_action:
        nf_al += self.n_actions

    move_feats = self.n_actions_move
    if self.obs_pathing_grid:
        move_feats += self.n_obs_pathing
    if self.obs_terrain_height:
        move_feats += self.n_obs_height

    enemy_feats = self.n_enemies * nf_en
    ally_feats = (self.n_agents - 1) * nf_al

    return move_feats + enemy_feats + ally_feats + own_feats
```

state_shape

```python
def get_state_size(self):
    """Returns the size of the global state."""
    if self.obs_instead_of_state:   # 默认为False，如果为True，则用局部观测替代全局状态
        return self.get_obs_size() * self.n_agents	# 智能体的局部观测size * 智能体数量

    nf_al = 4 + self.shield_bits_ally + self.unit_type_bits
    nf_en = 3 + self.shield_bits_enemy + self.unit_type_bits

    enemy_state = self.n_enemies * nf_en
    ally_state = self.n_agents * nf_al

    size = enemy_state + ally_state		# size = 所有敌军状态 + 所有盟军状态

    if self.state_last_action:	# qmix等算法，这里是需要为true的
        size += self.n_agents * self.n_actions
    if self.state_timestep_number:
        size += 1

    return size
```



以3m地图为例。

1. state_shape：48	

> 3盟军  ×（4 + 0 +  0） ×     +   3敌军  ×（3 + 0 +  0） +  3盟军  ×  9动作（这部分根据参数） +  0
>
> 盟军就是训练智能体
>
> 只有单位是P神族时，shield_bits_ally和shield_bits_enemy才为1，否则为0

2. obs_shape：30

> move_feats + enemy_feats + ally_feats + own_feats
>
> 移动feats，敌人的feats，盟军（不包括自己）的 feats，自己的feats
>
> 4  + （3  ×  4）+ （（3 - 1）×  4）+  6

3. n_agents：3
4. n_actions：9

>动作空间为9



## runner.py

```python
import numpy as np
import os
from common.rollout import RolloutWorker, CommRolloutWorker
from agent.agent import Agents, CommAgents
from common.replay_buffer import ReplayBuffer
import matplotlib.pyplot as plt


class Runner:
    def __init__(self, env, args):
        self.env = env

        if args.alg.find('commnet') > -1 or args.alg.find('g2anet') > -1:  # communication agent
            self.agents = CommAgents(args)
            self.rolloutWorker = CommRolloutWorker(env, self.agents, args)
        else:  # no communication agent
            self.agents = Agents(args)
            self.rolloutWorker = RolloutWorker(env, self.agents, args)
        if not args.evaluate and args.alg.find('coma') == -1 and args.alg.find('central_v') == -1 and args.alg.find('reinforce') == -1:  # these 3 algorithms are on-poliy
            self.buffer = ReplayBuffer(args)
        self.args = args
        self.win_rates = []
        self.episode_rewards = []

        # 用来保存plt和pkl
        self.save_path = self.args.result_dir + '/' + args.alg + '/' + args.map
        if not os.path.exists(self.save_path):
            os.makedirs(self.save_path)

    def run(self, num):
        time_steps, train_steps, evaluate_steps = 0, 0, -1
        while time_steps < self.args.n_steps:
            print('Run {}, time_steps {}'.format(num, time_steps))
            if time_steps // self.args.evaluate_cycle > evaluate_steps:
                win_rate, episode_reward = self.evaluate()
                # print('win_rate is ', win_rate)
                self.win_rates.append(win_rate)
                self.episode_rewards.append(episode_reward)
                self.plt(num)
                evaluate_steps += 1
            episodes = []
            # 收集self.args.n_episodes个episodes
            for episode_idx in range(self.args.n_episodes):
                episode, _, _, steps = self.rolloutWorker.generate_episode(episode_idx)
                episodes.append(episode)
                time_steps += steps
                # print(_)
            # episode的每一项都是一个(1, episode_len, n_agents, 具体维度)四维数组，下面要把所有episode的的obs拼在一起
            episode_batch = episodes[0]
            episodes.pop(0)
            for episode in episodes:
                for key in episode_batch.keys():
                    episode_batch[key] = np.concatenate((episode_batch[key], episode[key]), axis=0)
            if self.args.alg.find('coma') > -1 or self.args.alg.find('central_v') > -1 or self.args.alg.find('reinforce') > -1:
                self.agents.train(episode_batch, train_steps, self.rolloutWorker.epsilon)
                train_steps += 1
            else:
                self.buffer.store_episode(episode_batch)
                for train_step in range(self.args.train_steps):
                    mini_batch = self.buffer.sample(min(self.buffer.current_size, self.args.batch_size))
                    self.agents.train(mini_batch, train_steps)
                    train_steps += 1
        win_rate, episode_reward = self.evaluate()
        print('win_rate is ', win_rate)
        self.win_rates.append(win_rate)
        self.episode_rewards.append(episode_reward)
        self.plt(num)

    # 评估
    def evaluate(self):
        win_number = 0
        episode_rewards = 0
        for epoch in range(self.args.evaluate_epoch):
            _, episode_reward, win_tag, _ = self.rolloutWorker.generate_episode(epoch, evaluate=True)
            episode_rewards += episode_reward
            if win_tag:
                win_number += 1
        return win_number / self.args.evaluate_epoch, episode_rewards / self.args.evaluate_epoch

    def plt(self, num):
        plt.figure()
        plt.ylim([0, 105])
        plt.cla()
        plt.subplot(2, 1, 1)
        plt.plot(range(len(self.win_rates)), self.win_rates)
        plt.xlabel('step*{}'.format(self.args.evaluate_cycle))
        plt.ylabel('win_rates')

        plt.subplot(2, 1, 2)
        plt.plot(range(len(self.episode_rewards)), self.episode_rewards)
        plt.xlabel('step*{}'.format(self.args.evaluate_cycle))
        plt.ylabel('episode_rewards')

        plt.savefig(self.save_path + '/plt_{}.png'.format(num), format='png')
        np.save(self.save_path + '/win_rates_{}'.format(num), self.win_rates)
        np.save(self.save_path + '/episode_rewards_{}'.format(num), self.episode_rewards)
        plt.close()
```



## agent.py

> 智能体能做的就是根据策略选择动作，为了不陷入局部最优解，可使用如ε-贪婪策略来平衡探索与利用。

```python
class Agents:
	def __init__(self, args):
	
	def choose_action(self, obs, last_action, agent_num, avail_actions, epsilon, maven_z=None, evaluate=False):

	def _choose_action_from_softmax(self, inputs, avail_actions, epsilon, evaluate=False):
	
	def _get_max_episode_len:
	
	def train(self, batch, train_step, epsilon=None):
	
class CommAgents:
```

ε-贪婪策略

```python
q_value[avail_actions == 0.0] = - float("inf")
# 如果随机值在0~ε范围内，还是很激进的；超出ε，则很保守，猥琐发育，利用当前所谓的最优策略
if np.random.uniform() < epsilon:
    action = np.random.choice(avail_actions_ind)  # action是一个整数
else:
    action = torch.argmax(q_value)
```



## 网络

### base_net.py

> 智能体的效用网络，拟合局部Q值

```python
import torch.nn as nn
import torch.nn.functional as f


class RNN(nn.Module):
    # Because all the agents share the same network, input_shape=obs_shape+n_actions+n_agents
    def __init__(self, input_shape, args):
        super(RNN, self).__init__()
        self.args = args

        self.fc1 = nn.Linear(input_shape, args.rnn_hidden_dim)  # 一层MLP，编码
        self.rnn = nn.GRUCell(args.rnn_hidden_dim, args.rnn_hidden_dim)     # 一层GRU，为了得到更多的信息
        self.fc2 = nn.Linear(args.rnn_hidden_dim, args.n_actions)       # 一层MLP，输出n个动作对应的局部Q值

    def forward(self, obs, hidden_state):
        x = f.relu(self.fc1(obs))
        h_in = hidden_state.reshape(-1, self.args.rnn_hidden_dim)
        h = self.rnn(x, h_in)
        q = self.fc2(h)
        return q, h


# Critic of Central-V
class Critic(nn.Module):
    def __init__(self, input_shape, args):
        super(Critic, self).__init__()
        self.args = args
        self.fc1 = nn.Linear(input_shape, args.critic_dim)
        self.fc2 = nn.Linear(args.critic_dim, args.critic_dim)
        self.fc3 = nn.Linear(args.critic_dim, 1)        # 输出的是一个概率，只有一个

    def forward(self, inputs):
        x = f.relu(self.fc1(inputs))
        x = f.relu(self.fc2(x))
        q = self.fc3(x)
        return q
```

### vdn_net.py

```python
import torch.nn as nn
import torch


class VDNNet(nn.Module):
    def __init__(self):
        super(VDNNet, self).__init__()

    def forward(self, q_values):
        return torch.sum(q_values, dim=2, keepdim=True)
```







## 策略

> 最重要的三个函数：
>
> 1. 得到input
>    1. 做一些处理
> 2. 在1的基础上，处理q值
>    1. 
> 3. 在2的基础上，训练

```python
import torch
import os
from network.base_net import RNN
from network.vdn_net import VDNNet


class VDN:
    def __init__(self, args):
        self.n_actions = args.n_actions
        self.n_agents = args.n_agents
        self.state_shape = args.state_shape
        self.obs_shape = args.obs_shape
        input_shape = self.obs_shape    # input_shape作为效用网络的输入size
        # 根据参数决定RNN的输入维度
        if args.last_action:    # 如果记住上一时刻的动作，那么观测size还要 + 该智能体的动作空间（3m地图的动作空间为9）
            input_shape += self.n_actions
        if args.reuse_network:  # 如果参数共享
            input_shape += self.n_agents

        # 神经网络
        self.eval_rnn = RNN(input_shape, args)  # 每个agent选动作的网络
        self.target_rnn = RNN(input_shape, args)
        self.eval_vdn_net = VDNNet()  # 把agentsQ值加起来的网络，也是训练网络
        self.target_vdn_net = VDNNet()
        self.args = args
        if self.args.cuda:  # 使用cuda
            self.eval_rnn.cuda()
            self.target_rnn.cuda()
            self.eval_vdn_net.cuda()
            self.target_vdn_net.cuda()

        self.model_dir = args.model_dir + '/' + args.alg + '/' + args.map
        # 如果存在模型则加载模型
        if self.args.load_model:
            if os.path.exists(self.model_dir + '/rnn_net_params.pkl'):
                path_rnn = self.model_dir + '/rnn_net_params.pkl'
                path_vdn = self.model_dir + '/vdn_net_params.pkl'
                map_location = 'cuda:0' if self.args.cuda else 'cpu'
                self.eval_rnn.load_state_dict(torch.load(path_rnn, map_location=map_location))
                self.eval_vdn_net.load_state_dict(torch.load(path_vdn, map_location=map_location))
                print('Successfully load the model: {} and {}'.format(path_rnn, path_vdn))
            else:
                raise Exception("No model!")

        # 让target_net和eval_net的网络参数相同
        self.target_rnn.load_state_dict(self.eval_rnn.state_dict())
        self.target_vdn_net.load_state_dict(self.eval_vdn_net.state_dict())

        self.eval_parameters = list(self.eval_vdn_net.parameters()) + list(self.eval_rnn.parameters())
        if args.optimizer == "RMS":
            self.optimizer = torch.optim.RMSprop(self.eval_parameters, lr=args.lr)


        # 执行过程中，要为每个agent都维护一个eval_hidden
        # 学习过程中，要为每个episode的每个agent都维护一个eval_hidden、target_hidden
        self.eval_hidden = None
        self.target_hidden = None
        print('Init alg VDN')

    def learn(self, batch, max_episode_len, train_step, epsilon=None):  # train_step表示是第几次学习，用来控制更新target_net网络的参数
        '''
        在learn的时候，抽取到的数据是四维的，四个维度分别为 1——第几个episode 2——episode中第几个transition
        3——第几个agent的数据 4——具体obs维度。因为在选动作时不仅需要输入当前的inputs，还要给神经网络输入hidden_state，
        hidden_state和之前的经验相关，因此就不能随机抽取经验进行学习。所以这里一次抽取多个episode，然后一次给神经网络
        传入每个episode的同一个位置的transition
        '''
        episode_num = batch['o'].shape[0]
        self.init_hidden(episode_num)
        for key in batch.keys():  # 把batch里的数据转化成tensor
            if key == 'u':
                batch[key] = torch.tensor(batch[key], dtype=torch.long)
            else:
                batch[key] = torch.tensor(batch[key], dtype=torch.float32)
        # TODO pymarl中取得经验没有取最后一条，找出原因
        u, r, avail_u, avail_u_next, terminated = batch['u'], batch['r'],  batch['avail_u'], \
                                                  batch['avail_u_next'], batch['terminated']
        mask = 1 - batch["padded"].float()  # 用来把那些填充的经验的TD-error置0，从而不让它们影响到学习
        if self.args.cuda:
            u = u.cuda()
            r = r.cuda()
            mask = mask.cuda()
            terminated = terminated.cuda()
        # 得到每个agent对应的Q值，维度为(episode个数, max_episode_len， n_agents，n_actions)
        q_evals, q_targets = self.get_q_values(batch, max_episode_len)

        # 取每个agent动作对应的Q值，并且把最后不需要的一维去掉，因为最后一维只有一个值了
        q_evals = torch.gather(q_evals, dim=3, index=u).squeeze(3)

        # 得到target_q
        q_targets[avail_u_next == 0.0] = - 9999999
        q_targets = q_targets.max(dim=3)[0]

        q_total_eval = self.eval_vdn_net(q_evals)
        q_total_target = self.target_vdn_net(q_targets)

        targets = r + self.args.gamma * q_total_target * (1 - terminated)

        td_error = targets.detach() - q_total_eval
        masked_td_error = mask * td_error  # 抹掉填充的经验的td_error

        # loss = masked_td_error.pow(2).mean()
        # 不能直接用mean，因为还有许多经验是没用的，所以要求和再比真实的经验数，才是真正的均值
        loss = (masked_td_error ** 2).sum() / mask.sum()
        # print('Loss is ', loss)
        self.optimizer.zero_grad()
        loss.backward()
        torch.nn.utils.clip_grad_norm_(self.eval_parameters, self.args.grad_norm_clip)
        self.optimizer.step()

        if train_step > 0 and train_step % self.args.target_update_cycle == 0:
            self.target_rnn.load_state_dict(self.eval_rnn.state_dict())
            self.target_vdn_net.load_state_dict(self.eval_vdn_net.state_dict())

    def _get_inputs(self, batch, transition_idx):
        # 取出所有episode上该transition_idx的经验，u_onehot要取出所有，因为要用到上一条
        obs, obs_next, u_onehot = batch['o'][:, transition_idx], \
                                  batch['o_next'][:, transition_idx], batch['u_onehot'][:]
        episode_num = obs.shape[0]
        inputs, inputs_next = [], []
        inputs.append(obs)
        inputs_next.append(obs_next)

        # 给obs添加上一个动作、agent编号
        if self.args.last_action:
            if transition_idx == 0:  # 如果是第一条经验，就让前一个动作为0向量
                inputs.append(torch.zeros_like(u_onehot[:, transition_idx]))
            else:
                inputs.append(u_onehot[:, transition_idx - 1])
            inputs_next.append(u_onehot[:, transition_idx])
        if self.args.reuse_network:
            # 因为当前的obs三维的数据，每一维分别代表(episode，agent，obs维度)，直接在dim_1上添加对应的向量
            # 即可，比如给agent_0后面加(1, 0, 0, 0, 0)，表示5个agent中的0号。而agent_0的数据正好在第0行，那么需要加的
            # agent编号恰好就是一个单位矩阵，即对角线为1，其余为0
            inputs.append(torch.eye(self.args.n_agents).unsqueeze(0).expand(episode_num, -1, -1))
            inputs_next.append(torch.eye(self.args.n_agents).unsqueeze(0).expand(episode_num, -1, -1))
        # 要把obs中的三个拼起来，并且要把episode_num个episode、self.args.n_agents个agent的数据拼成episode_num*n_agents条数据
        # 因为这里所有agent共享一个神经网络，每条数据中带上了自己的编号，所以还是自己的数据
        inputs = torch.cat([x.reshape(episode_num * self.args.n_agents, -1) for x in inputs], dim=1)
        inputs_next = torch.cat([x.reshape(episode_num * self.args.n_agents, -1) for x in inputs_next], dim=1)
        return inputs, inputs_next

    def get_q_values(self, batch, max_episode_len):
        episode_num = batch['o'].shape[0]
        q_evals, q_targets = [], []
        for transition_idx in range(max_episode_len):
            inputs, inputs_next = self._get_inputs(batch, transition_idx)  # 给obs加last_action、agent_id
            if self.args.cuda:
                inputs = inputs.cuda()
                inputs_next = inputs_next.cuda()
                self.eval_hidden = self.eval_hidden.cuda()
                self.target_hidden = self.target_hidden.cuda()
            q_eval, self.eval_hidden = self.eval_rnn(inputs, self.eval_hidden)  # 得到的q_eval维度为(episode_num*n_agents, n_actions)
            q_target, self.target_hidden = self.target_rnn(inputs_next, self.target_hidden)

            # 把q_eval维度重新变回(episode_num, n_agents, n_actions)
            q_eval = q_eval.view(episode_num, self.n_agents, -1)
            q_target = q_target.view(episode_num, self.n_agents, -1)
            q_evals.append(q_eval)
            q_targets.append(q_target)
        # 得的q_eval和q_target是一个列表，列表里装着max_episode_len个数组，数组的的维度是(episode个数, n_agents，n_actions)
        # 把该列表转化成(episode个数, max_episode_len， n_agents，n_actions)的数组
        q_evals = torch.stack(q_evals, dim=1)
        q_targets = torch.stack(q_targets, dim=1)
        return q_evals, q_targets

    def init_hidden(self, episode_num):
        # 为每个episode中的每个agent都初始化一个eval_hidden、target_hidden
        self.eval_hidden = torch.zeros((episode_num, self.n_agents, self.args.rnn_hidden_dim))
        self.target_hidden = torch.zeros((episode_num, self.n_agents, self.args.rnn_hidden_dim))

    def save_model(self, train_step):
        num = str(train_step // self.args.save_cycle)
        if not os.path.exists(self.model_dir):
            os.makedirs(self.model_dir)
        torch.save(self.eval_vdn_net.state_dict(), self.model_dir + '/' + num + '_vdn_net_params.pkl')
        torch.save(self.eval_rnn.state_dict(),  self.model_dir + '/' + num + '_rnn_net_params.pkl')

```