# UTEC Stream — Plataforma de streaming en C++

**Programación III (CS2013) · Proyecto Final 2026‑2 · UTEC**

Buscador de películas sobre el corpus *Wiki Movie Plots* (34 886 registros, 81 MB).
Permite buscar por palabra, frase o sub‑palabra, filtrar por etiquetas
(director, reparto, género, origen, año), marcar **Me gusta** y **Ver más tarde**,
y recibir recomendaciones basadas en similitud de contenido.

Todo el programa —lectura del CSV, pre‑procesamiento, estructuras de datos,
búsqueda y recomendación— está escrito en **C++17 sin dependencias externas**.

## Integrantes

| Nombre y apellidos | Código | Responsabilidad principal |
|---|---|---|
| *(completar)* | | Pre‑procesamiento y lector CSV |
| Alvaro Quispe | 202510375 | Tries e índice invertido |
| *(completar)* | | Ranking y recomendaciones |
| *(completar)* | | Interfaz e integración |

---

## 1. Cómo compilar y ejecutar

### Con CMake (recomendado)

```bash
git clone <url-del-repositorio>
cd <repositorio>
# Coloca el CSV en data/ (ver data/LEEME.md)
cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/streaming data/wiki_movie_plots_deduped.csv
```

### Con Make (alternativa sin CMake)

```bash
make -j
./streaming data/wiki_movie_plots_deduped.csv
```

> Para subir el proyecto a GitHub (requisito del enunciado), sigue
> `docs/repositorio-github.md`.

### Pruebas

```bash
make pruebas && ./pruebas        # 64 pruebas unitarias y de integración
# o con CMake:  cmake --build build --target pruebas && cd .. && ./build/pruebas
```

Argumentos opcionales:

```bash
./streaming [ruta_csv] [ruta_estado] [--sin-cache]
```

El estado del usuario se guarda por defecto en `datos_usuario.txt`.
La primera ejecución construye los árboles (~5 s) y escribe un caché binario
junto al CSV; a partir de la segunda el arranque baja a **1 s**. `--sin-cache`
fuerza la reconstrucción desde el CSV.

---

## 2. Uso

```
> ship                        búsqueda por palabra
> ghost ship                  búsqueda por frase
> shi                         búsqueda por sub-palabra
> director:hitchcock          búsqueda por etiqueta
> reparto:tom hanks
> genero:horror anio:1960     etiquetas combinadas
> m                           siguientes 5 coincidencias
> 3                           abre la ficha del resultado 3
> r                           mis recomendaciones
> i                           volver al inicio
> q                           salir
```

Dentro de una ficha: `l` = Me gusta, `v` = Ver más tarde, `s` = similares,
`Enter` = volver.

---

## 3. Justificación de la estructura de árbol

El enunciado exige un árbol cuyos nodos guarden caracteres y pide justificar la
elección. Usamos **dos tries** que se complementan.

### 3.1 Qué se descartó y por qué

| Alternativa | Por qué no |
|---|---|
| **Suffix tree / suffix array del texto completo** | El corpus tiene 75 MB de sinopsis y 13.3 M de tokens. Un suffix tree sobre el texto crudo necesitaría del orden de 10–20 bytes por carácter: **1–1.5 GB**, inviable en una laptop. |
| **Trie de prefijos únicamente** | Resuelve palabra y prefijo, pero no la búsqueda por sub‑cadena que pide el enunciado (`"bar"` dentro de `"bartender"`). |
| **Índice de k‑gramas (3‑gramas)** | Funciona, pero deja de ser un árbol de caracteres y falla con patrones de menos de *k* letras. |
| **Tabla hash del vocabulario** | O(1) para palabra exacta, pero no soporta prefijo ni sub‑cadena, y el enunciado pide un árbol. |

### 3.2 La decisión clave: indexar el **vocabulario**, no el texto

El texto tiene 13 353 866 tokens, pero solo **143 535 palabras distintas**
(1 074 389 caracteres). Insertar todos los sufijos del *vocabulario* cuesta
4 991 100 caracteres en el peor caso, unas **250 veces menos** que hacerlo sobre
el texto completo. El resultado medido son 947 941 nodos.

Se apoya en una equivalencia sencilla:

> Un patrón `P` aparece en un documento `D` **si y solo si** existe una palabra
> `T` del vocabulario que contiene a `P` y que además aparece en `D`.

Así, la búsqueda por sub‑cadena se descompone en dos pasos:
resolver `P → {términos}` en el árbol, y `término → {películas}` en el índice
invertido.

### 3.3 `TriePrefijos` — diccionario del vocabulario

