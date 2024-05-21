# dbatools.ai: A Copilot for SQL Server Databases

dbatools.ai is a PowerShell module that acts as a helpful assistant for SQL Server databases. It lets developers and DBAs explore their databases using plain English commands. This project is a proof of concept designed to show PowerShell and .NET developers how to create a database assistant using OpenAI's advanced language models.

It works so well, though. Check this out -- I used the laziest language possible and it still came through.

![dbatools.ai example output](copilot.example.png)

As a developer,  you'll note that building a copilot is not 100% magic. The natural language part is magic, certainly, but it still requires a schema to be provided. The copilot/OpenAI doesn't magically go in and get that for you. To see how this works, scroll down to see the workflow.

## Supported Platforms
+ Windows PowerShell 5.1
+ PowerShell 7 or higher
+ Windows, macOS or Linux

## Prerequisites
You need to sign-up for an OpenAI account and generate an API key for authentication. Visit https://platform.openai.com/account/api-keys to create your API key.

Azure OpenAI Services support coming soon.

## Getting Started

To install dbatools.ai, simply run the following command in your PowerShell session:

```powershell
Install-Module dbatools.ai
```
This command will automatically install the required dependencies, including dbatools and PSOpenAI.

Next, set your OpenAI API key as an environment variable:

```powershell
$env:OPENAI_API_KEY = "sk-fake12345FAKE67890APIKEY12345"
```

Now, import the module. You don't always have to do it but sometimes you do if you JUST installed it from the Gallery and it hasn't been indexed.

```powershell
Import-Module dbatools.ai
```

Now, execute your first query. If the Assistant doesn't exist yet, it'll create it automatically. **NOTE**: the module defaults to using Northwind on localhost. This config exists on my local machine so this actually works:

```powershell
dbai what questions can i ask about the database
```
dbai is just a shortcut for `Invoke-DbaiQuery`. Here's how you'd use a database on another server:

```powershell
$parms = @{
    SqlInstance = "sql01"
    Database = "AdventureWerks"
    Message = "Any employee birthdays coming up?"
}

Invoke-DbaiQuery @parms
```

## Usage

dbatools.ai provides three main functions to interact with your SQL Server databases:

### New-DbaiAssistant

Creates a new AI assistant for a specified SQL Server database. The assistant is generated based on the database schema and can be customized with a name, description, and instructions.

Invoke-DbaiQuery automatically executes this if the given assistant hasn't been created yet.

```powershell
Get-DbaDatabase -SqlInstance sql01 -Database AdventureWerks |
New-DbaiAssistant -Name "AdventureWerks AI" -Description "AI assistant for AdventureWerks db"
```

By default, the assistant uses GPT-4o which has a 128k context. That's like 97,000 words so datatypes can easily be included in the schema. If you choose any other model, it'll likely have an 8k context so the module leaves that off when building the instruction string.

### ConvertTo-DbaiInstruction

Converts the schema of a SQL Server database to a specified format (JSON, SQL, or plain text). This function is used internally by `New-DbaiAssistant` to generate the schema representation for the AI assistant.


```powershell
Get-DbaDatabase -SqlInstance sql01 -Database AdventureWerks | ConvertTo-DbaiInstruction -Type SQL
```

I recommend using plain text as it uses the least amount of tokens.

As mentioned earlier, the assistant uses GPT-4o by default, which has a 128k context. That's like 97,000 words so datatypes can easily be included in the schema. If you choose any other model, it'll likely have an 8k context so the module leaves that off when building the instruction string.

### Invoke-DbaiQuery

Executes a natural language query on a specified SQL Server database. This function takes your query, passes it to the OpenAI language model, which determines whether it requires a direct response or a SQL query execution. If a SQL query is needed, the function checks the query's safety and executes it against the database, returning the results.


```powershell
$parms = @{
    SqlInstance = "sql01"
    Database = "AdventureWerks"
    Message = "What are the top 5 selling products?"
}

Invoke-DbaiQuery @parms
```

