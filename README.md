# Prediccion de sinergia entre farmacos

Este repositorio contiene el codigo desarrollado para el Trabajo de Fin de Grado **"Evaluacion de modelos de aprendizaje profundo para la prediccion de sinergia entre farmacos"**. El objetivo del proyecto es evaluar modelos de aprendizaje automatico y aprendizaje profundo para predecir si una combinacion de dos farmacos presenta comportamiento sinergico en una linea celular tumoral.

El trabajo integra informacion de combinaciones farmacologicas, estructura molecular de farmacos y perfiles de expresion genica de lineas celulares. A partir de estas fuentes se construye un conjunto de datos supervisado y se comparan distintos modelos bajo protocolos de validacion con diferente grado de exigencia, prestando especial atencion a la generalizacion ante farmacos y estudios no vistos.

## Resumen del proyecto

La sinergia farmacologica se plantea como una tarea de clasificacion binaria. Cada muestra esta formada por:

- Farmaco A
- Farmaco B
- Linea celular
- Etiqueta binaria de sinergia basada en el indice Loewe

La variable objetivo se construye a partir de la puntuacion de sinergia Loewe:

- `1`: combinacion sinergica, si `synergy_loewe > 10`
- `0`: combinacion no sinergica o antagonista, si `synergy_loewe < -10`
- Las combinaciones intermedias, `-10 <= synergy_loewe <= 10`, se eliminan para reducir ambiguedad en las etiquetas.

El proyecto compara modelos clasicos y arquitecturas neuronales, evaluandolos tanto en particiones estratificadas como en escenarios Leave-Drug-Out.

## Fuentes de datos

El notebook trabaja con datos preprocesados procedentes de:

- **DrugComb**: combinaciones de farmacos, puntuaciones de sinergia, identificadores moleculares y SMILES.
- **DepMap**: perfiles de expresion genica de lineas celulares y tejido de origen.

Los datos esperados por el notebook se cargan desde:

```text
../data/preprocessed/drugs_info.pkl
../data/preprocessed/cell_expr_depmap.pkl
../data/preprocessed/df_synergy_expr.pkl
../data/preprocessed/cellline2tissue.json
```

Por motivos de tamano y trazabilidad, los datos no se incluyen necesariamente en el repositorio. Para reproducir el flujo completo es necesario disponer de estos archivos en la ruta indicada o adaptar las rutas de carga en el notebook.

## Representacion de las muestras

La representacion principal utilizada por la mayoria de modelos esta formada por:

- Morgan Fingerprint del primer farmaco.
- Morgan Fingerprint del segundo farmaco.
- Perfil de expresion genica reducido de la linea celular.

Los fingerprints moleculares se generan con **RDKit**, usando un radio de 2 y 1024 bits. Para la expresion genica se seleccionan los 500 genes con mayor varianza dentro del conjunto de entrenamiento de cada fold, evitando fuga de informacion hacia validacion o test.

Ademas, para el modelo basado en grafos se construyen grafos moleculares a partir de SMILES, donde:

- Los nodos representan atomos.
- Las aristas representan enlaces quimicos.
- Las caracteristicas atomicas incluyen tipo de elemento, grado, carga formal, aromaticidad, hibridacion y numero total de hidrogenos.

## Modelos evaluados

Se evaluaron los siguientes modelos:

- **Random Forest**
- **XGBoost**
- **Early Fusion MLP**
- **Multimodal MLP**
- **Drug-Pair Transformer MLP**
- **Graph Drug Synergy Model**

Los modelos clasicos se emplean como baselines robustos sobre la representacion vectorial concatenada. Las arquitecturas de aprendizaje profundo exploran distintas formas de integrar informacion molecular y transcriptomica: fusion temprana, ramas multimodales, atencion entre farmacos y representacion molecular mediante grafos.

## Protocolos de validacion

El proyecto compara tres estrategias principales:

### Stratified K-Fold

Validacion cruzada estratificada con 10 folds. Mantiene la proporcion de clases en cada particion, pero permite que los mismos farmacos aparezcan en entrenamiento y test.

### Leave-Drug-Out parcial

Los farmacos se dividen en grupos. En cada fold se evaluan combinaciones que contienen al menos un farmaco retenido. Este protocolo reduce el solapamiento de farmacos entre entrenamiento y test, aunque una misma muestra puede aparecer en mas de un fold si sus dos farmacos pertenecen a grupos retenidos distintos.

### Leave-Drug-Out disjunto

Variante LDO a nivel de muestra. Cada muestra se asigna a un unico fold de test, evitando repeticion de instancias durante la evaluacion.

## Evaluacion externa

