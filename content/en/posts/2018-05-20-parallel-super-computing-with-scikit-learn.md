---
title: Parallel (super) computing with scikit-learn
author: hyamanieu
date: 2018-05-20T15:18:41+00:00
slug: parallel-super-computing-with-scikit-learn
featureimage: /img/2018/05/logo_AlsaCalcul.png
translationKey: 2018-05-20-parallel-super-computing-with-scikit-learn
categories:
  - Computing
tags:
  - hpc
  - machine learning
  - parallel computing
  - scikit-learn

---
I was granted an access to the [super computer of the University of Strasbourg][1], which is perfect for tackling machine learning challenges where many models need to be tried.

Using scikit-learn, it is relatively easy to take advantage of the multiple cores of a PC with the `n_jobs` option. Parallel computing on a remote machine using the Slurm queuing system is not as straightforward.

I want to achieve the following:
* Use python 3 and be able to install any library
* Run jobs in parallel for any custom function

With a little search and testing, I came up with the following solution.

# Connecting and setting up your environment

After registering, you will obtain a username and a password. If you are using Ubuntu like me, connecting through ssh is easy. Open a terminal and type

    ssh -Y username@hpc-login.u-strasbg.fr

followed by your password. You can also use `sshpass` if you don’t want to type your password in blind mode.

## Installing Anaconda

Anaconda is a widely used python package manager. It is pre-installed on the HPC and could be activated with `load module python/Anaconda3`. However, I was unable to add new libraries using the `conda install` command. Luckily, it can be installed without admin rights!
Simply do the following in the terminal:

```bash
wget "https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh"
bash Miniconda3-latest-Linux-x86_64.sh #installs in ~/conda
export PATH=$HOME/conda/bin:$PATH
conda install scikit-learn -y
```

you can add any python package you need with `conda install`.

# Running tests in parallel

## workflow

My workflow consists in developping on my PC and testing with little data. When I want to use the whole _train_ data set, I can then transfer everything using `scp`.

For testing purpose, you can make a `test_parallel` directory on both your PC and the remote super computer.

    mkdir test_parallel
    cd ./test_parallel

## Python code

We are going to use the standard “digits” data set included as an example in scikit-learn. We will not try to optimize for the perfect model, but only show how a custom function to classify this data set can be tested with several parameters in parallel.

As I said earlier, it is not as straightforward as calling the option `n_jobs` in most sklearn classes. We will need to use the package `ipyparallel` from the ipython project. Don’t forget to install it using `conda install ipyparallel`!

It requires functions to be [pickle-able][2], so we will have to create two files: one containing your script `script.py` and one containing your functions `some_funcs.py`.

_Note:_ unless you implement your own algorithm, using functions from the `sklearn.model_selection` rather than a custom function is certainly much better. For debugging and testing purpose, home-made functions can let you write information into files.

### script.py

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
Run it with:
    python3 script.py -p ipy_profile
