# Challenge Infraestructura
Una aplicación python que funciona en kubernetes utiliza 6 pods en un cluster de 8 nodos La aplicación utiliza una base de datos RDS postgres. El tamaño de las tablas de la base de datos es de 700GB. Seis de las cuales ocupan el 85% del espacio.
Se puede acceder a los servidores, logs de las aplicaciones y logs de la base de datos.
Se manifiesta una degradación en los tiempos de respuesta de la aplicación. Recientemente se implementaron nuevas funcionalidades solicitadas por un nuevo Operador (cliente)
En los últimos días también hubo un incremento del uso de la aplicación (se triplicó la carga habitual) a raíz del inicio de actividad de dicho Operador.

1) Elaborar un camino para obtener un diagnóstico ante el problema planteado.

2) Elegir dos de los posibles diagnósticos (uno que involucre a la base de datos y otro
que no) y definir soluciones/mejoras alternativas


1)

# RDS
Analizaria en el servicio RDS:

● Las metricas (cpu, memoria, storage) del servicio de RDS para detectar la fecha en la que comenzó el incidente.

● Ya con el dia y la hora detectada revisaría en la consola de RDS si Performance Insights esta activado. Con esto podemos ver rapidamente las queries que mas costo estan teniendo, ademas del usuario que las esta ejecutando.
Si desde la consola no se detecta facilmente, podemos ingresar en la base de datos de Postgres y hacer un select contra la tabla pg_stat_statements ordenando por total_time.
Con esto ya obtenemos la query y el usuario que esta afectando los tiempos de respuesta.

● Deberiamos ejecutar un explain plan de la query para ver el costo de la misma: 
Si el plan hace un full scan, la tabla podria necesitar un indice.
Tambien deberiamos ver si los join estan bien armados o si se deberia reformular la consulta. 
Si es una tabla muy grande como en este caso, podriamos limitar el numero de los registros con un LIMIT. 

● Otro de los posibles incidentes puede ser que la tabla este siendo lockeada, en ese caso habria que detectar cual es el proceso que la esta lockeando y proceder a killearlo siempre y cuando sea posible. Existen queries para detectar estos lockeos que se crean haciendo un join entre las tablas pg_stat_activity y pg_locks.


# Kubernetes 
En cuanto al cluster de kubernetes revisaria:

● Las metricas (kubectl top) para monitorear los recursos de los pods y los nodos del clúster. Si los pods no tienen suficientes recursos de CPU y memoria asignados, pueden experimentar cuellos de botella que afecten el rendimiento de la app.
● El escalamiento de los pods si son adecuados para la nueva carga que se implemento. El aumento en la carga de la app puede haber superado la capacidad asignada al cluster.
● Los events del cluster (kubectl get events)
● Los pods. Los mismos pueden estar crasheando si estan sobrecargados y el cluster no logra escalar. 


2) 

# RDS

Suponiendo que el incidente sea generado por una nueva query con un alto costo lo que haria seria:

● Asegurarme que cada aplicacion tenga su propio usuario en la db para poder detectar facilmente cual esta afectando.
● Analizaria el costo de la query ejecutando un explain analyze. Si la misma necesita un indice ver la posibilidad de crearlo. (Esto requiere un analisis y seguramente en ambientes productivos se necesite una ventana. La creacion de un indice puede generar lockeos) 
● Evaluaria particionar las tablas con muchos registros para asi aliviar las consultas. Tambie deberian tener un mantenimiento con frecuencia de vaccum analyze de al menos cada 3 meses como para ordenar los blocks.
● Si no es posible realizar mantenimiento, se podria crear instancias de solo lectura en RDS para que asi poder disminuir la carga en la instancia "writer" reduciendo el riesgo de poder tener respuestas lentas y caidas.
● Finalmente si no se pudo reducir los tiempos, habria que agregarle mas recursos a la instancia de RDS, optando por un instance type mayor.

# Kubernetes:

Suponiendo que tengamos un problema de escalamiento en los pods lo que haria seria:

● Revisaria los logs de los nuevos pods para descartar errores de codigo.
● Aumentaria la utilizacion de los recursos (memoria, cpu) en los pods editando el HPA (HorizontalPodAutoscaler) para que los mismos se escalen automaticamente. 
● Tambien podriamos probar aumentando el numero de replicas de los pods. Es muy probable que con el nuevo consumo, necesitemos mas pods para dividir la carga.
