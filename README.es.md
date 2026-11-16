Un agente de reservas por voz para una peluquería: 407 llamadas y un fallo silencioso
=====================================================================================

Desde septiembre de 2026, mi agente de IA contesta el teléfono de una peluquería y reserva en su agenda real. A finales de octubre: 407 llamadas, 212 reservas por teléfono. Aquí cuento cómo funciona, de dónde sale cada cifra y los tres días de octubre en que falló sin que mis pruebas lo vieran.

El problema
-----------

El dueño está cortando el pelo y suena el teléfono. O suelta las tijeras o pierde la cita.

Yo quería que el teléfono se contestara con su voz y que la cita quedara en la agenda del local.

Qué hace el sistema
-------------------

1. Teléfono: un agente contesta con la voz clonada del dueño y reserva en la agenda real.
2. WhatsApp: un segundo agente, con la API oficial de Meta, gestiona las mismas citas por escrito. Las confirma, las cambia o las cancela y manda recordatorios.

El asistente sigue un guion. Si quien llama pregunta otra cosa, contesta en breve y vuelve a la reserva.

Está hecho con TypeScript, Zod para revisar cada petición del modelo, ElevenLabs para la voz y Docker. Cada cliente va en su propio contenedor, con sus datos aparte.

La regla de diseño: el código decide, el modelo redacta
-------------------------------------------------------

El modelo es la parte que habla: entiende a quien llama y redacta la respuesta. Todo lo que tiene consecuencias lo decide código normal, escrito y probado por mí: quién llama, qué huecos existen, si se escribe algo en la agenda y qué mensaje sale.

Una reserva funciona así:

1. El modelo le pide huecos libres a mi servidor.
2. El servidor mira la agenda real y devuelve solo huecos que existen. Cada uno lleva un código firmado que caduca pronto, como una entrada con fecha sellada.
3. Para reservar, el modelo tiene que devolver ese código. El servidor lo vuelve a comprobar antes de escribir nada.

El modelo puede expresarse mal, pero no puede reservar un hueco que no existe.

Parte de esta regla salió de un error. El 10 de octubre, alguien pidió cita para el jueves 16, pero el 16 era viernes. El modelo calculó la fecha él solo, mandó el viernes al servidor y luego dijo jueves.

Ese mismo día pasé las fechas al servidor. Ahora el modelo solo pasa lo que oye, el día de la semana y el número, y el servidor calcula la fecha. Si las dos cosas no cuadran, el agente pregunta qué día quería decir.

Las cifras y de dónde salen
---------------------------

| Cifra | Valor | De dónde sale |
|---|---|---|
| Llamadas atendidas, septiembre a finales de octubre de 2026 | 407 | Un recuento a finales de octubre |
| Reservas por teléfono en el mismo periodo | 212 | El mismo recuento |
| Primeros 2 días en producción | 63 llamadas, 34 reservas | Mi recuento de esos 2 días |
| Duración de la llamada, primeros 2 días | Alrededor de minuto y medio | El mismo recuento |
| Errores técnicos, primeros 2 días | 0 | El mismo recuento |
| Llamadas con la versión del fallo, 20 a 22 de octubre | 101 | Registros y transcripciones en crudo |
| Tests del sistema en producción | Más de 5.700 | Repositorio privado |
| Tests de la plantilla pública | 4.767 + 277, todos en verde | Repositorio público, enlazado abajo |

Dos notas. No he repetido el recuento, así que 407 y 212 son una foto de finales de octubre. Y 0 errores técnicos significa que nada se rompió en esos 2 días; no dice si cada reserva cayó en el hueco correcto.

El fallo: tres días de silencios (del 20 al 22 de octubre)
----------------------------------------------------------

El 20 de octubre pasé el agente a una versión nueva con un modelo más reciente. En tres días atendió 101 llamadas. En algunas, el agente decía "Un momento" y no hacía nada, y quien llamaba se quedaba en silencio.

Cómo encontré la causa:

