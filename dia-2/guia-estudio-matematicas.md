# Guía de Estudio — Matemáticas
## Álgebra Lineal: Matrices, Determinantes y Sistemas de Ecuaciones

**Nivel**: Primer año universitario / Bachillerato avanzado

---

## 1. Matrices

### ¿Qué es una Matriz?

Una **matriz** es un arreglo rectangular de números organizados en filas y columnas. Se denota con letras mayúsculas (A, B, C...).

Una matriz de **m filas** y **n columnas** se llama matriz de orden m×n (se lee "m por n").

```
        columna 1  columna 2  columna 3
fila 1  [  a₁₁       a₁₂       a₁₃  ]
fila 2  [  a₂₁       a₂₂       a₂₃  ]
```

Esta sería una matriz 2×3 (2 filas, 3 columnas).

### Tipos de Matrices

| Tipo | Definición | Ejemplo |
|---|---|---|
| **Cuadrada** | m = n (mismo número de filas y columnas) | 3×3 |
| **Fila** | Una sola fila (1×n) | [1  2  3] |
| **Columna** | Una sola columna (m×1) | [1; 2; 3] |
| **Nula** | Todos sus elementos son 0 | [[0,0],[0,0]] |
| **Identidad** | Diagonal principal = 1, resto = 0 | [[1,0],[0,1]] |
| **Diagonal** | Solo la diagonal tiene valores distintos de 0 | — |
| **Triangular superior** | Ceros debajo de la diagonal | — |
| **Triangular inferior** | Ceros encima de la diagonal | — |
| **Simétrica** | A = Aᵀ (igual a su transpuesta) | — |

### Operaciones con Matrices

#### Suma y Resta
Solo se pueden sumar matrices del **mismo orden**. Se suman elemento a elemento.

```
A = [[1, 2], [3, 4]]
B = [[5, 6], [7, 8]]

A + B = [[1+5, 2+6], [3+7, 4+8]] = [[6, 8], [10, 12]]
```

**Propiedades de la suma**: Conmutativa (A+B = B+A), Asociativa, Elemento neutro (matriz nula).

#### Multiplicación por un Escalar
Se multiplica cada elemento de la matriz por el escalar k.

```
3 × [[1, 2], [3, 4]] = [[3, 6], [9, 12]]
```

#### Multiplicación de Matrices
Para multiplicar A×B, el número de **columnas de A** debe ser igual al número de **filas de B**.

Si A es m×n y B es n×p → A×B es m×p.

**El elemento (i,j) del producto** es el producto escalar de la fila i de A por la columna j de B.

```
A = [[1, 2], [3, 4]]    B = [[5, 6], [7, 8]]

A×B = [[1×5+2×7, 1×6+2×8], [3×5+4×7, 3×6+4×8]]
    = [[5+14, 6+16], [15+28, 18+32]]
    = [[19, 22], [43, 50]]
```

**¡IMPORTANTE!**: La multiplicación de matrices **NO es conmutativa**. En general, A×B ≠ B×A.

#### Transpuesta de una Matriz
Se obtiene intercambiando filas por columnas. Si A es m×n, Aᵀ es n×m.

```
A = [[1, 2, 3], [4, 5, 6]]  →  Aᵀ = [[1, 4], [2, 5], [3, 6]]
```

---

## 2. Determinantes

El **determinante** es un número real asociado a una matriz cuadrada. Se denota det(A) o |A|.

### Determinante de una Matriz 2×2

```
|a  b|
|c  d|  = a×d - b×c
```

**Ejemplo**:
```
|3  2|
|1  4|  = 3×4 - 2×1 = 12 - 2 = 10
```

### Determinante de una Matriz 3×3 (Regla de Sarrus)

Para una matriz 3×3:
```
|a₁ b₁ c₁|
|a₂ b₂ c₂|
|a₃ b₃ c₃|
```

Se calcula como:
```
det = a₁(b₂c₃ - b₃c₂) - b₁(a₂c₃ - a₃c₂) + c₁(a₂b₃ - a₃b₂)
```

**Regla de Sarrus**: Escribe las dos primeras columnas a la derecha de la matriz, suma las diagonales hacia la derecha y resta las diagonales hacia la izquierda.

### Propiedades del Determinante

