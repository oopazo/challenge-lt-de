## Configuración

La siguiente configuración ha sido probada en un entorno Linux:
El entorno virtual ha sido configurado para trabajar con Python 3.12.3

1. Clonar el repositorio

```
git clone https://github.com/oopazo/challenge-lt-de.git
```

2. Configuración de entorno virtual

```
# mover a folder del proyecto
cd challenge-lt-de

# copiar datos de prueba suministrados a folder data/ 
# Por razones de confidencialidad he decidido NO subirlos al repositorio público.
mkdir data
cp path_to_test_data/contrato_latam_fakesoft.md data/
cp path_to_test_data/contrato_latam_ficticia.md data/
cp path_to_test_data/contrato_latam_ficticia_anexo_b_especificaciones_tecnicas.md data/

# creación de entorno virtual
python3 -m venv .venv
source .venv/bin/activate

# Configuración de api key OpenAI. Agregar tu key al archivo .env
touch .env
OPENAI_API_KEY=sk-..

# instalar dependencias
pip install -r requirements.txt

# Ejecutar proyecto
jupyter-lab
```

##### Nota-1: para ejecutar correctamente el proyecto, se debe ejecutar en primer lugar el notebook *utils.ipynb*, puesto que es dependencia para todos los demás notebooks.

##### Nota-2: se han incluido las colecciones de chroma utilizadas durante el desarrollo de este challenge en el repositorio. Esto se traduce en que los embeddings no serán creados si no que serán cargados desde estas colecciones. Si deseas recrear estas colecciones, puedes eliminar o mover el folder chroma_index:.

```
# opción 1:
mv chroma_index chroma_index_backup
# opción 2:
rm -fr chroma_index
```


### Elecciones técnicas

Para el desarrollo de este challenge he realizado las siguientes elecciones técnicas:

- **Llamaindex** para parsing/chunking.
- **OpenAI gpt-4.1-nano** como llm.
- **text-embedding-3-small** como embedding model.
- **Chromadb** como base de datos vectorial.
- **Jupyterlab** para implementación de funcionalidades.

Las justificaciones de la elección se basan en los siguientes criterios:

1. Simplicidad de uso.
2. Documentación disponible sobre las librerías seleccionadas.
3. Soporte para los formatos de datos otorgados.
4. Costos de ejecución (para llm y embedding models).
5. En el caso de jupyterlab en lugar de scripts python, la elección se basa en que la visualización y evolución entre los distintos enfoques son más sencillos de explicar.


## Estrategia 

El challenge se abordará de manera incremental, demostrando en cada iteración las mejoras obtenidas luego de aplicar distintas técnicas. En general, los entregables son:

1. src/utils.ipynb: contiene implementación de todas las funciones requeridas para armar los escenarios *basic*, *augmented* y *advanced_augmented*.
2. src/basic.ipynb: notebook con estrategia básica.
3. src/augmented.ipynb: notebook con estrategia básica de aumento de metadatos.
4. src/advanced_augmented.ipynb: notebook con estrategia básica y estrategia avanzada de aumento de metadatos.
5. src/nodes.ipynb: notebook de ejemplo que muestra como cambia la estructura de los *nodes* al aplicar las distintas técnicas.

## Retrievers

Se han implementado 2 tipos de retrievers: 

1. función *retrieve(index, query)*: retriever básico donde, a partir de un index, se crea un query_index que permite consultar sobre los datos indexados utilizando un llm.
2. función *custom_retrieve(index, query)*: retriever avanzado donde, a partir de un index, se arma paso a paso el retriever. Este approach permite customizar, entre otros, el llm a utilizar, la cantidad de documentos a obtener desde la base de datos vectorial luego de consultar o el modo de retorno de respuestas.

## ChromaDB

