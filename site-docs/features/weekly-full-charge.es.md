# Carga semanal completa

Carga las baterÃ­as al **100 % una vez por semana** para equilibrar las celdas y mantener la salud de la baterÃ­a (cell balancing).

## Comportamiento

1. El dÃ­a configurado de la semana, si el SOC mÃ¡ximo habitual es inferior al 100 %, la integraciÃ³n eleva temporalmente el lÃ­mite de corte de carga al 100 %.
2. La baterÃ­a carga hasta que todas las baterÃ­as disponibles alcanzan el 100 % de SOC o el BMS corta claramente la carga cerca de la parte alta.
3. Una vez alcanzada la parte alta, la integraciÃ³n inicia balanceo activo en lugar de volver inmediatamente al SOC mÃ¡ximo configurado.
4. El balanceo activo usa el perfil por tension de celda documentado en [Monitor de equilibrio de celdas](cell-balance-monitor.md): carga a 50 W hasta 3.55 V, espera de 60 s para medir balance y descarga a 25 W.
5. La carga semanal termina tras la descarga final a 25 W hasta que `max_cell_voltage` baja a 3.42 V.
6. Tras finalizar, el lÃ­mite de SOC mÃ¡ximo vuelve automÃ¡ticamente al valor configurado por el usuario.

Si los datos de tensiÃ³n de celda no estÃ¡n disponibles para una baterÃ­a durante la fase de balanceo, esa baterÃ­a queda a 0 W hasta que los datos vuelvan o finalice la ventana de 4 horas.

## Monitor de equilibrio de celdas

El paso de configuraciÃ³n de carga semanal completa incluye una opciÃ³n para activar el **monitor de equilibrio de celdas**. Cuando estÃ¡ activo, la integraciÃ³n mide la diferencia de tensiÃ³n entre la celda mÃ¡s y menos cargada despuÃ©s de cada carga completa, para hacer seguimiento de la salud de la baterÃ­a a lo largo del tiempo.

Consulta [Monitor de equilibrio de celdas](cell-balance-monitor.md) para mÃ¡s detalles.

## InteracciÃ³n con el retraso de carga solar

Si el [retraso de carga solar](solar-charge-delay.md) estÃ¡ activo, la carga semanal se postpone mientras la producciÃ³n solar prevista sea suficiente para alcanzar el 100 %. La baterÃ­a solo empieza a cargar desde la red cuando el modelo solar determina que el sol no completarÃ¡ la carga.

Cuando el monitor de equilibrio de celdas estÃ¡ activado, el retraso de carga solar se omite automÃ¡ticamente el dÃ­a de la carga semanal para que la baterÃ­a pueda alcanzar la parte alta y ejecutar la fase de balanceo activo antes de tomar la lectura OCV.

## Registro Modbus implicado

La funciÃ³n manipula el registro **44000** (charging cutoff) de la baterÃ­a para elevar temporalmente el lÃ­mite.

!!! info
    Esta funciÃ³n estÃ¡ disponible para todas las versiones de baterÃ­a compatibles (v2, v3, vA, vD).

![ConfiguraciÃ³n de carga semanal completa](../assets/screenshots/features/weekly-full-charge-config.png){ width="650"  style="display: block; margin: 0 auto;"}