Cada palabra distinta se inserta una vez y recibe un `idTermino`.
El nodo terminal guarda ese id. Resuelve:

* **palabra exacta** → descenso directo, **O(|palabra|)**;
* **prefijo** → descenso + recorrido del subárbol.

Este trie *es* el diccionario: no hay `unordered_map` de vocabulario en ninguna
parte del proyecto, ni siquiera durante la construcción del índice.

### 3.4 `TrieSufijos` — trie generalizado de sufijos del vocabulario

Para cada palabra `T` se insertan sus `|T|` sufijos y se marca cada nodo final
con el id de `T`. Entonces:

```
términos que contienen P  =  términos marcados en el subárbol al que se llega descendiendo P
```

porque `P` ocurre en `T` en la posición `i` ⟺ el sufijo `T[i..]` empieza con `P`.
Costo: **O(|P| + tamaño del subárbol)**.

### 3.5 Representación de los nodos

Los hijos son una **lista enlazada de hermanos**, no un arreglo indexado por
carácter:

```cpp
struct Nodo {
  int32_t primerHijo = -1;
  int32_t hermano    = -1;
  int32_t cabezaLista = -1;  // ó idTermino en el trie de prefijos
  uint8_t c = 0;             // el carácter del nodo
};                           // 16 bytes
```

Con un arreglo de 36 punteros por nodo, los ~1.33 M de nodos costarían más de
190 MB solo en punteros vacíos. Con lista de hermanos son **21 MB**. Como el
alfabeto normalizado es `[a-z0-9]` (36 símbolos), recorrer la lista es O(36) en
el peor caso y ~3 comparaciones en la práctica, porque la mayoría de los nodos
tiene pocos hijos.

Los nodos viven en un `std::vector` y se referencian por índice `int32_t` en
lugar de punteros: la mitad de memoria en 64 bits y mucha mejor localidad de
caché.

---

## 4. Arquitectura

```
CSV (81 MB)
   │  csv.cpp          lector RFC 4180 (comillas escapadas, CRLF incrustados)
   ▼
Catalogo               limpieza, deduplicación, separación de reparto/géneros
   │  catalogo.cpp
   ▼
Indice                 tokeniza → TriePrefijos (ids) → listas de publicaciones
   │  indice.cpp       y luego llena el TrieSufijos con el vocabulario final
   ▼
Buscador               patrón → expansión en los tries → BM25F → re-ranking
   │  buscador.cpp
   ▼
Recomendador           perfil tf-idf del usuario → similitud del coseno
   │  recomendador.cpp
   ▼
main.cpp               interfaz de consola + EstadoUsuario (persistencia)
```

| Archivo | Responsabilidad |
|---|---|
| `normalizador.*` | Plegado UTF‑8→ASCII, limpieza de referencias `[12]`, tokenización, stopwords |
| `csv.*` | Lector CSV conforme a RFC 4180 |
| `catalogo.*` | Modelo `Pelicula` y todas las reglas de pre‑procesamiento |
| `trie.*` | `TriePrefijos` y `TrieSufijos` |
| `indice.*` | Índice invertido con campos, longitudes de documento, perfiles tf‑idf |
| `buscador.*` | Parseo de consultas, expansión léxica, BM25F, re‑ranking por frase |
| `recomendador.*` | Similares a una película y recomendaciones por perfil |
| `binario.hpp` | Serialización binaria en bloque para el caché |
| `estado_usuario.*` | Me gusta / Ver más tarde con persistencia en disco |
| `main.cpp` | Interfaz de consola |

---

## 5. Pre‑procesamiento

El CSV crudo trae problemas que rompen cualquier parser ingenuo. Estas son las
reglas aplicadas y lo que descartó cada una en la corrida real:

| Problema en el dato | Regla aplicada | Registros afectados |
|---|---|---|
| Comillas escapadas `""` y saltos `\r\n` dentro de `Plot` | Lector RFC 4180 real (no `split(',')` ni `getline`) | 23 425 sinopsis contenían CRLF |
| Referencias de Wikipedia `[1]`, `[cita requerida]` | Se eliminan en `limpiarTextoBruto` | 4 125 sinopsis |
| Tildes y diacríticos (`Fantôme`, `Ríos`) | Plegado UTF‑8 → ASCII, para que `capitan` encuentre `capitán` | 9 411 sinopsis, 417 títulos |
| `Director = "Unknown"` | Se traduce a campo vacío en vez de indexar la palabra "unknown" | 1 132 |
| `Genre = "unknown"` | Igual que arriba | 6 107 |
| `Cast` vacío (`NaN`) | Reparto vacío | 1 450 |
| `Cast` con separadores mixtos (`,`, `and`, `/`) | Se homogeneizan a coma y se separa | — |
| Título vacío | Se descarta la fila | 1 |
| Sinopsis de menos de 40 caracteres | Se descarta la fila | 30 |
| Filas duplicadas (mismo título + año + director) | Se descartan | 41 |

