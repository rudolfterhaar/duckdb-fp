# DuckDB.DataFrame API Reference

## Table of Contents

- [DuckDB.DataFrame API Reference](#duckdbdataframe-api-reference)
  - [Table of Contents](#table-of-contents)
  - [TDuckFrame](#tduckframe)
    - [Properties](#properties)
    - [Column Information (TDuckDBColumn)](#column-information-tduckdbcolumn)
      - [Data Types](#data-types)
      - [Date/Time Handling](#datetime-handling)
        - [Date Values (dctDate)](#date-values-dctdate)
        - [Time Values (dctTime)](#time-values-dcttime)
        - [Timestamp Values (dctTimestamp)](#timestamp-values-dcttimestamp)
  - [TDuckFrame Methods](#tduckframe-methods)
    - [1. Constructor and Destructor](#1-constructor-and-destructor)
      - [Advanced Examples (CSV)](#advanced-examples-csv)
      - [Common Issues and Solutions (CSV)](#common-issues-and-solutions-csv)
      - [Best Practices using Constructors](#best-practices-using-constructors)
    - [Data Access Methods](#data-access-methods)
    - [Data Manipulation Methods](#data-manipulation-methods)
    - [Output Methods](#output-methods)
    - [Data Analysis Methods](#data-analysis-methods)
    - [Histogram Generation](#histogram-generation)
    - [Statistical Analysis](#statistical-analysis)
  - [File Operations](#file-operations)
    - [CSV Export](#csv-export)
  - [DataFrame Combination Methods](#dataframe-combination-methods)
    - [DataFrame Combination Operations](#dataframe-combination-operations)
      - [Union Modes](#union-modes)
      - [Union](#union)
      - [UnionAll](#unionall)
      - [Distinct](#distinct)
    - [Examples](#examples)
    - [Implementation Details](#implementation-details)
    - [Type Conversion](#type-conversion)
    - [Notes](#notes)
    - [Missing Data Handling](#missing-data-handling)
    - [Unique Value Analysis](#unique-value-analysis)

## TDuckFrame

A data structure for handling DuckDB query results, similar to R's data frame or pandas DataFrame in Python.

### Properties

```pascal
property RowCount: Integer;
```
- Number of rows in the DataFrame
- Read-only
- Returns the current number of rows

```pascal
property ColumnCount: Integer;
```
- Number of columns in the DataFrame
- Read-only
- Returns the current number of columns

```pascal
property Columns[Index: Integer]: TDuckDBColumn;
```
- Access column by index
- Read-only
- Raises `EDuckDBError` if index out of bounds
- Returns column information including name, type, and data

```pascal
property ColumnsByName[Name: string]: TDuckDBColumn;
```
- Access column by name
- Read-only
- Raises `EDuckDBError` if column name not found
- Returns column information including name, type, and data

```pascal
property Values[Row, Col: Integer]: Variant;
```
- Access value by row and column index
- Read/write
- Raises `EDuckDBError` if:
  - Row index out of bounds
  - Column index out of bounds
- Automatic type conversion attempted when setting values

```pascal
property ValuesByName[Row: Integer; ColName: string]: Variant;
```
- Access value by row and column name
- Read/write
- Raises `EDuckDBError` if:
  - Row index out of bounds
  - Column name not found
- Automatic type conversion attempted when setting values

### Column Information (TDuckDBColumn)

| Property | Type | Description |
|----------|------|-------------|
| `Name` | `string` | Column name |
| `DataType` | `TDuckDBColumnType` | Column data type |
| `Data` | `array of Variant` | Column values |

#### Data Types

```pascal
  TDuckDBColumnType = (
    dctUnknown,    // Unknown or unsupported type
    dctBoolean,    // Boolean (true/false)
    dctTinyInt,    // 8-bit integer
    dctSmallInt,   // 16-bit integer
    dctInteger,    // 32-bit integer
    dctBigInt,     // 64-bit integer
    dctFloat,      // Single-precision floating point
    dctDouble,     // Double-precision floating point
    dctDate,       // Date without time (YYYY-MM-DD)
    dctTime,       // Time without date (HH:MM:SS.SSS)
    dctTimestamp,  // Date with time (YYYY-MM-DD HH:MM:SS.SSS)
    dctInterval,   // Time interval/duration
    dctString,     // Variable-length string
    dctBlob,       // Binary large object
    dctDecimal,    // Decimal number with precision and scale
    dctUUID,       // Universally Unique Identifier
    dctJSON        // JSON data
  );
```

#### Date/Time Handling

The DataFrame supports three date/time-related types:

##### Date Values (dctDate)
- Stored internally as TDate (integer part of TDateTime)
- Default/null value is 0 (30/12/1899)
- Displayed in 'dd/mm/yyyy' format
- Access via `TDuckDBColumn.AsDateArray`

##### Time Values (dctTime)
- Stored internally as TTime (fractional part of TDateTime)
- Default/null value is 0 (00:00:00)
- Displayed in 'hh:nn:ss' format
- Access via `TDuckDBColumn.AsTimeArray`

##### Timestamp Values (dctTimestamp)
- Stored internally as TDateTime (full date and time)
- Default/null value is 0 (30/12/1899 00:00:00)
- Displayed in 'dd/mm/yyyy hh:nn:ss' format
- Access via `TDuckDBColumn.AsDateTimeArray`

Example:
```pascal
var
  Frame: TDuckFrame;
begin
  Frame := TDuckFrame.CreateBlank(
    ['Timestamp', 'Date', 'Time'],
    [dctTimestamp, dctDate, dctTime]
  );
  
  Frame.AddRow([
    EncodeDateTime(2023, 11, 25, 10, 30, 0, 0),  // Timestamp
    EncodeDate(2023, 11, 25),                     // Date
    EncodeTime(10, 30, 0, 0)                      // Time
  ]);
  
  Frame.Print;  // Will display formatted date/time values
end;
```

## TDuckFrame Methods

### 1. Constructor and Destructor

```pascal
constructor Create;
```
- Creates an empty DataFrame
- No initial columns or rows
- Basic initialization of internal structures

```pascal
constructor CreateFromDuckDB(const AFileName: string; const ATableName: string);
```
- Creates a DataFrame from a DuckDB database table
- Parameters:
  - `AFileName`: Path to the DuckDB database file
  - `ATableName`: Name of the table to load
- Raises `EDuckDBError` if:
  - Database file not found
  - Table doesn't exist
  - Connection fails
  - Query execution fails

```pascal
constructor CreateBlank(const AColumnNames: array of string;
                       const AColumnTypes: array of TDuckDBColumnType);
```
- Creates an empty DataFrame with predefined structure
- Parameters:
  - `AColumnNames`: Array of column names
  - `AColumnTypes`: Array of column data types
- Features:
  - Initializes DataFrame with specified columns but no rows
  - Ready for data addition via AddRow
- Raises `EDuckDBError` if:
  - Column names array and types array have different lengths
  - Duplicate column names exist


```pascal
constructor CreateFromCSV(const AFileName: string; 
                         AHasHeaders: Boolean = True;
                         const ADelimiter: Char = ',');
```
- Creates a DataFrame by loading data from a CSV file
- Parameters:
  - `AFileName`: Path to the CSV file
  - `AHasHeaders`: Whether the first row contains column names (default: True)
  - `ADelimiter`: Character used to separate fields (default: comma)
- Features:
  - Automatic type detection
  - Handles quoted strings
  - Supports custom delimiters
- Raises `EDuckDBError` if:
  - File not found
  - Invalid file format
  - Database or connection errors

Example usage:

```pascal
// Create from DuckDB table
var
  DF1: TDuckFrame;
begin
  DF1 := TDuckFrame.CreateFromDuckDB('customers.db', 'sales');
  try
    // Use DF1...
  finally
    DF1.Free;
  end;
end;

// Create from CSV file
var
  DF2: TDuckFrame;
begin
  DF2 := TDuckFrame.CreateFromCSV('data.csv', True, ',');
  try
    // Use DF2...
  finally
    DF2.Free;
  end;
end;

// Create blank DataFrame with structure
var
  DF3: TDuckFrame;
begin
  DF3 := TDuckFrame.CreateBlank(
    ['id', 'name', 'age'],
    [dctInteger, dctString, dctInteger]
  );
  try
    // Add rows to DF3...
    DF3.AddRow([1, 'John', 25]);
    DF3.AddRow([2, 'Jane', 30]);
  finally
    DF3.Free;
  end;
end;
```


```pascal
constructor CreateFromDuckDB(const ADatabase, ATableName: string); overload;
``` 
- Creates a DataFrame by connecting to a DuckDB database and loading a specific table.
- Parameters:
  - `ADatabase`: Path to the DuckDB database file.
  - `ATableName`: Name of the table to load into the DataFrame.
- Raises `EDuckDBError` if:
  - Database connection fails.
  - Table does not exist in the specified database.
  - Query execution encounters an error.
    
Example Usage:
    
```pascal
// Create from DuckDB with database and table name
var
  DF4: TDuckFrame;
begin
  DF4 := TDuckFrame.CreateFromDuckDB('analytics.db', 'transactions');
  try
    // Use DF4...
    DF4.Print;
  finally
    DF4.Free;
  end;
end;
```

```pascal
constructor CreateFromParquet(const AFileName: string); overload;
constructor CreateFromParquet(const Files: array of string); overload;
``` 

- Creates a DataFrame by loading data from Parquet files.
- Parameters:
  - `AFileName`: Path to the Parquet file to load into the DataFrame.
  - `Files`: Array of paths to Parquet files to load and combine into the DataFrame.
- Raises `EDuckDBError` if:
  - Any specified file does not exist or cannot be accessed.
  - Any file is not a valid Parquet file or cannot be parsed.

Example Usage:

```pascal
// Create from a single Parquet file
var
  DF5: TDuckFrame;
begin
  DF5 := TDuckFrame.CreateFromParquet('data/sales.parquet');
  try
    // Use DF5...
    DF5.Print;
  finally
    DF5.Free;
  end;
end;

// Create from multiple Parquet files
var
  DF6: TDuckFrame;
  ParquetFiles: array of string;
begin
  
  SetLength(ParquetFiles, 2);
  ParquetFiles[0] := 'data/sales_january.parquet';
  ParquetFiles[1] := 'data/sales_february.parquet';

  DF6 := TDuckFrame.CreateFromParquet(ParquetFiles);
  try
    // Use DF6...
    DF6.Print;
  finally
    DF6.Free;
  end;
end;
```

> [!NOTE] 
> 
> All constructors create a new instance that must be freed when no longer needed.

#### Advanced Examples (CSV)

```pascal
program CSVExamples;

{$mode objfpc}{$H+}

uses
  SysUtils, DuckDB.DataFrame;
  
procedure LoadCSVWithOptions;
var
  Frame: TDuckFrame;
begin
  // Example 1: CSV with semicolon delimiter
  Frame := TDuckFrame.CreateFromCSV('european_data.csv', True, ';');
  try
    WriteLn('Loaded CSV with semicolon delimiter:');
    Frame.Head(3).Print;  // Show first 3 rows
  finally
    Frame.Free;
  end;
  
  // Example 2: CSV without headers
  Frame := TDuckFrame.CreateFromCSV('raw_data.csv', False);
  try
    WriteLn('Loaded CSV without headers:');
    Frame.Print;
  finally
    Frame.Free;
  end;
  
  // Example 3: Tab-delimited file
  Frame := TDuckFrame.CreateFromCSV('tab_data.txt', True, #9);
  try
    WriteLn('Loaded tab-delimited file:');
    Frame.Print;
  finally
    Frame.Free;
  end;
end;

// Example with error handling and data validation
procedure LoadAndValidateCSV(const FileName: string);
var
  Frame: TDuckFrame;
begin
  WriteLn('Loading ', FileName, '...');
  
  try
    Frame := TDuckFrame.CreateFromCSV(FileName);
    try
      // Basic validation
      if Frame.RowCount = 0 then
      begin
        WriteLn('Warning: CSV file is empty');
        Exit;
      end;
      
      // Display structure information
      WriteLn(Format('Loaded %d rows and %d columns', [Frame.RowCount, Frame.ColumnCount]));
      Frame.Info;
      
      // Check for missing values
      WriteLn('Checking for missing values:');
      Frame.NullCount.Print;
      
      // Display data preview
      WriteLn('Data preview:');
      Frame.Head(5).Print;
      
    finally
      Frame.Free;
    end;
    
  except
    on E: EDuckDBError do
    begin
      WriteLn('DuckDB Error loading CSV:');
      WriteLn('  ', E.Message);
    end;
    on E: Exception do
    begin
      WriteLn('Unexpected error:');
      WriteLn('  ', E.Message);
    end;
  end;
end;

begin
  // Example usage
  LoadAndValidateCSV('sample_data.csv');
end.
```

#### Common Issues and Solutions (CSV)

1. **Wrong Delimiter**: If your CSV isn't loading correctly, check if it uses a different delimiter:
```pascal
// Try with different delimiters
Frame := TDuckFrame.CreateFromCSV('data.csv', True, ';');  // Semicolon
Frame := TDuckFrame.CreateFromCSV('data.csv', True, #9);   // Tab
```

2. **No Headers**: If your CSV doesn't have headers:
```pascal
Frame := TDuckFrame.CreateFromCSV('data.csv', False);
```

3. **Error Handling**: Always use try-finally blocks:
```pascal
Frame := nil;
try
  Frame := TDuckFrame.CreateFromCSV('data.csv');
  // Use Frame here...
finally
  Frame.Free;
end;
```

#### Best Practices using Constructors

1. Always use try-finally blocks for proper memory management
2. Validate the data after loading
3. Check for missing values
4. Preview the data to ensure it loaded correctly
5. Use appropriate error handling

### Data Access Methods

```pascal
function FindColumnIndex(const Name: string): Integer;
```
- Returns the index of a column by name
- Returns -1 if column not found

```pascal
function GetColumn(Index: Integer): TDuckDBColumn;
```
- Returns column information by index
- Raises exception if index out of bounds

```pascal
function GetColumnByName(const Name: string): TDuckDBColumn;
```
- Returns column information by name
- Raises exception if column not found

### Data Manipulation Methods

```pascal
procedure Clear;
```
- Removes all data from the DataFrame

```pascal
function Head(Count: Integer = 5): TDuckFrame;
```
- Returns a new DataFrame with the first N rows
- Default count is 5
- Caller must free the returned DataFrame

```pascal
function Tail(Count: Integer = 5): TDuckFrame;
```
- Returns a new DataFrame with the last N rows
- Default count is 5
- Caller must free the returned DataFrame

```pascal
function Select(const Columns: array of string): TDuckFrame;
```
- Returns a new DataFrame with only the specified columns
- Caller must free the returned DataFrame

**Example**

```pascal
var
  Frame: TDuckFrame;
begin
  Frame := TDuckFrame.CreateBlank(
    ['Timestamp', 'Date', 'Time'],
    [dctTimestamp, dctDate, dctTime]
  );
  
  Frame.AddRow([
    EncodeDateTime(2023, 11, 25, 10, 30, 0, 0),  // Timestamp
    EncodeDate(2023, 11, 25),                     // Date
    EncodeTime(10, 30, 0, 0)                      // Time
  ]);
  
  Frame.Print;  // Will display formatted date/time values
end;
```

```pascal
procedure AddColumn(const AName: string; AType: TDuckDBColumnType);
```
- Adds a new column to the DataFrame
- Parameters:
  - `AName`: Name of the new column
  - `AType`: Data type for the new column
- New column is initialized with NULL values
- Raises `EDuckDBError` if column name already exists

```pascal
procedure AddRow(const AValues: array of Variant);
```
- Adds a new row to the DataFrame
- Parameters:
  - `AValues`: Array of values matching column count and types
- Automatic type conversion is attempted
- Raises `EDuckDBError` if value count doesn't match column count

```pascal
procedure SetValue(const ARow: Integer; const AColumnName: string; const AValue: Variant);
```
- Sets a single value in the DataFrame
- Parameters:
  - `ARow`: Row index
  - `AColumnName`: Column name
  - `AValue`: New value
- Raises `EDuckDBError` for invalid row/column

### Output Methods

```pascal
procedure Print;
```
- Prints the DataFrame to console in a formatted table

### Data Analysis Methods

```pascal
procedure Describe;
```
- Displays comprehensive summary statistics with separate analysis for:
  
  Factor (Categorical) Variables:
  - skim_variable: Column name
  - n_missing: Number of missing values
  - complete_rate: Percentage of non-missing values
  - ordered: Whether the categorical variable is ordered
  - n_unique: Number of unique values
  - top_counts: Most frequent values and their counts

  Numeric Variables:
  - skim_variable: Column name
  - n_missing: Number of missing values
  - complete_rate: Percentage of non-missing values
  - mean: Average value
  - sd: Standard deviation
  - min: Minimum value
  - q1: First quartile (25th percentile)
  - median: Median (50th percentile)
  - q3: Third quartile (75th percentile)
  - max: Maximum value
  - skew: Skewness (measure of distribution asymmetry)
  - kurt: Kurtosis (measure of distribution "tailedness")

Example output:

```
Statistical Summary:
-------------------
Number of rows: 4
Number of columns: 4

Column type frequency:
  factor    1
  numeric   3

-- Variable type: factor
skim_variable    n_missing  complete_rate ordered   n_unique
name             0          1.000         FALSE     4
    Top counts: Bob: 1, Alice: 1, John: 1

-- Variable type: numeric
skim_variable    n_missing  complete_rate mean      sd        min       q1        median    q3        max       skew      kurt
id               0          1.000         2.500     1.291     1.000     1.750     2.500     3.250     4.000     0.000     -2.080
age              1          0.750         33.333    10.408    25.000    27.500    30.000    37.500    45.000     0.996     0.000
salary           1          0.750         68333.333 16072.751 50000.000 62500.000 75000.000 77500.000 80000.000 -1.190     0.000
```

```pascal
function NullCount: TDuckFrame;
```
- Returns a new DataFrame with a count of null values for each column
- Result has a single row with the same column names as the original
- Caller must free the returned DataFrame

```pascal
procedure Info;
```
- Displays basic information about the DataFrame including:
  - Number of rows and columns
  - Column names and their data types
  - Number of null values per column
  - Memory usage in both bytes and MB

### Histogram Generation

```pascal
procedure PlotHistogram(const ColumnName: string; Bins: Integer = 10);
```
- Generates a text-based histogram for numeric columns
- Parameters:
  - `ColumnName`: Name of the numeric column to plot
  - `Bins`: Number of bins to divide the data into (default: 10)
- Features:
  - Automatically calculates appropriate bin ranges
  - Shows distribution of values with ASCII bar charts
  - Displays bin ranges and counts
  - Proportional bar lengths for better visualization
- Raises `EDuckDBError` if column is not numeric

Example output:
```
Histogram of age
Range: 28.00 to 49.00
Bin width: 4.20
Total count: 5

[28.00-32.20)   |##########   2
[32.20-36.40)   |###   1
[36.40-40.60)   |   0
[40.60-44.80)   |###   1
[44.80-49.00]   |###   1
```


### Statistical Analysis

```pascal
function CorrPearson: TDuckFrame;
```
- Calculates the Pearson correlation matrix for all numeric columns
- Returns a new DataFrame containing the correlation coefficients
- Measures linear correlation between variables
- Best for linear relationships between variables
- Sensitive to outliers
- Caller must free the returned DataFrame

```pascal
function CorrSpearman: TDuckFrame;
```
- Calculates the Spearman rank correlation matrix for all numeric columns
- Returns a new DataFrame containing the correlation coefficients
- Measures monotonic relationships (including non-linear)
- More robust to outliers than Pearson correlation
- Better for ordinal data and non-linear relationships
- Caller must free the returned DataFrame

Example usage:
```pascal
var
  PearsonCorr, SpearmanCorr: TDuckFrame;
begin
  // Calculate Pearson correlation
  PearsonCorr := DF.CorrPearson;
  try
    WriteLn('Pearson Correlation:');
    PearsonCorr.Print;
  finally
    PearsonCorr.Free;
  end;
  
  // Calculate Spearman correlation
  SpearmanCorr := DF.CorrSpearman;
  try
    WriteLn('Spearman Correlation:');
    SpearmanCorr.Print;
  finally
    SpearmanCorr.Free;
  end;
end;
```

When to use each correlation method:

Pearson Correlation:
- Variables have linear relationships
- Data is normally distributed
- No significant outliers
- Variables are continuous

Spearman Correlation:
- Non-linear but monotonic relationships
- Ordinal data
- Presence of outliers
- Non-normal distributions
- More robust general-purpose correlation

## File Operations

### CSV Export

```pascal
procedure SaveToCSV(const FileName: string);
```
Saves the DataFrame to a CSV file following RFC 4180 specifications.

Features:
- Full RFC 4180 compliance
- Handles special cases:
  - Multi-line text fields (preserved with proper quoting)
  - Fields containing commas
  - Fields containing quotes (escaped with double quotes)
  - NULL values (written as empty fields)
  - Empty string values
- Uses standard CRLF line endings
- Properly escapes and quotes field values when needed

Example:
```pascal
var
  DF: TDuckFrame;
begin
  // ... populate DataFrame ...
  
  // Save to CSV file
  DF.SaveToCSV('output.csv');
end;
```

Output format example:
```csv
id,name,description
1,"Simple text","Normal field"
2,"Text with
multiple lines","Another field"
3,"Text with ""quotes""","Text with, comma"
4,,"Empty field (NULL)"
```

## DataFrame Combination Methods

### DataFrame Combination Operations

#### Union Modes
The `TUnionMode` enumeration controls how DataFrame combinations handle column matching:

```pascal
type
  TUnionMode = (
    umStrict,    // Most conservative: requires exact match of column names and types
    umCommon,    // Intersection mode: only includes columns that exist in both frames
    umAll        // Most inclusive: includes all columns from both frames
  );
```

#### Union
```pascal
function Union(const Other: TDuckFrame; Mode: TUnionMode = umStrict): TDuckFrame;
```
Combines two DataFrames and removes duplicate rows (similar to SQL's UNION).
- Internally calls `UnionAll` followed by `Distinct`
- Returns a new DataFrame with combined unique rows

#### UnionAll
```pascal
function UnionAll(const Other: TDuckFrame; Mode: TUnionMode = umStrict): TDuckFrame;
```
Combines two DataFrames keeping all rows including duplicates (similar to SQL's UNION ALL).
- Different union modes affect how columns are combined:
  - `umStrict`: Requires exact column match (names and types)
  - `umCommon`: Only includes columns present in both frames
  - `umAll`: Includes all columns, fills missing values with NULL
- Returns a new DataFrame with all rows from both frames

#### Distinct
```pascal
function Distinct: TDuckFrame;
```
Removes duplicate rows from the DataFrame.
- Uses efficient hash-based deduplication
- Considers all columns when determining uniqueness
- Returns a new DataFrame with unique rows only

### Examples

```pascal
var
  DB: TDuckDBConnection;
  DF1, DF2, Combined: TDuckFrame;
begin
  DB := TDuckDBConnection.Create(':memory:');
  try
    // Create DataFrames with different structures
    DF1 := DB.Query('SELECT 1 as id, ''A'' as name, 25 as age');
    DF2 := DB.Query('SELECT 2 as id, ''B'' as name, ''HR'' as department');
    
    // Union with common columns only
    Combined := DF1.Union(DF2, umCommon);
    try
      Combined.Print;  // Shows only 'id' and 'name' columns
    finally
      Combined.Free;
    end;
    
    // UnionAll with all columns
    Combined := DF1.UnionAll(DF2, umAll);
    try
      Combined.Print;  // Shows all columns with NULL for missing values
    finally
      Combined.Free;
    end;
    
    // Remove duplicates from a single DataFrame
    Combined := DF1.Distinct;
    try
      Combined.Print;
    finally
      Combined.Free;
    end;
  finally
    DF1.Free;
    DF2.Free;
    DB.Free;
  end;
end;
```

### Implementation Details

The combination operations use `THashSet` for efficient unique value tracking:
- Column name uniqueness in `umAll` mode
- Row uniqueness in `Distinct` operation
- Hash-based lookups provide O(1) average case complexity

### Type Conversion
When combining DataFrames with different column types:
- Compatible types are automatically converted (e.g., Integer to Float)
- Incompatible conversions result in NULL values
- String columns can accept any type through string conversion
- Type precedence follows SQL conventions

### Notes
- All union operations create a new DataFrame that must be freed by the caller
- NULL values are preserved and handled properly in all operations
- Column name matching is case-sensitive
- Type conversions are attempted when possible but may result in NULL values if incompatible

### Missing Data Handling

```pascal
function FillNA(const Value: Variant): TDuckFrame;
```
- Creates new DataFrame with NULL values replaced
- Parameters:
  - `Value`: Value to use for replacement
- Returns new DataFrame with filled values

```pascal
function DropNA: TDuckFrame;
```
- Creates new DataFrame with rows containing any NULL values removed
- Returns new DataFrame with complete cases only

### Unique Value Analysis

```pascal
function UniqueCounts(const ColumnName: string): TDuckFrame;
```
- Creates frequency table for a column
- Parameters:
  - `ColumnName`: Column to analyze
- Returns DataFrame with value counts