## Workflow

The workflow of `Invoke-DbaiQuery` can be summarized as follows:

0. You build an assistant just once. This assistant contains the schema of your db.
1. The user asks the assistant a question.
2. The assistant determines the appropriate response type (direct answer or SQL query).
3. If a SQL query is required, the function checks the query's safety.
  - If the query is safe, it is executed, and the results are returned.
  - If the query is unsafe, it is rejected, and an error is handled.
4. If a direct answer is sufficient, the assistant returns a natural language response.
5. The function checks the completion status and returns the data-augmented natural language response.

Visually, this is what it looks like

```mermaid
graph TD
Z(Developer Builds Assistant) --> A([User Asks Assistant a Question])
A --> D{Determine Response Type}
D -->|Return SQL Query| E[SQL Query Returned]
E -->|Requires Further Action| F[Check SQL Query Safety]
D -->|Direct Answer| G[Return Natural Language Response]
F -->|Safe| H[Execute SQL Query]
H --> I[Send to API]
I --> J[Check Completion Status]
J -->|Requires Further Action| K[Handle Required Action]
J -->|Completed| L[Return Data-Driven Natural Language Response]
F -->|Unsafe| M[Reject Query]
M --> N[Error Handling]
K --> L
classDef operation fill:#E1F5FE,stroke:#01579B,stroke-width:2px;
classDef decision fill:#FFE0B2,stroke:#EF6C00,stroke-width:2px;
classDef final fill:#C8E6C9,stroke:#388E3C,stroke-width:2px;
classDef positiveResponse fill:#A5D6A7,stroke:#2E7D32,stroke-width:2px;
classDef errorHandling fill:#FFCDD2,stroke:#D32F2F,stroke-width:2px;
class Z,A,H,I,J operation
class D,F,K decision
class E,G,L positiveResponse
class M,N errorHandling
linkStyle default stroke:#333,stroke-width:2px,fill:none;
```

## The Assistant

If you're curious what the assistant actually looks like, this is one that was created for Northwind.

