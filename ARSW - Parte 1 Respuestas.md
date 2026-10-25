# ARSW Lab Escalamiento Azure: Parte 1, respuestas a las preguntas

Escalabilidad vertical con una máquina virtual en Azure ·

**Nota sobre el entorno:** por la política de regiones de la suscripción Azure for Students no fue posible usar `B1ls` ni `B2ms`. La VM se creó en **Chile Central** con `B2ats_v2` (2 vCPU, 1 GiB RAM) y se escaló a `B4als_v2` (4 vCPU, 8 GiB RAM). El objetivo del experimento es el mismo: comparar una VM pequeña con una de mayor capacidad. Cada prueba de carga se corrió una sola vez, por lo que los resultados son indicativos.


## Paso 12: evaluación del escenario de calidad

**Escenario:** con N > 1.000.000 consultado de forma concurrente en condiciones normales, todas las peticiones deben responderse y la CPU no debe superar el 70 %.

| Configuración | Peticiones | Fallos | CPU máxima | ¿Cumple? |
| --- | --- | --- | --- | --- |
| `B2ats_v2`, 2 Newman | 20 | 5 (25 %) | 49.22 % | No: hay peticiones sin respuesta |
| `B4als_v2`, 2 Newman | 20 | 0 (0 %) | 24.89 % | Sí, en esta corrida |
| `B4als_v2`, 4 Newman | 40 | 5 (12.5 %) | 25.16 % | No: hay peticiones sin respuesta |

**Conclusión:** el escalamiento vertical **no cumple de forma confiable** el escenario. Con 2 clientes en la VM grande se cumplió, pero con 4 clientes volvieron los fallos. Además, la CPU total se ve baja porque la app usa un solo hilo: un núcleo saturado de cuatro marca \~25 %, así que el límite del 70 % no se alcanza aunque la aplicación ya esté al límite. Por eso la métrica de CPU total no sirve aquí para saber si el sistema está saturado.

## Preguntas

### 1. ¿Cuántos y cuáles recursos crea Azure junto con la VM?

Al crear la VM por el portal, Azure genera en el grupo `SCALABILITY_LAB`:

1. La máquina virtual (`VERTICAL-SCALABILITY`)
2. El disco del sistema operativo (OS disk, 30 GiB, SSD premium LRS)
3. La interfaz de red (NIC)
4. La dirección IP pública (`VERTICAL-SCALABILITY-ip`)
5. El grupo de seguridad de red (`VERTICAL-SCALABILITY-nsg`)
6. La red virtual (`vnet-chilecentral-...`), con su subred

Son 6 recursos. 

<img width="948" height="916" alt="Captura de pantalla 2026-10-25 165805" src="https://github.com/user-attachments/assets/fde2c594-1bc7-41c5-9494-cd4aa7d38925" />


### 2. ¿Para qué sirve cada recurso?

| Recurso | Función |
| --- | --- |
| Máquina virtual | El servidor que ejecuta el sistema operativo y la aplicación |
| Disco del SO | Almacenamiento persistente donde vive Ubuntu y los archivos de la VM |
| Interfaz de red (NIC) | Conecta la VM con la red virtual; tiene la IP privada y se asocia a la IP pública y al NSG |
| IP pública | Dirección con la que se accede a la VM desde Internet |
| Network Security Group | Firewall con reglas de entrada y salida que permiten o bloquean tráfico por puerto y protocolo |
| Red virtual y subred | Red privada de Azure donde se ubica la VM; define el rango de direcciones IP privadas |

### 3. ¿Por qué se cae la app al cerrar SSH con `npm FibonacciApp.js`? ¿Por qué crear un *Inbound port rule*?

**Caída de la app:** el proceso se lanza como hijo de la sesión de la terminal. Al cerrar SSH, el sistema le envía a esa sesión una señal de terminación (SIGHUP) y los procesos hijos mueren con ella. `forever` evita esto porque ejecuta la app como un proceso independiente (daemon) que sobrevive a la sesión y, además, la reinicia si se cae.

**Regla de entrada:** el NSG bloquea por defecto el tráfico entrante que no esté permitido explícitamente. Al crear la VM solo quedó abierto el puerto 22 (SSH). La aplicación escucha en el 3000, así que sin una regla que permita ese puerto, las peticiones desde Internet nunca llegan.

