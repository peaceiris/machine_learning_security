# Transferable Clean-Label Poisoning Attacks on Deep Neural Nets
[original paper](https://arxiv.org/abs/1905.05897)  

## Abstract
Clean-label poisoning攻撃は、一見すると無害に見える**汚染画像**を学習データに注入し、この学習データで作成されたモデルに**特定の画像を攻撃者の意図したクラスに誤分類**させることができる。本論文では、標的となるモデルの出力、ネットワーク構造、そして学習データにアクセスせずに攻撃を行うことが可能な手法を検討する。 この目標を達成するために、我々は**Polytope攻撃**と呼ぶ新たなpoisoning攻撃を提案する。この攻撃では、標的とするベース画像を特徴空間で囲むように汚染画像が設計される。また、汚染画像の作成中にDropoutを使用することで、この攻撃の汎用性（様々なモデルに対して攻撃が成功する）が向上することを示す。本手法の成功率は**50％**以上であるが、この時の学習データの汚染率は僅か**1％**である。  

## 1. Introduction
Deep Neural Network（以下、DNN）では、モデルの学習とパラメータチューニングのために大規模なデータセットが必要である。このため、多くのモデル作成者は学習データをWebから収集することが多く、これをWebスクレイピングで自動的に行っている。しかし、最近の研究では、このようなデータ収集方法が脆弱性につながる可能性があることが実証されている。特に信頼できないソースからデータを収集すると、作成するモデルがpoisoning攻撃に対して脆弱になり、攻撃者が悪意を持って作成したデータをデータセットに挿入することで、モデルを乗っ取ることが可能となる。  

本論文では、画像分類のタスクに対する**効果的で汎用的**なClean-label poisoning攻撃を検討する。一般的に、poisoning攻撃は学習データを変更することでモデルの推論（予測）動作を制御することを目的としている。従来のEvasion攻撃やバックドア攻撃とは対照的に、我々は攻撃対象のモデルの推論中に学習データが変更されないケースを調査する。  

Clean-label poisoning攻撃は、従来のpoisoning攻撃とは大きな違いがある。それは、学習データのラベル付け作業をユーザが制御する必要はないことである。よって、汚染画像はユーザによって正しくラベル付けされた場合でも、（誤分類を引き起こす）摂動を維持する必要がある。このような攻撃は、攻撃者がWeb上に汚染画像を配置するだけでデータセットを汚染することができ、WebスクレイピングやSNSの利用者、または汚染に気づかないユーザによって汚染画像が収集されるのを待つだけで良い。収集された汚染画像はユーザによって正しくラベル付けされ、学習データに挿入される。更に、標的型のClean-label攻撃は、モデルの分類精度を無作為に低下させるのではなく、特定のクラスのみをターゲットにすることが可能である。つまり、モデル全体の精度は低下しないため、攻撃の検知を困難にすることができる。  

従来のClean-label poisoning攻撃は、攻撃者が攻撃対象モデルに対する完全な知識を持ち、汚染画像を作成する過程でこの知識を使用するという**ホワイトボックス**設定でのみ実証されている。一方、我々は攻撃対象モデルの内部構造が未知（**ブラックボックス**）の状態で汚染画像を作成することを目指す。  

ブラックボックスのClean-label poisoning攻撃は、以下2つの理由で非常に挑戦的なタスクである。  
 
 * 汚染画像を含むデータセットで学習された攻撃対象モデルの決定境界は予測不能である。  
 * 攻撃者は攻撃対象モデルに直接アクセスすることができないため、汚染画像をモデルに依存しないようにする必要がある。  

ここでは、汎用性の高いClean-label poisoning攻撃を提案する。攻撃者は攻撃対象モデルの出力やハイパーパラメータにアクセスできないが、**攻撃対象モデルと同様のデータセットを収集できる**ことを想定している。攻撃者はこのデータセットで**代替モデルを作成**し、**標的空間をConvexの内部に閉じ込めるPolytope**を特徴空間内に形成するNovel Objectiveを最適化する。この汚染画像に過学習するクラスは、標的とする画像を汚染画像と同じクラスに分類する。このNovel Objectiveは、従来のブラックボックス攻撃であるFeature Collision攻撃よりも成功率が高く、ネットワーク内の複数の中間層に適用すると更に攻撃成功率が高まり、転送学習とEnd-to-Endで学習したモデルの両方で高い成功率を示す。また、汚染画像の作成時にDropoutを使用すると、攻撃の汎用性が向上することも示す。  

## 2. The Threat Model
Shafahiらののpoisoning攻撃のように、我々が想定する攻撃者は、少数の汚染画像を攻撃対象モデルが使用する学習データに注入する。攻撃者のゴールは、この学習データで作成された攻撃対象モデルに、テストデータ（当然ながら学習データに含まれない）を特定のクラスに分類させることである。攻撃者が摂動`δ`をベース画像に加えてゴールを達成する画像分類のケースを考えると、汚染画像`x+δ`が正常画像`x`と同じクラスを共有し、特定の方法でDNNの決定境界を変更できるように作成される。  

攻撃対象モデルへの完全なアクセス、またはクエリアクセスを必要とするShafahiらの研究やPapernotらの研究とは異なり、我々は自動運転自動車や監視システムのように、攻撃者が攻撃対象モデルにアクセスできないケースを想定する。その代わりに、攻撃者は攻撃対象モデルの学習データ分布に関する知識を必要とする。これにより、代替モデルの学習用に同様のデータセットを収集できるものとする。  

我々は、攻撃対象モデルが採用する可能性のある2つの学習アプローチを検討する。  

最初の学習アプローチは**転送学習**であり、CIFARやImageNetなどのパブリックなデータセットで学習した**ResNet**のConv層など、（事前学習された後に）Freezeされた特徴抽出器`φ`が画像に適用される。また、重み`W`やバイアス`b`を持つ線形分類器は、別のデータセット`X`の特徴`φ(X)`で微調整される。このタイプの転移学習は、モデル作成時に大規模なラベル付きデータセットが利用できない産業用アプリケーションでは一般的である。 転送学習に対するpoisoning攻撃は、KohとLiangらの研究やShafahiらの研究のように、特徴抽出器の内部構造が明らかになっているホワイトボックス設定で検証されている。  

2番目の学習アプローチは、（特徴抽出と線形分類器が同時に学習される）**End-to-End学習**である。これは明らかに、注入する摂動は特徴抽出器のパラメータ更新に影響を与えるため、このアプローチでは転送学習よりも摂動に対する厳しい制約がある。Shafahiらの10クラス分類設定で約60％の成功率を達成するためにのみ、ターゲットとなるベース画像の最大30％に、約40個の汚染画像に重ね合わせる**透かし戦略**を使用する。  

## 3. Transferable Targeted Poisoning Attacks
### 3.1. Difficulties of Targeted Poisoning Attacks
標的型poisoning攻撃は標的型Evasion攻撃よりも実現困難である。  
画像分類のタスクにおいて、標的型Evasion攻撃は攻撃対象モデルに汚染画像を特定のクラスに誤分類させるのみである。攻撃対象モデルは摂動に適応しないため、攻撃者は制約付き最適化問題を解くことで、決定境界への最短の摂動Pathを見つけるだけで済む。例えば、norm-boundedのホワイトボックス設定では、攻撃者は`δ`を直接最適化するために`δt =  argmin||δ||∞≤ε LCE`をProjected勾配降下法で解き、ターゲットラベル`yt`のクロスエントロピー損失`LCE(xt +δ, yt)`を最小化することができる。クエリアクセスを使用したnorm-boundedのブラックボックス設定では、攻撃者は代替モデルをトレーニングし、蒸留を介して攻撃対象モデルの挙動をシミュレートし、代替モデルに対して同じ最適化を実行して、汎用的な`δ`を見つけることができる。  

Targeted poisoning attacks, however, face a more challenging problem. The attacker needs to get the victim to classify the target sample xt into the alternative target class `y˜t` after being trained on the modified data distribution. One simple approach is to select the poisons from class `y˜t`, and make the poisons as close to xt as possible in the feature space. A rational victim will usually overfit the training set, since it is observed in practice that generalization keeps improving even after training loss saturates (Zhang et al., 2016). As a result, when the poisons are close to the target in the feature space, a rational victim is likely to classify xt into `y˜t` since the space near the poisons are classified as `y˜t`. However, as shown in Figure 1, smaller distance to the target does not always lead to a successful attack. In fact, being close to the target might be too restrictive for successful attacks. Indeed, there exists conditions where the poisons can be farther away from the target, but the attack is more successful.  

 <div align="center">
 <figure>
 <img src='./img/Fig1.png' alt='Figure1'><br>
 <figurecaption>Figure 1. An illustrative toy example of a linear SVM trained on a two-dimensional space with training sets poisoned by Feature Collision Attack and Convex Polytope Attack respectively. The two striped red dots are the poisons injected to the training set, while the striped blue dot is the target, which is not in the training set. All other points are in the training set. Even when the poisons are the closest points to the target, the optimal linear SVM will classify the target correctly in the left figure. The Convex Polytope attack will enforce a small distance of the line segment formed by the two poisons to the target. When the line segment’s distance to the target is minimized, the target’s negative margin in the retrained model is also minimized if it overfits.
 </figurecaption><br>
 <br>
 </figure>
 </div>

### 3.2. Feature Collision Attack
Feature collision attacks, as originally proposed in (Shafahi et al., 2018), are a reliable way of producing targeted cleanlabel poisons on white-box models. The attacker selects a base example `xb` from the targeted class for crafting the poisons `xp`, and tries to make `xp` become the same as the target `xt` in the feature space by adding small adversarial perturbations to `xb`. Specifically, the attacker solves the following optimization problem (1) to craft the poisons:  

 <div align="center">
 <figure>
 <img src='./img/Formula1.png' alt='Formula1'><br>
 <br>
 </figure>
 </div>

where `φ` is a pre-trained neural feature extractor. The first term enforces the poison to lie near the base in input space, and therefore maintains the same label as `xb` to a human labeler. The second term forces the feature representation of the poison to collide with the feature representation of the target. The hyperparameter `µ > 0` trades off the balance between these terms.  

If the poison example `xp` is correctly labeled as a member of the targeted class and placed in the victim’s training dataset `χ`, then after training on `χ`, the victim classifier learns to classify the poison’s feature representation into the targeted class. Then, if `xp`’s feature distance to xt is smaller than the margin of `xp`, xt will be classified into the same class as `xp` and the attack is successful. Sometimes more than one poison image is used to increase the success rate.  

Unfortunately for the attacker, different feature extractors `φ` will lead to different feature spaces, and a small feature space distance `d_φ(x_p, x_t) = ||φ(x_p) − φ(x_t)||` for one feature extractor does not guarantee a small distance for another.  

Fortunately, the results in (Tramer et al., 2017) have demonstrated that for each input sample, there exists an adversarial subspace for different models trained on the same dataset, such that a moderate perturbation can cause the models to misclassify the sample, which indicates it is possible to find a small perturbation to make the poisons `xp` close to the target `xt` in the feature space for different models, as long as the models are trained on the same dataset or similar data distributions.  

With such observation, the most obvious approach to forge a black-box attack is to optimize a set of poisons `{x_p^(j)}_j=1^k` to produce feature collisions for an ensemble of models `{φ^(i)}_i=1^m`, where m is the number of models in the ensemble. Such a technique was also used in black-box evasion attacks (Liu et al., 2016). Because different extractors produce feature vectors with different dimensions and magnitudes, we use the following normalized feature distance to prevent any one network from dominating the objective due to such biases:  

 <div align="center">
 <figure>
 <img src='./img/Formula2.png' alt='Formula2'><br>
 <br>
 </figure>
 </div>

### 3.3. Convex Polytope Attack
One problem with the feature collision attack is the emergence of obvious patterns of the target in the crafted perturbations. Unlike prevalent objectives for evasion attacks which maximize single-entry losses like cross entropy, feature collision (Shafahi et al., 2018) enforces each entry of the poison’s feature vector `φ(xp)` to be close to `φ(xt)`, which usually results in hundreds to thousands of constraints on each poison image `xp`. What is worse, in the black-box setting as Eq.2, the poisoning objective forces a collision over an ensemble of m networks, which further increases the number of constraints on the poison images. With such a large number of constraints, the optimizer often resorts to pushing the poison images in a direction where obvious patterns of the target will occur, therefore making `xp` look like the target class. As a result, human workers will notice the difference and take actions. Figure 9 shows a qualitative example in the process of crafting poisons from images of hook to attack a fish image with Eq.2. Elements of the target image that are evident in the poison images include the fish’s tail in the top image and almost a whole fish in the bottom image in column 3.  

 <div align="center">
 <figure>
 <img src='./img/Fig2.png' alt='Figure2'><br>
 <figurecaption>Figure 2. A qualitative example of the difference in poison images generated by Feature Collision (FC) Attack and Convex Polytope (CP) Attack. Both attacks aim to make the model mis-classify the target fish image on the left into a hook. We show two of the five hook images that were used for the attack, along with their perturbations and the poison images here. Both attacks were successful, but unlike FC, which demonstrated strong regularity in the perturbations and obvious fish patterns in the poison images, CP tends to have no obvious pattern in its poisons. More details are provided in the supplementary.
 </figurecaption><br>
 <br>
 </figure>
 </div>

Another problem with Feature Collision Attack is its lack of transferability. Feature Collision Attack tends to fail in the black-box setting because it is difficult to make the poisons close to `xt` for every model in the feature space. The feature space of different feature extractors should be very different, since neural networks are using one linear classifier to separate the deep features `φ(x)`, which has a unique solution if `φ(x)` is given, but different networks usually have different accuracies. Therefore, even if the poisons `xp` collide with `xt` in the feature space of the substitute models, they probably do not collide with `xt` in the unknown target model, due to the generalization error. As demonstrated by Figure 1, the attack is likely to fail even when `xp` has smaller distance to `xt` than its intra-class samples. It is also impractical to ensemble too many substitute models to reduce such error. We provide experimental results with the ensemble Feature Collision Attack defined by Eq.2 to show it can be ineffective.  

We therefore seek a looser constraint on the poisons, so that the patterns of the target are not obvious in the poison images and the requirements on generalization are reduced. Noticing (Shafahi et al., 2018) usually use multiple poisons to attack one target, we start by deriving the necessary and sufficient conditions on the set of poison features `{φ(x_p^(j))}_j=1^k` such that the target `xt` will be classified into the poison’s class.  

 * Proposition 1. The following statements are equivalent:  
 <div align="center">
 <figure>
 <img src='./img/Proposition1.png' alt='Proposition1'><br>
 <br>
 </figure>
 </div>

The proof of Proposition 1 is given in the supplementary material. In words, a set of poisons from the same class is guaranteed to alter the class label of a target into theirs  if that target’s feature vector lies in the convex polytope of the poison feature vectors. We emphasize that this is a far  more relaxed condition than enforcing a feature collision— it enables the poisons to lie much farther away from the  target while altering the labels on a much larger region of space. As long as `xt` lives inside this region in the unknown target model, and `{x_p^(j)}` are classified as expected, `xt` will be classified as the same label as `{x_p^(j)}`.  

With this observation, we optimize the set of poisons towards forming a convex polytope in the feature space such that the target’s feature vector will lie within or at least close to the convex polytope. Specifically, we solve the following optimization problem:  

 <div align="center">
 <figure>
 <img src='./img/Formula3.png' alt='Formula3'><br>
 <br>
 </figure>
 </div>

where `x_b^(j)` is the clean image of the `j-th` poison, and `ε` is the maximum allowable perturbation such that the perturbations are not immediately perceptible.  

Eq.3 simultaneously finds a set of poisons `{x_p^(j)}`, and a set of convex combination coefficients `{c_j^(i)}` such that the target lies in or close to the convex polytope of the poisons in the feature space of the `m` models. Notice the coefficients `{c_j^(i)}` are untied, i.e., they are allowed to vary across different models, which does not require `φ^(i)(xt)` to be close to any specific point in the polytope, including the vertices `{x_p^(j)}`. Given the same amount of perturbation, such an objective is also more relaxed than Feature Collision Attack (Eq.2) since Eq.2 is a special case of Eq.3 when we fix `c_j^(i) = 1/k`. As a result, the poisons demonstrate almost no patterns of the target, and the imperceptibility of the attack is enhanced compared with feature collision attack, as shown in Figure 9.  

The most important benefit brought by the convex polytope objective is the improved transferability. For Convex Polytope Attack, `xt` does not need to align with a specific point in the feature space of the unknown target model. It only needs to lie within the convex polytope formed by the poisons. In the case where this condition is not satisfied, Convex Polytope Attack still has advantages over Feature Collision Attack. Suppose for a given target model, a residual1 smaller than `ρ` will guarantee a successful attack2. For Feature Collision Attack, the target’s feature needs to lie within a ρ-ball centered at `φ^(t)(x_p^(j∗))`, where `j∗ = argminj||φ^(t)(x_p^(j)) − φ^(t)(xt)||`. For Convex Polytope Attack, the target’s feature could lie within the ρ-expansion of the convex polytope formed by `{φ^(t)(x_p^(j))}_j=1^k`, which has a larger volume than the aforementioned ρ-ball, and thus tolerates larger generalization error.  

### 3.4. An Efficient Algorithm for Convex Polytope Attack
We optimize the non-convex and constrained problem (3) using an alternating method that side-steps the difficulties posed by the complexity of `{φ^(i)}` and the convex polytope constraints on `{c^(i)}`. Given `{x_p^(j)}_j=1^k`, we use forwardbackward splitting (Goldstein et al., 2014) to find the the optimal sets of coefficients `{c^(i)}`. This step takes much less computation than back-propagation through the neural network, since the dimension of `c^(i)` is usually small (in our case a typical value is 5). Then, given the optimal `{c^(i)}` with respect to `{x_p^(j)}`, we take one gradient step to optimize `{x_p^(j)}`, since back-propagation through the `m` networks is relatively expensive. Finally, we project the poison images to be within `ε` units of the clean base image so that the perturbation is not obvious, which is implemented as a clip operation. We repeat this process to find the optimal set of poisons and coefficients, as shown in Algorithm 1.  

 <div align="center">
 <figure>
 <img src='./img/Formula.png' alt='Formula'><br>
 <br>
 </figure>
 </div>

In our experiments, we find that after the first iteration, initializing `{c^(i)}` to the value from the last iteration accelerates its convergence. We also find the loss in the target network to bear high variance without momentum optimizers. Therefore, we choose Adam (Kingma & Ba, 2014) as the optimizer for the perturbations as it converges more reliably than SGD in our case. Although the constraint on the perturbation is `ℓ∞` norm, in contrast to (Dong et al., 2018) and the common practices for crafting adversarial perturbations such as FGSM (Kurakin et al., 2016), we do not take the sign of the gradient, which further reduces the variance caused by the flipping of signs when the update step is already small.  

 <div align="center">
 <figure>
 <img src='./img/Algorithm1.png' alt='Algorithm1'><br>
 <br>
 </figure>
 </div>

### 3.5. Multi-Layer Convex Polytope Attack
When the victim trains its feature extractor `φ` in addition to the classifier (last layer), enforcing Convex Polytope Attack only on the feature space of `φ^(i)` is not enough for a successful attack as we will show in experiments. In this setting, the change in feature space caused by a model trained on the poisoned data will also make the polytope more difficult to transfer.  

Unlike linear classifiers, deep neural networks have much better generalization on image datasets. Since the poisons all come from one class, the whole network can probably generalize well enough to discriminate the distributions of xt and xp, such that after trained with the poisoned dataset, it will still classify xt correctly. As shown in Figure 7, when CP attack is applied to the last layer’s features, the lower capacity models like SENet18 and ResNet18 are more susceptible to the poisons than other larger capacity models. However, we know there probably exist poisons with small perturbations that are transferable to networks trained on the poison distribution, as empirical evidence from (Tramer et al., 2017) have shown there exist a common adversarial subspace for different models trained on the same dataset, and naturally trained networks usually have large enough Lipschitz to cause mis-classification (Szegedy et al., 2013), which hopefully will also be capable of shifting the polytope into such a subspace to lie close to xt.  

One strategy to increase transferability to models trained end-to-end on the poisoned dataset is to jointly apply Convex Polytope Attack to multiple layers of the network. The deep network is broken into shallow networks `φ1, ..., φ_n` by depth, and the objective now becomes  

 <div align="center">
 <figure>
 <img src='./img/Formula4.png' alt='Formula4'><br>
 <br>
 </figure>
 </div>

where `φ_1:l^(i)` is the concatenation from `φ_l^(i)` to `φ_l^(i)`. Networks similar to ResNet are broken into blocks separated by pooling layers, and we let `φ_l^(i)` be the l-th layer of such blocks. The optimal linear classifier trained with the features up to `φ1:l (l < n)` will have worse generalization than the optimal linear classifier trained with features of `φ`, and therefore the feature of xt should have higher chance to deviate from the features of the same class after training, which is a necessary condition for a successful attack. Meanwhile, with such an objective, the perturbation is optimized towards fooling models with different depths, which further increases the variety of the substitute models and adds to the transferability.  

### 3.6. Improved Transferability via Network Randomization
Even when trained on the same dataset, different models have different accuracy and therefore different distributions of the samples in the feature space. Ideally, if we craft the poisons with arbitrarily many networks from the function class of the target network then we should be able effectively minimize Eq. 3 in the target network. It is, however, impractical to ensemble a large number of networks due to memory constraints.  

To avoid ensembling too many models, we randomize the networks with Dropout (Srivastava et al., 2014), turning it on when crafting the poisons. In each iteration, each substitute network `φ^(i)` is randomly sampled from its function class by shutting off each neuron with probability p, and multiplying all the “on” neurons with `1/(1 − p)` to keep the expectation unchanged. In this way, we can get an exponential (in depth) number of different networks for free in terms of memory. Such randomized networks increase transferability in our experiments. One qualititative example is given in Figure 3.  

 <div align="center">
 <figure>
 <img src='./img/Fig3.png' alt='Figure3'><br>
 <figurecaption>Figure 3. Loss curves of Feature Collision and Convex Polytope Attack on the substitute models and the victim models, tested using the target with index 2. Dropout improved the minimum achievable test loss for the FC attack, and improved the test loss of the CP attack significantly.
 </figurecaption><br>
 <br>
 </figure>
 </div>

 <div align="center">
 <figure>
 <img src='./img/Fig4.png' alt='Figure4'><br>
 <figurecaption>Figure 4. Qualitative results of the poisons crafted by FC and CP. Each group shows the target along with five poisons crafted to attack it, where the first row is the poisons crafted with FC, and the second row is the poisons crafted with CP. In the first group, the CP poisons fooled a DenseNet121 but FC poisons failed, in the second group both succeeded. The second target’s image is more noisy and is probably an outlier of the frog class, so it is easier to attack. The poisons crafted by CP contain fewer patterns of xt than with FC, and are harder to detect.  
 </figurecaption><br>
 <br>
 </figure>
 </div>

## 4. Experiments
In the following, we will use CP and FC as abbreviations for Convex Polytope Attacks and Feature Collision attacks respectively. The code for the experiments is available at [our github](https://github.com/zhuchen03/ConvexPolytopePosioning).  

### Datasets
In this section, all images come from the CIFAR10 dataset. If not explicitly specified, we take the first 4800 images from each of the 10 classes (a total of 48000 images) in the training set to pre-train the victim models and the substitute models `(φ^(i))`. We leave the test set intact so that the accuracy of these models under different settings can be evaluated on the standard test set and compared directly with the benchmarks. A successful attack should not only have unnoticeable image perturbations, but also unchanged test accuracy after fine-tuning on the clean and poisoned datasets. As shown in the supplementary, our attack preserves the accuracy of the victim model compared with the accuracy of those tuned on the corresponding clean dataset.  

The remaining 2000 images of the training set serve as a pool for selecting the target, crafting the poisons, and finetuning the victim networks. We take the first 50 images from each class (a total of 500 images) in this pool as the clean fine-tuning dataset. This resembles the scenario where pretrained models on large datasets like Imagenet (Krizhevsky et al., 2012) are fine-tuned on a similar but usually disjoint dataset. We randomly selected “ship” as the target class, and “frog” as the targeted image’s class, i.e., the attacker wants to cause a particular frog image to be misclassified as a ship. The poison images `x_p^(j)` across all experiments are crafted from the first 5 images of the ship class in the 500-image fine-tuning dataset. We evaluate the poison’s efficacy on the next 50 images of the frog class. Each of these images is evaluated independently as the target xt to collect statistics. Again, the target images are not included in the training and fine-tuning set.  

### Networks
Two sets of substitute model architectures are used in this paper. Set S1 includes SENet18 (Hu et al., 2018), ResNet50 (He et al., 2016), ResNeXt29-2x64d (Xie et al., 2017), DPN92 (Chen et al., 2017b), MobileNetV2 (Sandler et al., 2018) and GoogLeNet (Szegedy et al., 2015). Set S2 includes all the architectures of S1 except for MobileNetV2 and GoogLeNet. S1 and S2 are used in different experiments as specified below. ResNet18 and DenseNet121 (Huang et al., 2017) were used as the black-box model architectures. The poisons are crafted on models from the set of substitute model architectures. We evaluate the poisoning attack against victim models from the 6 different substitute model architectures as well as from the 2 black-box model architectures. Each victim model, however, was trained with different random seeds than the substitute models. If the victim’s architecture appears in the substitute models, we call it a gray-box setting; otherwise, it is a black-box setting.  

We add a Dropout layer at the output of each Inception block for GoogLeNet, and in the middle of the convolution layers of each Residual-like blocks for the other networks. We train these models from scratch on the aforementioned 48000-image training set with Dropout probabilities of 0, 0.2, 0.25 and 0.3, using the same architecture and hyperparameters (except for Dropout) of a public [repository](https://github.com/kuangliu/pytorch-cifar). The victim models that we evaluate were not trained with Dropout.  

### Attacks
We use the same 5 poison ship images to attack each frog image. For the substitute models, we use 3 models trained with Dropout probabilities of 0.2, 0.25, 0.3 from each architecture, which results in 18 and 12 substitute models for S1 and S2 respectively. When crafting the poisons, we use the same Dropout probability as the models were trained with. For all our experiments, we set `ε = 0.1`. We use Adam (Kingma & Ba, 2014) with a relatively large learning rate of 0.04 for crafting the poisons, since the networks have been trained to have small gradients on images similar to the training set. We perform no more than 4000 iterations on the poison perturbations in each experiment. Unless specified, we only enforce Eq. 3 on the
features of the last layer.  

For the victim, we choose its hyperparameters during finetuning such that it overfits the 500-image training set, which satisfies the aforementioned rational victim assumption. In the transfer learning setting, where only the final linear classifier is fine-tuned, we use Adam with a large learning rate of 0.1 to overfit. In the end-to-end setting, we use Adam with a small learning rate of 10−4 to overfit.  

### 4.1. Comparison with Feature Collision
We first compare the transferability of poisons generated by FC and CP in the transfer learning training context. The results are shown in Figure 5. We use set S1 of substitute architectures. FC never achieves a success rate higher than 0.5, while CP achieves success rates higher or close to 0.5 in most cases. A qualitative example of the poisons crafted by the two approaches is shown in Figure 4.  

 <div align="center">
 <figure>
 <img src='./img/Fig5.png' alt='Figure5'><br>
 <figurecaption>Figure 5. Success rates of FC and CP attacks on various models. Notice the first six entries are the gray-box setting where the models with same architecture but different weights are in the substitute networks, while the last two entries are the black-box setting.
 </figurecaption><br>
 <br>
 </figure>
 </div>

### 4.2. Importance of Training Set
Despite being much more successful than FC, questions remain about how reliable CP will be when we have no knowledge of the victim’s training set. In the last section, we trained the substitute models on the same training set as the victim. In Figure 6 we provide results for when the substitute models’ training sets are similar to (but mostly different from) that of the victim. Such a setting is sometimes more realistic than the setting where no knowledge of the victim’s training set is required, but query access to the victim model is needed (Papernot et al., 2017), since query access is not available for scenarios like surveillance. We use the less ideal S2, which has 12 substitute models from 4 different architectures. Results are evaluated in the transfer learning setting. Even with no data overlap, CP can still transfer to models with very different structure than the substitute models in the black-box setting. In the 0% overlap setting, the poisons transfer better to models with higher capacity like DPN92 and DenseNet121 than to lowcapacity ones like SENet18 and MobileNetV2, probably because high capacity models overfit more to their training set. Overall, we see that CP may remain powerful without access to the training data in the transfer learning setting, as long as the victim’s model has good generalization.  

 <div align="center">
 <figure>
 <img src='./img/Fig6.png' alt='Figure6'><br>
 <figurecaption>Figure 6. Success rates of Convex Polytope Attack, with poisons crafted by substitute models trained on the first 2400 images of each class of CIFAR10. The models corresponding to the three settings are trained with samples indexed from 1201 to 3600, 1 to 4800 and 2401 to 4800, corresponding to the settings of 50%, 50% and 0% training set overlaps respectively.
 </figurecaption><br>
 <br>
 </figure>
 </div>

### 4.3. End-to-End Training
A more challenging setting is when the victim adopts endto-end training. Unlike the transfer learning setting where models with better generalization turn out to be more vulnerable, here good generalization may help such models classify the target correctly despite the poisons. As shown in Figure 7, CP attacks on the last layer’s feature is not enough for transferability, leading to almost zero successful attacks. It is interesting to see that the success rate on ResNet50 is 0, which is an architecture in the substitute models, while the success rate on ResNet18 is the highest, which is not an architecture in the substitute models but should have worse generalization.  

Therefore, we enforce CP in multiple layers of the substitute models, which breaks the models into lower capacitie ones and leads to much better results. In the gray-box setting, all of the attacks achieved more than 0.6 success rates. However, it remains very difficult to transfer to GoogLeNet, which has a more different architecture than the substitute models. It is therefore more difficult to find a direction to make the convex polytope survive end-to-end training.  

 <div align="center">
 <figure>
 <img src='./img/Fig7.png' alt='Figure7'><br>
 <figurecaption>Figure 7. Success rates of Convex Polytope Attack in the end-toend training setting. We use S2 for the substitute models.
 </figurecaption><br>
 <br>
 </figure>
 </div>

## 5. Conclusion
In summary, we have demonstrated an approach to enhance the transferability of clean-labeled targeted poisoning attacks. The main contribution is a new objective which constructs a convex polytope around the target image in feature space, so that a linear classifier which overfits the poisoned dataset is guaranteed to classify the target into the poisons’ class. We provided two practical ways to further improve transferability. First, turn on Dropout while crafting poisons, so that the objective samples from a variety (i.e. ensemble) of networks with different structures. Second, enforce the convex polytope objective in multiple layers, which enables attack success even in end-to-end learning contexts. Additionally, we found that transferability can depend on the data distribution used to train the substitute model.  

## A. Proof of Proposition 1
`2 ⇒ 1`  

For multi-class problems, the condition for `φ(x)` to be classified as `ℓ_p` is  

 <div align="center">
 <figure>
 <img src='./img/A-1.png' alt='FormulaA-1'><br>
 <br>
 </figure>
 </div>

Each of these constraints is linear, and is satisfied by a convex half-space. The region that satisfies all of these constraints in an intersection of convex half-spaces, and so is convex. Under condition (2), `φ(xt)` is a convex combination of points in this convex region, and so `φ(xt)` is itself in this convex region.  

`2 ⇒ 1`  

Suppose that (1) holds. Let  

 <div align="center">
 <figure>
 <img src='./img/A-2.png' alt='FormulaA-2'><br>
 <br>
 </figure>
 </div>

be the convex hull of the points `φ(x_p^j)}_j=1^k` Let `t = argmin_u∈S||u − φ(x_t)||` be the closest point to `φ(x_t)` in `S`. If `||u_t − φ(x_t)|| = 0`, then (2) holds and the proof is complete. If `||u_t − φ(x_t)|| > 0`, then define the classifier function.  

 <div align="center">
 <figure>
 <img src='./img/A-3.png' alt='FormulaA-3'><br>
 <br>
 </figure>
 </div>

Clearly `f(φ(x_t)) < 0`. By condition (1), there is some `j` with `f(φ(x_p^j)) < 0` as well. Consider the function  

 <div align="center">
 <figure>
 <img src='./img/A-4.png' alt='FormulaA-4'><br>
 <br>
 </figure>
 </div>

Because ut is the closest point to `φ(x_t)` in `S`, and `g` is smooth, the derivative of `g` with respect to `η`, evaluated at `η = 0`, is `0`. We can write this derivative condition as  

 <div align="center">
 <figure>
 <img src='./img/A-5.png' alt='FormulaA-5'><br>
 <br>
 </figure>
 </div>

However this statement is a contradiction, since `f(φ(x_p^j)) < 0`.  

## B. Comparison of Validation Accuracies
To make data poisoning attacks undetectable, in addition to making the perturbations to nonobvious, the accuracy of the model fine-tuned on the poisoned dataset shall not drop too significantly, compared with fine-tuning on the same (except for the poisons) clean dataset. Figure 8 shows that the drop in accuracy is indeed not obvious.

## C. Details of the qualitative example
Both the target fish image and the five hook images used for crafting poisons come from the WebVision (Li et al., 2017) dataset, which has the same taxonomy as the ImageNet dataset. Figure 9 gives all the five poison examples.

 <div align="center">
 <figure>
 <img src='./img/Fig8.png' alt='Figure8'><br>
 <figurecaption>Figure 8. Accuracies on the whole CIFAR10 test for models trained or fine-tuned on different datasets. The fine-tuned models are initialized with the network trained on the first 4800 images of each class. There is little accuracy drop after fine-tuned on the poisoned datasets, compared with fine-tuning on the clean 500-image set directly.
 </figurecaption><br>
 <br>
 </figure>
 </div>

 <div align="center">
 <figure>
 <img src='./img/Fig9.png' alt='Figure9'><br>
 <figurecaption>Figure 9.  All the 5 poison images.
 </figurecaption><br>
 <br>
 </figure>
 </div>