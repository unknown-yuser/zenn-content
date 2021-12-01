---
title: "Stable-baselines3 x Soft Actor-Critic x KukaDiverseObject"
emoji: ""
type: "tech"
topics: ["Stable-baselines3", "Soft Actor-Critic", "Kuka", "PyBullet", "強化学習", "深層強化学習"]
published: false
---

# この記事は何？

シミュレーション上のKukaロボットを Soft Actor-Critic という強化学習アルゴリズムで操作してみよう、という記事です。
Stable-baselines3 は容易に強化学習アルゴリズムを使えるようにした素晴らしいライブラリですが、 Soft Actor-Critic を使った紹介が多くないように感じたので
本記事は Stable-baselines3 で Soft Actor-Critic を使った一例というつもりでいます。

なお、筆者は専門家でもないので至らないところがあります。もし間違い等あればご指摘をおねがいします。

# 前提知識

## Stable-Baselines3

- Github: [リンク](https://github.com/DLR-RM/stable-baselines3)
- Document: [リンク](https://stable-baselines3.readthedocs.io/en/master/)
- Blog: [リンク](https://araffin.github.io/post/sb3/)

Stable-Baselines3 はメジャーな深層強化学習まとまった実装のセットで「Stable-Baselines」の次にメジャーバージョンとなります。 ( Stable-Baselines は OpenAI Baselines をフォークして大規模なリファクタリングを実行し、さらにいくつかの強化学習アルゴリズムを追加したものになります。 )

arxiv などで最新の強化学習アルゴリズムの発表を目にすることができますが、実装の詳細が記載されていないこともあって再現させることが難しい場合があります。Stable-Baselines3 が信頼性の高い高品質な実装を提供することで再現性の難しさを乗り越える手助けをしてくれます。

Stable-Baselines3 は次の特徴を備えており、非常に使いやすいライブラリとなっています。

- すべてのアルゴリズムに統一した構造
- 統一された PEP8コードスタイル
- 充実したドキュメント
- しっかりとテストされている。
  - コードカバレッジ 95%
- 型ヒント
- Clean Code
  - 整ったシンプルなインタフェースを提供している
- Tensorboard をサポート
- それぞれのアルゴリズムのパフォーマンスがテストされている
  - _Stable-Baselines3 (SB3) is a library providing reliable implementations of reinforcement learning algorithms in PyTorch._
  - 例えば Blog の __High-Quality Implementations__ を見ると Stable-baselines3を使った実行結果とオリジナルの実行結果の比較が載っています。

## Soft Actor-Critic

- arxiv: [リンク1](https://arxiv.org/abs/1801.01290), [リンク2](https://arxiv.org/abs/1812.05905)
- Github: [リンク](https://github.com/haarnoja/sac)

Soft Actor-Critic は有名な強化学習アルゴリズムの一つだとおもいます。このアルゴリズムの詳細な説明は他にわかりやすい適切な記事が沢山存在するのでここでは簡単な特徴のみ触れようと思います。

### 方策ON型(On-policy) vs 方策OFF型(Off-policy)

強化学習アルゴリズムを分類する切り口はいくつかありますが、その内の一つに方策ON型と方策OFF型というものがあります。方策とは行動を決定するルールのことをいいますが、方策ON型はこのルールのもとでの行動価値関数を推定し、推定した行動価値をもとに方策を改善する手法です。一方で方策OFF型は行動を決定する方策と推定する方策を分けて考えることで、行動を決定する方策をもとにサンプルを収集し、有効なサンプルだけをつかって最適な方策を推定することが出来ます。

方策ON型は行動価値関数を更新するために直前の方策のサンプルのみしか使えないというサンプル効率の悪さがありますが、学習が安定しやすい傾向があります。一方で方策OFF型は目標とする方策が行動の方策に束縛されないので過去の行動のサンプルを使いまわすことができてサンプル効率が良いのですが、学習が安定しにくい可能性があります。

Soft Actor-Critic は方策OFF型の強化学習アルゴリズムになります。

### 行動の種類

行動には離散値をとるものと連続値をとるものがあります。離散値をとるものには例えばゲームのコマンド入力があります。一方で連続値をとるものには例えばロボットのモータなどのアクチュエータの制御があります。

強化学習アルゴリズムの中には連続値をとる行動しか扱えないものや逆に離散値をとる行動しか使えないものもあります。Stable-baselines3 に含まれるアルゴリズムにおいては[こちら](https://github.com/DLR-RM/stable-baselines3#implemented-algorithms)にまとまっています。

Box, Discrete, MultiDiscrete, MultiBinary の説明に関しては[こちら](https://note.com/npaka/n/n784a13c44fa7)を参照すると良いと思います。Stable-baselines3 においてSoft Actor-Critic は行動が連続値をとる場合でしか使えないことになります。

ただし [SAC-Discrete](https://arxiv.org/abs/1910.07207) という論文はあり、実際は離散行動に Soft Actor-Critic を適用することも可能という考えもあります。

## PyBullet (Kuka)

### PyBullet

[PyBullet](https://pybullet.org/wordpress/) は物理エンジンの一つで C++ で書かれた Bullet Physics を Python で使えるようにしたものです。ロボティクスや強化学習、VR で使うことが出来ます。[PyBullet Quick Guide](https://docs.google.com/document/d/10sXEhzFRSnvFcl3XxNGhnD4N2SedqwdAvK3dsihxVUA/edit#heading=h.2ye70wns7io3)に PyBullet の関数やクラスの一覧を見ることができたり、強化学習のための PyBullet環境の使い方の説明が記載されています。PyBullet環境を使ってみようという動機としてはロボットシミュレーション環境を試してみたいというのがあったりします。

強化学習で使えるロボットシミュレーション環境には他に Mujoco がありますが、以前は商用ライセンスが必要でした。そして研究用途以外で個人で出来るだけ無償で使える環境を使いたいとなれば Mujoco の代わりに PyBullet を使うという選択肢がありました。最近はライセンスの購入が必要なく [Apache License 2.0](https://github.com/deepmind/mujoco/blob/1f7eaae62e0cd45a71cf7a593fa5605720765167/LICENSE) で利用できるようになっています。(mujoco-pyも[マージ済み](https://github.com/openai/mujoco-py/pull/640)です。)

### Kuka

PyBullet は 「KukaBulletEnv-v0」 や 「KukaCamBulletEnv-v0」、「KukaDiverseObjectGrasping-v0」 があります。 「KukaBulletEnv-v0」 や　「KukaCamBulletEnv-v0」 に関しては [こちら](https://docs.google.com/document/d/10sXEhzFRSnvFcl3XxNGhnD4N2SedqwdAvK3dsihxVUA/edit#heading=h.y55px8b66p6t) に説明があるのでご覧ください。ここでは KukaDiverseObjectGrasping-v0 のみ深堀りします。

KukaDiverseObjectGrasping-v0 はトレイに入っているオブジェクトを上手く掴むというタスクになります。(以下の画像をクリックするとYoutubeに飛びます)。

[![KukaDiverseObjectGrasping-v0](http://img.youtube.com/vi/y7scmgsGQdA/0.jpg)](http://www.youtube.com/watch?v=y7scmgsGQdA)

KukaDiverseObjectGrasping-v0 は実際には「[KukaDiverseObjectEnv](https://github.com/bulletphysics/bullet3/blob/master/examples/pybullet/gym/pybullet_envs/bullet/kuka_diverse_object_gym_env.py#L17)」というクラスを使うわけですが、このクラスのコンストラクタではいくつかの引数が定義されています。

例えば

- `renders`: GUIの表示。
- `isDiscrete`: 行動を離散値とするか連続値とするか。
- `maxSteps`: 最大ステップ数をいくつにするか。
- `removeHeightHack`: エンドエフェクタの高さ方向の制御を自動にするか、入力にするか。
- `isTest`: トレイに入れるオブジェクトをtrain setから選ぶか、test setから選ぶか。

があります。この記事で紹介する実装では連続的な行動を扱うので `isDiscrete=True`としました。また、少し複雑なタスクを試してみようと思い `removeHeightHack=True` としています。 またロボットアームが左右に振り続けてエビソードの終端につかない可能性もあるので `maxSteps=20` と制限をかけることにしまいした。

KukaDiverseObjectGrasping-v0 における状態はデフォルトではそれぞれの画素値が0~255でサイズが48x48, 3チャンネルの画像になります。また行動はそれぞれ最小 -1, 最大 -1 の4次元ベクトルでエンドエフェクタのxyzの位置と回転角度の変化量となります。

ロボットアームはオブジェクトをつかめる高さまでになるとエンドエフェクタは自動的に閉じる動作を行うようになります。また、報酬は物体をつかめた時だけ 1 と返し、そのほかは 0 を返します。

# 実装と解説

## 課題と学習結果

ロボットアームがトレイに入っているオブジェクトを掴む課題を解きます。
作成した実装は雑ですが[Github](https://github.com/unknown-yuser/SAC_KukaDiverseObject)に残しています。

### ビデオ

10 回中 6 回成功しています。実用できないレベルですが、それなりに学習はできているようです。

<video width="320" height="240" controls>
  <source src="/res/kuka_grasp_sac.mp4" type="video/mp4">
</video>


### グラフ

Stable-Baselines3 では学習が進んでいる様子を TensorBoard を使って記録することが容易です。
[TensorBoard.dev](https://tensorboard.dev/experiment/yzZqd0JxR2Guv8ry1nSGiA/#scalars) にアップロードしているのでご覧ください。

およそ 30000 ステップ超えるあたりまで行動の価値の学習が進み、30000 ステップ進んだ後あたりから徐々に適切な行動を学び始めている感じがします。

## 環境の実装

環境の実装は次のようにしています。

```python
class AssignTypeWrapper(gym.Wrapper):
    def __init__(self, env: gym.Env):
        super(AssignTypeWrapper, self).__init__(env)
        low = env.observation_space.low
        high = env.observation_space.high
        shape = env.observation_space.shape
        self.observation_space = gym.spaces.Box(low=low, high=high, shape=shape, dtype=np.uint8)

    env = gym.make("KukaDiverseObjectGrasping-v0", maxSteps=20, isDiscrete=False, renders=False, removeHeightHack=True)
    env = AssignTypeWrapper(env)
    env = Monitor(env, filename=filename)
    env = VecTransposeImage(DummyVecEnv([lambda: env]))
```

ここで `AssignTypeWrapper` を簡単に説明しますと、Kuka の状態空間は確かに 0-255 の整数値をとるように設計されているのですが、型が`np.float32`となっています。画像を入力とする場合は入力の型を `np.uint8`にしなければなりません [(Noteを参照)](https://stable-baselines3.readthedocs.io/en/master/guide/custom_env.html#using-custom-environments)。そのため `AssignTypeWrapper` では明示的に型が `np.uint8` であることを指定するようにしています。


## 学習

[こちら](https://github.com/swagatk/RL-Projects-SK/blob/master/src/stable_baselines/kukacam/SB3_KukaDiverseObjectEnv.ipynb)を参考に実装しています。

### Custum Policy

Stable-baselines3では自前のニューラルネットワークを使うときは次のような実装をします。

```python
from stable_baselines3.common.torch_layers import NatureCNN

sac_net_policy_kwargs = dict(
    features_extractor_class=NatureCNN,
    features_extractor_kwargs=dict(features_dim=128),
    net_arch=dict(qf=[128, 64, 32], pi=[128, 64])
)
```

それぞれのキーの意味は↓のようになっています。

- `features_extractor_class`: 画像から特徴量を抽出するニューラルネットワークの class。
- `features_extractor_class_kwargs`: `features_extractor_class` に指定したクラスの argument に入力する値を指定。
- `net_arch`: `features_extractor_class` で出力された特徴量から行動価値や方策を出力するネットワークを構成。SAC や DDPG, TD3 では `qf`, `pi`ともに必須。

`features_extractor_class` の値である `NatureCNN` は Stable-baselines3 に実装されている CNN で3つの畳み込み層と1つのFC層によって構成されたニューラルネットワークです。

最終的にこのネットワークは次のような構成になっています。

![](/res/dnn_arch.JPG)

(図はStable-Baseline3 の画像を参照)

Custom Policy の詳細な説明については[こちら](https://stable-baselines3.readthedocs.io/en/master/guide/custom_policy.html)をご覧ください。

### Stable-baseline3 の Soft Actor-Critic を使う

Stable-baselines3 で Soft Actor-Critic を使いたい場合は次にように実装します。

```python
from stable_baselines3 import SAC

result_path = "path/to/output_directory"
tensorflow_board_name = "board_name"

sac_model = SAC(
    policy='CnnPolicy', 
    env=env,
    verbose=1, 
    buffer_size=30000, 
    batch_size=256,
    policy_kwargs=sac_net_policy_kwargs,
    tensorboard_log=os.path.join(result_path, tensorflow_board_name)
    )
```

`policy` に指定するものは大体次のような使い分け方になります。
- `'CNNPolicy'`: 画像を入力とする場合
- `'MlpPolicy'`: 画像以外を入力とする場合
- `'MultiInputPolicy'`: 複数のタイプを入力とする場合

今回は画像を入力としているので `policy='CnnPolicy'` となっています。 

## 推論

学習したSoft Actor-Criticモデルで実際に物体を掴むことに成功するか確認します。

### 環境

ロボットアームが物体を掴めるかの様子を動画にしていますが、注意が必要です。
gymパッケージを使った場合、1 ステップのactionを実行する場合は `env.step(action)` という呼び出し方をしますが、`KukaDiverseObjectEnv`オブジェクト内部ではさらに（デフォルトで）80ステップの行動が実行されています。そのため、`env.step(action)` 毎にスナップショットを撮ったものを動画にすると早送りされた動画になってしまいます。

ここではロボットアームの動きを録画するスレッドを用意して撮るようにしました。

```python
class KukaVideoRecorder(Wrapper):
    def __init__(self, env, filename, video_folder):
        super(KukaVideoRecorder, self).__init__(env)
        self.recording = False
        self.in_playing = False

        os.makedirs(video_folder, exist_ok=True)
        self.file_path = os.path.join(video_folder, filename)
        self.video_recorder = video_recorder.VideoRecorder(env=self.env, base_path=self.file_path)

        def snapshot_worker(recorder):
            while recorder.recording:
                if recorder.in_playing:
                    recorder.video_recorder.capture_frame()

        self.capture_runner_thread = threading.Thread(target=snapshot_worker, args=(self,), daemon=True)

    def reset(self, **kwargs):
        self.in_playing = False
        observation = self.env.reset(**kwargs)
        self.in_playing = True

        if not self.recording:
            self.recording = True
            self.capture_runner_thread.start()

        return observation

    def __del__(self):
        if self.recording:
            self.recording = False
            self.video_recorder.close()
            self.capture_runner_thread.join()


env = gym.make("KukaDiverseObjectGrasping-v0", maxSteps=20, isDiscrete=False, renders=True, removeHeightHack=True, isTest=True)
env = KukaVideoRecorder(env=env, filename=filename, video_folder=dir)
```

### 推論の実行

学習したモデルは次のようにしてロードして実行することができます。

```python
model = SAC.load(path=model_path)

obs = target_env.reset()
while True:
    action, _ = model.predict(obs)
    obs, _, done, _ = target_env.step(action)
    if done:
        break
```

# まとめ

この記事では Stable-Baselines3 で実装された Soft Actor-Critic を使ってロボットアームが物体を掴むことを学習する方法を紹介しました。ロボットアームの物体を掴むタスクのシミュレータ環境は pybullet を使っています。

実際に学習させる実装を紹介する前に Stable-Baseline3 がどのようなライブラリなのか、また強化学習アルゴリズムの一つである Soft Actor-Critic の特徴がどのようなものかを簡単に紹介しました。さらに物理エンジン bullet の上でシミュレーションされるロボットアームの環境、KukaDiverseObjectGrasping-v0 の使い方や状態空間、行動空間そして報酬の与えられ方について紹介しました。

Soft Actor-Critic でロボットアームの把持タスクを解く方法を Stable-Baselines3 とカスタムニューラルネットワークの設計方法を紹介し、実際にこの設計で解くことが可能であることを示しました。今回はシミュレータ上で試しましたが、次は実機（こんな[実例](https://masato-ka.hatenablog.com/entry/2020/04/29/153505)もありますね）でも試してみたいなと思います。