<img width="500" height="165" alt="Captura de pantalla 2026-10-25 171636" src="https://github.com/user-attachments/assets/c6d47508-a4a2-4f43-b7c0-7f160bf619dc" />


### 4. Tabla de tiempos e interpretación

Medidos en la consola del navegador (pestaña Network, columna Time), con caché desactivada y una petición a la vez:

| N | `B2ats_v2` (s) | `B4als_v2` (s) |
| --- | --- | --- |
| 1000000 | 6.55 | 6.03 |
| 1010000 | 6.60 | 6.05 |
| 1020000 | 6.62 | 6.21 |
| 1030000 | 6.82 | 6.35 |
| 1040000 | 7.00 | 6.51 |
| 1050000 | 7.08 | 6.75 |
| 1060000 | 7.26 | 6.90 |
| 1070000 | 7.48 | 6.88 |
| 1080000 | 7.47 | 6.96 |
| 1090000 | 7.62 | 7.16 |

**Interpretación:** el tiempo crece con N. Entre 1.000.000 y 1.090.000 (+9 % en N) el tiempo pasó de 6.55 s a 7.62 s (+16 %). Es coherente con un algoritmo iterativo que, en cada paso, suma números cada vez más grandes (el resultado tiene cientos de miles de dígitos), de modo que cada iteración cuesta más que la anterior. Además Node.js ejecuta el cálculo en un solo hilo y bloquea el *event loop* mientras dura. **Verifiquen esto leyendo `FibonacciApp.js`** y confirmen si usa `BigInt` o una librería de enteros grandes, porque aquí es una inferencia a partir de las mediciones. Parte del tiempo también es la descarga de \~209 kB de respuesta.

<img width="954" height="1001" alt="Captura de pantalla 2026-10-25 172115" src="https://github.com/user-attachments/assets/c399e88c-e691-49c6-b4ab-ddd411485fba" />


### 5. Imagen del consumo de CPU e interpretación

<img width="1568" height="655" alt="Captura de pantalla 2026-10-25 173201" src="https://github.com/user-attachments/assets/4c403140-636a-4a55-994d-f9f412e6b4e6" />


<img width="1587" height="649" alt="Captura de pantalla 2026-10-25 173320" src="https://github.com/user-attachments/assets/f052d3b0-307e-41cd-a2d9-a74e9d6e66de" />


**Interpretación:** los picos corresponden a las tandas de peticiones secuenciales. Entre una y otra la CPU vuelve a casi 0 %. El máximo fue 23.46 % porque Azure agrega la métrica por intervalo (aprox. un minuto) y cada cálculo dura 6 a 7 s, así que se diluye. Mientras corre un cálculo, un núcleo está al 100 %: la función es puro cómputo (CPU-bound), no espera disco ni red. Con 2 vCPU, un solo hilo equivale como máximo a \~50 % de la CPU total, y es lo que se observó después con carga concurrente.

### 6. Resumen de la ejecución de Postman (Newman), `B2ats_v2`

Dos ejecuciones en paralelo, 10 iteraciones cada una, iniciadas a las 5:46 p. m.

|  | Newman A | Newman B |
| --- | --- | --- |
| Peticiones fallidas | 2 | 3 |
| Duración total | 1 m 31 s | 1 m 25.4 s |
| Tiempo promedio | 9.8 s | 9.9 s |
| Mínimo | 6.3 s | 6.3 s |
| Máximo | 17.7 s | 12.3 s |
| Desviación estándar | 3.8 s | 2.7 s |

<img width="904" height="723" alt="Captura de pantalla 2026-10-25 174745" src="https://github.com/user-attachments/assets/f2ea597e-7059-4e65-a96d-28ad860d5704" />


<img width="906" height="717" alt="Captura de pantalla 2026-10-25 174757" src="https://github.com/user-attachments/assets/61eb327d-fe8f-4bb4-b6c9-886b9656fd20" />


<img width="1545" height="654" alt="Captura de pantalla 2026-10-25 174907" src="https://github.com/user-attachments/assets/0e66fa1b-c0e8-4f0f-90df-14a0b9d9907c" />


<img width="954" height="384" alt="Captura de pantalla 2026-10-25 175202" src="https://github.com/user-attachments/assets/238d45da-cc23-4d27-88ae-42daa8be21d4" />


**Tiempos:** el mínimo (6.3 s) coincide con el de una petición aislada. El promedio sube a \~9.8 s y el máximo a 17.7 s porque, al ser un solo hilo, cuando dos peticiones coinciden una espera a que termine la otra.