1. Mi script de recuento marcó 32 frases de espera sin acción detrás. Leí en crudo las transcripciones de las 22 llamadas afectadas. 3 de las 32 no eran fallos, porque la frase llegaba después de la acción. Quedaban 29.
2. En 24 de las 29, el turno del modelo se paró justo en 300 tokens. Un token es un trozo pequeño de texto, más o menos parte de una palabra, y 300 era el tope de cada respuesta.
3. El tope venía de una versión anterior, pensada para el modelo de antes. Ese modelo gastaba de 119 a 149 tokens en escribir una petición de reserva. El nuevo necesitaba de 315 a 350 en una simulación con datos reales. Por encima de 300, la petición se cortaba y la plataforma de voz la descartaba.

¿Por qué no se pararon las reservas? Tras un intento fallido entraba un modelo de respaldo un instante después. Hizo 17 de las 24 reservas de esas llamadas. En mis totales, el respaldo parecía una parte pequeña de los turnos. No tengo un recuento propio de esa parte ni del retraso, así que no doy cifra.

Por qué no lo vieron mis pruebas: la simulación del 20 de octubre usaba códigos de reserva falsos de 14 caracteres; los reales tienen 91. Con códigos cortos, la petición de reserva se quedaba en 266 a 272 tokens, por debajo del tope, y las pruebas pasaban.

El arreglo fue un solo ajuste: el tope pasó de 300 a 1.000, sin tocar el prompt ni la voz. Antes de ponerlo en producción lo probé así:

1. Copié los 18 casos de prueba del 20 de octubre con un código falso de 91 caracteres, el tamaño real.
2. Añadí 4 casos que reproducen el fallo, como un "sí" muy corto o dos citas en la misma llamada.
3. Pasé la versión antigua por los mismos casos, a la misma hora, para comparar.

En los casos que hicieron las dos versiones, las frases de espera sin acción pasaron de 1 de 41 a 0 de 41. También medí el tiempo de respuesta, pero los 1,53 segundos (9 de cada 10, sobre 10 escenarios que hicieron las dos versiones) y los 2,49 de antes salen de conjuntos de escenarios distintos, así que no los comparo. Horas inventadas: 0 en las dos.

Esa versión incluyó otro cambio. Si la agenda no responde, el agente ofrece pasarle el recado al dueño.

Entró en producción a las 21:15 del 22 de octubre, con las versiones anteriores listas para volver atrás. Al día siguiente seguía atendiendo todas las llamadas.

Qué puedes reutilizar
---------------------

1. Si un error puede costar una cita, quítale esa decisión al modelo. El mío oía bien el día de la semana y el número y aun así se equivocaba con la fecha.
2. Para todo lo que se escribe, dale al modelo opciones para elegir, nunca un hueco en blanco que rellenar. En las pruebas de octubre, con los huecos solo del servidor, el agente inventó 0 horas.
3. Prueba con datos del tamaño real. Códigos falsos de 14 caracteres frente a reales de 91 escondieron el fallo durante 101 llamadas.
4. Cuando cambies de modelo, revisa cada límite que pusiste para el anterior. Un tope de 300 iba bien con un modelo que gastaba hasta 149 tokens por reserva y rompió uno que necesitaba hasta 350.
5. Lee las transcripciones en crudo y vigila el respaldo. El respaldo hizo 17 de 24 reservas mientras mis totales lo hacían parecer una parte pequeña de los turnos. Un respaldo que contesta más de la cuenta es la alarma.

Lo que aún no he medido
-----------------------

1. Coste por llamada: no tengo la cifra.
2. Una línea base: no sé cuántas reservas cogía la peluquería por teléfono antes del agente, así que no puedo decir cuántas de las 212 se habrían perdido.
3. Calidad de las reservas: no he comprobado una a una que cada cita tuviera el servicio y el hueco correctos. El plan es un golden set: llamadas reales anonimizadas con la reserva esperada en cada una, revisadas primero por código y después por un modelo como juez, comparado con el criterio de una persona. Previsto para enero; aún no está hecho.
4. WhatsApp: solo tengo un recuento parcial, así que no está en la tabla.
5. Satisfacción: no he preguntado ni al dueño ni a quienes llaman.
6. Las pruebas de octubre son de texto, con una ejecución por caso. No miden el reconocimiento de voz ni los silencios reales, así que el 0 de 41 no sale de llamadas reales.

Plantilla pública con datos falsos y el mismo diseño de reservas: https://github.com/mariochasco78557/ai-booking-agent-template

Escrito por Mario Chasco, noviembre de 2026. https://www.mariochasco.com/es#sys-booking-agent
