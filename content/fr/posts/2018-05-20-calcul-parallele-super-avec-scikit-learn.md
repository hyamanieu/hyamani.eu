---
title: Calcul parallèle (super) avec scikit-learn
author: hyamanieu
date: 2018-05-20T15:18:41+00:00
slug: calcul-parallele-super-avec-scikit-learn
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
J'ai obtenu un accès au [supercalculateur de l'Université de Strasbourg][1], qui est parfait pour relever les défis d'apprentissage automatique où de nombreux modèles doivent être testés.

Avec scikit-learn, il est relativement facile de tirer parti des multiples cœurs d'un PC grâce à l'option `n_jobs`. Le calcul parallèle sur une machine distante utilisant le système de files d'attente Slurm est moins direct.

Je veux parvenir à ce qui suit :
* utiliser python 3 et pouvoir installer n'importe quelle bibliothèque
* exécuter des tâches en parallèle pour n'importe quelle fonction personnalisée

Après quelques recherches et essais, j'ai abouti à la solution suivante.

# Se connecter et préparer son environnement

Après votre inscription, vous obtiendrez un identifiant et un mot de passe. Si vous utilisez Ubuntu comme moi, la connexion en ssh est simple. Ouvrez un terminal et tapez

    ssh -Y username@hpc-login.u-strasbg.fr

suivi de votre mot de passe. Vous pouvez aussi utiliser `sshpass` si vous ne voulez pas saisir votre mot de passe en aveugle.

## Installer Anaconda

Anaconda est un gestionnaire de paquets python très utilisé. Il est préinstallé sur le HPC et peut être activé avec `load module python/Anaconda3`. Cependant, je n'ai pas réussi à ajouter de nouvelles bibliothèques avec la commande `conda install`. Heureusement, il peut être installé sans droits administrateur !
Faites simplement ceci dans le terminal :

```bash
wget "https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh"
bash Miniconda3-latest-Linux-x86_64.sh #installs in ~/conda
export PATH=$HOME/conda/bin:$PATH
conda install scikit-learn -y
```

vous pouvez ajouter n'importe quel paquet python avec `conda install`.

# Exécuter les tests en parallèle

## flux de travail

Mon flux de travail consiste à développer sur mon PC et à tester avec peu de données. Quand je veux utiliser l'ensemble complet de données d'entraînement (_train_), je peux tout transférer avec `scp`.

Pour les tests, vous pouvez créer un répertoire `test_parallel` sur votre PC comme sur le supercalculateur distant.

    mkdir test_parallel
    cd ./test_parallel

## Code Python

Nous allons utiliser le jeu de données « digits » fourni comme exemple dans scikit-learn. Nous n'allons pas chercher le modèle parfait, mais seulement montrer comment une fonction personnalisée de classification de ce jeu de données peut être testée avec plusieurs paramètres en parallèle.

Comme je l'ai dit plus haut, ce n'est pas aussi simple que d'appeler l'option `n_jobs` dans la plupart des classes sklearn. Nous devrons utiliser le paquet `ipyparallel` du projet ipython. N'oubliez pas de l'installer avec `conda install ipyparallel` !

Il exige que les fonctions puissent être [sérialisées (pickle)][2], nous devrons donc créer deux fichiers : l'un contenant votre script `script.py`, l'autre vos fonctions `some_funcs.py`.

_Remarque :_ à moins d'implémenter votre propre algorithme, utiliser les fonctions de `sklearn.model_selection` plutôt qu'une fonction personnalisée est certainement bien meilleur. Pour le débogage et les tests, des fonctions maison permettent d'écrire des informations dans des fichiers.

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

Veillez à importer les fonctions de joblib depuis `sklearn.external`, et non depuis joblib directement.

Si vous préférez utiliser `GridSearchCV` de sklearn plutôt qu'une fonction personnalisée (fortement conseillé), vous pouvez remplacer la fin par :

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

Et voilà !

## Exécuter le code en local

Testons-le d'abord sur votre propre PC. Vous devrez ouvrir 1+n terminaux, n étant le nombre de cœurs que vous voulez utiliser, et lancer :

```bash
ipcontroller --profile=ipy_profile
ipengine --profile=ipy_profile (*n)
```

Vous pouvez aussi lancer un cluster dans un seul terminal avec

```bash
ipcluster start --profile=ipy_profile &
```

Il démarrera automatiquement autant de moteurs qu'il y a de cœurs. Vous pouvez ensuite l'arrêter en tapant :

```bash
ipcluster stop --profile=ipy_profile
```

Vous trouverez d'autres options dans la [documentation][3]. Vous pouvez maintenant exécuter votre script python avec `python3 script.py`. Si vous avez utilisé `ipcontroller`+`ipengine`(s), vous devriez voir le contrôleur distribuer les tâches aux moteurs. Les fichiers journaux seront également créés.

## L'exécuter sur le HPC distant

La première chose à faire est de copier les deux fichiers python vers le HPC :

```bash
#run this on your local computer
scp *.py username@hpc-login.u-strasbg.fr:~/test_parallel
```

Vous devez maintenant créer un fichier bash pour démarrer les moteurs et votre script, et définir les ressources nécessaires dans l'en-tête pour Slurm.

```bash
#run this after ssh-ing into the HPC
cd ~/test_parallel
nano start_script.sh
```

Vous pouvez ensuite insérer :

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

Je n'ai alloué que 4 cœurs pour comparer les performances avec mon PC à ressources similaires.

Quittez maintenant (ctrl+X) et lancez-le :

```bash
sbatch start_script.sh script.py
```

Vous pouvez voir où il en est dans la file d'attente avec `queue -u username`. Cela ne devrait pas être long : 4 cœurs et 1 heure de calcul, c'est peu, la tâche devrait être priorisée. Le script devrait se terminer en quelques minutes seulement.

Vous pouvez maintenant comparer les résultats en lisant les fichiers `*.log` ! Une fois les tests terminés, je conseille d'utiliser `GridSearchCV` et de lancer bien plus de 12 paramètres ! Il y aura un seul fichier journal et, bien sûr, les résultats dans un `.csv`.

# Et ensuite

J'utilise actuellement le HPC Alsacalcul pour chercher une solution à l'un des défis de l'ENS, [jetez-y un œil][4] ! Je publierai bientôt ma première solution, ainsi que la façon dont j'ai combiné le travail sur le notebook jupyter et les calculs sur le HPC.

_quelques sources utilisées :_
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