**34 814 películas útiles** de 34 886 filas leídas.

Nota sobre el apóstrofe: `don't` se normaliza como `dont` y no como `don` + `t`,
para no ensuciar el vocabulario con miles de tokens `t` y `s`.

---

## 6. Algoritmo de importancia (ranking)

El enunciado pide "un algoritmo para determinar qué película tiene más
importancia en una búsqueda". El nuestro tiene cuatro componentes.

### 6.1 Expansión léxica del patrón

Cada término de la consulta se resuelve contra los dos árboles y produce un
conjunto de términos del vocabulario, cada uno con un peso:

* **coincidencia exacta** (trie de prefijos): peso léxico `1.0`;
* **coincidencia por sub‑cadena** (trie de sufijos): peso
  `0.85 · (|P| / |T|)²`.

El cuadrado penaliza fuerte las coincidencias diluidas: para `P = "bar"`,
`T = "bar"` pesa 0.85 y `T = "bartender"` pesa 0.094.

### 6.2 Corrección del IDF en coincidencias parciales

Este fue el error más interesante que encontramos al probar. Buscando `ship`,
el top‑1 era una película cuyo único vínculo era un actor apellidado **Shipp**.
La causa: `shipp` aparece en 1 de 34 814 películas, así que su IDF es ~10, casi
cuatro veces el de `ship` (~2.9), y ese factor se comía la penalización léxica.

La corrección: **una coincidencia parcial no puede reclamar el IDF de su término
aislado**. Todas las expansiones de un patrón son *un solo concepto*, así que se
acota su IDF por el de la unión:

```
df_unión = Σ df(T)  sobre las expansiones
idf_tope = log(1 + (N − df_unión + 0.5) / (df_unión + 0.5))
idf_efectivo(T) = min( idf(T), idf_tope )
```

El patrón `ship` aparece en miles de películas (`relationship`, `friendship`,
`worship`…), así que discrimina poco y su tope es bajo. La coincidencia exacta
conserva su IDF completo. Después del arreglo, el top‑5 de `ship` son
*Flying Phantom Ship*, *Sail a Crooked Ship*, *The Devil‑Ship Pirates*,
*Ship Ahoy* y *Hell Ship Mutiny*.

### 6.3 BM25 con campos (BM25F)

Cada película es un documento con seis campos. La frecuencia efectiva pondera
dónde apareció el término:

```
tf' = 6·tf_título + 4·tf_etiquetas + 1·tf_sinopsis
```

y se aplica la saturación de BM25 (`k₁ = 1.2`, `b = 0.6`):

```
puntaje(T,D) = idf_efectivo(T) · tf'·(k₁+1) / (tf' + k₁·(1 − b + b·|D|/media))
```

BM25 y no TF‑IDF simple por dos razones: la **saturación** evita que una sinopsis
que repite `ship` veinte veces aplaste a un título que se llama *Ghost Ship*, y
la **normalización por longitud** compensa que las sinopsis van de 40 a 36 773
caracteres.

Cuando un término de consulta se expande a varios términos del vocabulario, el
aporte del término es el **máximo** entre sus expansiones, no la suma: si no,
`bar` sumaría cuatro veces en un documento que contiene *bar*, *barn*, *barber*
y *barrel*.

### 6.4 Cobertura y bonus de frase

El enunciado pide semántica **"y/o"**: `"barco fantasma"` debe devolver
películas con `barco` **y/o** `fantasma`. Devolvemos la unión, pero las que
cumplen más términos suben:

```
puntaje_final = Σ_T puntaje(T,D) · (1 + 0.60 · términos_cubiertos / términos_totales)
```

Además, sobre los 400 mejores candidatos se hace un **re‑ranking por frase
literal**: `+12` si la frase completa está en el título, `+3.5` si está en la
sinopsis. Solo sobre esa ventana, porque exige normalizar la sinopsis y no vale
la pena hacerlo sobre las 34 814 películas.

El resultado para `ghost ship` son las tres películas tituladas *Ghost Ship*
primero, y recién después las coincidencias de una sola palabra.

Cada resultado muestra **por qué** salió (`frase exacta en el título`,
`2/3 en el título`, `1/1 en sinopsis/etiquetas`), lo que facilita defender el
ranking en la exposición.

