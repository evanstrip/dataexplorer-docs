---
title:  scan operator
description: Learn how to use the scan operator to scan data, match, and build sequences based on the predicates.
ms.reviewer: alexans
ms.topic: reference
ms.date: 01/22/2023
---
# scan operator

Scans data, matches, and builds sequences based on the predicates.

Matching records are determined according to predicates defined in the operator’s steps. A predicate can depend on the state that is generated by previous steps.
The output for the matching record is determined by the input record and assignments defined in the operator's steps.

## Syntax

*T* `| scan` [ `with_match_id` `=` *MatchIdColumnName* ] [ `declare` `(` *ColumnDeclarations* `)` ] `with` `(` *StepDefinitions* `)`

### *ColumnDeclarations* syntax

*ColumnName* `:` *ColumnType*[`=` *DefaultValue* ] [`,` ... ]

### *StepDefinition* syntax

`step` *StepName* [ `output` = `all` | `last` | `none`] `:` *Condition* [ `=>` *Column* `=` *Assignment* [`,` ... ] ] `;`

[!INCLUDE [syntax-conventions-note](../../includes/syntax-conventions-note.md)]

## Parameters

| Name | Type | Required | Description |
|--|--|--|--|
| *MatchIdColumnName* | string | |  The name of a column of type `long` that is appended to the output as part of the scan execution. Indicates the 0-based index of the match for the row. |
| *ColumnDeclarations* | string | | Declares an extension to the schema of the operator’s source. Additional columns are assigned in the steps or *DefaultValue* if not assigned. *DefaultValue* is `null` if not specified. |
| *StepName* | string | &check; | Used to reference values in the state of scan for conditions and assignments. The step name must be unique.|
| *Condition* | string | &check; | An expression that evaluates to a `bool`, `true` or `false`, that defines which records from the input matches the step. A record matches the step when the condition is `true` with the step’s state or with the previous step’s state.|
| *Assignment* | string | | A scalar expression that is assigned to the corresponding column when a record matches a step.|
| `output` | string | | Controls the output logic of the step on repeated matches. `all` outputs all records matching the step, `last` outputs only the last record in a series of repeating matches for the step, and `none` does not output records matching the step. The default is `all`.|

## Returns

A record for each match of a record from the input to a step. The schema of the output is the schema of the source extended with the column in the `declare` clause.

## Scan logic

`scan` goes over the serialized input data, record by record, comparing each record against each step’s condition while taking into account the current state of each step.

### Scan's state

The state that is used behind the scenes by `scan` is a set of records, with the same schema of the output, including source and declared columns.
Each step has its own state, the state of step *k* has *k* records in it, where each record in the step’s state corresponds to a step up to *k*.

For example, if a scan operator has *n* steps named *s_1*, *s_2*, ..., *s_n* then step *s_k* would have *k* records in its state corresponding to *s_1*, *s_2*, ..., *s_k*.
Referencing a value in the state is done in the form *StepName*.*ColumnName*. For example, `s_2.col1` references column `col1` that belongs to step *s_2* in the state of *s_k*.

### Matching logic

Each record from the input is evaluated against all of scan’s steps, starting from last to first. When a record *r* is considered against some step *s_k*, the following logic is applied:

* If the state of the previous step isn't empty and the record *r* satisfies the condition of *s_k* using the state of the previous step *s_(k-1)*, then the following happens:
    1. The state of *s_k* is deleted.
    1. The state of *s_(k-1)* becomes ("promoted" to be) the state of *s_k*, and the state of *s_(k-1)* becomes empty.
    1. All the assignments of *s_k* are calculated and extend *r*.
    1. The extended *r* is added to the output (if *s_k* is defined as `output=all`) and to the state of *s_k*.
* If *r* doesn't satisfy the condition of *s_k* with the state of *s_(k-1)*, *r* is then checked with the state of *s_k*. If *r* satisfies the condition of *s_k* with the state of *s_k*, the following happens:
    1. The record *r* is extended with the assignments of *s_k*.
    1. If *s_k* is defined as `output=all`, the extended record r is added to the output.
    1. The last record in the state of *s_k* (which represents *s_k* itself in the state) is replaced by the extended record *r*.
    1. Whenever the first step is matched while its state is empty, a new match begins and the match ID is increased by `1`. This only affects the output when `with_match_id` is used.
* If r doesn't satisfy the condition *s_k* with the state *s_k*, evaluate *r* against condition *s_k-1* and repeat the logic above.

## Examples

### Cumulative sum

