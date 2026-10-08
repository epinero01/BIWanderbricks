# BIWanderbricks

Semantic Layer artesanal para Genie
Objetivo. Diseñar una capa semántica mantenible para consultar el dataset Wanderbricks mediante Databricks Genie en Free Edition, evitando concentrar toda la lógica de negocio en un system prompt enorme.
1. Contexto y decisiones de partida
    • Se utiliza el sample Wanderbricks de Databricks como dataset de referencia.
    • El dataset se ha copiado íntegramente a la instancia propia de Databricks Free Edition.
    • La copia se realizó reproduciendo los CREATE TABLE y cargando los datos mediante INSERT SELECT, manteniendo la estructura y los datos del sample.
    • La copia local permite administrar las tablas directamente y añadir comentarios, descripciones y otra metadata que no era viable gestionar de la misma manera sobre el sample alojado externamente.
    • La capa semántica se construirá separada de las tablas de datos originales.
2. Principio arquitectónico
La lógica semántica debe vivir principalmente fuera de Genie. Genie debe recibir una configuración mínima y estable, mientras que las definiciones de negocio, métricas, relaciones y reglas se mantienen en objetos versionables de Databricks.
Principio rector:
Tablas de datos = fuente de datos · Capa semántica = fuente de significado · Genie = interfaz de lenguaje natural y motor de consulta
3. Arquitectura propuesta
La arquitectura inicial se organizará en dos zonas lógicas: el dataset Wanderbricks copiado y un esquema semántico propio.
    • Dataset: tablas originales de Wanderbricks, conservadas como espejo del sample.
    • Semantic layer: objetos propios destinados a documentar y normalizar el significado de negocio.
    • Genie: espacio configurado con instrucciones breves, conocimiento semántico y ejemplos SQL cuando sean necesarios.
4. Business Glossary artesanal
La primera versión del sustituto del Business Glossary será un conjunto pequeño de tablas Delta. No se pretende replicar toda una plataforma de gobierno, sino disponer de una fuente de verdad semántica sencilla, consultable y mantenible.
semantic_terms
Definiciones de conceptos de negocio y su correspondencia con el modelo físico.
Campos iniciales:
    • term_id
    • term
    • business_definition
    • synonyms
    • entity
    • source_table
    • source_column
    • notes
semantic_metrics
Catálogo de métricas oficiales y sus reglas de cálculo.
Campos iniciales:
    • metric_id
    • metric_name
    • definition
    • formula
    • default_filter
    • grain
    • synonyms
    • notes
semantic_relationships
Relaciones entre entidades y joins técnicos autorizados.
Campos iniciales:
    • relationship_id
    • from_entity
    • to_entity
    • relationship
    • join_condition
    • cardinality
    • description
semantic_rules
Reglas de interpretación y comportamiento de negocio que no son simples definiciones de términos.
Campos iniciales:
    • rule_id
    • rule_name
    • description
    • scope
    • priority
    • notes
5. Ejemplos de semántica
Ejemplo de término:
Booking — Reserva realizada por un huésped para una propiedad. Sinónimos posibles: reservation.
Ejemplo de métrica:
Revenue — Valor monetario generado por reservas válidas. La fórmula y los filtros deberán definirse contra las columnas reales del dataset una vez validado el modelo.
Ejemplos de reglas:
    • El recuento de reservas debe utilizar booking_id de forma distinta.
    • Revenue debe excluir reservas canceladas si esa es la definición de negocio acordada.
    • “Booking date” debe distinguirse de check-in y check-out.
    • Las relaciones entre entidades deben utilizar los joins documentados.
6. Papel del system prompt de Genie
El system prompt no será el repositorio principal de conocimiento. Solo establecerá el comportamiento general del agente.
Contenido previsto: utilizar las definiciones semánticas como autoridad, no inventar métricas, respetar las relaciones documentadas y pedir aclaración cuando una definición sea realmente ambigua.
7. Evolución posterior
    • Añadir semantic_examples para almacenar preguntas representativas y SQL esperado.
    • Evaluar vistas de negocio sobre las tablas Wanderbricks cuando el modelo físico esté validado.
    • Evaluar Unity Catalog Metric Views si están disponibles y resultan adecuadas para la edición utilizada.
    • Añadir versionado o estado a las definiciones si el proyecto crece.
    • Probar qué parte de la semántica puede exponerse directamente mediante metadata de Unity Catalog y qué parte requiere configuración específica de Genie.
8. Plan de trabajo
1. Inventariar las tablas y columnas reales de la copia local de Wanderbricks.
2. Identificar entidades, dimensiones y relaciones.
3. Definir el vocabulario de negocio.
4. Definir las primeras métricas y reglas.
5. Crear el esquema y las tablas semantic_*.
6. Añadir comentarios y metadata a tablas/columnas donde aporte valor.
7. Configurar Genie con un conjunto mínimo de instrucciones.
8. Crear ejemplos SQL solo donde las pruebas demuestren que son necesarios.
9. Validar el comportamiento con un conjunto de preguntas de negocio.
10. Iterar la capa semántica, no inflar el system prompt.
9. Estado actual
Este documento define la arquitectura inicial. Todavía no se han fijado las entidades, métricas ni reglas definitivas: esas decisiones se tomarán después de inspeccionar la copia local de Wanderbricks.
