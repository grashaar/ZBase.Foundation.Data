# ZBase.Foundation.Data

A code-first data management workflow for C# and Unity, powered by [BakingSheet](https://www.github.com/cathei/BakingSheet) and [Source Generators](https://docs.microsoft.com/en-us/dotnet/csharp/roslyn-sdk/source-generators-overview).

## Features

- Clear separation between runtime data models and authoring data sources.
    - Authoring code, data sources and configuration will NOT be included in the build.
- Data sources can be Google Sheets or CSV files (powered by BakingSheet).
- Support for complex data types (powered by BakingSheet).
    - There are only a few [Limitations](#limitations).
- Automatic mapping between data sources and data models (powered by Source Generators).
- Code-first approach with minimal configuration on Unity Inspector.
- Flexible and automatic data type conversion mechanism.

## Installation

### Requirements

- Unity 2022.3 or later

### Unity Package Manager

1. Open menu `Window` -> `Package Manager`.
2. Click the `+` button at the top-left corner, then choose `Add package from git URL...`.

![add package by git url](imgs/add-package-by-git-url-1.png)

3. Enter the package URL `https://github.com/Zitga-Tech/ZBase.Foundation.Data/tree/main/Packages/ZBase.Foundation.Data`.

![enter git url then press add button](imgs/add-package-by-git-url-2.png)

### OpenUPM

1. Install [OpenUPM CLI](https://openupm.com/docs/getting-started.html#installing-openupm-cli).
2. Run the following command in your Unity project root directory:

```sh
openupm add com.zbase.foundation.data
```

## General Workflow

At high level, the usage workflow usually consists of the following steps:
1. **Data Authoring**: Create data sources in Google Sheets or CSV files.
2. **Data Modeling**: Design `IData` models in C# code along with the table assets to store them.
3. **Data Exporting**: Leverage **BakingSheet** to import authored data from step 1 into each corresponding table asset. This step requires a piece of bridging code and a config asset.

## Tutorial

For this tutorial:
- Data sources are defined in [Google Sheets](https://docs.google.com/spreadsheets/d/19BtCJ6GqEE0rKCVFcfgX8-rjLdPTK8KQbE7gHonjdJ4/edit?usp=sharing)
- Data models and table assets located at [Assets/Samples](./Assets/Samples/)
- Data exporting configs located at [Assets/Samples.Authoring](./Assets/Samples.Authoring/)

### Step 1. Data Authoring

#### 1.1. Create Data Sources

Data sources can be either Google Sheets or CSV files.

<picture id="fig_1">
  <source media="(prefers-color-scheme: dark)" srcset="imgs/table-map-regions-dark.png">
  <source media="(prefers-color-scheme: light)" srcset="imgs/table-map-regions-light.png">
  <img alt="table map regions" src="imgs/table-map-regions-light.png">
</picture>

**Figure 1:** [`map_regions` table](https://docs.google.com/spreadsheets/d/19BtCJ6GqEE0rKCVFcfgX8-rjLdPTK8KQbE7gHonjdJ4/edit?gid=1055644696#gid=1055644696)

#### 1.2. Use a Consistent Naming Strategy

You must choose one of these strategies and apply it consistently for all sheets, columns, and CSV files.

| Pascal        | Camel         | Snake          | Kebab          |
| ------------- | ------------- | -------------- | -------------- |
| `SheetName`   | `sheetName`   | `sheet_name`   | `sheet-name`   |
| `ColumnName`  | `columnName`  | `column_name`  | `column-name`  |
| `FileName.csv`| `fileName.csv`| `file_name.csv`| `file-name.csv`|

### Step 2. Data Modeling

#### 2.1. Define Data Models

- Define a data model, can be `struct` or `class`, and **must** implement `IData` interface.
- Any field that should be mapped to a column in the data source must be decorated with `[SerializeField]`.
    - A public property will be generated for such valid fields.
- In case you prefer writing properties, each should be decorated with `[DataProperty]`.
    - The underlying field and methods will be generated for such valid properties.
- The data model **must** be `partial` so that source generators can generate the underlying implementation.
- Fields or properties are matched to columns in the data source by name, after applying the naming strategy.
- The ID of a data model can be a complex structure, consists of multiple fields.
    - These field named `_id` or `id` or the property named `Id` will be recognized as the ID of that model.

<br/>

```csharp
public partial struct MapRegionIdData : IData
{
    [SerializeField]
    private int _mapId;

    [SerializeField]
    private int _region;

    // IData source generator will generate
    // a property for each field.
    // ===

    // public int MapId { get => _mapId; init => _mapId = value; }

    // public int Region { get => _region; init => _region = value; }
}
```

<p id="list_1"><b>Listing 1:</b> Model for the ID of a map region entry</p>

```cs
public partial class MapRegionData : IData
{
    [DataProperty]
    public MapRegionIdData Id => Get_Id();

    [DataProperty]
    public int UnlockCost => Get_UnlockCost();

    // IData source generator will generate
    // a field and a `Get_XXX()` method for each property.
    // ===

    // [SerializeField]
    // private MapRegionIdData _id;

    // private readonly MapRegionIdData Get_Id() => _id;

    // [SerializeField]
    // private int _unlockCost;

    // private readonly int Get_UnlockCost() => _unlockCost;
}
```

<p id="list_2"><b>Listing 2:</b> Model for the map region entry</p>

#### 2.2. Define Data Table Assets

- Each data table asset type should inherit from either `DataTableAsset<TEntryId, TEntry>` or `DataTableAsset<TEntryId, TEntry, TConvertedId>`.
    - `TEntryId` is the type of the `Id` property of `TEntry`.
    - `TEntry` is the data model, corresponding to a row in the data source.
    - `TConvertedId` is the type of the `Id` property of `TEntry` after being converted from `TEntryId`.
- It is **required** to implement `IDataTableAsset` interface so source generator can generate additional but necessary code.
- Ultimately this is a `ScriptableObject` to store the imported data.

```cs
public sealed partial class MapRegionDataTableAsset
    : DataTableAsset<MapRegionIdData, MapRegionData, MapRegionId>
    , IDataTableAsset
{
    protected override MapRegionId Convert(MapRegionIdData value)
        => value;

    // IDataTableAsset source generator will generate
    // a constant field `NAME` and a `GetId()` method.
    // ===

    // public const string NAME = nameof(MapRegionDataTableAsset);

    // protected override MapRegionIdData GetId(in MapRegionData data)
    // {
    //     return data.Id;
    // }
}
```

<p id="list_3"><b>Listing 3:</b> Data table asset for map region</p>

### Step 3. Data Exporting

#### 3.1. Declare a Bridge to BakingSheet




## Limitations

- Nested vertical list is [not supported](https://github.com/cathei/BakingSheet/issues/36).
- [Cross-Sheet Reference](https://github.com/cathei/BakingSheet?tab=readme-ov-file#using-cross-sheet-reference) is not supported.