---

## 7. Algoritmo de recomendación

### 7.1 Perfil de cada película

Al construir el índice se calcula, para cada película, su vector de los **40
términos de mayor peso tf‑idf**, con las etiquetas pesando más que la sinopsis
(`tf = 1 + 2·tf_título + 1·tf_sinopsis + 3·tf_etiquetas`). Se descartan
stopwords, palabras de menos de 3 letras y términos presentes en más del 12.5 %
del catálogo. Quedarse con 40 términos en vez del vector completo reduce la
memoria de gigabytes a ~11 MB y mejora la calidad: los términos de cola son ruido.

### 7.2 Perfil del usuario

Es el **centroide** de los vectores de las películas con Me gusta, con las más
recientes pesando algo más (de 0.6 a 1.0) para que el gusto actual mande sobre
el histórico.

### 7.3 Vecinos más cercanos

Similitud del **coseno**. Para no comparar contra las 34 814 películas, se
recorren las listas de publicaciones de los 80 términos más fuertes del perfil y
se acumula el producto punto solo sobre los documentos que **comparten al menos
un término**. Los términos presentes en más del 5 % del catálogo se saltan: no
discriminan y multiplicarían el trabajo.

Se excluyen las películas ya marcadas con Me gusta o en Ver más tarde, y cada
recomendación se explica con el título que más la motivó
(`porque te gustó "Destroy All Monsters"`).

**Prueba real:** con Me gusta en *Ghost Ship* (1952) y *Destroy All Monsters*
(1968), las recomendaciones fueron *Ghidorah, the Three‑Headed Monster*,
*Invasion of Astro‑Monster*, *Son of Godzilla*, *Mothra vs. Godzilla* y
*Godzilla, Mothra & King Ghidorah*.

---

## 7bis. Caché binario del índice

Construir todo desde el CSV cuesta ~4.9 s de CPU: 1.6 s de parseo y limpieza y
3.3 s de construcción de árboles, índice y perfiles. Para una demo en la que se
arranca el programa varias veces, eso se nota.

La primera ejecución vuelca a `<ruta_csv>.cache` el catálogo ya limpio y todas
las estructuras ya construidas: los dos tries, las listas de publicaciones, las
longitudes de documento y los perfiles tf‑idf. Las siguientes ejecuciones son
una lectura secuencial de disco.

**Validación.** La cabecera guarda una magia, un número de versión de formato,
el tamaño del CSV de origen y `sizeof(Publicacion)`. Si el CSV cambia, si se
modifica el formato o si el proyecto se recompila con otro compilador que altere
el padding de las estructuras, el caché se descarta y se reconstruye solo. No
hay forma de quedarse con un índice desactualizado sin darse cuenta.

**Implementación.** `binario.hpp` acumula la salida en un `std::string` y vuelca
el archivo de una vez; al leer carga el archivo completo y avanza sobre el
buffer. Con 201 MB, eso es mucho más rápido que miles de llamadas pequeñas a
`ostream`. Los vectores de tipos POD se copian en bloque con `memcpy`.

Un detalle que obligó a cambiar el código: `std::pair<int32_t,float>` **no es
trivialmente copiable** en libstdc++, porque su operador de asignación está
definido por el usuario. Volcarlo en bloque es comportamiento indefinido, así
que los perfiles pasaron a usar un POD propio, `TerminoPeso`.

**El costo.** El caché cambia 3.6 s de CPU por 201 MB de disco y 158 MB más de
memoria pico, porque el archivo se carga entero antes de repartirlo entre las
estructuras. Es un intercambio deliberado y reversible con `--sin-cache`.

| | Sin caché | Con caché |
|---|---|---|
| Arranque (CPU) | 4.89 s | **1.33 s** |
| Memoria residente pico | 268 MB | 426 MB |
| Disco adicional | 0 | 201 MB |

Una prueba automatizada verifica que el ranking obtenido desde el caché es
idéntico al obtenido reconstruyendo desde el CSV, y que una huella distinta
invalida el archivo.

---

## 8. Complejidad

| Operación | Complejidad | Nota |
|---|---|---|
| Insertar palabra en `TriePrefijos` | O(\|w\|·σ) | σ = 36, en la práctica ~3 |
| Buscar palabra exacta | **O(\|w\|)** | independiente del tamaño del corpus |
| Buscar por prefijo | O(\|P\| + nodos del subárbol) | |
| Insertar palabra en `TrieSufijos` | O(\|w\|²) | \|w\| ≤ 24 por construcción |
| Buscar por sub‑cadena | O(\|P\| + nodos del subárbol) | acotado a 3 000 términos |
| Consulta completa | O(Σ_T \|publicaciones(T)\| + k log k) | k = 400 candidatos reordenados |
| Recomendación | O(80 · \|publicaciones\| + m log m) | m = candidatos con términos en común |
| Construcción del índice | O(total de tokens) | una pasada |

