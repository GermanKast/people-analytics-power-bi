# Medidas DAX --- People Analytics Dashboard

Este documento describe las medidas DAX implementadas en
`NovaTalento_People_Analytics.pbix`. El proyecto utiliza datos ficticios
y fue desarrollado como ejercicio práctico de portafolio.

## Absent Workdays

``` dax
Absent Workdays =
SUM ( AbsencesTable[workdays_absent] )
```

Suma los días laborables registrados como ausencia dentro del contexto
de filtros actual. Constituye el numerador de la tasa de absentismo.

## Scheduled Workdays

``` dax
Scheduled Workdays =
SUM ( MonthlyScheduleTable[scheduled_workdays] )
```

Suma los días laborables programados dentro del contexto actual.
Funciona como denominador de la tasa de absentismo.

## Absenteeism Rate

``` dax
Absenteeism Rate =
DIVIDE (
    [Absent Workdays],
    [Scheduled Workdays]
)
```

Calcula la proporción de días laborables ausentes respecto de los días
laborables programados:

``` text
Tasa de absentismo = Días laborables de ausencia / Días laborables programados
```

En este proyecto se utiliza una referencia corporativa ficticia de **≤
3,00 %**. `DIVIDE` permite manejar de forma segura denominadores iguales
a cero o vacíos.

## Headcount

``` dax
Headcount =
VAR CutoffDate =
    MAX ( CalendarTable[month_end] )
RETURN
CALCULATE (
    DISTINCTCOUNT ( ContractsTable[employee_id] ),
    REMOVEFILTERS ( CalendarTable ),
    CROSSFILTER (
        CalendarTable[month_start],
        ContractsTable[hire_month],
        NONE
    ),
    FILTER (
        ALL (
            ContractsTable[start_date],
            ContractsTable[end_date]
        ),
        ContractsTable[start_date] <= CutoffDate
            &&
        (
            ISBLANK ( ContractsTable[end_date] )
                || ContractsTable[end_date] > CutoffDate
        )
    )
)
```

Calcula el número de empleados con contrato vigente en la fecha de corte
correspondiente al final del período seleccionado.

Un empleado se considera activo cuando `start_date <= fecha de corte` y
`end_date` está vacío o es posterior a la fecha de corte.

La medida desactiva temporalmente la propagación del filtro del
calendario hacia `hire_month`, porque el headcount es un indicador de
estado (*snapshot*) y no debe limitarse a quienes fueron contratados
durante el período seleccionado.

## Headcount Start

``` dax
Headcount Start =
VAR StartDate =
    MIN ( CalendarTable[month_start] )
VAR PreviousDate =
    StartDate - 1
RETURN
CALCULATE (
    DISTINCTCOUNT ( ContractsTable[employee_id] ),
    REMOVEFILTERS ( CalendarTable ),
    CROSSFILTER (
        CalendarTable[month_start],
        ContractsTable[hire_month],
        NONE
    ),
    FILTER (
        ALL (
            ContractsTable[start_date],
            ContractsTable[end_date]
        ),
        ContractsTable[start_date] <= PreviousDate
            &&
        (
            ISBLANK ( ContractsTable[end_date] )
                || ContractsTable[end_date] > PreviousDate
        )
    )
)
```

Calcula los empleados activos inmediatamente antes del inicio del
período seleccionado. Se utiliza para obtener la plantilla promedio y
medir el crecimiento neto de la plantilla.

## Average Headcount

``` dax
Average Headcount =
DIVIDE (
    [Headcount Start] + [Headcount],
    2
)
```

Calcula:

``` text
Plantilla promedio = (Plantilla inicial + Plantilla final) / 2
```

Se utiliza como denominador de las tasas de rotación y contratación.

## Hires

``` dax
Hires =
CALCULATE (
    DISTINCTCOUNT ( ContractsTable[contract_id] ),
    NOT ISBLANK ( ContractsTable[hire_month] )
)
```

Cuenta los contratos de contratación dentro del contexto temporal
actual. El modelo utiliza la relación activa entre el calendario y
`ContractsTable[hire_month]`.

**Nota:** al contar `contract_id`, la medida representa técnicamente
eventos o contratos de contratación, aunque en el dataset actual el
resultado pueda coincidir con el número de personas contratadas.

## Hiring Rate

``` dax
Hiring Rate =
DIVIDE (
    [Hires],
    [Average Headcount]
)
```

Calcula:

``` text
Tasa de contratación = Contrataciones / Plantilla promedio
```

Permite comparar el volumen de contratación entre períodos con diferente
tamaño de plantilla.

## Terminations

``` dax
Terminations =
CALCULATE (
    DISTINCTCOUNT ( ContractsTable[contract_id] ),
    CROSSFILTER (
        CalendarTable[month_start],
        ContractsTable[hire_month],
        NONE
    ),
    USERELATIONSHIP (
        CalendarTable[month_start],
        ContractsTable[termination_month]
    ),
    NOT ISBLANK ( ContractsTable[termination_month] )
)
```

Cuenta los contratos finalizados durante el período seleccionado.

`CROSSFILTER` desactiva temporalmente la relación utilizada para
contrataciones y `USERELATIONSHIP` activa la relación con
`termination_month`. Así, las salidas se analizan según su fecha de
terminación.

