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









## 网络

### base_net.py

```python
import torch.nn as nn
import torch.nn.functional as f


class RNN(nn.Module):
    # Because all the agents share the same network, input_shape=obs_shape+n_actions+n_agents
    def __init__(self, input_shape, args):
        super(RNN, self).__init__()
        self.args = args

        self.fc1 = nn.Linear(input_shape, args.rnn_hidden_dim)
        self.rnn = nn.GRUCell(args.rnn_hidden_dim, args.rnn_hidden_dim)
        self.fc2 = nn.Linear(args.rnn_hidden_dim, args.n_actions)

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
        self.fc3 = nn.Linear(args.critic_dim, 1)

    def forward(self, inputs):
        x = f.relu(self.fc1(inputs))
        x = f.relu(self.fc2(x))
        q = self.fc3(x)
        return q
```