---

## 9. Resultados medidos

Medido con `g++ 13.3 -O2`, un solo núcleo, dataset completo:

| Métrica | Valor |
|---|---|
| Filas leídas | 34 886 |
| Películas útiles tras la limpieza | 34 814 |
| Vocabulario | 164 015 palabras distintas |
| Publicaciones (pares término–película) | 6 839 286 |
| Nodos del trie de prefijos | 386 145 |
| Nodos del trie de sufijos | 947 941 |
| Memoria del índice | ~135 MB |
| Memoria residente pico del proceso | 268 MB (426 MB con caché) |
| Tiempo de carga y limpieza del CSV | 1.6 s |
| Tiempo de construcción de árboles e índice | 3.3 s |
| Arranque total desde el CSV (CPU) | 4.89 s |
| Arranque desde el caché binario (CPU) | **1.33 s** |
| Latencia de consulta | **0.1 – 10.7 ms** |

Latencias por tipo de consulta:

| Consulta | Tiempo |
|---|---|
| `director:hitchcock` | 0.1 ms |
| `genero:horror anio:1960` | 0.1 ms |
| `ship` (palabra) | 0.8 ms |
| `bar` (sub‑palabra) | 0.9 ms |
| `shi` (sub‑palabra corta) | 2.0 ms |
| `reparto:tom hanks` | 6.0 ms |
| `submarine escape prison` (3 términos) | 8.1 ms |
| `ghost ship` (frase) | 8.6 ms |

> El vocabulario del índice (164 015) es mayor que el de un conteo solo sobre
> título y sinopsis (143 535) porque el índice incorpora además los nombres de
> directores, del reparto, los géneros y los orígenes.

---

## 10. Cobertura de los requisitos del enunciado

| Requisito | Dónde | Estado |
|---|---|---|
| Leer la base de datos en `.csv` | `csv.cpp` | ✅ |
| Pre‑procesamiento a cargo del grupo | `catalogo.cpp`, `normalizador.cpp` | ✅ §5 |
| Árbol con caracteres en los nodos | `trie.hpp` | ✅ §3 |
| Elección del árbol justificada y documentada | README §3 | ✅ |
| Búsqueda por palabra | `buscador.cpp` | ✅ |
| Búsqueda por frase (semántica y/o) | bonus de cobertura, §6.4 | ✅ |
| Búsqueda por sub‑palabra | `TrieSufijos`, §3.4 | ✅ |
| Búsqueda por tag (director, casting, género…) | máscara de campos, §6.3 | ✅ |
| Cinco resultados + opción de ver los siguientes cinco | comando `m` | ✅ |
| Algoritmo propio de importancia | §6 | ✅ |
| Ver sinopsis + Like + Ver más tarde | ficha de película | ✅ |
| Al iniciar: Ver más tarde + similares a los Like | pantalla de inicio | ✅ |
| Todo el programa en C++ | C++17, sin dependencias | ✅ |
| Documentación en el repositorio | este README + `docs/` | ✅ |
| Subir el programa a un repositorio en GitHub | `docs/repositorio-github.md` | ✅ |
| Pruebas automatizadas | `tests/pruebas.cpp` (64 pruebas) | ✅ |

---

## 11. Limitaciones conocidas

* **Sin índice posicional.** El bonus de frase busca la cadena literal en el
  texto normalizado del top‑400, así que una frase que solo aparezca en la
  película 401 no recibe el bonus. Un índice posicional lo resolvería a costa de
  ~3× más memoria.
* **La expansión por sub‑cadena está topada en 3 000 términos.** Con patrones de
  2 letras muy comunes el recorrido se corta; los términos descartados son los
  más largos, es decir, los de menor peso léxico.
* **Sin stemming.** `ship` y `ships` son términos distintos; la búsqueda por
  sub‑cadena los conecta parcialmente, pero un stemmer de Porter mejoraría el
  recall.
* **El caché ocupa 201 MB** y se carga entero en memoria antes de repartirse,
  lo que sube el pico de RSS a 426 MB. Un `mmap` del archivo evitaría la copia.
* **Corpus en inglés.** Las stopwords y los ejemplos son en inglés; buscar
  `barco` devuelve casi nada porque el dataset no está en español.