Calculate the cumulative sum for an input column. The result of this example is equivalent to using [row_cumsum()](rowcumsumfunction.md).

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA02NOwqAMBBE+5xiSkUQLWyUeBUJcf1ATCTZaAoPb7ByqmEej/HKroSExbsDLdihQ2A6cxcPglYWM2mjPKHQ8YhG8X7RlHrj7CqbEvfOG0QhkPOJoe3BPhLkiL8BmW+qjOv/OojyBSZSTeODAAAA" target="_blank">Run the query</a>

```kusto
range x from 1 to 5 step 1 
| scan declare (cumulative_x:long=0) with 
(
    step s1: true => cumulative_x = x + s1.cumulative_x;
)
```

**Output**

|x|cumulative_x|
|---|---|
|1|1|
|2|3|
|3|6|
|4|10|
|5|15|

### Cumulative sum on multiple columns with a reset condition

Calculate the cumulative sum for two input column, reset the sum value to the current row value whenever the cumulative sum reached 10 or more.

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA22OQQrCMBBF9znFXzZaxAhuKulVJLSTWmhTSdKagIc3FIVo/fzVG+bNWGU6QoC20wgBP+EM5+kOwZ6g4Mm0iJA4YYeQkGuUQUvNoCyhaOZxHpTvF7qGaphMJ48lMhjfkOPR+xtYwZCy+p2o4O1MkHW+EdKtXuvCicMXrSVEcodU7PEz5SVW8Sb5K3/F8SOOqRtx5BfGX1hK/bgiAQAA" target="_blank">Run the query</a>

```kusto
range x from 1 to 5 step 1
| extend y = 2 * x
| scan declare (cumulative_x:long=0, cumulative_y:long=0) with 
(
    step s1: true => cumulative_x = iff(s1.cumulative_x >= 10, x, x + s1.cumulative_x), 
                     cumulative_y = iff(s1.cumulative_y >= 10, y, y + s1.cumulative_y);
)
```

**Output**

|x|y|cumulative_x|cumulative_y|
|---|---|---|---|
|1|2|1|2|
|2|4|3|6|
|3|6|6|12|
|4|8|10|8|
|5|10|5|18|

### Fill forward a column

Fill forward a string column. Each empty value is assigned the last seen non-empty value.

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA01Qy2rDMBC86ysGnSwwoW5DCAkuNI8/yK2EoNjrViC7xrtNCPTjK8sOaPcyo50dRutJcLxRJ4wStZXQV0/ITryBuJa4t10+KTZgGVz3ZfCpEOqlzaE/dB5JMZIZv454N5O3ZLBM8GrE+5msk0ERrQ5PFs20OqutmnKqP/DPILg+cGJYrsaHynaoqfJ2CNmj7tI476l+hi61Nrg7+YbKojEL9eAi/HL4JZTvSLfCLVzTZI6p7eUxGZo8yBepar6L2SrzD0ocTu9HAQAA" target="_blank">Run the query</a>

```kusto
let Events = datatable (Ts: timespan, Event: string) [
    0m, "A",
    1m, "",
    2m, "B",
    3m, "",
    4m, "",
    6m, "C",
    8m, "",
    11m, "D",
    12m, ""
]
;
Events
| sort by Ts asc
| scan declare (Event_filled: string="") with 
(
    step s1: true => Event_filled = iff(isempty(Event), s1.Event_filled, Event);
)
```

**Output**

|Ts|Event|Event_filled|
|---|---|---|
|00:00:00|A|A|
|00:01:00||A|
|00:02:00|B|B|
|00:03:00||B|
|00:04:00||B|
|00:06:00|C|C|
|00:08:00||C|
|00:11:00|D|D|
|00:12:00||D|

### Sessions tagging

Divide the input into sessions: a session ends 30 minutes after the first event of the session, after which a new session starts. Note the use of `with_match_id` flag which assigns a unique value for each distinct match (session) of *scan*. Also note the special use of two *steps* in this example, `inSession` has `true` as condition so it captures and outputs all the records from the input while `endSession` captures records that happen more than 30m from the `sessionStart` value for the current match. The `endSession` step has `output=none` meaning it doesn't produce output records. The `endSession` step is used to advance the state of the current match from `inSession` to `endSession`, allowing a new match (session) to begin, starting from the current record.

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA3WQy2rDMBBF9/qKS1Y2qCVt+sLGgT7yBcmulKDYSiOQZaMZtxT68ZVsp3EWkTY6M/fOQ1YzVl/aMaFApTjcndVINpSBTa2pVU4OigzE3rjPFO8C4cxridnzTPZwM4XbCC8jLCK8HeE89RDp9UhPU+VdX3E10uP9UF98iFwM84pfUOMZux9sCIrKGCiVw7fhw7ZWXB62pipIE5nGhScqXVrlw3JjbM3K82nNtHdCJH1HYt3CuPUgDSrfaRRLTL3hy8x+nxhynbXJv/h6qkllGE/iQjI/9dKuGiVoOm47LlzjdBaXu7pgxxKLeZ2L9A9Yk5hPxgEAAA==" target="_blank">Run the query</a>