1. **det(Iₙ) = 1** (determinante de la identidad es 1)
2. **det(Aᵀ) = det(A)** (la transpuesta tiene el mismo determinante)
3. **det(A×B) = det(A) × det(B)**
4. Si se intercambian dos filas, el determinante cambia de signo.
5. Si una fila es múltiplo de otra, det(A) = 0.
6. Si una fila es todo ceros, det(A) = 0.
7. **Una matriz tiene inversa si y solo si det(A) ≠ 0** (se llama *matriz no singular*).

---

## 3. Matriz Inversa

La **inversa** de A (denotada A⁻¹) cumple: **A × A⁻¹ = A⁻¹ × A = I**

Solo existe si **det(A) ≠ 0**.

### Método para matrices 2×2:

Si A = [[a, b], [c, d]], entonces:

```
A⁻¹ = (1/det(A)) × [[d, -b], [-c, a]]
```

**Ejemplo**:
```
A = [[3, 2], [1, 4]]
det(A) = 3×4 - 2×1 = 10

A⁻¹ = (1/10) × [[4, -2], [-1, 3]] = [[0.4, -0.2], [-0.1, 0.3]]
```

**Verificación**: A × A⁻¹ debe dar la matriz identidad.

---

## 4. Sistemas de Ecuaciones Lineales

Un **sistema de ecuaciones lineales** es un conjunto de ecuaciones de primer grado con varias incógnitas.

### Formas de Representación

**Sistema de ecuaciones**:
```
2x + 3y - z = 4
x - y + 2z = 1
3x + 2y + z = 7
```

**Forma matricial**: A × X = B

```
[[2,  3, -1],    [x]     [4]
 [1, -1,  2], ×  [y]  =  [1]
 [3,  2,  1]]    [z]     [7]
```

### Tipos de Soluciones

- **Sistema Compatible Determinado (SCD)**: Solución única. det(A) ≠ 0.
- **Sistema Compatible Indeterminado (SCI)**: Infinitas soluciones. Ecuaciones redundantes.
- **Sistema Incompatible (SI)**: Sin solución. Ecuaciones contradictorias.

### Método de Gauss (Eliminación Gaussiana)

Se transforma la matriz aumentada [A|B] en una forma escalonada mediante operaciones elementales de filas:
1. Intercambio de dos filas.
2. Multiplicar una fila por un escalar ≠ 0.
3. Sumar a una fila un múltiplo de otra.

**Ejemplo**:
```
Sistema:
x + y + z = 6
2x - y + 3z = 14
x + 2y - z = 2

Matriz aumentada:
[1  1  1 | 6 ]
[2 -1  3 | 14]
[1  2 -1 | 2 ]

F2 → F2 - 2×F1:
[1  1  1 | 6 ]
[0 -3  1 | 2 ]
[1  2 -1 | 2 ]

F3 → F3 - F1:
[1  1  1 | 6 ]
[0 -3  1 | 2 ]
[0  1 -2 |-4 ]

Continuar hasta forma escalonada y resolver por sustitución regresiva.
Solución: x = 1, y = 2, z = 3
```

### Método de Cramer

Para sistemas n×n con det(A) ≠ 0:

**xᵢ = det(Aᵢ) / det(A)**

Donde Aᵢ es la matriz A con la columna i reemplazada por el vector B.

**Útil para sistemas pequeños (2×2, 3×3).**

---

## 5. Aplicaciones Prácticas

### Criptografía (Matrices)
Las matrices se usan para encriptar mensajes. Se multiplica el vector del mensaje por una matriz clave.

### Economía (Modelo de Leontief)
Los sistemas de ecuaciones describen las relaciones entre sectores económicos: cuánto insumo requiere cada industria de las otras.

### Gráficos Computacionales
Las transformaciones geométricas (rotaciones, traslaciones, escalados) se aplican mediante multiplicación de matrices.

### Análisis de Redes
Los grafos y redes (internet, redes sociales) se representan con matrices de adyacencia.

---

## Fórmulas para Memorizar

| Concepto | Fórmula |
|---|---|
| det(A) 2×2 | ad - bc |
| Inversa 2×2 | (1/det) × [[d,-b],[-c,a]] |
| Cramer | xᵢ = det(Aᵢ)/det(A) |
| Producto A×B elemento (i,j) | Σ aᵢₖ × bₖⱼ |

---

*Sube este archivo a NotebookLM y pregunta: "¿Cuándo no existe la inversa de una matriz?" o "Explícame el método de Gauss paso a paso con el ejemplo del documento."*