where ipy_profile is the name of the ipython profile.
@author: hyamanieu
"""
import argparse
import logging
import os
import sys
from sklearn.externals.joblib import Parallel, parallel_backend
from sklearn.externals.joblib import register_parallel_backend
from sklearn.externals.joblib import delayed
from sklearn.externals.joblib import cpu_count
from sklearn.datasets import load_digits
from sklearn.model_selection import train_test_split
from ipyparallel import Client
from ipyparallel.joblib import IPythonParallelBackend
import numpy as np
import datetime
#module in the same directory
from some_funcs import SVC_rbf

FILE_DIR = os.path.dirname(os.path.abspath(__file__))
sys.path.append(FILE_DIR)

#prepare the logger
parser = argparse.ArgumentParser()
parser.add_argument("-p", "--profile", default="ipy_profile",
                 help="Name of IPython profile to use")
args = parser.parse_args()
profile = args.profile
logging.basicConfig(filename=os.path.join(FILE_DIR,profile+'.log'),
                    filemode='w',
                    level=logging.DEBUG)
logging.info("number of CPUs found: {0}".format(cpu_count()))
logging.info("args.profile: {0}".format(profile))

#prepare the engines
c = Client(profile=profile)
#The following command will make sure that each engine is running in
# the right working directory to access the custom function(s).
c[:].map(os.chdir, [FILE_DIR]*len(c))
logging.info("c.ids :{0}".format(str(c.ids)))
bview = c.load_balanced_view()
register_parallel_backend('ipyparallel', 
                          lambda : IPythonParallelBackend(view=bview))

#Get data
digits = load_digits()
#prepare it for the custom function
X_train, X_test, y_train, y_test = train_test_split(digits.data,
                                                    digits.target,
                                                    test_size=0.3)
#some parameters to test in parallel
param_space = {
    'C': np.logspace(-6, 6, 12),
}

gamma = 0.1

with parallel_backend('ipyparallel'):
    scores = Parallel(n_jobs=len(c))(delayed(SVC_rbf)(C,
                      gamma,
                      X_train, 
                      X_test, 
                      y_train, 
                      y_test, 
                      profile)
                               for C in param_space['C'])
    #write down the scores in a file.
with open('scores_rbf_digits.csv', 'w') as f:
    f.write('C,gamma,MAE\n')
    f.write("\n".join(','.join(str(c) for c in l) for l in scores))
    f.write('\n')
```

Make sure to import joblib functions from `sklearn.external`, not from joblib itself.

If you prefer using the `GridSearchCV` from sklearn rather than a custom function (strongly advised), you can replace the last bit with:

```python
from sklearn.model_selection import GridSearchCV
svc_rbf = SVC(kernel='rbf',
              shrinking=False)
search = GridSearchCV(svc_rbf, 
                      param_space,
                      return_train_score=True,
                      n_jobs=len(c))
with parallel_backend('ipyparallel'):
    search.fit(X_train, y_train)
results = search.cv_results_
results = pd.DataFrame(results)
results.to_csv(os.path.join(FILE_DIR,'scores_rbf_digits.csv'))
```

### some_funcs.py

```python
import os
import datetime
from sklearn import metrics
from sklearn.svm import SVC
import numpy as np
now = datetime.datetime.now

FILE_DIR = os.path.dirname(os.path.abspath(__file__))

def SVC_rbf(C,gamma, X_train, X_test, y_train, y_test, profile):
    file_path = os.path.join(os.getcwd(),
                             '{0}_C{1:06}.log'.format(profile,C))
    #logging will not work from the HPC engines
    #need to write into a file manualy.
    with open(file_path, 'a+') as f:
        f.write('new task for C='+str(C)+' and gamma='+str(gamma)+'\n')

    t0 = now()
    with open(file_path, 'a+') as f:
        f.write('Start learning at {0}\n'.format(t0.isoformat()))
    svc_rbf_reg = SVC(kernel='rbf',
                       C=C,
                       gamma=gamma, 
                       shrinking=False)
    svc_rbf_reg.fit(X_train,y_train)
    predictions = svc_rbf_reg.predict(X_test)
    score = metrics.mean_absolute_error(y_test,predictions)

    ##uncomment if you want to make the function slower
    #with open(file_path, 'a+') as f:
    #    f.write('Compute complex matrix inverse')
    #    m = np.random.randn(10000,10000)
    #    np.linalg.inv(m)

    t1 = now()
    h = (t1-t0).total_seconds()//3600
    m = (t1-t0).total_seconds()//60 - h*60
    s = (t1-t0).total_seconds() -m*60 - h*60
    with open(file_path, 'a+') as f:
        f.write('Finished at {0} after '
                '{1}h {2}min {3:0.2f}s\n'.format(t1.isoformat(),
                                                 h,m,s))
        f.write('SVC rbf\n')
        f.write('C: {0}, gamma: {1}\n'.format(str(C),str(gamma)))
        f.write('MAE: {0}\n'.format(str(score)))

    return (C, gamma, score)
```

And voila!

## Run your code locally

Let’s first test it on your own PC. You’ll need to open 1+n terminals, where n is the number of cores you can to use and run:

```bash
ipcontroller --profile=ipy_profile
ipengine --profile=ipy_profile (*n)
```

Alternatively, you can also run a cluster in a single terminal with

```bash
ipcluster start --profile=ipy_profile &
```

It will automatically start as many engines as there are cores. You can then stop it by typing:

```bash
ipcluster stop --profile=ipy_profile
```

You can find other options in the [documentation][3]. You can now run your python script with `python3 script.py`. If you used `ipcontroller`+`ipengine`(s), you should see the controller dispatching the jobs to the engines. Also, the log files will be created.

## Run it on the remote HPC

The first thing you need to do is to copy both python files to the HPC:

```bash
#run this on your local computer
scp *.py username@hpc-login.u-strasbg.fr:~/test_parallel
```

Now you need to make a bash file to start the engines, your script, and define the needed resources in the header for Slurm.

```bash
#run this after ssh-ing into the HPC
cd ~/test_parallel
nano start_script.sh
```

You can then insert:

```bash
#! /bin/bash
#SBATCH -p public           #available partition, depends on your HPC 
#SBATCH -J ipy_engines      #job name
#SBATCH -n 4                # 4 cores, you can increase it
#SBATCH -N 1                # 1 node, you can increase it
#SBATCH -t 1:00:00         # Job is killed after 1h
#SBATCH --mail-type=END     # Sends an email when the job finishes
#SBATCH --mail-user=nickname@domain.com#your email address

#the following deletes all the ipython profiles older than 1 day.
find ~/.ipython/profile_job* -maxdepth 0 -type d -ctime +1 | xargs rm -r
#create a new ipython profile appended with the job id number
profile=job_${SLURM_JOB_ID}

echo "Creating profile_${profile}"
$HOME/conda/bin/ipython profile create ${profile}

$HOME/conda/bin/ipcontroller --ip="*" --profile=${profile} &
sleep 10

#srun: runs ipengine on each available core
srun $HOME/conda/bin/ipengine --profile=${profile} --location=$(hostname) &
sleep 25

echo "Launching job for script $1"
$HOME/conda/bin/python $1 -p ${profile}
```

I have allocated only 4 cores to compare the performance with my PC with similar resources.

Now exit (ctrl+X) and run it:

```bash
sbatch start_script.sh script.py
```

You can check where it is in the queue with `queue -u username`. It should not take that long: 4 cores and 1 hours computing time is rather little, it should be put on priority. The script should finish within a couple minutes only.

You can now compare the results on each by reading the `*.log` files! Once you’re done with testing, I would advise using `GridSearchCV` and run many more parameters than only 12! There will be a single log file and of course the results in a `.csv`.

# Next

I am currently using the Alsacalcul HPC to find a solution from one of the ENS challenges, [check them out][4]! I will post my first solution soon, as well as how I combined working on the jupyter notebook and ran calculations on the HPC.

_some sources I used:_
– _https://services-numeriques.unistra.fr/les-services-aux-usagers/hpc.html_
– _http://www.makeloft.org/2016/05/work-in-progress-parallel-ipython-from.html_
– _http://twiecki.github.io/blog/2014/02/24/ipython-nb-cluster/_
– _https://github.com/scikit-learn/scikit-learn/issues/7168 (ivdorelian 10 Aug 2016)_
– _https://stackoverflow.com/questions/38601026/easy-way-to-use-parallel-options-of-scikit-learn-functions-on-hpc_
– _image copyright owner: AlsaCalcul Services, http://alsacalcul.unistra.fr/_

[1]: http://alsacalcul.unistra.fr/en/about-us/
[2]: https://docs.python.org/3/library/pickle.html#what-can-be-pickled-and-unpickled
[3]: http://ipyparallel.readthedocs.io/en/latest/process.html#using-ipcluster
[4]: http://challengedata.ens.fr/
