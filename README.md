# TP3 - Compilador del lenguaje Micro con Flex y Bison

Trabajo práctico de la materia **Sintaxis y Semántica de los Lenguajes** (UTN FRBA). Implementación de un compilador para el lenguaje **Micro** utilizando las herramientas **Flex** (analizador léxico) y **Bison** (analizador sintáctico), con rutinas semánticas básicas y manejo personalizado de errores.

---

## Tabla de contenidos

- [Análisis del problema](#análisis-del-problema)
- [Diseño de la solución](#diseño-de-la-solución)
  - [Analizador léxico (Flex)](#analizador-léxico-flex)
  - [Analizador sintáctico (Bison)](#analizador-sintáctico-bison)
  - [Rutinas semánticas](#rutinas-semánticas)
  - [Decisiones de diseño](#decisiones-de-diseño)
- [Manual de usuario](#manual-de-usuario)
- [Pruebas realizadas](#pruebas-realizadas)
- [Integrantes](#integrantes)
- [Bibliografía](#bibliografía)

---

## Análisis del problema

El objetivo del trabajo práctico es desarrollar un compilador para el lenguaje **Micro** utilizando Flex y Bison. El compilador debe realizar análisis léxico, sintáctico y semántico del código fuente, personalizar los mensajes de error e implementar al menos dos rutinas semánticas.

El lenguaje Micro incluye:

- Estructuras de control básicas (`inicio` / `fin`)
- Operaciones de entrada/salida (`leer` / `escribir`)
- Expresiones aritméticas con operadores aditivos (`+`, `-`)
- Asignaciones de variables
- Identificadores y constantes numéricas

Requisitos funcionales del compilador:

1. Reconocer los tokens del lenguaje mediante un analizador léxico.
2. Verificar la estructura gramatical mediante un analizador sintáctico.
3. Implementar semántica básica: tabla de símbolos y validaciones.
4. Generar mensajes de error personalizados (léxicos y sintácticos).
5. Producir una salida que refleje los análisis realizados.

---

## Diseño de la solución

### Analizador léxico (Flex)

Se definieron patrones regulares para cada categoría léxica:

```
DIGITO  [0-9]
LETRA   [a-zA-Z]
ID      {LETRA}({LETRA}|{DIGITO})*
CTE     {DIGITO}+
```

Tokens reconocidos:

- **Palabras reservadas:** `INICIO`, `FIN`, `LEER`, `ESCRIBIR`
- **Operadores:** `:=`, `+`, `-`
- **Delimitadores:** `;`, `,`, `(`, `)`
- **Identificadores y constantes numéricas**

Cuando aparece un carácter no válido, se genera un mensaje con número de línea y carácter problemático:

```c
. { fprintf(stderr, "Error léxico linea %d: '%s'\n", yylineno, yytext); 
    return ERRORLEX; }
```

Espacios en blanco y saltos de línea se ignoran automáticamente, manteniendo actualizado el contador de líneas para reportar errores con precisión.

### Analizador sintáctico (Bison)

La gramática cubre:

- Programa completo (`INICIO ... FIN`)
- Listas de sentencias
- Sentencias de asignación, lectura y escritura
- Expresiones aritméticas
- Elementos primarios (identificadores, constantes, expresiones entre paréntesis)

Se utilizó `%union` para definir distintos tipos de valores semánticos (cadenas para identificadores, enteros para constantes) y se asignaron números de línea para mejorar el reporte de errores.

```c
void yyerror(const char *s) {
    fprintf(stderr, "Error sintáctico linea %d: %s\n", yylineno, s);
}
```

### Rutinas semánticas

#### 1. Tabla de símbolos y declaración de identificadores

La tabla de símbolos (`TS`) almacena por cada identificador:

- Nombre
- Valor (para constantes y temporales)
- Bandera de inicialización (`0` = no inicializada, `1` = inicializada)

La rutina `declarar_id()` verifica si un identificador ya existe; si no, lo agrega e imprime la instrucción de declaración:

```c
void declarar_id(const char* id) {
    if (strlen(id) > 10) {
        fprintf(stderr, "Error semántico linea %d: identificador '%s' demasiado largo (max 10).\n",
                yylineno, id);
    }
    if (buscarTS(id) == -1) {
        int pos = insertarTS(id);
        printf("Declara %s, Entera\n", TS[pos].id);
    }
}
```

Además se valida la longitud máxima de identificadores (10 caracteres).

#### 2. Control de uso de identificadores no declarados

La rutina `usar_id()` verifica que un identificador esté declarado antes de usarse. Si no lo está, reporta error y lo declara implícitamente para permitir continuar el análisis:

```c
void usar_id(const char* id) {
    int pos = buscarTS(id);
    if (pos == -1) {
        fprintf(stderr, "Error semántico linea %d: uso de identificador no declarado '%s'.\n",
                yylineno, id);
        declarar_id(id); // declaración implícita para continuar
    }
}
```

#### Rutinas adicionales

- `procesar_cte()` — procesa constantes numéricas
- `generar_asigna()` — genera código para asignaciones
- `buscarTS()` / `insertarTS()` — operaciones sobre la tabla de símbolos

### Decisiones de diseño

1. **Tabla de símbolos global** — simplifica el acceso durante todo el análisis y facilita el seguimiento de variables.
2. **Recuperación de errores** — ante errores semánticos no se detiene la ejecución, sino que se declaran variables implícitamente. Esto permite detectar múltiples errores en una sola corrida.
3. **Validación de longitud de identificadores** — máximo 10 caracteres.
4. **Código de salida representativo** — en lugar de generar código ejecutable, se producen instrucciones de bajo nivel (`Declara`, `Almacena`, `Sumar`, etc) que reflejan las operaciones realizadas.
5. **Campo de inicialización** — se incorporó el campo `inicializada` para detectar a futuro usos de variables no inicializadas.
6. **Estructura modular** — responsabilidades separadas entre el analizador léxico (tokens), el sintáctico (estructura) y las rutinas semánticas (validaciones y generación de código).

---

## Manual de usuario

### Requisitos

- Sistema operativo Linux o Windows con WSL
- Compilador C (`gcc`)
- Herramientas **Flex** y **Bison** instaladas

El compilador genera:

1. Mensajes de error (léxicos, sintácticos o semánticos) en `stderr`.
2. Instrucciones de bajo nivel en `stdout`.
3. Al finalizar, el contenido de la tabla de símbolos.

### Formato de entrada

Archivos con extensión `.m` y código válido en lenguaje Micro. Ejemplo:

```
inicio
    a := 10;
    b := a + 5;
    leer(c);
    d := b + c;
    escribir(d);
fin
```

---

## Pruebas realizadas

### Prueba 1: Programa básico sin errores

```
inicio
    a := 10;
    b := 20;
    c := a + b;
    escribir(c);
fin
```

Verifica que el compilador reconozca la sintaxis básica, declare correctamente las variables y genere el código para asignación y operación aritmética.

### Prueba 2: Errores léxicos

```
inicio
    %a := 10;
    b := 2$0;
fin
```

Verifica el reporte de errores léxicos ante caracteres inválidos en identificadores y constantes.

### Prueba 3: Errores semánticos

```
inicio
    a := b + 10;
    leer(a, a);
    c := 20 + d;
fin
```

Verifica:

- Uso de variables no declaradas (`b`, `d`)
- Lectura repetida de la misma variable (`a`)
- Recuperación de errores semánticos para continuar el análisis

### Prueba 4: Programa completo

```
inicio
    leer(x, y);
    z := x + y;
    suma := z + 5;
    resta := z - 3;
    escribir(suma, resta);
fin
```

Verifica el funcionamiento integral del compilador: lectura de múltiples variables, operaciones aritméticas y escritura de resultados.

> Las capturas de pantalla de las pruebas se encuentran en el informe `TP3sic_-_Flex_y_Bison.pdf`.

---

## Integrantes

**Grupo 8 — Comisión K2102**

- Ignacio Farid Juri
- Gustavo Ekiel
- Juan Ignacio Galzerano
- Felipe Carosella
- Juan Cruz Marotti

**Docente:** Ing. Roxana Leituz
**Fecha de entrega:** 16/11/2025

---

## Bibliografía

- [Flex - The Fast Lexical Analyzer](https://github.com/westes/flex/)
- [GNU Bison Manual](https://www.gnu.org/software/bison/manual/)
- [SI413: Lexical analysis with flex - U.S. Naval Academy](https://www.usna.edu/Users/cs/wcbrown/courses/Su20SI413/lab/l04/lab.html)
- [Representing Code - Crafting Interpreters](https://craftinginterpreters.com/representing-code.html)