---
title: Paralleles (Super-)Computing mit scikit-learn
author: hyamanieu
date: 2018-05-20T15:18:41+00:00
slug: paralleles-super-computing-mit-scikit-learn
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
Ich habe Zugang zum [Supercomputer der Universität Straßburg][1] erhalten, ideal für Machine-Learning-Herausforderungen, bei denen viele Modelle durchprobiert werden müssen.

Mit scikit-learn ist es relativ einfach, die mehreren Kerne eines PCs über die Option `n_jobs` zu nutzen. Paralleles Rechnen auf einem entfernten Rechner mit dem Warteschlangensystem Slurm ist dagegen nicht ganz so einfach.

Ich möchte Folgendes erreichen:
* Python 3 verwenden und beliebige Bibliotheken installieren können
* Jobs für beliebige eigene Funktionen parallel ausführen

Mit etwas Recherche und Ausprobieren kam ich zu folgender Lösung.

# Verbinden und Einrichten der Umgebung

Nach der Registrierung erhalten Sie einen Benutzernamen und ein Passwort. Wenn Sie wie ich Ubuntu verwenden, ist die Verbindung per ssh einfach. Öffnen Sie ein Terminal und tippen Sie

    ssh -Y username@hpc-login.u-strasbg.fr

gefolgt von Ihrem Passwort. Sie können auch `sshpass` verwenden, wenn Sie Ihr Passwort nicht blind eintippen möchten.

## Anaconda installieren

Anaconda ist ein weit verbreiteter Python-Paketmanager. Er ist auf dem HPC vorinstalliert und kann mit `load module python/Anaconda3` aktiviert werden. Allerdings konnte ich keine neuen Bibliotheken mit dem Befehl `conda install` hinzufügen. Zum Glück lässt er sich auch ohne Admin-Rechte installieren!
Führen Sie einfach Folgendes im Terminal aus:

```bash
wget "https://repo.continuum.io/miniconda/Miniconda3-latest-Linux-x86_64.sh"
bash Miniconda3-latest-Linux-x86_64.sh #installs in ~/conda
export PATH=$HOME/conda/bin:$PATH
conda install scikit-learn -y
```

Sie können jedes benötigte Python-Paket mit `conda install` hinzufügen.

# Tests parallel ausführen

## Ablauf

Mein Ablauf: Entwicklung auf meinem PC, Tests mit kleinen Datenmengen. Wenn ich den vollständigen _Train_-Datensatz verwenden will, übertrage ich alles mit `scp`.

Zum Testen können Sie auf Ihrem PC und auf dem entfernten Supercomputer ein Verzeichnis `test_parallel` anlegen.

    mkdir test_parallel
    cd ./test_parallel

## Python-Code

Wir verwenden den Standard-Datensatz „digits“, der als Beispiel in scikit-learn enthalten ist. Wir versuchen nicht, das perfekte Modell zu finden, sondern zeigen nur, wie eine eigene Funktion zur Klassifikation dieses Datensatzes mit mehreren Parametern parallel getestet werden kann.

Wie schon gesagt ist es nicht so einfach wie die Option `n_jobs` in den meisten sklearn-Klassen. Wir brauchen das Paket `ipyparallel` aus dem IPython-Projekt. Vergessen Sie nicht, es mit `conda install ipyparallel` zu installieren!

Es setzt voraus, dass Funktionen [pickle-fähig][2] sind, daher brauchen wir zwei Dateien: eine mit Ihrem Skript `script.py` und eine mit Ihren Funktionen `some_funcs.py`.

_Hinweis:_ Solange Sie keinen eigenen Algorithmus implementieren, ist die Verwendung von Funktionen aus `sklearn.model_selection` statt einer eigenen Funktion sicherlich besser. Zum Debuggen und Testen ermöglichen selbst geschriebene Funktionen, Informationen in Dateien zu schreiben.

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

Importieren Sie die joblib-Funktionen unbedingt aus `sklearn.external`, nicht aus joblib selbst.

Wenn Sie lieber `GridSearchCV` von sklearn statt einer eigenen Funktion verwenden (dringend empfohlen), können Sie den letzten Teil ersetzen durch:

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

Und fertig!

## Den Code lokal ausführen

Testen wir es zuerst auf Ihrem eigenen PC. Sie müssen 1+n Terminals öffnen, wobei n die Anzahl der Kerne ist, die Sie nutzen möchten, und Folgendes ausführen:

```bash
ipcontroller --profile=ipy_profile
ipengine --profile=ipy_profile (*n)
```

Alternativ können Sie einen Cluster auch in einem einzigen Terminal starten mit

```bash
ipcluster start --profile=ipy_profile &
```

Er startet automatisch so viele Engines, wie Kerne vorhanden sind. Stoppen können Sie ihn mit:

```bash
ipcluster stop --profile=ipy_profile
```

Weitere Optionen finden Sie in der [Dokumentation][3]. Jetzt können Sie Ihr Python-Skript mit `python3 script.py` ausführen. Wenn Sie `ipcontroller`+`ipengine`(s) verwendet haben, sollten Sie sehen, wie der Controller die Jobs an die Engines verteilt. Außerdem werden die Logdateien angelegt.

## Auf dem entfernten HPC ausführen

Zuerst müssen Sie beide Python-Dateien auf den HPC kopieren:

```bash
#run this on your local computer
scp *.py username@hpc-login.u-strasbg.fr:~/test_parallel
```

Jetzt müssen Sie eine Bash-Datei erstellen, die die Engines und Ihr Skript startet, und die benötigten Ressourcen im Header für Slurm definieren.

```bash
#run this after ssh-ing into the HPC
cd ~/test_parallel
nano start_script.sh
```

Sie können dann Folgendes einfügen:

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

Ich habe nur 4 Kerne zugeteilt, um die Leistung mit meinem PC mit ähnlichen Ressourcen zu vergleichen.

Jetzt beenden (ctrl+X) und starten:

```bash
sbatch start_script.sh script.py
```

Mit `queue -u username` können Sie prüfen, wo der Job in der Warteschlange steht. Es sollte nicht lange dauern: 4 Kerne und 1 Stunde Rechenzeit sind recht wenig, der Job sollte bevorzugt behandelt werden. Das Skript sollte nur wenige Minuten dauern.

Jetzt können Sie die Ergebnisse vergleichen, indem Sie die `*.log`-Dateien lesen! Wenn Sie mit den Tests fertig sind, empfehle ich, `GridSearchCV` zu verwenden und deutlich mehr als nur 12 Parameter auszuprobieren! Es wird eine einzige Logdatei geben und natürlich die Ergebnisse in einer `.csv`.

# Weiter geht's

Zurzeit nutze ich den Alsacalcul-HPC, um eine Lösung für eine der ENS-Challenges zu finden: [schauen Sie vorbei][4]! Ich werde bald meine erste Lösung veröffentlichen, ebenso wie die Art, wie ich die Arbeit im Jupyter-Notebook mit Berechnungen auf dem HPC kombiniert habe.

_Quellen, die ich verwendet habe:_
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