```json
{
    "id": "asst_FAKEFGOXz08Rt65w9mKcgwbm",
    "object": "assistant",
    "name": "query-Northwind",
    "description": "Copilot for the Northwind database.",
    "model": "gpt-4o",
    "instructions": "You are an friendly assistant that specializes in translating natural language queries into MSSQL queries. Your task is to analyze the provided database schema, including tables, columns, data types, views, and relationships, and generate the appropriate SQL query based on the user\u0027s natural language input. Ensure that the generated SQL query is optimized, efficient, and accurately retrieves the desired data from the specified database. If the natural language query is ambiguous or lacks necessary information, ask clarifying questions to refine the query.. and a blurb about how to format.",
    "tools": [
        {
            "type": "function",
            "function": {
                "name": "ask_database",
                "description": "Use this function to answer user questions about the database. Input should be a fully formed SQL query.",
                "parameters": {
                    "properties": {
                        "query": {
                            "type": "string",
                            "description": "SQL query extracting info to answer the user\u0027s question. SQL should be written using this database schema:\r\n                                Table: dbo.Categories\nColumns: CategoryID (int) CategoryName (nvarchar) Description (ntext) Picture (image)\n\nTable: dbo.CustomerCustomerDemo\nColumns: CustomerID (nchar) CustomerTypeID (nchar)\nForeign Keys: CustomerTypeID -\u003e dbo.CustomerDemographics(CustomerTypeID), CustomerID -\u003e dbo.Customers(CustomerID)\n\nTable: dbo.CustomerDemographics\nColumns: CustomerTypeID (nchar) CustomerDesc (ntext)\n\nTable: dbo.Customers\nColumns: CustomerID (nchar) CompanyName (nvarchar) ContactName (nvarchar) ContactTitle (nvarchar) Address (nvarchar) City (nvarchar) Region (nvarchar) PostalCode (nvarchar) Country (nvarchar) Phone (nvarchar) Fax (nvarchar)\n\nTable: dbo.Employees\nColumns: EmployeeID (int) LastName (nvarchar) FirstName (nvarchar) Title (nvarchar) TitleOfCourtesy (nvarchar) BirthDate (datetime) HireDate (datetime) Address (nvarchar) City (nvarchar) Region (nvarchar) PostalCode (nvarchar) Country (nvarchar) HomePhone (nvarchar) Extension (nvarchar) Photo (image) Notes (ntext) ReportsTo (int) PhotoPath (nvarchar)\nForeign Keys: ReportsTo -\u003e dbo.Employees(EmployeeID)\n\nTable: dbo.EmployeeTerritories\nColumns: EmployeeID (int) TerritoryID (nvarchar)\nForeign Keys: EmployeeID -\u003e dbo.Employees(EmployeeID), TerritoryID -\u003e dbo.Territories(TerritoryID)\n\nTable: dbo.Order Details\nColumns: OrderID (int) ProductID (int) UnitPrice (money) Quantity (smallint) Discount (real)\nForeign Keys: OrderID -\u003e dbo.Orders(OrderID), ProductID -\u003e dbo.Products(ProductID)\n\nTable: dbo.Orders\nColumns: OrderID (int) CustomerID (nchar) EmployeeID (int) OrderDate (datetime) RequiredDate (datetime) ShippedDate (datetime) ShipVia (int) Freight (money) ShipName (nvarchar) ShipAddress (nvarchar) ShipCity (nvarchar) ShipRegion (nvarchar) ShipPostalCode (nvarchar) ShipCountry (nvarchar)\nForeign Keys: CustomerID -\u003e dbo.Customers(CustomerID), EmployeeID -\u003e dbo.Employees(EmployeeID), ShipVia -\u003e dbo.Shippers(ShipperID)\n\nTable: dbo.Products\nColumns: ProductID (int) ProductName (nvarchar) SupplierID (int) CategoryID (int) QuantityPerUnit (nvarchar) UnitPrice (money) UnitsInStock (smallint) UnitsOnOrder (smallint) ReorderLevel (smallint) Discontinued (bit)\nForeign Keys: CategoryID -\u003e dbo.Categories(CategoryID), SupplierID -\u003e dbo.Suppliers(SupplierID)\n\nTable: dbo.Region\nColumns: RegionID (int) RegionDescription (nchar)\n\nTable: dbo.Shippers\nColumns: ShipperID (int) CompanyName (nvarchar) Phone (nvarchar)\n\nTable: dbo.Suppliers\nColumns: SupplierID (int) CompanyName (nvarchar) ContactName (nvarchar) ContactTitle (nvarchar) Address (nvarchar) City (nvarchar) Region (nvarchar) PostalCode (nvarchar) Country (nvarchar) Phone (nvarchar) Fax (nvarchar) HomePage (ntext)\n\nTable: dbo.Territories\nColumns: TerritoryID (nvarchar) TerritoryDescription (nchar) RegionID (int)\nForeign Keys: RegionID -\u003e dbo.Region(RegionID)\n\nView: dbo.Alphabetical list of products\nColumns: ProductID (int) ProductName (nvarchar) SupplierID (int) CategoryID (int) QuantityPerUnit (nvarchar) UnitPrice (money) UnitsInStock (smallint) UnitsOnOrder (smallint) ReorderLevel (smallint) Discontinued (bit) CategoryName (nvarchar)\n\nView: dbo.Category Sales for 1997\nColumns: CategoryName (nvarchar) CategorySales (money)\n\nView: dbo.Current Product List\nColumns: ProductID (int) ProductName (nvarchar)\n\nView: dbo.Customer and Suppliers by City\nColumns: City (nvarchar) CompanyName (nvarchar) ContactName (nvarchar) Relationship (varchar)\n\nView: dbo.Invoices\nColumns: ShipName (nvarchar) ShipAddress (nvarchar) ShipCity (nvarchar) ShipRegion (nvarchar) ShipPostalCode (nvarchar) ShipCountry (nvarchar) CustomerID (nchar) CustomerName (nvarchar) Address (nvarchar) City (nvarchar) Region (nvarchar) PostalCode (nvarchar) Country (nvarchar) Salesperson (nvarchar) OrderID (int) OrderDate (datetime) RequiredDate (datetime) ShippedDate (datetime) ShipperName (nvarchar) ProductID (int) ProductName (nvarchar) UnitPrice (money) Quantity (smallint) Discount (real) ExtendedPrice (money) Freight (money)\n\nView: dbo.Order Details Extended\nColumns: OrderID (int) ProductID (int) ProductName (nvarchar) UnitPrice (money) Quantity (smallint) Discount (real) ExtendedPrice (money)\n\nView: dbo.Order Subtotals\nColumns: OrderID (int) Subtotal (money)\n\nView: dbo.Orders Qry\nColumns: OrderID (int) CustomerID (nchar) EmployeeID (int) OrderDate (datetime) RequiredDate (datetime) ShippedDate (datetime) ShipVia (int) Freight (money) ShipName (nvarchar) ShipAddress (nvarchar) ShipCity (nvarchar) ShipRegion (nvarchar) ShipPostalCode (nvarchar) ShipCountry (nvarchar) CompanyName (nvarchar) Address (nvarchar) City (nvarchar) Region (nvarchar) PostalCode (nvarchar) Country (nvarchar)\n\nView: dbo.Product Sales for 1997\nColumns: CategoryName (nvarchar) ProductName (nvarchar) ProductSales (money)\n\nView: dbo.Products Above Average Price\nColumns: ProductName (nvarchar) UnitPrice (money)\n\nView: dbo.Products by Category\nColumns: CategoryName (nvarchar) ProductName (nvarchar) QuantityPerUnit (nvarchar) UnitsInStock (smallint) Discontinued (bit)\n\nView: dbo.Quarterly Orders\nColumns: CustomerID (nchar) CompanyName (nvarchar) City (nvarchar) Country (nvarchar)\n\nView: dbo.Sales by Category\nColumns: CategoryID (int) CategoryName (nvarchar) ProductName (nvarchar) ProductSales (money)\n\nView: dbo.Sales Totals by Amount\nColumns: SaleAmount (money) OrderID (int) CompanyName (nvarchar) ShippedDate (datetime)\n\nView: dbo.Summary of Sales by Quarter\nColumns: ShippedDate (datetime) OrderID (int) Subtotal (money)\n\nView: dbo.Summary of Sales by Year\nColumns: ShippedDate (datetime) OrderID (int) Subtotal (money)\n\r\n                                The query should be returned in plain text, not in JSON."
                        }
                    },
                    "type": "object",
                    "required": [
                        "query"
                    ]
                }
            }
        },
        {
            "type": "function",
            "function": {
                "name": "examine_sql",
                "description": "Check if a SQL query is valid and if potentially dangerous.",
                "parameters": {
                    "properties": {
                        "danger_reason": {
                            "type": "string",
                            "description": "If the query is dangerous, why?"
                        },
                        "dangerous": {
                            "type": "boolean",
                            "description": "Does this sql query modify data or is it potentially dangerous?"
                        },
                        "valid_sql": {
                            "type": "boolean",
                            "description": "Is this a valid SQL statement?"
                        }
                    },
                    "type": "object",
                    "required": [
                        "dangerous",
                        "valid_sql"
                    ]
                }
            }
        }
    ],
    "top_p": 1.0,
    "temperature": 1.0,
    "tool_resources": {},
    "metadata": {},
    "response_format": "auto",
    "created_at": "\/Date(1919296032000)\/"
}
```

## Limitations and Considerations

This module is a proof of concept -- don't run this in prod.