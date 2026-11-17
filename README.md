# Étude, détection et remédiation contre l'empoisonnement des données

Expérimentation d'une attaque par porte dérobée (*backdoor*) sur un classifieur d'images MNIST, puis détection des données empoisonnées et remédiation par réentraînement.

Le rapport technique et la présentation sont livrés séparément ; ce dépôt contient le code et tout ce qu'il faut pour reproduire les résultats.

## Contenu du dépôt

| Fichier | Rôle |
| --- | --- |
| `projet_backdoor_mnist.ipynb` | Notebook complet : code commenté, explications et résultats de l'exécution |
| `config.json` | Ensemble des paramètres des expériences |
| `requirements.txt` | Bibliothèques Python et versions utilisées |
| `README.md` | Ce fichier |

Le notebook peut être lu directement sur GitHub : toutes les sorties (tableaux, figures, matrices de confusion) sont conservées.

## Installation

### Option 1 — Google Colab (recommandée)

1. Ouvrir [Google Colab](https://colab.research.google.com) et importer `projet_backdoor_mnist.ipynb` (*Fichier → Importer un notebook*, onglet *GitHub* ou *Importer*).
2. Activer le GPU : *Exécution → Modifier le type d'exécution → GPU*.
3. PyTorch, torchvision, NumPy, scikit-learn et matplotlib sont préinstallés. ART est installé par la première cellule du notebook (`pip install adversarial-robustness-toolbox`).

### Option 2 — Machine locale

```bash
python -m venv venv
source venv/bin/activate          # Windows : venv\Scripts\activate
pip install -r requirements.txt
pip install jupyter
jupyter notebook projet_backdoor_mnist.ipynb
```

Sans GPU, le notebook fonctionne sur CPU, mais l'exécution complète est nettement plus longue.

## Configuration

Les paramètres sont définis dans les cellules du notebook et récapitulés dans `config.json`. Principaux réglages :

| Paramètre | Valeur | Section du notebook |
| --- | --- | --- |
| Graine aléatoire | 42 | 0 — Environnement |
| Époques / taille de lot / taux d'apprentissage | 5 / 128 / 0,001 | 1.6 |
| Classe source → classe cible | 7 → 1 | 2.1 |
| Taux d'empoisonnement | 10 % de la classe source | 2.1 |
| Déclencheur | Carré blanc 4×4, marge 1 px, coin inférieur droit | 2.1 |
| Seuils de détection | taille < 35 %, silhouette > 0,35, \|z\| > 3,5 | 3.2 à 3.10 |
| Nombre de groupes (K-means) | 2 (méthode initiale), 3 (vérification aux faibles taux) | 3.2, 5.6 |
| Taux étudiés | 1 %, 2 %, 5 %, 10 % | 5.2, 5.6 |

Pour tester une autre configuration (autre couple source/cible, autre taux, autre déclencheur), modifier les constantes correspondantes dans la section 2.1 puis réexécuter le notebook depuis le début.

## Exécution

Exécuter toutes les cellules dans l'ordre : *Exécution → Tout exécuter* (Colab) ou *Kernel → Restart & Run All* (Jupyter). L'ordre est important : chaque étape réutilise les variables des précédentes.

Durée indicative sur GPU Colab : environ 20 à 25 minutes, dont la majeure partie pour l'étude des taux (5.2) et la vérification à trois groupes (5.6 et 5.7), qui réentraînent chacune plusieurs modèles.

### Organisation du notebook

| Section | Contenu |
| --- | --- |
| 0 | Installation d'ART, graines, déterminisme GPU, versions des bibliothèques |
| 1.1 – 1.8 | **Étape 1** — Chargement et description de MNIST, séparation entraînement / validation / test, CNN, entraînement et évaluation du modèle de référence |
| 2.1 – 2.5 | **Étape 2** — Empoisonnement (déclencheur + changement d'étiquette), exemples avant / après, modèle compromis, accuracy et taux de succès de l'attaque (ASR) |
| 3.1 – 3.11 | **Étape 3** — Activations de la dernière couche cachée, activation clustering (critère de taille, puis taille + silhouette), ActivationDefence d'ART, signatures spectrales, comparaison des méthodes |
| 4.1 – 4.3 | **Étape 4** — Retrait des échantillons suspects, réentraînement, comparaison des trois modèles |
| 5.1 – 5.7 | **Étape 5** — Erreurs de détection, influence du taux d'empoisonnement, cause de l'échec aux faibles taux, autres réglages d'ART, vérification de la détection à trois groupes et remédiation associée |

### Fichiers produits pendant l'exécution

| Fichier | Contenu |
| --- | --- |
| `modele_reference.pt`, `modele_compromis.pt`, `modele_remediation.pt` | Poids des trois modèles |
| `indices_empoisonnes.npy`, `journal_empoisonnement.csv` | Trace des images empoisonnées (vérité terrain) |
| `suspects_detection.npz` | Images signalées par chaque méthode de détection |
| `indices_retires.npy` | Images retirées lors de la remédiation |
| `resultats_reference.json`, `resultats_compromis.json`, `resultats_remediation.json`, `resultats_taux.json` | Métriques de chaque étape |

Sur Colab, ces fichiers sont supprimés à la fin de la session : les télécharger depuis le panneau *Fichiers* si besoin.

## Reproductibilité

La graine est fixée à 42 pour Python, NumPy et PyTorch, et les options de déterminisme de PyTorch et cuDNN sont activées. Deux exécutions complètes dans le même environnement produisent donc les mêmes résultats. Avec d'autres versions de bibliothèques ou un autre GPU, de légères différences numériques restent possibles.

## Principaux résultats

| Modèle | Accuracy (test propre) | ASR (7 + déclencheur → 1) |
| --- | --- | --- |
| Référence | 99,02 % | 0,19 % |
| Compromis | 98,97 % | 99,61 % |
| Après remédiation | 98,92 % | 0,10 % |

- **Détection retenue** (activation clustering, critère taille + silhouette) : 560 images empoisonnées retrouvées sur 564, 1 faux positif (F1 = 0,996).
- **Influence du taux** : avec deux groupes, la détection échoue dès 5 % d'empoisonnement. Avec trois groupes, elle retrouve 280 images empoisonnées sur 282 à 5 %, sans fausse alerte ; après remédiation, l'ASR tombe à 0,39 % pour une accuracy de 98,65 %.
- **Limite** : en dessous de 5 %, le poison reste indétectable, alors que l'attaque fonctionne encore à environ 90 %.
