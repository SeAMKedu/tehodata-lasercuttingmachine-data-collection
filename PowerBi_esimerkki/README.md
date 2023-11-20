# Esimerkki tuloksesta, joka saatiin aikaan.

Tiedostoissa on Power BI raportti "Aikalaskenta 2023.pbix" sekä JSON -tiedosto johon on kerätty dataa Raspberry PI -laitteesta kytkimien avulla.

## Esimerkki raportin testaaminen


Avatessa Power BI esimerkki raportin täytyy avata yläpalkista Transfer Data / Muunna tiedot painikkeesta Power Query Editori ![transfer data](https://github.com/SeAMKedu/tehodata-lasercuttingmachine-data-collection/assets/35451517/488caf89-7e35-4ee7-8971-5d90b212620f) 

Advanced Editorista pääset muokkaamaan M-Query tiedostoa jossa on listattu eri asetuksia. ![image](https://github.com/SeAMKedu/tehodata-lasercuttingmachine-data-collection/assets/35451517/563d0ade-23e7-4ee4-ba91-8e6b9ab5353d)

Esiin tulee ruutu, jossa on määritetty `Json.Document(File.Contents("C:\Users\SaKa\Desktop\sshVSC\jsonBackupMachine1.json"))` Tänne muokkaamalla ladatun json tiedoston polun, voidaan tuoda data esiin ja raporttia pystytään käsittelemään.
```
let
    jsonBackupMachine1 = let
    // Reading and initial processing of the JSON file
    Source = Json.Document(File.Contents("C:\Users\SaKa\Desktop\sshVSC\jsonBackupMachine1.json")),
    ConvertedToTable = Table.FromList(Source, Splitter.SplitByNothing(), null, null, ExtraValues.Error),
    ExpandedColumn1 = Table.ExpandRecordColumn(ConvertedToTable, "Column1", {"Machine ID", "Start", "End", "Duration", "isFault"}, {"machine_id", "Start", "End", "Duration", "isFault"}),
    CorrectedDuration = Table.TransformColumns(ExpandedColumn1, {"Duration", each Duration.FromText(_), type duration}),

    // Your existing transformations...
    RemovedErrors = Table.RemoveRowsWithErrors(CorrectedDuration, {"Duration", "Start", "End"}),
    SortedRows = Table.Sort(RemovedErrors,{{"Start", Order.Descending}}),
    
    // Adding the new steps
    ChgType = Table.TransformColumnTypes(#"SortedRows", {{"Start", type datetime}, {"End", type datetime}}),
    #"Removed Errors1" = Table.RemoveRowsWithErrors(ChgType, {"Start"}),
    #"Removed Errors" = Table.RemoveRowsWithErrors(#"Removed Errors1", {"Start"}),
    ListOfDates = Table.AddColumn(#"Removed Errors", "Dates", each List.Transform({Number.From(Date.From([Start]))..Number.From(Date.From([End]))}, Date.From)),
    ListOfDurations = Table.AddColumn(ListOfDates, "Durations", each 
    let
        StartDate = Date.From([Start]),
        EndDate = Date.From([End]),
        StartDateTime = DateTime.From([Start]),
        EndDateTime = DateTime.From([End]),
        DateRange = [Dates],
        DurationList = List.Transform(DateRange, (DateValue) => 
            if DateValue = StartDate and DateValue = EndDate then EndDateTime - StartDateTime
            else if DateValue = StartDate then #duration(1,0,0,0) - (StartDateTime - DateTime.From(StartDate))
            else if DateValue = EndDate then EndDateTime - DateTime.From(EndDate)
            else #duration(1,0,0,0)
        )
    in DurationList
),

Result = Table.AddColumn(ListOfDurations, "Result", each Table.FromColumns({[Dates], [Durations]}, {"Date", "Duration"})),

ExpandedResult = Table.ExpandTableColumn(Result, "Result", {"Date", "Duration"}, {"Result.Date", "Result.Duration"}),
    #"Sorted Rows" = Table.Sort(ExpandedResult,{{"Result.Date", Order.Descending}}),
    #"Changed Type2" = Table.TransformColumnTypes(#"Sorted Rows",{{"Result.Duration", type duration}}),
    #"Duplicated Column5" = Table.DuplicateColumn(#"Changed Type2", "Result.Date", "Result.Date - Copy"),
    #"Renamed Columns" = Table.RenameColumns(#"Duplicated Column5",{{"Result.Date - Copy", "Result.DateValue"}}),
    #"Changed Type3" = Table.TransformColumnTypes(#"Renamed Columns",{{"Result.DateValue", type date}}),
    #"Duplicated Column4" = Table.DuplicateColumn(#"Changed Type3", "Result.Duration", "Result.Duration - Copy"),
    #"Changed Type4" = Table.TransformColumnTypes(#"Duplicated Column4",{{"Result.Duration - Copy", type duration}})
in
    #"Duplicated Column4",
    #"Filtered Rows" = Table.SelectRows(jsonBackupMachine1, each true)
in
    #"Filtered Rows"
```