ChromaDB se utiliza como almacén de vectores. Se ha configurado para escribir las colecciones en el sistema de archivos (*path: chroma_index/*).
Se ha implementado una lógica para evitar recrear o modificar un index que ya ha sido creado previamente y en su lugar poder cargar esos embeddings desde el sistema de archivos. De esta manera se puede ahorrar tanto en tiempo como en costos al no requerir el proceso completo (loading/parsing/chunking/augmentation) cada vez que se ejecuta un notebook si su base de datos vectorial ya fue creada.

## Basic

Esta estrategia contempla solamente el *parsing* de los documentos.
Debido a que los documentos suministrados se encuentran bien estructurados, el parser de llamaindex es capaz de procesar de manera eficiente, manteniendo las relaciones entre títulos/subtítulos y párrafos al momento de generar *nodes*, lo que otorga buena información al realizar análisis semántico.
No obstante, se evidencia pérdida de información al momento de analizar las respuestas y a su vez tanto el modelo de embedding como el llm deben trabajar con *nodes* más grandes, lo que a escala, degradaría la performance.

Los resultados se pueden revisar en:
- results/basic.txt: approach básico utilizando retriever básico
- results/basic_custom_retriever.txt: approach básico utilizando retriever custom

## Augmented

Esta estrategia contempla, además del *parsing* de los documentos, una capa simple de manejo de metadata. En específico, para cada *node*, se asocia la empresa y se excluyen metadatos que no agregan valor en este contexto, como lo es, la extensión de los archivos.
Para la asociación de documento-empresa, se simula un diccionario de datos. En un contexto productivo este approach es bastante realista, puesto que esta clase de información se puede considerar en el layer de ingesta de datos (preservando detalles del source y utilizando estos detalles como metadata).
No se aplica ninguna estrategia de chunking ni de generación avanzada de metadata.
Si bien es cierto los resultados siguen siendo incompletos, a escala se puede intuir una mejora importante en cuanto a performance. Incluir información de las empresas dueñas de los documentos en la metadata de éstos, generará un mejor indexado y, considerando que las preguntas a realizar tienden a tener sentido al segmentar por empresas, la obtención de documentos desde la base de datos vectorial tenderá a ser más eficiente.

Los resultados se pueden revisar en:
- results/augmented.txt: approach augmented utilizando retriever básico
- results/augmented_custom_retriever.txt: approach augmented utilizando retriever custom


## Advanced Augmented

Esta estrategia contempla todas las estrategias utilizadas en el approach *Augmented*, pero además, con la finalidad de poder responder de manera completa a todas las preguntas, se contemplan los siguientes puntos:

1. Chunking de *nodes*: luego de obtener los *nodes* a partir del *parsing* de documentos, se utiliza *SentenceSplitter* para "partir" esos *nodes* en unidades más pequeñas, intentando preservar el significado semántico de ellos. La configuración utilizada es la siguiente: chunk_size=200, chunk_overlap=25. La decisión se basa en resultados obtenidos y en inspecciones sobre la data, donde prevalecen párrafos cortos y bien estructurados.
2. Aumento de metadatos avanzado: Una vez generado el nuevo set de *nodes*, se obtienen resúmenes de cada *node*, utilizando *SummaryExtractor* y el llm seleccionado. La idea general de esto es que cada node tenga un resumen asociado, mejorando así el indexado. Para generar los resúmenes se ha suministrado un *prompt* customizado por razones que se explican en la sección de [Conclusiones](#conclusiones).

La configuración de estos pasos se realiza utilizando *IngestionPipeline*, definiendo cada paso como un *transformer*. Es importante destacar el order en el cual estos transformers son implementados. En este caso (y probablemente en general) *chunking* debe ser ejecutado previo a *metadata augmentation*

Los resultados de esta configuración son completos y coherentes y se pueden revisar en:
- results/advanced_augmented.txt: approach advanced augmented utilizando custom retriever


## Lecciones aprendidas

- SummaryExtractor prompt: al momento de generar *summaries* de los chunks, se ha experimentado una mejora importante en tiempo de retrieval al generarlos en el idioma de los documentos (en este caso, español). Esto puede ser un problema si se cuenta con documentos en distintos idiomas. Algunos enfoques que podrían ser útiles serían capas de homologación de idiomas o el uso de bases de datos vectoriales mejor optimizadas para este tipo de caso de uso.

- Sobrecarga de índices: si una base de datos vectorial no maneja de manera correcta la repetición de documentos (utilizando estrategias como *updates* o *upserts*) se pueden generar degradaciones en tiempos de retrieval. Esto sucede porque el mismo *node* podría aparecer varias veces como resultado a una query, previniendo otros *nodes* diferentes y potencialmente valiosos de ser incluidos. 
Una opción para superar este problema sería utilizar *rerankers*, pero definitivamente el cuidado debe estar en manejar correctamente el reindexado de documentos.

- Queries con múltiples preguntas: para la query *"¿Cómo es el reajuste por IPC en Fakesoft? ¿Cómo es el reajuste por IPC en Ficticia?"* o en general, queries compuestas por más de una pregunta, dependiendo de la configuración del retriever podrían existir problemas de completitud si *K* es demasiado bajo. Esto se debe a que el retriever podría retornar solo *nodes* relevantes para una parte de la pregunta y no para todas las preguntas incluidas. En este escenario algunas posibles soluciones serían trabajar con *K* más grande + reranker o seguir approaches de *splitting* de queries y unión de resultados (esto es, internamente la query se divide en *N* subqueries, el retriever se llama *N* veces -1 vez por cada subquery- y antes de retornar, se genera un único resultado a partir de todos los resultados obtenidos).


## Conclusiones

En un contexto de RAGs, es crucial generar buenas estrategias de parsing/chunking, puesto que incluso los llms más potentes no serán capaces de dar buenas respuestas si su contexto y base de conocimiento es insuficiente o no está correctamente preprocesada.
Es por ello que desde una perspectiva de *data engineering*, hace bastante sentido contar con layers de validación o scoring que prevengan que datos de baja calidad alimenten y potencialmente degraden las bases de conocimiento de este tipo de sistemas. A su vez, hace sentido contar con layers de homologación que permitan implementar parsers potentes, para que, desde un punto en adelante e independientemente del formato de entrada, los datos salgan con estructuras y niveles de calidad aceptables, maximizando de esa forma las capacidades de los actuales modelos de embedding y llms.
En términos de escalabilidad y performance de RAGs, es fundamental aplicar estrategias correctas para indexar la data de forma eficiente. Esto permite por un lado mantener vectores diferentes lo suficientemente alejados entre sí y por otro lado mejorar la calidad del retriever. Sobre este punto, la experimentación demuestra que la metadata funciona como un mecanismo eficiente para colocar vectores semánticamente similares en espacios vectoriales cercanos.
Mediante un approach incremental, este trabajo demuestra el beneficio del uso de metadatos y de estrategias apropiadas de chunking, obteniendo cada vez mejores respuestas a las preguntas planteadas.