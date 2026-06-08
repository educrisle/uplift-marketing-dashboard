# Análisis de Rentabilidad de Campañas Promocionales
### Optimización del targeting mediante Uplift Modeling

---

## Contexto y problema de negocio

Las campañas de marketing masivo generan tres ineficiencias simultáneas que erosionan el margen de forma silenciosa:

- Se regala descuento a clientes que habrían comprado de todos modos.
- Se gasta en contactar a quienes no responderán nunca, independientemente de la oferta.
- No se distingue qué mecánica de promoción (Descuento directo vs. BOGO) es más efectiva para cada perfil de cliente.

La pregunta de negocio que motiva este proyecto: **¿a quién le cambia realmente la oferta la decisión de compra, y cuál de las dos ofertas genera mayor rentabilidad incremental?**

---

## Objetivo

Construir un sistema de scoring de persuadibilidad que puntúe a cada cliente según su probabilidad de ser persuadido por una oferta —es decir, que su compra ocurra *gracias* a la oferta y no habría ocurrido sin ella— para focalizar el presupuesto de campaña exclusivamente donde genera retorno real.

---

## Dataset

| Característica | Detalle |
|---|---|
| Registros | 64.000 clientes |
| Variables | 9 (7 features + 1 variable de tratamiento + 1 target) |
| Valores nulos | 0 — dataset completo sin depuración adicional |
| Tasa de conversión global | 14,68% |
| Gasto histórico medio | 441,98 € por cliente |

**Variables del dataset:**

| Variable | Tipo | Descripción |
|---|---|---|
| `recency` | int | Días desde la última compra |
| `history` | float | Gasto histórico acumulado en € |
| `used_discount` | bool | Uso previo de descuento directo |
| `used_bogo` | bool | Uso previo de promoción BOGO |
| `zip_code` | cat | Zona geográfica: Suburban / Rural / Urban |
| `is_referral` | bool | Si el cliente fue referido por otro cliente |
| `channel` | cat | Canal de contacto: Phone / Web / Multichannel |
| `offer` | cat | Tratamiento recibido: Discount / Buy One Get One / No Offer |
| `conversion` | bool | Variable objetivo: realizó una compra (1) o no (0) |

---

## Marco conceptual: los cuatro tipos de cliente

La clave del Uplift Modeling es distinguir entre clientes que convierten *gracias a* la oferta y los que lo hacen por otros motivos. Cruzando la variable de tratamiento con el resultado de conversión, cada cliente se clasifica en una de cuatro categorías causales:

| Clase | Nombre | Comportamiento | Acción recomendada | % estimado |
|---|---|---|---|---|
| TR | Persuadible | Solo compra si recibe la oferta | Contactar siempre | ~4,6% |
| CR | Comprador natural | Compra aunque no reciba la oferta | No ofertar (regala margen) | ~3,5% |
| TN | Insensible | Recibe la oferta y no compra | Excluir (coste sin retorno) | ~29% |
| CN | Inactivo | Sin oferta tampoco compra | Excluir | ~31% |

Solo el ~4,6% de la base (~2.900 clientes) son verdaderamente persuadibles. Focalizar la inversión en ese segmento es el objetivo central del proyecto.

---

## Metodología

### Preprocesamiento

- Renombrado y mapeo de la variable de tratamiento: `Buy One Get One → -1`, `No Offer → 0`, `Discount → 1`.
- One-hot encoding de variables categóricas (`zip_code`, `channel`).
- Normalización de variables numéricas (`recency`, `history`) con `StandardScaler`, aplicada dentro del pipeline para evitar data leakage.
- Split estratificado por variable de tratamiento (70/30) para mantener proporciones de tratamiento y control en train y test.

### Modelado

Se entrenaron **dos modelos XGBoost independientes** —uno para la oferta Discount y otro para BOGO— dado que cada mecánica atrae perfiles de cliente distintos. El enfoque es de clasificación multiclase (4 clases: CN, CR, TN, TR), donde la clase TR representa al cliente persuadible.

**Pipeline de entrenamiento:**
```
ColumnTransformer (StandardScaler | passthrough)
        ↓
XGBClassifier (objective: multi:softprob, num_class: 4)
```

**Ajuste de hiperparámetros:**
- `RandomizedSearchCV` con 30 iteraciones sobre 9 hiperparámetros (`n_estimators`, `max_depth`, `learning_rate`, `subsample`, `colsample_bytree`, `min_child_weight`, `gamma`, `reg_alpha`, `reg_lambda`).
- Métrica de optimización: `f1_macro` (balanceada para clases desiguales).
- Corrección del desbalanceo de clase con `scale_pos_weight` (ratio clase mayoritaria / clase TR).