## Turnover Rate

``` dax
Turnover Rate =
DIVIDE (
    [Terminations],
    [Average Headcount]
)
```

Calcula:

``` text
Tasa de rotación = Salidas / Plantilla promedio
```

En el proyecto se utiliza una referencia corporativa ficticia de **≤
2,50 % mensual**.

Una tasa elevada constituye una señal para profundizar el análisis, pero
por sí sola no permite determinar las causas de la rotación.

## Average Tenure Years

``` dax
Average Tenure Years =
VAR CutoffDate =
    MAX ( CalendarTable[month_end] )
RETURN
AVERAGEX (
    FILTER (
        ALL ( ContractsTable ),
        ContractsTable[start_date] <= CutoffDate
            &&
        (
            ISBLANK ( ContractsTable[end_date] )
                || ContractsTable[end_date] > CutoffDate
        )
    ),
    DIVIDE (
        DATEDIFF (
            ContractsTable[start_date],
            CutoffDate,
            DAY
        ),
        365.25
    )
)
```

Calcula la antigüedad promedio, expresada en años, de los contratos
activos en la fecha de corte. Para cada contrato vigente calcula los
días transcurridos desde `start_date` y los divide entre `365.25`.

**Nota técnica:** `ALL(ContractsTable)` elimina filtros aplicados
directamente sobre la tabla de contratos al construir la población
evaluada. Esto debe considerarse si en el futuro se desea que la medida
responda a nuevos filtros provenientes directamente de esa tabla.

## Active Employees by Tenure

``` dax
Active Employees by Tenure =
VAR CutoffDate =
    MAX(CalendarTable[month_end])

VAR MinYears =
    SELECTEDVALUE(TenureRanges[MinYears])

VAR MaxYears =
    SELECTEDVALUE(TenureRanges[MaxYears])

RETURN
CALCULATE(
    DISTINCTCOUNT(ContractsTable[employee_id]),

    CROSSFILTER(
        CalendarTable[month_start],
        ContractsTable[hire_month],
        NONE
    ),

    REMOVEFILTERS(CalendarTable),

    FILTER(
        ALL(
            ContractsTable[start_date],
            ContractsTable[end_date]
        ),
        VAR StartDate = ContractsTable[start_date]
        VAR EndDate = ContractsTable[end_date]

        VAR TenureYears =
            DIVIDE(
                DATEDIFF(StartDate, CutoffDate, DAY),
                365.25
            )

        RETURN
            StartDate <= CutoffDate
                && (
                    ISBLANK(EndDate)
                    || EndDate > CutoffDate
                )
                && TenureYears >= MinYears
                && TenureYears < MaxYears
    )
)
```

Distribuye los empleados activos entre los rangos definidos en
`TenureRanges`. Obtiene los límites inferior y superior mediante
`SELECTEDVALUE`, calcula la antigüedad a la fecha de corte y conserva
los empleados que pertenecen al rango correspondiente.

La tabla `TenureRanges` permite además controlar el orden lógico de las
categorías y evitar un orden alfabético incorrecto.

## Workforce Growth

``` dax
Workforce Growth =
DIVIDE (
    [Headcount] - [Headcount Start],
    [Headcount Start]
)
```

Calcula:

``` text
Crecimiento de plantilla =
(Plantilla final - Plantilla inicial) / Plantilla inicial
```

Un resultado positivo representa crecimiento neto, uno negativo
reducción neta y cero indica que la plantilla inicial y final tienen el
mismo tamaño. No debe confundirse con la tasa de contratación ni con la
tasa de rotación.

## Contexto de filtro

Una parte importante del modelo consiste en distinguir indicadores de
**flujo** de indicadores de **estado**.

Las contrataciones y terminaciones representan eventos ocurridos durante
un período. El headcount representa el estado de la plantilla en una
fecha determinada.

Por esta razón, algunas medidas modifican explícitamente el contexto de
filtro mediante funciones como:

-   `CALCULATE`
-   `FILTER`
-   `REMOVEFILTERS`
-   `CROSSFILTER`
-   `USERELATIONSHIP`
-   `ALL`

Esto evita, por ejemplo, que al seleccionar agosto de 2026 el cálculo de
empleados activos considere únicamente a quienes fueron contratados
durante agosto.

## Validación

Para **agosto de 2026**, las principales medidas del dashboard
presentan:

  Indicador               Resultado
  --------------------- -----------
  Empleados activos             415
  Contrataciones                  9
  Salidas                         9
  Tasa de absentismo         3,17 %
  Tasa de rotación           2,17 %
  Antigüedad promedio     2,41 años

La distribución por rangos de antigüedad suma igualmente **415 empleados
activos**, lo que proporciona una comprobación adicional de consistencia
para ese período.

## Consideraciones de interpretación

Las referencias de **3,00 % para absentismo** y **2,50 % mensual para
rotación** son parámetros ficticios definidos exclusivamente para este
ejercicio de portafolio y no se presentan como estándares universales de
People Analytics.

Los indicadores permiten identificar comportamientos y áreas que
requieren análisis adicional, pero no deben utilizarse de forma aislada
para atribuir causas.

------------------------------------------------------------------------

[Volver al README principal](../README.md)