**Fallos:** 5 de 20 peticiones (25 %) terminaron en `read ECONNRESET`, es decir, la conexión fue cerrada por el otro extremo mientras se esperaba la respuesta. El `forever list` muestra un uptime de unos 38 minutos con el mismo PID, por lo que **el proceso no se cayó**. La causa exacta del corte no se verificó (por ejemplo con los logs del servidor). Una hipótesis razonable es que las conexiones que esperan mucho en cola son cortadas por la red o por los tiempos de espera del servidor HTTP de Node.

### 7. Diferencia entre `B2ms` y `B1ls`

En infraestructura: `B1ls` tiene 1 vCPU y 0.5 GiB de RAM; `B2ms` tiene 2 vCPU y 8 GiB. Pero la diferencia importante está en el modelo de la **serie B (burstable)**: estas VMs no dan el 100 % de la CPU de forma sostenida. Tienen un rendimiento base (una fracción de la CPU) y acumulan **créditos** cuando están por debajo de esa base, que luego gastan para "explotar" hasta el 100 %. Cuando los créditos se agotan, la VM queda limitada al rendimiento base. `B1ls` tiene un rendimiento base mucho menor que `B2ms`, así que se queda sin créditos mucho antes bajo carga continua.  y cítenlos aquí.

Aclaración: por la restricción de la suscripción, en la práctica se comparó `B2ats_v2` con `B4als_v2`, que también son *burstable* (Serie B v2). Para una carga sostenida de CPU como esta, ese modelo es relevante: una VM burstable no es la más adecuada para cómputo continuo.

<img width="1643" height="481" alt="Captura de pantalla 2026-10-25 175845" src="https://github.com/user-attachments/assets/bc357dfd-8e43-4ed4-addc-fe7d9f8f1bfd" />


### 8. ¿Aumentar el tamaño de la VM es una buena solución? ¿Qué pasa con la FibonacciApp al cambiar el tamaño?

**No, no es una buena solución para este problema.** La app ejecuta el cálculo en un solo hilo, así que los núcleos adicionales quedan sin usar (la CPU se mantuvo en \~25 % con 4 vCPU). El tiempo de una petición individual mejoró poco (de 6.55 s a 6.03 s en N = 1.000.000), mejora que no se puede atribuir con certeza a los vCPU extra. Con 2 clientes en paralelo dejó de haber fallos en esta corrida, pero con 4 reaparecieron, así que la solución solo retrasa el problema.

**Qué pasa con la app:** al cambiar el tamaño la VM se reinicia y la aplicación **no volvió a arrancar sola** (`forever list` indicó *No forever processes running*). Hubo que entrar por SSH y ejecutar `forever start FibonacciApp.js` otra vez. Mientras tanto el servicio estuvo caído.

<img width="957" height="255" alt="Captura de pantalla 2026-10-25 180341" src="https://github.com/user-attachments/assets/1a8766a2-ff36-430d-93a1-faa1204ee284" />


### 9. ¿Qué pasa con la infraestructura al cambiar el tamaño? ¿Efectos negativos?

- La VM se **reinicia**, lo que implica downtime y pérdida de la sesión SSH.
- Los procesos que no están configurados para iniciar con el sistema (como el lanzado con `forever` a mano) no se recuperan solos.
- Puede ocurrir que cambie la IP pública si es dinámica. En este caso **no cambió**, pero no está garantizado.
- Solo se puede escalar a tamaños disponibles en esa región y para esa suscripción, y hay un límite máximo de tamaño.
- El costo sube (de \~US$10/mes a \~US$135/mes en este caso) y hay que acordarse de volver al tamaño inicial.
- Sigue habiendo un único punto de falla: una sola VM.

### 10. ¿Hubo mejora en el consumo de CPU o en los tiempos de respuesta?

**CPU: sí en el porcentaje, pero es engañoso. Tiempos: no de forma significativa.**

|  | `B2ats_v2` (2 Newman) | `B4als_v2` (2 Newman) |
| --- | --- | --- |
| Fallos | 5 de 20 (25 %) | 0 de 20 (0 %) |
| Tiempo promedio | \~9.8 s | \~10.6 s |
| Tiempo máximo | 17.7 s / 12.3 s | 11.3 s / 11.6 s |
| CPU máxima | 49.22 % | 24.89 % |