```kusto
let Events = datatable (Ts: timespan, Event: string) [
    0m, "A",
    1m, "A",
    2m, "B",
    3m, "D",
    32m, "B",
    36m, "C",
    38m, "D",
    41m, "E",
    75m, "A"
]
;
Events
| sort by Ts asc
| scan with_match_id=session_id declare (sessionStart: timespan) with 
(
    step inSession: true => sessionStart = iff(isnull(inSession.sessionStart), Ts, inSession.sessionStart);
    step endSession output=none: Ts - inSession.sessionStart > 30m;
)
```

**Output**

|Ts|Event|sessionStart|session_id|
|---|---|---|---|
|00:00:00|A|00:00:00|0|
|00:01:00|A|00:00:00|0|
|00:02:00|B|00:00:00|0|
|00:03:00|D|00:00:00|0|
|00:32:00|B|00:32:00|1|
|00:36:00|C|00:32:00|1|
|00:38:00|D|00:32:00|1|
|00:41:00|E|00:32:00|1|
|01:15:00|A|01:15:00|2|

### Events between Start and Stop

Find all sequences of events between the event `Start` and the event `Stop` that occur within 5 minutes. Assign a match ID for each sequence.

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA3WPYWvCMBCGv+dXvPOTQidr3cZozYc5/QX6TYbENsxCE0vvcAj+eJM0FR0sgRzPe3fvXRrNWJ20ZYJEpdjdfaMx3lAOro2mVtmkr8hB3NX2Z4KtgDsvJsHoc5QESD2sWXUchcwLiwgzD8sIr33psY387vkrwsdfnzQ4rwbKbs3iWxSiX11cQMeOsT9jQ1BUeqFUFr81H3ZGcXnY1ZU07gkSxDi4EesWlOb9/yDlMLq4S2dD+umWhrLVg+jWCZqb/uwMpy7OJd7Mvc/sccz/LZMrJ20JNJQBAAA=" target="_blank">Run the query</a>

```kusto
let Events = datatable (Ts: timespan, Event: string) [
    0m, "A",
    1m, "Start",
    2m, "B",
    3m, "D",
    4m, "Stop",
    6m, "C",
    8m, "Start",
    11m, "E",
    12m, "Stop"
]
;
Events
| sort by Ts asc
| scan with_match_id=m_id with 
(
    step s1: Event == "Start";
    step s2: Event != "Start" and Event != "Stop" and Ts - s1.Ts <= 5m;
    step s3: Event == "Stop" and Ts - s1.Ts <= 5m;
)
```

**Output**

|Ts|Event|m_id|
|---|---|---|
|00:01:00|Start|0|
|00:02:00|B|0|
|00:03:00|D|0|
|00:04:00|Stop|0|
|00:08:00|Start|1|
|00:11:00|E|1|
|00:12:00|Stop|1|

### Calculate a custom funnel of events

Calculate a funnel completion of the sequence  `Hail` -> `Tornado` -> `Thunderstorm Wind` by `State` with custom thresholds on the times between the events (`Tornado` within `1h` and `Thunderstorm Wind` within `2h`). This example is similar to the [funnel_sequence_completion plugin](funnel-sequence-completion-plugin.md), but allows greater flexibility.

> [!div class="nextstepaction"]
> <a href="https://dataexplorer.azure.com/clusters/help/databases/Samples?query=H4sIAAAAAAAAA2VQsa7CMBDb+QqrEwwgPUZ4HZHYqcR8NNHLSfRSJVeeivh40lBBgQw32D774oP60OwuVjTObmgpKCt7gWPRVdRAav/6Ukj5YnHqcdCEYIb05nlGH3QkglbcWFCsM3NDrEnwz+qmC3lJbQtHfN4gZ1d9a1GWKPYJK7bvunShkPGf0uoBFyAxk/Rl9l29gN8SP+7T0nVibIjD548s5st7wmMQfKeMV70HrcegxThTBV3TUOCrhal9JzrPBS6Gxp6RdzkEW8CEAQAA" target="_blank">Run the query</a>

```kusto
StormEvents
| partition hint.strategy=native by State 
    (
    sort by StartTime asc
    | scan with 
    (
        step hail: EventType == "Hail";
        step tornado: EventType == "Tornado" and StartTime - hail.StartTime <= 1h;
        step thunderstormWind: EventType == "Thunderstorm Wind" and StartTime - tornado.StartTime <= 2h;
    )
    )
| summarize dcount(State) by EventType
```

**Output**

|EventType|dcount_State|
|---|---|
|Hail|50|
|Tornado|34|
|Thunderstorm Wind|32|
