# Compilador MiniLang 🔧

Un compilador didáctico completo para el lenguaje **MiniLang**, implementado en Python 3 puro. El proyecto cubre las cuatro fases clásicas del front-end de un compilador: **Análisis Léxico**, **Análisis Sintáctico**, **Análisis Semántico** y **Generación de Código Intermedio**.

---

## 📋 Tabla de Contenidos

1. [Descripción del Lenguaje](#descripción-del-lenguaje)
2. [Arquitectura](#arquitectura)
3. [Fase 1 – Analizador Léxico](#fase-1--análisis-léxico)
4. [Fase 2 – Analizador Sintáctico](#fase-2--análisis-sintáctico--ast)
5. [Fase 3 – Analizador Semántico](#fase-3--análisis-semántico)
6. [Fase 4 – Generador de Código](#fase-4--generación-de-código-intermedio)
7. [Manejo de Errores](#manejo-de-errores)
8. [Uso](#uso)
9. [Ejemplo](#ejemplo)

---

## Descripción del Lenguaje

MiniLang es un lenguaje pequeño, tipado estáticamente e imperativo que soporta:

| Característica | Sintaxis |
|---|---|
| Declaración de variable entera | `int x = 5;` |
| Asignación | `x = x + 1;` |
| Aritmética | `+  -  *  /` |
| Comparación | `==  <  >` |
| Condicional | `if (cond) { … } else { … }` |
| Salida por pantalla | `print(expr);` |
| Comentarios de línea | `// esto es ignorado` |

---

## Arquitectura

```
Código Fuente
    │
    ▼
┌─────────────┐     Lista[Token]
│    LEXER    │ ──────────────────►
└─────────────┘
                                  ┌──────────────┐     AST
                                  │    PARSER    │ ──────────►
                                  └──────────────┘
                                                           ┌──────────────────┐    Tabla de Símbolos
                                                           │ ANALIZ. SEMÁNTICO│ ──────────────────►
                                                           └──────────────────┘
                                                                                           ┌───────────────┐    TAC
                                                                                           │ GENER. CÓDIGO │ ────►
                                                                                           └───────────────┘
```

---

## Fase 1 – Análisis Léxico

**Archivo:** `compiler.py` → clase `Lexer`

El analizador léxico convierte el texto fuente en una secuencia plana de **tokens**. Cada token contiene:
- `type` — uno de los valores del enum `TT` (p. ej. `TT.KW_INT`, `TT.IDENT`, `TT.PLUS`)
- `value` — el texto reconocido (o un entero para `INT_LIT`)
- `line` / `col` — ubicación en el fuente para reportar errores

**Categorías de tokens soportadas:**

| Categoría | Ejemplos |
|---|---|
| Palabras reservadas | `int` `if` `else` `print` |
| Identificadores | `x`, `miVar`, `resultado_1` |
| Literales enteros | `0`, `42`, `1000` |
| Operadores | `+` `-` `*` `/` `=` `==` `<` `>` |
| Delimitadores | `;` `{` `}` `(` `)` |

El lexer utiliza un **escaneo manual carácter a carácter** en lugar de expresiones regulares, lo que proporciona un control total sobre el seguimiento de línea y columna. Omite silenciosamente espacios en blanco y comentarios de línea `//`.

---

## Fase 2 – Análisis Sintáctico / AST

**Archivo:** `compiler.py` → clase `Parser`, dataclasses de nodos AST

El parser implementa una estrategia de **Descenso Recursivo / LL(1)**. Cada regla gramatical se corresponde con un método Python (`_stmt`, `_expr`, `_factor`, etc.). La gramática completa es:

```
program     → stmt*  EOF
stmt        → var_decl | assign_stmt | if_stmt | print_stmt
var_decl    → 'int' IDENT ( '=' expr )? ';'
assign_stmt → IDENT '=' expr ';'
if_stmt     → 'if' '(' expr ')' block ( 'else' block )?
print_stmt  → 'print' '(' expr ')' ';'
block       → '{' stmt* '}'
expr        → comparison
comparison  → addition ( ( '==' | '<' | '>' ) addition )*
addition    → term ( ( '+' | '-' ) term )*
term        → factor ( ( '*' | '/' ) factor )*
factor      → NUMBER | IDENT | '(' expr ')'
```

La salida es un **Árbol de Sintaxis Abstracta (AST)** compuesto por dataclasses ligeras de Python (`Program`, `VarDecl`, `Assign`, `BinOp`, `IfStmt`, `PrintStmt`, `Number`, `Identifier`).

---

## Fase 3 – Análisis Semántico

**Archivo:** `compiler.py` → clases `SemanticAnalyser`, `SymbolTable`

El analizador semántico realiza un **recorrido recursivo único** del AST y ejecuta:

1. **Población de la tabla de símbolos** — cada declaración `int` registra el nombre y tipo de la variable.
2. **Detección de variables no declaradas** — cualquier referencia a un identificador que no esté en la tabla lanza un `SemanticError`.
3. **Detección de declaraciones duplicadas** — declarar la misma variable dos veces es un error.
4. **Propagación de tipos** — los nodos `BinOp` retornan `"int"` para operadores aritméticos y `"bool"` para operadores de comparación (`==`, `<`, `>`), habilitando una verificación de tipos rudimentaria.

---

## Fase 4 – Generación de Código Intermedio

**Archivo:** `compiler.py` → clase `CodeGenerator`

El generador de código produce **Código de Tres Direcciones (TAC)**, una representación intermedia común en compiladores donde cada instrucción tiene como máximo un operador y tres operandos (dos fuentes y un resultado). Ejemplos de instrucciones:

```
t0 = y * 2
t1 = x + t0
z = t1
PRINT z
IF_FALSE t2 GOTO L0
GOTO L1
LABEL L0
```

Las variables temporales (`t0`, `t1`, …) se generan automáticamente. Las etiquetas (`L0`, `L1`, …) se crean para las instrucciones de flujo de control (ramas `if`/`else`).

---

## Manejo de Errores

Las tres fases de análisis lanzan excepciones tipadas y descriptivas:

| Excepción | Fase | Mensaje de ejemplo |
|---|---|---|
| `LexerError` | Léxica | `[Lexer Error] Line 3, Col 7: Unknown character '@'` |
| `ParseError` | Sintáctica | `[Parse Error] Line 5, Col 1: Expected SEMI (got IDENT 'x')` |
| `SemanticError` | Semántica | `[Semantic Error] Line 8: Undeclared variable 'foo'` |

Cada mensaje de error incluye el **número de línea**, la **columna** (cuando aplica), la **clase de error** y una **descripción legible**.

---

## Uso

```bash
# Ejecutar sobre un archivo fuente
python compiler.py example.ml

# El compilador imprimirá:
#   === TOKENS ===
#   === AST ===
#   === SYMBOL TABLE ===
#   === THREE-ADDRESS CODE ===
```

También se puede usar la función `compile_source(source: str) -> dict` de forma programática:

```python
from compiler import compile_source

result = compile_source("int x = 5; print(x);")
print(result["tac"])      # Código de tres direcciones
print(result["symbols"])  # Entradas de la tabla de símbolos
print(result["errors"])   # Errores encontrados
```

---

## Ejemplo

**Entrada (`example.ml`):**
```c
int x = 10;
int y = 3;
int z = x + y * 2;
print(z);

if (x > y) {
    int result = x - y;
    print(result);
} else {
    print(0);
}
```

**TAC generado:**
```
x = 10
y = 3
t0 = y * 2
t1 = x + t0
z = t1
PRINT z
t2 = x > y
IF_FALSE t2 GOTO L0
t3 = x - y
result = t3
PRINT result
GOTO L1
LABEL L0
PRINT 0
LABEL L1
```

---

## Requisitos

- Python 3.8 o superior
- Sin dependencias externas — biblioteca estándar pura

---

## Licencia

MIT