**Validación:**
- `StratifiedKFold` con 5 splits, garantizando que los resultados son generalizables a campañas futuras.

### Evaluación por deciles y AUUC

Una vez generados los scores de persuadibilidad, la base se dividió en 10 deciles de igual tamaño. Para cada decil se midió el uplift real: diferencia de tasa de conversión entre clientes que recibieron la oferta y los que no, dentro del mismo grupo.

La métrica global de comparación entre modelos es el **AUUC (Area Under Uplift Curve)**, que resume la capacidad de ordenar correctamente a los persuadibles en toda la base.

### Explicabilidad con SHAP

Se aplicaron SHAP values sobre ambos modelos para identificar las variables de mayor peso en la persuadibilidad de cada cliente, haciendo los modelos interpretables para el equipo de negocio.

---

## Resultados

### KPIs del proyecto

| Métrica | Valor |
|---|---|
| ROI sobre grupo de control | **+38,25%** |
| Conversiones incrementales atribuibles a las ofertas | **2.599** |
| Contactos evitados (D8-D10, 30% de la base) | **19.200** |
| Clientes persuadibles estimados | **~2.900 (~4,6%)** |
| Ventas totales acumuladas | **9.394** |

### Comparativa de modelos

| Oferta | AUUC | Deciles rentables | Uplift en D1 | Veredicto |
|---|---|---|---|---|
| Discount | 236,87 | D1–D7 (70% de la base) | +7,84% | Ganadora global (+30% vs BOGO) |
| BOGO | 181,79 | D1–D6 (60% de la base) | +9,06% | Mayor intensidad en top segmento |

### Análisis por deciles

| Decil | Perfil | Uplift Discount | Uplift BOGO | Acción |
|---|---|---|---|---|
| D1 | Máxima persuadibilidad | +7,84% | +9,06% | Ambas ofertas |
| D2–D4 | Alta persuadibilidad | Positivo | Positivo | Ambas ofertas |
| D5–D6 | Persuadibilidad media | Positivo | Positivo | Ambas ofertas |
| D7 | Persuadibilidad baja | +1,29% | −0,94% | Solo Discount |
| D8–D10 | Sin persuadibilidad | Negativo | Negativo | No contactar |

### Estrategia óptima resultante

```
D1–D2  →  BOGO      (mayor intensidad de conversión donde más importa)
D3–D7  →  Discount  (cobertura amplia con ROI positivo)
D8–D10 →  Excluir   (ROI neto negativo — 19.200 clientes, 30% de la base)
```

---

## Entrega y puesta en producción

Los scores individuales de persuadibilidad de los 64.000 clientes fueron exportados a CSV (separador `;`, decimal `,`, encoding `utf-8-sig` para compatibilidad con Power BI en español) e integrados en un **dashboard interactivo de Power BI** para uso operativo del equipo de marketing.

El dashboard permite filtrar y priorizar audiencias por decil, oferta recomendada y score individual antes de cada campaña, sin necesidad de conocer el modelo.

🔗 **[Ver dashboard interactivo →](https://educrisle.github.io/uplift-marketing-dashboard/)**

---

## Stack tecnológico

| Categoría | Herramientas |
|---|---|
| Lenguaje | Python 3 |
| Modelado | XGBoost, Scikit-learn |
| Explicabilidad | SHAP |
| Análisis y preprocesamiento | Pandas, NumPy |
| Visualización | Matplotlib, Seaborn |
| Validación | StratifiedKFold, RandomizedSearchCV |
| Entrega | Power BI |

---

## Estructura del repositorio

```
uplift-marketing-dashboard/
├── README.md                        ← este archivo
├── Optimizacion_UPM.ipynb           ← notebook completo con todo el análisis
├── marketing_promotion_sample.csv   ← muestra representativa (2.000 registros)
├── Informe_rentabilidad_UPM.pdf     ← informe ejecutivo para negocio
├── index.html                       ← punto de entrada para GitHub Pages
└── uplift_dashboard.html            ← dashboard interactivo
```

---

## Próximos pasos

El análisis actual opera a nivel de oferta y decil de persuadibilidad. El siguiente paso recomendado es un **análisis de uplift segmentado por canal de contacto** (Teléfono / Web / Multicanal), que permitiría optimizar no solo qué oferta enviar, sino cómo contactar a cada cliente para maximizar el impacto, sin necesidad de aumentar el presupuesto.
