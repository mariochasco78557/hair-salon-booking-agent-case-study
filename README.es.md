# Un agente de reservas para una peluquería

El problema: el dueño está cortando el pelo y suena el teléfono. O suelta las tijeras o pierde la cita.

Qué hace: desde septiembre de 2026, mi agente contesta el teléfono con la voz clonada del dueño y reserva en la agenda real del local. Por WhatsApp, un segundo agente confirma o cambia las reservas y manda recordatorios.

El código decide; el modelo lee y redacta: habla con quien llama, pero no puede inventarse un hueco. Mi servidor solo ofrece huecos reales, cada uno con un código firmado que caduca pronto, y vuelve a comprobar ese código antes de escribir nada en la agenda.

Las cifras: a finales de octubre de 2026, 407 llamadas y 212 reservas por teléfono. Los dos primeros días hubo 63 llamadas y 34 reservas, con alrededor de minuto y medio por llamada y 0 errores técnicos.

Un fallo real: del 20 al 22 de octubre tuve en producción una versión nueva con un modelo más reciente. En 101 llamadas, el agente a veces decía "Un momento", no hacía nada y quien llamaba se quedaba en silencio.

La causa era un tope de longitud para cada respuesta: 300 tokens, heredado de una versión pensada para otro modelo. El modelo nuevo necesitaba más espacio para escribir la petición de reserva, y esta se cortaba a medias. En 24 de los 29 silencios, la respuesta se paró justo en 300. Mi simulación no lo vio porque sus códigos de reserva falsos tenían 14 caracteres; los reales tienen 91.

Subí el tope a 1.000, puse en las pruebas códigos del tamaño real y añadí cuatro casos que reproducen el fallo. En las mismas pruebas, los silencios pasaron de 1 de 41 a 0 de 41, y el arreglo entró en producción la noche del 22 de octubre. Lo que me llevo: pruebo con datos del tamaño real y leo las transcripciones en crudo, no solo los contadores.

Lo que aún no he medido: no tengo el coste por llamada. No sé cuántas reservas cogía la peluquería por teléfono antes del agente, así que no puedo decir cuántas de las 212 se habrían perdido. Cero errores técnicos significa que nada se rompió, no que cada reserva cayera en el hueco correcto. El 0 de 41 sale de pruebas de texto, no de llamadas reales.

Lo siguiente que haría:
1. Montar un golden set con llamadas reales anonimizadas: una lista fija de casos con la reserva esperada, que primero revisa el código y después un modelo como juez, comparado con el criterio de una persona.
2. Medir el coste por llamada.
3. Acordar con el dueño una línea base, para que las 212 reservas se puedan comparar con algo.

Plantilla pública, con datos falsos (4.767 + 277 tests en verde): https://github.com/mariochasco78557/ai-booking-agent-template
Todos mis sistemas: https://www.mariochasco.com
