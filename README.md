## 一、比赛介绍
近年来，随着深度学习的重燃以及海量大数据的支撑，NLP 领域迎来了蓬勃发展，百度拥有全球最大的中文知识图谱，拥有数亿实体、千亿事实，具备丰富的知识标注与关联能力，不仅构建了通用知识图谱，还构建了汉语语言知识图谱、关注点图谱、以及包含业务逻辑在内的行业知识图谱等多维度图谱。我们希望通过开放百度的数据，邀请学界和业界的青年才俊共同推进算法进步，激发更多灵感和火花。

面向中文短文本的实体链指，简称 EL（Entity Linking），是NLP领域的基础任务之一，即对于给定的一个中文短文本（如搜索 Query、微博、对话内容、文章/视频/图片的标题等），EL将其中的实体提及(*mention、指称*)与给定知识库中对应的实体进行关联。如下图所示：
	![](https://ai-studio-static-online.cdn.bcebos.com/488a764881b44ff3acf03a424c1179f1d1612f7c90114156ba7dc6b8055c2f61)
    

传统的实体链指任务主要是针对长文档，长文档拥有在写的上下文信息能辅助实体的歧义消解并完成链指。相比之下，针对中文短文本的实体链指存在很大的挑战，主要原因如下：
1. 口语化严重，导致实体歧义消解困难；
2. 短文本上下文语境不丰富，须对上下文语境进行精准理解；
3. 相比英文，中文由于语言自身的特点，在短文本的链指问题上更有挑战。

本次评测任务旨在借助实体链指技术，拓展其对应的AI智能应用需求，并将技术成果实践于更多的现实场景。

+ [进入比赛网页查看数据集、赛题任务等详细介绍](https://aistudio.baidu.com/aistudio/competition/detail/58)  
+ [数据集链接](https://aistudio.baidu.com/aistudio/datasetdetail/60899)  

## 二、思路介绍
### 2.1 实体链指基本思路
目前关于实体链接已经有较长时间的研究积累，这些方法基本由如下三
个步骤构成：  
1. **候选实体召回:** 根据实体提及字符串与知识库中所有实体的字符串按
照一定规则匹配，生成候选实体集。良好的候选实体集要在保证覆盖目
标实体的同时，尽可能的小。  
2. **候选实体排序:** 通过算法比较待链接实体与候选实体集中每个实体中的
关联度，这个过程通常需要利用实体提及的上下文以及候选实体的描述
基于多任务层投影 ERNIE 的短文本实体链接及实体分类方法 3
信息。候选实体排序方法包括有监督和无监督两大类，其中有监督方法
有二分类 [5]、排序学习 [6]、基于图 [7] 的方法等。本文使用基于二分类
的排序方法。  
3. **NIL 实体判断:** 根据候选实体排序结果，如果候选实体集为空或者排名
较高的几个候选实体均不能匹配，那么就将实体预测为一个 NIL（Not
In the List）实体。  

对于 CCKS2020 中文短文本实体链接任务1，除了需要进行以上实体链
接任务外，还需要对 NIL 实体进行细粒度分类，也就是本文3.4节描述的子
任务二。

### 2.2 本文方案及亮点
针对本次中文短文本实体链接任务，将 **实体链接** 和 **NIL实体概念分类** 作
为两个子任务。**基于百度 ERNIE 和 BERT-PALs，使用飞桨框架实现多
任务模型，即层投影 ERNIE。该模型节省多任务场景下模型参数量的同
时，提升模型在多任务下的泛化性能**。任务一利用待链接实体上下文以及
目标实体描述信息，计算待链接实体与目标实体的关联度，提升实体链接
的准确率。任务二利用实体所在文本以及实体位置信息，进行 NIL 实体分
类。

## 三、方案细节

### 3.1 思路框架图
![](https://ai-studio-static-online.cdn.bcebos.com/4b35fe3d5cd8499a9404658e96648e17d4e8414b798642759d6aafc250d86002)

### 3.2 多任务层投影ERNIE介绍
#### 3.2.1 模型结构
+ 本文通过修改ERNIE模型内部结构，实现了能够进行多任务扩展的ERNIE模型。百度开源的中文预训练 ERNIE 模型的基本结构为
12 层堆叠的 Transformer 作为编码器，每一层的输入和输出均为 [序列长度 × 768]的向量。如图所示：
![](https://ai-studio-static-online.cdn.bcebos.com/a1c839237f854d3fb36f1f99a80fe8a12de502d6a19e4be581da3273338fb496)

+ 修改方式为：设 n 个子任务构成集合为 $\{task_1, task_2, \cdots , task_n\}$，如图2，在 ERNIE
模型每一层增加 n 个降维层 $\{FC_{D1}, FC_{D2}, \cdots , FC_{Dn}\}$ 以及 n 个升维层
$\{FC_{U1}, FC_{U2}, \cdots, FC_{Un}\}$。理论上降维层和升维层实现方式并不受限制，本
文实验使用的为普通的全连接层。模型进行任务 $task_i(i \in n)$ 的训练或推理
时，只有 $FC_{Di}$ 和 $FC_{Ui}$ 参与运算。其中全连接层 $FC_{Di}$ 输入神经元个数为
768，这与 ERNIE 的隐藏层大小相等，输出神经元的个数为 $h(h < 768)$，本
文实验取 $h = 200$。
![](https://ai-studio-static-online.cdn.bcebos.com/6b56aef80e5f4557af2c13bec305d0dcc24a4c2570f94c21ad4857e0e6da53c3)  
+ 修改后的encoder相关代码如下：
  ```py
  class ErnieEncoderStackForMutiTask(D.Layer):
      def __init__(self, cfg, name=None, task_num=0, d_proj=0):
          super(ErnieEncoderStackForMutiTask, self).__init__()

          assert task_num > 1 and d_proj > 0
          self.task_num = task_num

          initializer = F.initializer.TruncatedNormal(scale=cfg['initializer_range'])
          d_model = cfg['hidden_size']
          act = cfg['hidden_act']

          n_layers = cfg['num_hidden_layers']
          self.block = D.LayerList(
              [ErnieBlockForMutiTask(cfg, append_name(name, 'layer_%d' % i)) for i in range(n_layers)])

          ## task相关的投影层（FC_U、FC_D）参数，对ernie的每一层、每个task都要初始化一个独立的全连接层。
          self.proj_down_layers = D.LayerList([
              D.LayerList([
                  _build_linear(d_model, d_proj, append_name(name, f'task_{j}_proj_down_{i}'), initializer)
                  for i in range(n_layers)
              ])
              for j in range(task_num)
          ])
          self.proj_up_layers = D.LayerList([
              D.LayerList([
                  _build_linear(d_proj, d_model, append_name(name, f'task_{j}_proj_up_{i}'), initializer) for i in
                  range(n_layers)
              ]) for j in range(task_num)
          ])

      def forward(self, inputs, attn_bias=None, past_cache=None, task_id=0):
          assert 0 <= task_id <= self.task_num

          if past_cache is not None:
              assert isinstance(past_cache, tuple), 'unknown type of `past_cache`, expect tuple or list. got %s' % repr(
                  type(past_cache))
              past_cache = list(zip(*past_cache))
          else:
              past_cache = [None] * len(self.block)
          cache_list_k, cache_list_v, hidden_list = [], [], [inputs]

          # 计算时，根据当前的task_id选择对应的投影层参数，最终将task投影输出与transforers的输出相加
          for b, p, d, u in zip(self.block, past_cache, self.proj_down_layers[task_id], self.proj_up_layers[task_id]):
              projected_down = d(inputs)
              projected_up = u(projected_down)

              inputs, cache = b(inputs, projected_up, attn_bias=attn_bias, past_cache=p)

              cache_k, cache_v = cache
              cache_list_k.append(cache_k)
              cache_list_v.append(cache_v)
              hidden_list.append(inputs)

          return inputs, hidden_list, (cache_list_k, cache_list_v)
  ```
  
 

#### 3.2.2 训练及推理过程
每个任务输入需要同时提供当前任务的编号，图2表示当前输入为 $task_1$
时的运行示意图，只有当前任务编号对应的投影层，即 $FC_{D1}$ 和 $FC_{U1}$ 会进
行前向计算。投影层的输出与 Transformer 的输出维度相同，通过简单的相
加后作为当前编码器层的输出。在反向传播过程中，根据飞桨框架的自动求
导机制，每一次迭代只有参与计算的投影层的参数会被更新，因此一个任务
的推理或训练不会影响其他任务的投影层，这使得模型的任务数目可以方便
的扩展。  
多任务训练时，每个 batch 使用随机采样，同一个 batch 中的样本来自
一个任务。对于训练样本数目分别为 $|X_1|, |X_2|,\cdots , |X_n|$ 的任务集合，每个
batch 选择第 $i$ 个任务的概率 $p_i$ 是个超参数。本文默认使用的概率计算方式：  
$p_{i}=\frac{\sqrt{\left|X_{i}\right|}}{\sum_{i=1}^{n} \sqrt{\left|X_{i}\right|}}$

使用层投影ERNIE模型时，前向传播forward()函数中要根据task_id选择对应的计算流，训练时也要根据task_id选择不同的损失函数。前向传播实例代码如下：
  ```python
  class MutiTaskWithErnieModel(D.Layer):
      def __init__(self, args, task_dict, version_cfg):
          super(MutiTaskWithErnieModel, self).__init__()
          self.NIL_DICT = json.load(open(join(args.data_dir, 'type2id.json')))  # length = 23
          self.task_dict = task_dict
          self.args = args
          self.version_cfg = version_cfg
          ernie_cfg = json.load(open(join(args.load_from, 'ernie-config.json')))
          self.ernie_mt = ErnieModelForMutiTask(ernie_cfg, task_num=len(task_dict), name='ccks', d_proj=256)

          self.task0_ffn = D.Linear(768, 1)  # sim
          self.task1_ffn = D.Linear(768, len(self.NIL_DICT))

      def forward(self, inputs: dict, r_adv=None):
          if r_adv is not None:
              src_embed = self.ernie_mt.word_emb(inputs['src_ids'])
              inputs['src_embed'] = src_embed + 1 * r_adv
          if inputs['task_id'] == self.task_dict['sim']:
              # add entity type info
              # .......	
              pooled, _ = self.ernie_mt(**inputs)
              return L.tanh(L.squeeze(self.task0_ffn(pooled), [1]))

          elif inputs['task_id'] == self.task_dict['nil']:
              pooled, _ = self.ernie_mt(**inputs)
              return L.softmax(self.task1_ffn(pooled))
          else:
              raise Exception(f"error task_id:{inputs['task_id']} in inputs batch!")
  ```

### 3.3  候选实体排序输入输出
+ **样本标签及损失函数**： 对数据集中全部实体提及构成的集合 $M$ 中的每一个元素 $m$，在知识库
中搜索得到它的候选实体集 $E_m$ 后，将实体提及与候选实体集中的每一个实
体两两组合，每一个组合 $<m, e_m >$ 为一个输入样本。训练阶段，如果组合
中的候选实体与训练数据中的标签一致，则将当前组合作为正例，正例标签
值为 1，该实体提及构成的其他组合作为负例，标签值为-1，模型对这些组
合进行二分类。使用 $tanh(\cdot)$ 激活函数以及均方误差损失函数。  
+ **模型输入**：如图，使用特殊符号#对mention添加位置标记后的短文本 **+** 候选实体特征拼接后构成的文本。
![](https://ai-studio-static-online.cdn.bcebos.com/a13b103db0d5434a8649f6cc5a8651a3bfd9008cff93446db67a37724f53352b)

### 3.4 NIL实体分类
对训练集中每一个实体提及，如果为 NIL 实体，则以对应的标注 NIL
类别作为标签，否则将其目标实体的类别作为该实体提及类别标签。和候选
实体排序任务中的预处方法类似，对每一个实体提及在其前后分别插入 #
作为位置标记，作为一个分类样本。如果一句话中出现多个实体提及，则分
别处理为多个样本，使模型能够学习到句子中不同位置上的实体的差异。模
型最后一层接 $softmax(\cdot)$ 激活函数，对每一个样本进行分类，分类结果作
为当前被标记的实体提及的细粒度类别。


## 四、实验分析
### 4.1 多任务层投影ERNIE与原版ERNIE的对比
为了验证本文模型的有效性，使用ERNIE作为单任务模型进行对比。用于对比的模型结构图如下：
![](https://ai-studio-static-online.cdn.bcebos.com/bd5e07f13bb447e8a782efff484f3b710931080e9001465983a994155e943e3d)  

实验结果如下表：
| 模型 | A榜测试F1值 |
| -------- | -------- |
| 多任务层投影ERNIE | 0.8742 |
| 单任务原版ERNIE  | 0.86605 |

### 4.2 候选实体特征选取
| 特征类别 | 验证集f1值 |
| -------- | -------- |
|摘要 + 义项描述 | 0.87096|
|摘要 + 义项描述 + 属性值 + 别名 | 0.87420|
|摘要 + 义项描述 + 属性值 + 别名 + 属性名 + 实体名 | 0.8721|
|加权投票融合 |0.88187|

## 五、总结与展望
本文将 CCKS2020 中文短文本实体链指任务划分为候选实体排序和
NIL 实体细粒度分类两个子任务，并构建了多任务层投影 ERNIE，使用同
一个编码器分别训练和预测这两个子任务。本文
实现的模型可以将一个基于 ERNIE 的编码器扩展到更多的子任务上，并且
几乎不受任务数量的限制。

在子任务处理细节上，本文方案仍有提升空间。还可以使用基于知识表示的方法、利用短文本中共现实体的关联概率、实体标签文本等信息进一步提升候选实体排序和实体分类的准确率。

## 六、飞桨框架使用体验及新手建议
笔者参加这个比赛时也是首次接触飞桨框架，使用的版本为1.8.0。框架的API设计对新手比较友好，动态图下与Pytorch比较相似，因此从pytorch切换过来并没有废太大力气。结合百度的AI studio使用十分顺畅。这里告诉大家一些使用过程中的技巧：
1. 可以在本地调试，通过git仓库将代码同步到aistudio。
2. 框架上手时多翻阅paddlepaddle的官方文档，利用文档网站的搜索功能可以快速找到自己需要的API。
3. pytorch用户建议使用动态图开发。
4. 遇到问题可以积极到paddle的github仓库提issue，一般都能很快得到回应。

## 七、主要参考资料
+ **本文对应的论文pdf**：https://bj.bcebos.com/v1/conference/ccks2020/eval_paper/ccks2020_eval_paper_2_8.pdf
+ **Transformers**：Vaswani A, Shazeer N, Parmar N, et al. Attention is all you need[C]. Advances in
neural information processing systems. 2017: 5998-6008.
+ **ERNIE**：Sun Y, Wang S, Li Y, et al. Ernie: Enhanced representation through knowledge
integration[J]. arXiv preprint arXiv:1904.09223, 2019.
+ **层投影ERNIE的想法来源**：
	- Stickland A C, Murray I. Bert and pals: Projected attention layers for efficient
adaptation in multi-task learning[J]. arXiv preprint arXiv:1902.02671, 2019.
	- Jawahar G, Sagot B, Seddah D. What Does BERT Learn about the Structure
of Language?[C]. Proceedings of the 57th Annual Meeting of the Association for
Computational Linguistics. 2019: 3651-3657.

请点击[此处](https://ai.baidu.com/docs#/AIStudio_Project_Notebook/a38e5576)查看本环境基本用法.  <br>
Please click [here ](https://ai.baidu.com/docs#/AIStudio_Project_Notebook/a38e5576) for more detailed instructions. 
