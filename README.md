# GoogleSheetsWrapper
[![Nuget](https://img.shields.io/nuget/v/GoogleSheetsWrapper)](https://www.nuget.org/packages/GoogleSheetsWrapper/) [![GitHub](https://img.shields.io/github/license/SteveWinward/GoogleSheetsWrapper)](LICENSE) ![build status](https://github.com/SteveWinward/GoogleSheetsWrapper/actions/workflows/dotnet.yml/badge.svg)

## Google Sheets API .NET Wrapper Library

This library allows you to use strongly typed objects against a Google Sheets spreadsheet without having to have knowledge on the Google Sheets API methods and protocols. 

The following Google Sheets API operations are supported: 

* Reading all rows
* Appending new rows
* Deleting rows
* Updating specific cells

All operations above are encapsulated in the SheetHelper class. 

There are also base classes, BaseRecord and BaseRepository to simplify transforming raw Google Sheets rows into .NET objects. 

A really simple console application using this library is included in this project below,

[GoogleSheetsWrapper.SampleClient Project](src/GoogleSheetsWrapper.SampleClient/Program.cs)

To setup the sample application, you also need to configure [User Secrets in Visual Studio](https://docs.microsoft.com/en-us/aspnet/core/security/app-secrets?view=aspnetcore-5.0&tabs=windows) to run it locally.

## Extend BaseRecord and BaseRepository

Extending the BaseRecord class you can decorate properties with the SheetFieldAttribute to describe the column header name, the column index and the field type (ie string, DateTime, etc)

> The column index is 1 based and not 0 based. The first column 'A' is equivalent to the column ID of 1. 

```csharp
public class TestRecord : BaseRecord
{
    [SheetField(
        DisplayName = "Name",
        ColumnID = 1,
        FieldType = SheetFieldType.String)]
    public string Name { get; set; }

    [SheetField(
        DisplayName = "Number",
        ColumnID = 2,
        FieldType = SheetFieldType.PhoneNumber)]
    public long? PhoneNumber { get; set; }

    [SheetField(
        DisplayName = "Price Amount",
        ColumnID = 3,
        FieldType = SheetFieldType.Currency)]
    public double? PriceAmount { get; set; }

    [SheetField(
        DisplayName = "Date",
        ColumnID = 4,
        FieldType = SheetFieldType.DateTime)]
    public DateTime? DateTime { get; set; }

    [SheetField(
        DisplayName = "Quantity",
        ColumnID = 5,
        FieldType = SheetFieldType.Number)]
    public double? Quantity { get; set; }

    public TestRecord() { }

    public TestRecord(IList<object> row, int rowId, int minColumnId = 1)
        : base(row, rowId, minColumnId)
    {
    }
}
```

Extending the BaseRepository allows you to define your own access layer to the Google Sheets tab you want to work with. 

```csharp
public class TestRepository : BaseRepository<TestRecord>
{
    public TestRepository() { }

    public TestRepository(SheetHelper<TestRecord> sheetsHelper)
        : base(sheetsHelper) { }
}
```

## Core Operations  

Before you run the following code you will need to setup a Google Service Account and create a service account key.  You also need to decide how to store your environment variables and secrets (ie the Google service account key)

More details can be found [here](#authentication)

```csharp
// You need to implement your own configuration management solution here!
var settings = AppSettings.FromEnvironment();

// Create a SheetHelper class for the specified Google Sheet and Tab name
var sheetHelper = new SheetHelper<TestRecord>(
    settings.GoogleSpreadsheetId,
    settings.GoogleServiceAccountName,
    settings.GoogleMainSheetName);

sheetHelper.Init(settings.JsonCredential);

// Get all rows from the sheet starting with the 1st row, between the 1st and 8th columns
// Leaving the last row blank tells Google Sheets to return all rows from the Sheet
var allRows = sheetHelper.GetRows("TabName", 1, 1, 8);

// Create a Repository for the TestRecord class
var repository = new TestRepository(sheetHelper);

// Validate that the header names match the expected format defined with the SheetFieldAttribute values
var result = repository.ValidateSchema();

if(!result.IsValid)
{
    throw new ArgumentException(result.ErrorMessage);
}

// Get all rows from the Google Sheet
var allRecords = repository.GetAllRecords();

// Get the first record
var firstRecord = allRecords.First();

// Update the PriceAmount field and save it back to Google Sheets
firstRecord.PriceAmount = 99.99;
repository.SaveField(firstRecord, (r) => r.PriceAmount);

// Delete the first record from Google Sheets
repository.DeleteRecord(firstRecord);

// Add a new record to the end of the Google Sheets tab
repository.AddRecord(new TestRecord()
{
    Name = "John",
    Number = 2021112222,
    PriceAmount = 250.50,
    Date = DateTime.Now(),
    Quantity = 123
});

```

## Append a CSV to Google Sheets

```csharp
var appender = new SheetAppender(
    settings.GoogleSpreadsheetId, 
    settings.GoogleServiceAccountName, 
    settings.GoogleMainSheetName);

appender.Init(settings.JsonCredential);

var filepath = @"C:\Input\input.csv";

using (var stream = new FileStream(filepath, FileMode.Open))
{
    // Append the csv file to Google sheets, include the header row 
    // and wait 1000 milliseconds between batch updates 
    // Google Sheets API throttles requests per minute so you may need to play
    // with this setting. 
    appender.AppendCsv(
        stream, // The CSV FileStrem 
        true, // true indicating to include the header row
        1000); // 1000 milliseconds to wait every 100 rows that are batch sent to the Google Sheets API
}

```
## Import a CSV to Google Sheets and Purge Existing Rows

If you wanted to delete the existing rows in your tab first, you can use
```SheetHelper``` methods to do that.  Below is a sample of that,

```csharp
// Create a SheetHelper class
var sheetHelper = new SheetHelper(
    settings.GoogleSpreadsheetId,
    settings.GoogleServiceAccountName,
    settings.GoogleMainSheetName);

sheetHelper.Init(settings.JsonCredential);

// Get the total row count for the existing sheet
var rows = sheetHelper.GetRows(new SheetRange("", 1, 1, 1));

// Delete all of the rows
sheetHelper.DeleteRows(1, rows.Count);

// Create the SheetAppender class
var appender = new SheetAppender(sheetHelper);

var filepath = @"C:\Input\input.csv";

using (var stream = new FileStream(filepath, FileMode.Open))
{
    // Append the csv file to Google sheets, include the header row 
    // and wait 1000 milliseconds between batch updates 
    // Google Sheets API throttles requests per minute so you may need to play
    // with this setting. 
    appender.AppendCsv(
        stream, // The CSV FileStrem 
        true, // true indicating to include the header row
        1000); // 1000 milliseconds to wait every 100 rows that are batch sent to the Google Sheets API
}

```

## Export Google Sheet to CSV

```csharp
var exporter = new SheetExporter(
    settings.GoogleSpreadsheetId, 
    settings.GoogleServiceAccountName, 
    settings.GoogleMainSheetName);

exporter.Init(settings.JsonCredential);

var filepath = @"C:\Output\output.csv";

using (var stream = new FileStream(filepath, FileMode.Create))
{
    // Query the range A1:G (ie 1st column, 1st row, 8th column and last row in the sheet)
    var range = new SheetRange("TAB_NAME", 1, 1, 8);
    exporter.ExportAsCsv(range, stream);
}

```

## Export Google Sheet to Excel File

```csharp
var exporter = new SheetExporter(
    settings.GoogleSpreadsheetId, 
    settings.GoogleServiceAccountName, 
    settings.GoogleMainSheetName);

exporter.Init(settings.JsonCredential);

var filepath = @"C:\Output\output.xlsx";

using (var stream = new FileStream(filepath, FileMode.Create))
{
    // Query the range A1:G (ie 1st column, 1st row, 8th column and last row in the sheet)
    var range = new SheetRange("TAB_NAME", 1, 1, 8);
    exporter.ExportAsExcel(range, stream);
}

```

## Authentication
You need to setup a Google API Service Account before you can use this library.  

1. Create a service account.  Steps to do that are documented below,

    https://cloud.google.com/docs/authentication/production#create_service_account

2. After you download the JSON key, you need to decide how you want to store it and load it into the application.  

3. Use the service account identity that is created and add that email address to grant it permissions to the Google Sheets Spreadsheet you want to interact with.

4. Configure your code with the following parameters to initialize a SheetHelper object

```csharp
// You need to implement your own configuration management solution here!
var settings = AppSettings.FromEnvironment();

// Create a SheetHelper class for the specified Google Sheet and Tab name
var sheetHelper = new SheetHelper<TestRecord>(
    settings.GoogleSpreadsheetId,
    settings.GoogleServiceAccountName,
    settings.GoogleMainSheetName);

sheetHelper.Init(settings.JsonCredential);
```