La CPU bajó de \~49 % a \~25 % porque el mismo hilo ocupa 1 de 4 núcleos en lugar de 1 de 2; el trabajo no se redujo. El tiempo promedio por petición bajo concurrencia incluso subió ligeramente (aprox. 6 s de cálculo × 2 clientes en cola). Lo que sí cambió en esta corrida fue que no hubo fallos. No se verificó la causa; puede deberse a la mayor RAM, al cambio de host tras el reinicio o a variabilidad, así que conviene repetir la prueba antes de afirmar una explicación.

<img width="909" height="672" alt="Captura de pantalla 2026-10-25 181453" src="https://github.com/user-attachments/assets/f1d1b3a6-3f30-48d1-a9cf-be177f48a54d" />


<img width="902" height="709" alt="Captura de pantalla 2026-10-25 181501" src="https://github.com/user-attachments/assets/adb458d7-68fc-46ac-bb06-45efb784e213" />


<img width="1577" height="672" alt="Captura de pantalla 2026-10-25 181553" src="https://github.com/user-attachments/assets/ee643df2-ecea-42c2-8665-68c7c52ce125" />


### 11. Con 4 ejecuciones paralelas de Newman, ¿el comportamiento es porcentualmente mejor?

Prueba con `B4als_v2`, iniciada a las 6:17 p. m.:

|  | Newman A | Newman B | Newman C | Newman D |
| --- | --- | --- | --- | --- |
| Fallos | 1 | 2 | 0 | 2 |
| Duración total | 2 m 44 s | 3 m 0.5 s | 2 m 48.6 s | 3 m 10 s |
| Tiempo promedio | 16.1 s | 19.3 s | 16.7 s | 22 s |
| Máximo | 27.5 s | 33 s | 32.4 s | 38.5 s |

|  | `B2ats_v2` (2 Newman) | `B4als_v2` (2 Newman) | `B4als_v2` (4 Newman) |
| --- | --- | --- | --- |
| Peticiones totales | 20 | 20 | 40 |
| Fallos | 5 (25 %) | 0 (0 %) | 5 (12.5 %) |
| Tiempo promedio | \~9.8 s | \~10.6 s | \~18.5 s |
| CPU máxima | 49.22 % | 24.89 % | 25.16 % |

**Respuesta:** depende de contra qué se compare. Frente a la VM pequeña con 2 Newman, el porcentaje de fallos con 4 Newman en la VM grande es menor (12.5 % frente a 25 %), aunque con el doble de carga. Pero frente a la misma VM grande con 2 Newman, el comportamiento es **peor**: aparecen fallos (0 % → 12.5 %), el tiempo promedio casi se duplica (\~10.6 s → \~18.5 s) y el máximo llega a 38.5 s. La CPU se queda en \~25 %, lo que confirma que solo un núcleo trabaja. Al ser corridas únicas, no se puede decir que la VM grande sea "mejor" de forma concluyente; solo que tolera algo más de carga sin resolver el cuello de botella del hilo único.

<img width="904" height="660" alt="Captura de pantalla 2026-10-25 182101" src="https://github.com/user-attachments/assets/00c78e11-7fce-415a-a1de-7f439caf1e2a" />


<img width="887" height="709" alt="Captura de pantalla 2026-10-25 182112" src="https://github.com/user-attachments/assets/28c60c74-a5e9-40fd-bdf6-4446e5148dce" />


<img width="904" height="688" alt="Captura de pantalla 2026-10-25 182122" src="https://github.com/user-attachments/assets/d126871b-f796-4e12-8496-781e177b49aa" />


<img width="899" height="690" alt="Captura de pantalla 2026-10-25 182131" src="https://github.com/user-attachments/assets/b5d6c794-84a4-46c1-91aa-313ae3c7203f" />


<img width="1576" height="661" alt="Captura de pantalla 2026-10-25 182204" src="https://github.com/user-attachments/assets/1a501a7c-587c-4b73-b05f-77613b82f1b9" />


## Conclusión general

El escalamiento vertical tiene límites claros para esta aplicación: la función es intensiva en CPU y usa un solo hilo, por lo que aumentar vCPU y RAM no la acelera y solo ofrece algo más de holgura. Además implica downtime al reiniciar, un costo mayor y un único punto de falla. Esto motiva la Parte 2: escalar horizontalmente con varias instancias detrás de un balanceador, de modo que cada petición pueda atenderse en una VM distinta.