Ademas de la validacion interna, se reserva un conjunto externo formado por estudios no utilizados durante el desarrollo del modelo. Este conjunto permite evaluar la generalizacion ante una distribucion distinta, con un porcentaje elevado de farmacos y lineas celulares no observados previamente.

Para cada protocolo interno se selecciona el modelo con mayor AUPRC media y se evalua sobre el conjunto externo completo. Las probabilidades finales se obtienen promediando las predicciones de los 10 modelos entrenados en los folds internos.

## Metricas

Debido al desbalanceo de clases, la metrica principal es:

- **AUPRC**: area bajo la curva Precision-Recall.

Tambien se calculan:

- ROC-AUC
- Precision
- Recall
- F1-score
- MCC

La AUPRC se compara con el valor basal definido como la proporcion de muestras positivas en el conjunto evaluado.

Para obtener etiquetas binarias a partir de probabilidades, se exploran dos criterios de umbral en validacion:

- Umbral que maximiza F1-score.
- Umbral que maximiza precision.

El umbral seleccionado en validacion se aplica posteriormente al conjunto de test, evitando ajustar decisiones sobre datos no vistos.

## Analisis post-hoc

El notebook incluye analisis adicionales para estudiar posibles sesgos y patrones de error:

- AUPRC por estudio.
- AUPRC por tejido.
- Errores por linea celular.
- Tasas de falsos positivos y falsos negativos por estudio.
- Dificultad de folds y rendimiento.
- Evaluacion sobre estudios externos retenidos.

Estos analisis ayudan a interpretar si el rendimiento global oculta diferencias asociadas a la composicion del conjunto de datos.

## Estructura esperada

Una estructura recomendada del repositorio es:

```text
.
├── README.md
├── notebooks/
│   └── SynergyDrugsProject.ipynb
├── data/
│   └── preprocessed/
│       ├── drugs_info.pkl
│       ├── cell_expr_depmap.pkl
│       ├── df_synergy_expr.pkl
│       └── cellline2tissue.json
├── results/
│   ├── shared_splits/
│   ├── random_forest_stratified_cv/
│   ├── xgboost_stratified_cv/
│   ├── triple_branch_mlp_stratified_cv/
│   ├── zzz_disjoint_random_forest/
│   └── analysis/
└── requirements.txt
```

Los directorios `data/` y `results/` pueden excluirse del repositorio si contienen archivos pesados o datos que no deben publicarse.

## Instalacion

Se recomienda utilizar un entorno virtual:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

En Windows:

```bash
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

Principales dependencias utilizadas:

- `numpy`
- `pandas`
- `scikit-learn`
- `xgboost`
- `torch`
- `torch-geometric`
- `rdkit`
- `matplotlib`
- `seaborn`
- `joblib`
- `tensorboard`


## Ejecucion

El flujo principal se encuentra en el notebook:

```text
SynergyDrugsProject.ipynb
```

Orden general de ejecucion:

1. Carga de datos preprocesados.
2. Preparacion de la variable objetivo basada en Loewe.
3. Filtrado de muestras y resolucion de duplicados.
4. Generacion de Morgan Fingerprints y grafos moleculares.
5. Construccion de particiones Stratified K-Fold y Leave-Drug-Out.
6. Entrenamiento de modelos.
7. Evaluacion interna y seleccion de umbrales.
8. Evaluacion externa sobre estudios retenidos.
9. Analisis post-hoc y generacion de figuras/tablas.

Los resultados se guardan en subdirectorios dentro de `results/`, incluyendo:

- metricas por fold;
- resumenes de test;
- mejores umbrales;
- predicciones;
- modelos entrenados;
- figuras y analisis post-hoc.

## Notas de reproducibilidad

- Las particiones se generan con semilla fija cuando procede.
- La seleccion de genes se realiza dentro de cada fold usando solo entrenamiento.
- El conjunto externo no se utiliza para entrenar, seleccionar modelos ni ajustar umbrales.
- Algunas arquitecturas neuronales pueden producir pequenas variaciones dependiendo de GPU, version de CUDA y configuracion de PyTorch.

## Resultados principales

De forma resumida, los resultados muestran que:

- Los modelos superan claramente el baseline AUPRC dentro del conjunto de desarrollo.
- El rendimiento es mayor en validacion estratificada que en escenarios Leave-Drug-Out.
- La generalizacion a farmacos no vistos es considerablemente mas dificil.
- Random Forest muestra un comportamiento competitivo en los protocolos LDO.
- La evaluacion externa sobre estudios retenidos revela una caida marcada de rendimiento, indicando dificultades de generalizacion ante distribuciones diferentes.
- El rendimiento no es homogeneo entre estudios, tejidos y lineas celulares.



