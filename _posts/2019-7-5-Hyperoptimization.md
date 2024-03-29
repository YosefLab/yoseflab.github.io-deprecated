---
layout:     post
title:      Hyperparameter search for scVI
date:       2019-07-05 12:00:00
summary:    A study of the effects of hyperparameter optimization on scVI models.
author:     Gabriel Misrachi, Jeffrey Regier, Romain Lopez, Nir Yosef
visible:    True
---

While stochastic gradient-based optimization is highly successful for setting weights and other differentiable parameters of a neural network, it is in general useless for setting hyperparameters -- non-differentiable parameters that control the structure of the network (e.g. the number of hidden layers, or the dropout rate) or settings of the optimizer itself (e.g., the learning rate schedule). Yet finding good settings for hyperparameters is essential for good performance for deep methods like [scVI](https://www.nature.com/articles/s41592-018-0229-2). Furthermore, as pointed out by [Hu and Greene (2019)](https://www.worldscientific.com/doi/pdf/10.1142/9789813279827_0033?download=true&) selecting hyperparameters is nessary in order to compare different machine learning models, especially if those are substantially sensitive to hyperparameter variations. 

Generally, hyperparameter search is known to be challenging for the end-user and time-consuming. Therefore, we added in [`scVI`](https://github.com/YosefLab/scVI) a module based on the Bayesian optimization framework [`hyperopt`](https://github.com/hyperopt/hyperopt). This new feature makes effective use of multiple GPUs and is a ready-to-use solution for tuning hyperparameters in scVI. This blog post presents this feature and investigates how end-users will benefit from it.

# Theory : Bayesian Optimization and `Hyperopt`

First and foremost, one needs to carefully select a metric for which to search for optimal hyperparameters, keeping in mind that this one might depend on the downstream task, as explained in [Theis et al. (2016)](https://arxiv.org/abs/1511.01844). Since we train a generative model, the held-out negative log-likelihood is a natural criterion for model selection. We therefore rely on it for hyperparameters tuning. 

Historically, the very first methods introduced to tackle hyperparameter search were grid search and random search. Such approaches are limited in the sense that they scale exponentially with the size of the search space. A more scalable and elegant approach to search is Bayesian Optimization (BO), whose main idea is to learn and use the structure of the objective function. The overall process can be decomposed into two unordered tasks. One is to define a tractable surrogate prior for the underlying black-box objective (e.g., a Gaussian Process). The other is to define a criterion for which to optimize this surrogate process.

In the special case of `hyperopt` the criterion used is the expectation of the improvement over a certain baseline which can be the current best or lower. The surrogate is called a Tree-strctured Parzen Estimator. The TPE divides the sample set into two subsets based on their value compared to the current baseline and estimates two corresponding conditional densities over the search space using non-parametric kernel-density estimators. Then, good samples should have a high conditional density for improving samples and vice versa.

As suggested in the original article [Bergsta et al. (2011)](https://papers.nips.cc/paper/4443-algorithms-for-hyper-parameter-optimization.pdf), this hyper-optimization process can be used asynchronously by mocking the result for the pending trials with the mean of the available samples. Still, it should be noted that if the budget is a single evaluation per process running in parallel, then a random search should be preferred.

There are a certain number of recent approaches competing with `hyperopt`. [Li et al. (2018)](https://arxiv.org/abs/1603.06560) proposes Hyperband: an alternative to BO with bandit-based principled early stopping, which scales better with the number of workers.  These can be also combined for trial selection and principled early stopping, as explained in [Wang et al. (2018)](https://arxiv.org/abs/1801.01596). Interested readers might also take a look at two black-box optimizers based on different paradigms, namely the [CMA-ES](https://en.wikipedia.org/wiki/CMA-ES) and [DFO-TR](https://github.com/TheClimateCorporation/dfo-algorithm) algorithms - both have available code. BO enthusiasts can take a look at [scikit-optimize](https://github.com/scikit-optimize/scikit-optimize) as well as [Ax](https://github.com/facebook/Ax), Facebook's brand new pytorch-powered ML platform .


# Practice :  scVI's autotune module

We are excited to release our new `autotune` module for scVI, based on `hyperopt`. We thank `hyperopt` developers for their maintained codebase as well as their intelligent MongoDb-based communication between workers. Using this new feature on the Cortex dataset is as simple as running the code snippet below. 

```python
import logging
logger = logging.getLogger("scvi.inference.autotune")
logger.setLevel(logging.DEBUG)

if __name__ == "__main__":
	best_trainer, trials = auto_tune_scvi_model(
	    gene_dataset=dataset,
	    parallel=True,
	    exp_key="cortex_test",
	    max_evals=100,
	)
``` 

Note that the default behaviour of `auto_tune_scvi_model` is as follows:
* A parameter search space containing the learning rate, the network architecture and the dropout rate is provided. 
* The objective metric is set to be the held-out log-likelihood of the data computed on a test set which size equals 25% of the whole dataset. 
* All available GPUs are used. 

Of course, the behaviour of the auto-tuning process can be widely customized. For example, one can select the number of CPU processes, the ids of the GPUs to use, one can also input their own hyperparameter search space or even their own objective function.  For a more advanced usage, we expose functions to launch your own workers, which could be used to run the search on multiple machines.

# Results

In order to evaluate the autotune procedure, we ran it on several standard datasets, directly available using our codebase. Namely, we used the Cortex, Pbmc and Brain Large datasets.

### Datasets
Let us first describe these three datasets in more detail.
* **CORTEX** The mouse cortex dataset from [Zeisel et al. (2015)](https://science.sciencemag.org/content/347/6226/1138) contains 3005 mouse cortex cells and gold-standard labels for seven distinct cell types. We retain the top 558 genes ordered by variance.
* **PBMC** As in the scVI manuscript, we concatenated two datasets of peripheral blood mononuclear cells (PBMCs) from a healthy donor from [Zheng et al. (2017)](https://www.nature.com/articles/ncomms14049). After filtering, we extract 11,990 cells with 3446 genes.
* **BRAIN LARGE** This dataset contains 1.3 million brain cells from [10x Genomics](https://support.10xgenomics.com/single-cell-gene-expression/datasets). We retain the 720 most variable genes.

For each of these, we ran our autotune process using a default search space and a budget of 100 trainings. The raw outcome of these experiments can be found [here](https://docs.google.com/spreadsheets/d/1IEKDr1bJcNjcnFbitmN8EVPizMzfWQUdiQIfE5hVtCw/edit?usp=sharing) in the form of a table containing the parameters used and the resulting performance.

### Runtime information

We report the runtime metadata of our experiments in Table 1. This should give the user an idea of the time it takes to use our new feature, depending on the size of the dataset, the number of evaluations, the max number of epochs allowed and the number of GPUs available.

<center>
<table style="width:70%; font-size:80%" align="center">
  <thead>
    <tr>
      <th>Dataset Name</th>
      <th>Nb cells</th>
      <th>Nb genes</th>
      <th>Wall time</th>
      <th>Avg epochs</th>
      <th>N° of GPUs</th>
      <th>Max epoch</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>cortex</th>
      <td>3005</td>
      <td>558</td>
      <td>9 hours</td>
      <td>532 </td>
      <td>1</td>
      <td>1000</td>
    </tr>
    <tr>
      <th>pbmc</th>
      <td>11990</td>
      <td>3346</td>
      <td>3 hours</td>
      <td>387 </td>
      <td>16</td>
      <td>1000</td>
    </tr>
    <tr>
      <th>brain large</th>
      <td>1303182</td>
      <td>720</td>
      <td>21.5 hours</td>
      <td>43 </td>
      <td>16</td>
      <td>50</td>
    </tr>
  </tbody>
</table>
</center>

Table 1: Runtime Table for 100 trainings. Wall time is the total time the experiment took. Avg epoch is the average number of epochs per training.

#### Hyperparameter sensitivity 

Our first concern was to investigate scVI's sensitivity to hyperparameters. One ideal way to investigate this matter would have been to study the variance of the performance for each set of hyperparameters and collapse the runs who are not significantly different before analyzing the remaining representants. As this is too computationally intensive, we instead focus on the ten best and ten worst runs for each dataset.

<font size="1" face="Courier New" >
<center>
<table style="width:70%; font-size:80%" align="center">
  <summary> Cortex </summary>
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>marginal_ll</th>
      <th>n_layers</th>
      <th>n_hidden</th>
      <th>n_latent</th>
      <th>reconstruction_loss</th>
      <th>dropout_rate</th>
      <th>lr</th>
      <th>n_epochs</th>
      <th>n_params</th>
      <th>run index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>1218.52</td>
      <td>1</td>
      <td>256</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.01</td>
      <td>248</td>
      <td>290816</td>
      <td>92</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1218.7</td>
      <td>1</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.01</td>
      <td>382</td>
      <td>145920</td>
      <td>80</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1219.7</td>
      <td>1</td>
      <td>256</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.01</td>
      <td>365</td>
      <td>290816</td>
      <td>85</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1220.06</td>
      <td>1</td>
      <td>256</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.01</td>
      <td>275</td>
      <td>290816</td>
      <td>91</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1223.09</td>
      <td>1</td>
      <td>128</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.01</td>
      <td>440</td>
      <td>145408</td>
      <td>83</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1223.2</td>
      <td>1</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.005</td>
      <td>703</td>
      <td>145920</td>
      <td>38</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1223.53</td>
      <td>1</td>
      <td>256</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.001</td>
      <td>514</td>
      <td>290816</td>
      <td>97</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1223.94</td>
      <td>1</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>542</td>
      <td>145920</td>
      <td>74</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1224.37</td>
      <td>1</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>524</td>
      <td>145920</td>
      <td>76</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1224.37</td>
      <td>1</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>497</td>
      <td>145920</td>
      <td>71</td>
    </tr>
    <tr>
      <th>91</th>
      <td>1554.98</td>
      <td>2</td>
      <td>64</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.7</td>
      <td>0.005</td>
      <td>256</td>
      <td>80896</td>
      <td>45</td>
    </tr>
    <tr>
      <th>92</th>
      <td>1559.01</td>
      <td>5</td>
      <td>64</td>
      <td>7</td>
      <td>nb</td>
      <td>0.5</td>
      <td>0.0001</td>
      <td>457</td>
      <td>105088</td>
      <td>37</td>
    </tr>
    <tr>
      <th>93</th>
      <td>1601.53</td>
      <td>3</td>
      <td>64</td>
      <td>10</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.001</td>
      <td>88</td>
      <td>89088</td>
      <td>15</td>
    </tr>
    <tr>
      <th>94</th>
      <td>1612.9</td>
      <td>4</td>
      <td>64</td>
      <td>14</td>
      <td>zinb</td>
      <td>0.7</td>
      <td>0.005</td>
      <td>71</td>
      <td>97792</td>
      <td>49</td>
    </tr>
    <tr>
      <th>95</th>
      <td>1615.22</td>
      <td>2</td>
      <td>256</td>
      <td>9</td>
      <td>nb</td>
      <td>0.9</td>
      <td>0.0001</td>
      <td>197</td>
      <td>421376</td>
      <td>20</td>
    </tr>
    <tr>
      <th>96</th>
      <td>1746.25</td>
      <td>3</td>
      <td>128</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.001</td>
      <td>134</td>
      <td>211456</td>
      <td>52</td>
    </tr>
    <tr>
      <th>97</th>
      <td>1818.82</td>
      <td>1</td>
      <td>64</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.0005</td>
      <td>54</td>
      <td>72960</td>
      <td>60</td>
    </tr>
    <tr>
      <th>98</th>
      <td>6574.57</td>
      <td>1</td>
      <td>128</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.0001</td>
      <td>4</td>
      <td>144896</td>
      <td>61</td>
    </tr>
    <tr>
      <th>99</th>
      <td>10680.4</td>
      <td>5</td>
      <td>64</td>
      <td>12</td>
      <td>zinb</td>
      <td>0.3</td>
      <td>0.0001</td>
      <td>2</td>
      <td>105728</td>
      <td>1</td>
    </tr>
    <tr>
      <th>100</th>
      <td>NaN</td>
      <td>2</td>
      <td>64</td>
      <td>6</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.0001</td>
      <td>31</td>
      <td>80384</td>
      <td>13</td>
    </tr>
  </tbody>
</table>
</center>
</font>

Table 2: Hyperoptimization results for the Cortex dataset.

<font size="1" face="Courier New" >
<center>
<table style="width:70%; font-size:80%" align="center">
  <summary> PBMC </summary>
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>marginal_ll</th>
      <th>n_layers</th>
      <th>n_hidden</th>
      <th>n_latent</th>
      <th>reconstruction_loss</th>
      <th>dropout_rate</th>
      <th>lr</th>
      <th>n_epochs</th>
      <th>n_params</th>
      <th>run index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>1323.44</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>170</td>
      <td>1720320</td>
      <td>70</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1323.88</td>
      <td>1</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>130</td>
      <td>1717248</td>
      <td>54</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1324.12</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>178</td>
      <td>1720320</td>
      <td>85</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1324.18</td>
      <td>1</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>155</td>
      <td>1717248</td>
      <td>65</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1324.2</td>
      <td>1</td>
      <td>256</td>
      <td>15</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>140</td>
      <td>1720832</td>
      <td>84</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1324.23</td>
      <td>1</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.005</td>
      <td>170</td>
      <td>1717248</td>
      <td>42</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1324.24</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>227</td>
      <td>1720320</td>
      <td>87</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1324.25</td>
      <td>1</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>176</td>
      <td>1717248</td>
      <td>67</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1324.29</td>
      <td>1</td>
      <td>128</td>
      <td>15</td>
      <td>zinb</td>
      <td>0.3</td>
      <td>0.01</td>
      <td>176</td>
      <td>860416</td>
      <td>73</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1324.32</td>
      <td>1</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.01</td>
      <td>133</td>
      <td>1717248</td>
      <td>69</td>
    </tr>
    <tr>
      <th>91</th>
      <td>1347.3</td>
      <td>3</td>
      <td>64</td>
      <td>6</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.01</td>
      <td>285</td>
      <td>445440</td>
      <td>37</td>
    </tr>
    <tr>
      <th>92</th>
      <td>1350.02</td>
      <td>2</td>
      <td>128</td>
      <td>13</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.001</td>
      <td>350</td>
      <td>892672</td>
      <td>8</td>
    </tr>
    <tr>
      <th>93</th>
      <td>1350.54</td>
      <td>1</td>
      <td>64</td>
      <td>5</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.0005</td>
      <td>492</td>
      <td>428928</td>
      <td>0</td>
    </tr>
    <tr>
      <th>94</th>
      <td>1350.59</td>
      <td>1</td>
      <td>128</td>
      <td>13</td>
      <td>nb</td>
      <td>0.9</td>
      <td>0.01</td>
      <td>55</td>
      <td>859904</td>
      <td>5</td>
    </tr>
    <tr>
      <th>95</th>
      <td>1350.9</td>
      <td>2</td>
      <td>128</td>
      <td>13</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.0005</td>
      <td>710</td>
      <td>892672</td>
      <td>15</td>
    </tr>
    <tr>
      <th>96</th>
      <td>1352.32</td>
      <td>3</td>
      <td>256</td>
      <td>14</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.01</td>
      <td>201</td>
      <td>1982464</td>
      <td>88</td>
    </tr>
    <tr>
      <th>97</th>
      <td>1355.68</td>
      <td>3</td>
      <td>256</td>
      <td>5</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.001</td>
      <td>668</td>
      <td>1977856</td>
      <td>49</td>
    </tr>
    <tr>
      <th>98</th>
      <td>1365.46</td>
      <td>3</td>
      <td>256</td>
      <td>10</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.01</td>
      <td>42</td>
      <td>1980416</td>
      <td>75</td>
    </tr>
    <tr>
      <th>99</th>
      <td>1367.7</td>
      <td>2</td>
      <td>128</td>
      <td>15</td>
      <td>zinb</td>
      <td>0.9</td>
      <td>0.0001</td>
      <td>999</td>
      <td>893184</td>
      <td>13</td>
    </tr>
    <tr>
      <th>100</th>
      <td>1370.3</td>
      <td>3</td>
      <td>256</td>
      <td>8</td>
      <td>nb</td>
      <td>0.9</td>
      <td>0.01</td>
      <td>55</td>
      <td>1979392</td>
      <td>81</td>
    </tr>
  </tbody>
</table>
</center>
</font>

Table 3: Hyperoptimization results for the Pbmc dataset.
  
<font size="1" face="Courier New" >
<center>
<table style="width:70%; font-size:80%" align="center">
<summary> Brain Large </summary>
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>marginal_ll</th>
      <th>n_layers</th>
      <th>n_hidden</th>
      <th>n_latent</th>
      <th>reconstruction_loss</th>
      <th>dropout_rate</th>
      <th>lr</th>
      <th>n_epochs</th>
      <th>n_params</th>
      <th>run index</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>1</th>
      <td>1143.79</td>
      <td>1</td>
      <td>256</td>
      <td>15</td>
      <td>zinb</td>
      <td>0.1</td>
      <td>0.0005</td>
      <td>50</td>
      <td>376320</td>
      <td>92</td>
    </tr>
    <tr>
      <th>2</th>
      <td>1144.11</td>
      <td>1</td>
      <td>256</td>
      <td>15</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.0005</td>
      <td>42</td>
      <td>376320</td>
      <td>72</td>
    </tr>
    <tr>
      <th>3</th>
      <td>1144.22</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.005</td>
      <td>48</td>
      <td>375808</td>
      <td>70</td>
    </tr>
    <tr>
      <th>4</th>
      <td>1144.84</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.005</td>
      <td>43</td>
      <td>375808</td>
      <td>86</td>
    </tr>
    <tr>
      <th>5</th>
      <td>1144.97</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.005</td>
      <td>48</td>
      <td>375808</td>
      <td>68</td>
    </tr>
    <tr>
      <th>6</th>
      <td>1145.08</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.005</td>
      <td>43</td>
      <td>375808</td>
      <td>51</td>
    </tr>
    <tr>
      <th>7</th>
      <td>1145.2</td>
      <td>1</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.005</td>
      <td>47</td>
      <td>375808</td>
      <td>85</td>
    </tr>
    <tr>
      <th>8</th>
      <td>1145.86</td>
      <td>2</td>
      <td>256</td>
      <td>13</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.0001</td>
      <td>49</td>
      <td>506368</td>
      <td>55</td>
    </tr>
    <tr>
      <th>9</th>
      <td>1146.05</td>
      <td>2</td>
      <td>256</td>
      <td>13</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.0001</td>
      <td>49</td>
      <td>506368</td>
      <td>42</td>
    </tr>
    <tr>
      <th>10</th>
      <td>1146.11</td>
      <td>2</td>
      <td>256</td>
      <td>13</td>
      <td>nb</td>
      <td>0.1</td>
      <td>0.0001</td>
      <td>49</td>
      <td>506368</td>
      <td>58</td>
    </tr>
    <tr>
      <th>91</th>
      <td>1185.27</td>
      <td>5</td>
      <td>128</td>
      <td>13</td>
      <td>zinb</td>
      <td>0.5</td>
      <td>0.0005</td>
      <td>49</td>
      <td>318720</td>
      <td>40</td>
    </tr>
    <tr>
      <th>92</th>
      <td>1188.06</td>
      <td>5</td>
      <td>256</td>
      <td>14</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.0001</td>
      <td>44</td>
      <td>900096</td>
      <td>24</td>
    </tr>
    <tr>
      <th>93</th>
      <td>1188.98</td>
      <td>4</td>
      <td>128</td>
      <td>15</td>
      <td>nb</td>
      <td>0.5</td>
      <td>0.005</td>
      <td>49</td>
      <td>286464</td>
      <td>4</td>
    </tr>
    <tr>
      <th>94</th>
      <td>1193.21</td>
      <td>5</td>
      <td>256</td>
      <td>8</td>
      <td>zinb</td>
      <td>0.7</td>
      <td>0.005</td>
      <td>47</td>
      <td>897024</td>
      <td>30</td>
    </tr>
    <tr>
      <th>95</th>
      <td>1202.7</td>
      <td>5</td>
      <td>128</td>
      <td>11</td>
      <td>nb</td>
      <td>0.5</td>
      <td>0.001</td>
      <td>46</td>
      <td>318208</td>
      <td>11</td>
    </tr>
    <tr>
      <th>96</th>
      <td>1203.35</td>
      <td>2</td>
      <td>64</td>
      <td>13</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.0001</td>
      <td>49</td>
      <td>102016</td>
      <td>56</td>
    </tr>
    <tr>
      <th>97</th>
      <td>1205.6</td>
      <td>2</td>
      <td>64</td>
      <td>9</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.0001</td>
      <td>48</td>
      <td>101504</td>
      <td>82</td>
    </tr>
    <tr>
      <th>98</th>
      <td>1206.69</td>
      <td>4</td>
      <td>128</td>
      <td>12</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.0001</td>
      <td>47</td>
      <td>285696</td>
      <td>61</td>
    </tr>
    <tr>
      <th>99</th>
      <td>1211.31</td>
      <td>5</td>
      <td>256</td>
      <td>7</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.005</td>
      <td>38</td>
      <td>896512</td>
      <td>74</td>
    </tr>
    <tr>
      <th>100</th>
      <td>1232.12</td>
      <td>5</td>
      <td>64</td>
      <td>15</td>
      <td>nb</td>
      <td>0.7</td>
      <td>0.0001</td>
      <td>26</td>
      <td>126848</td>
      <td>52</td>
    </tr>
  </tbody>
</table>
</center>
</font>

Table 4: Hyperoptimization results for the Brain Large dataset.

From Table 2, 3 and 4 we note that the obtained parameters agree on certain aspects such as the number of layers or the learning rate. Additionally, it is worthwhile noting that the best results often come from configurations with the maximal number of hidden neurons (256), in particular for the PBMC and Brain Large datasets. Perhaps, better performance could be obtained by increasing the search space on this component.

We purposedly do not use these results to conclude on which type of conditional distribution for the count distribution (NB, ZINB or other; parameterized by `reconstruction_loss`). Surprisingly, another tuning process run on the PBMC dataset consistently selected the negative binomial loss in the ten best runs. This likely is a bias introduced by the exploration process during the optimization procedure. See our group's [blog post](https://yoseflab.github.io/2019/06/25/ZeroInflation/) by O. Clivio and P. Boyeau for a complete analysis of the conditional distribution choice. 

Overall, the best performances are stable even though we notice that small changes can have large impacts when we look at the full results (e.g, the number of layers can lead to a sensible difference in held-out log-likelihood). Such stability is in accordance with our extended use of scVI and scANVI in our harmonization preprint [Xu et al. (2019)](https://www.biorxiv.org/content/10.1101/532895v1), where we keep the hyperparameters fixed and obtain competitive performance with state-of-the-art methods. However, we recognize that hyperparameter tuning is an important practice in machine learning method developements and benchmarking and advocate the use of our autotune module for model selection and comparaisons.  

#### The case of the Brain large dataset


Given the amount of data available for this dataset, we were expecting to reach optimality with more hidden layers in our neural networks (as shallow networks with bounded width represent a potentially small function class). Suprisingly enough, single-hidden-layer models came out as the best contenders. In Figure 1, we investigate  in further details the relationship between our `n_layers` parameter and the observed performance in terms of marginal negative log-likelihood.

<p style="text-align:center;">
<img src="/assets/blog-post-hyperoptimization/n_layers_brain_large.png" width="80%" alt="Synthetic" />
</p>

Figure 1: Marginal negative log-likelihood as a function of the number of hidden layers in scVI's encoder/decoder, colored by the number of epochs. The number of epochs was limited to 50 or less if the early stopping criterion was triggered.

It is clear from Figure 1 that the shallow versions of scVI reach better performances on average. Still, a significant portion of hyperparameter configurations reach the maximum number of epochs. Potentially, allowing for a higher budget in terms of the number of epochs could change the outcome of our experiment. Additionally, the fact that we keep only the 720 most variable genes may be too big of a restriction on the amount of signal we keep, making this filtered version of the dataset simple enough to be better and faster captured by a single-hidden-layer model.


#### Benchmarking

Our next concern was to investigate the kind of performance uplift we could yield with auto-tuning. Namely, we investigated how optimizing log likelihood could increase other metrics like imputation for example. Our results are reported in the table below.

<center>
<table style="width:70%; font-size:80%" align="center">
  <thead>
    <tr>
      <th rowspan="2">Dataset</th>
      <th rowspan="2">Run</th>
      <th colspan="3" halign="left">Likelihood</th>
      <th colspan="2" halign="left">Imputation score</th>
    </tr>
    <tr>
      <th>Marginal ll</th>
      <th>ELBO train</th>
      <th>ELBO test</th>
      <th>Median</th>
      <th>Mean</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th rowspan="2" valign="top">cortex</th>
      <th>tuned</th>
      <td><strong>1218.52</strong></td>
      <td><strong>1178.52</strong></td>
      <td><strong>1231.16</strong></td>
      <td><strong>2.08155</strong></td>
      <td><strong>2.87502</strong></td>
    </tr>
    <tr>
      <th>default</th>
      <td>1256.03</td>
      <td>1224.18</td>
      <td>1274.02</td>
      <td>2.30317</td>
      <td>3.25738</td>
    </tr>
    <tr>
      <th rowspan="2" valign="top">pbmc</th>
      <th>tuned</th>
      <td><strong>1323.44</strong></td>
      <td>1314.07</td>
      <td><strong>1328.04</strong></td>
      <td><strong>0.83942</strong></td>
      <td><strong>0.924637</strong></td>
    </tr>
    <tr>
      <th>default</th>
      <td>1327.61</td>
      <td><strong>1309.9</strong></td>
      <td>1334.01</td>
      <td>0.840617</td>
      <td>0.925628</td>
    </tr>
        <tr>
      <th rowspan="2" valign="top">brain large</th>
      <th>tuned</th>
      <td><strong>1143.79</strong></td>
      <td><strong>1150.27</strong></td>
      <td><strong>1150.7</strong></td>
      <td><strong>1.03542</strong></td>
      <td>1.48743</td>
    </tr>
    <tr>
      <th>default</th>
      <td>1160.68</td>
      <td>1164.83</td>
      <td>1165.12</td>
      <td>1.04519</td>
      <td><strong>1.48591</strong></td>
    </tr>
  </tbody>
</table>
</center>


Table 5: Comparison between the best and default runs of scVI on the Cortex, Pbmc and Brain Large datasets. **Held-out** marginal log-likelihood and imputation benchmarks are reported.

Note that the median (resp. mean) imputation score is the median (resp. mean) of the median absolute error per cell. Also, the held-out marginal negative log-likelihood is an importance sampling estimate of the negative marginal log-likelihood $-\log p(x)$ computed on a 25% test set and is the metric we optimize for. 

From Table 5, we see that the tuning process improves the held-out marginal negative log-likelihood on all three datasets. This model improvement, in turn, ameliorates the imputation metrics for all three datasets, albeit not significantly for the Pbmc and Brain Large datasets.

#### Efficiency of the tuning process

Our last concern was to investigate the efficiency of the tuning process. To do so we wanted to check how the held-out negative Evidence Lower BOund (ELBO) histories of the runs were evolving with the number of runs. Below is a plot of these histories.


<p style="text-align:center;">
<img src="/assets/blog-post-hyperoptimization/elbo_histories_all.png" width="80%" alt="Synthetic" />
</p>

Figure 2: Held-out negative ELBO history for each of the 100 trainings performed for each dataset. The lines are colored from red to green where red represents the first trial and green the last one.

In Figure 2, we see that most green runs (the last trials) are concentrated in the low negative ELBO regions which indicates that the TPE algorithm provides relevant trial suggestions. This would suggest that allowing a slightly higher budget might yield better results. The user should consider this when setting the number of evaluations to run.



# Discussion

As we illustrated, hyper-parameter tuning provides a improvement for the data fit and also for any metrics that are correlated with the held-out marginal log-likelihood. It is possible that some other scores such as clustering might not be improved by this process since scVI is not per se separating any form of clusters in its latent space. A case where improvement on log-likelihood should be correlated with better clustering scores is for example when using [scANVI](https://www.biorxiv.org/content/10.1101/532895v1), which has a Gaussian mixture model prior over the latent space.

In this set of experiments, the hyperparameter selection procedure retained for all datasets an architecture with a unique hidden layer. To make sure that scVI benefits from non-linearity (compared to its homolog [ZINB-WaVE](https://www.nature.com/articles/s41467-017-02554-5)), we investigated whether hyperopt would select a model with one layer compared to a linear model. We report our results in [this table](https://docs.google.com/spreadsheets/d/15VenLjjFKrIq1QfSMieKZGkh95TynkIucvL4JH0xGHc/edit?usp=sharing) and conclude that a non-linear model sensibly outperforms the linear alternative on both of the Cortex and Pbmc datasets. Brain Large was not part of this experiment as it would have required a considerable amount of additional computing ressources. Besides, it is worth mentioning that some contemporary results tend to indicate that deep architectures are good approximators for gene expression measurements. For instance, [Chen et al. (2016)](https://academic.oup.com/bioinformatics/article/32/12/1832/1743989) compare linear regression to their deep neural network D-GEX on gene expression inference. They show architectures with three fully-connected layers outperform shallower architectures and linear regression by a significant margin. Such evidence could very well motivate a more in-depth study of scVI's optimization procedure. Perhaps, techniques to improve and accelerate deep neural-networks training could have a significant impact on the results we are able to reach. For example, between-layer connections such as those proposed in the DenseNet architecture [Huang et al. (2018)](https://arxiv.org/pdf/1608.06993.pdf) might lead a path towards future improvement. In the meantime, we also plan to release a patch to scVI allowing for assymetric encoders and decoders as well as variable number of hidden units for each layer of each of scVI's neural networks.

Finally, we have seen how sensible scVI can be with respect to its set of hyperparameters on certain datasets.  Thus, our new hyperparameter tuning feature should be used whenever possible - especially when trying to compare other methods to scVI. 

# Acknowledgements

We note Qiwen Hu and Casey Greene's remarks on [hyperparameter selection](https://www.worldscientific.com/doi/pdf/10.1142/9789813279827_0033?download=true&) as well as their [comment on Nature Methods](https://www.nature.com/articles/s41592-018-0230-9) and aknowledge their work being our main motivation for incorporating this new tool into scVI. We acknowledge Valentine Svensson for proposing the linear decoder and contributing to the scVI codebase. 
