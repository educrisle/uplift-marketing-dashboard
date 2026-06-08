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
