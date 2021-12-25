---
title: "Saving Bot Activities in Azure SQL Database"
date: 2021-12-24
categories:
  - Blog
tags:
  - Azure Bot
  - Bot activities
  - Conversations
  - Azure SQL Database
  - Persistent State storage
---

In this blog post, we'll walkthrough on how to integrate Bot Framework v4 SDK with [Azure SQL Database](https://azure.microsoft.com/en-us/products/azure-sql/database/#overview) and save Bot activities/ conversations into database.

We'll be leverage [`Bot.Builder.Community.Storage.EntityFramework`](https://www.nuget.org/packages/Bot.Builder.Community.Storage.EntityFramework/) package to save conversations using EntityFrameworkTranscriptStore with TranscriptLoggerMiddleware into Azure SQL Database.

<h2>Let's start -</h2>

### Create Azure Sql Database:

If you are new to Azure SQL and looking to understand more about different offerings of Azure SQL, you can start with [this article](https://docs.microsoft.com/en-us/azure/azure-sql/azure-sql-iaas-vs-paas-what-is-overview).

Once you decided the type of Azure SQL Database suits your requirement, you can navigate to the [Select SQL Deployment options](https://ms.portal.azure.com/#create/Microsoft.AzureSQL) page and create your Database.

>For this demo, I have created a single database in the [serverless compute tier](https://docs.microsoft.com/en-us/azure/azure-sql/database/serverless-tier-overview).

You can follow [this article](https://docs.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?tabs=azure-portal) which explains steps of creating Azure SQL Database.

### Connect to the database:
Once your Azure sql database is created, you can use the [Query editor (preview)](https://docs.microsoft.com/en-us/azure/azure-sql/database/single-database-create-quickstart?tabs=azure-portal#query-the-database) in the Azure portal to connect to the database and execute queries. Alternatively you can also connect your database using [SSMS](https://docs.microsoft.com/en-us/sql/ssms/download-sql-server-management-studio-ssms?view=sql-server-ver15) and [Azure Data Studio](https://docs.microsoft.com/en-us/sql/azure-data-studio/download-azure-data-studio?view=sql-server-ver15) etc.

- In the Azure portal, search for and select SQL databases, and then select your database from the list.
- On the page for your database, select Query editor (preview) in the left menu.
- Enter your server admin login information and select OK.

<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/query-editor-login.png" width="600" border="0"><br/>

>Note: Click on the `Connection strings` link in the above blade and copy the connection string somewhere which we'll use later in this demo.

### Create tables in your Azure SQL database:
Now, you need to create tables in your Azure SQL database to store the Bot Activities.

- Execute the [CreateTableScript.sql](https://github.com/abhimantiwari/botbuilder-community-dotnet/blob/develop/samples/EntityFramework%20Storage%20Sample/CreateTableScript.sql) script in your Azure SQL Database. 
  <details>
  <summary>Alternatively you can also <b>Click Me</b> to see, `Create table script`</summary>

  CREATE TABLE [dbo].[BotDataEntity](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [RealId] [varchar](1024) NOT NULL UNIQUE,
    [Document] [nvarchar](max) NOT NULL,
    [CreatedTime] [datetimeoffset](7) Not NULL,
    [TimeStamp] [datetimeoffset](7) Not NULL,
  CONSTRAINT [PK_BotDataEntity] PRIMARY KEY CLUSTERED 
  (
    [Id] ASC
  )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
  ) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]
  GO
  ALTER TABLE [dbo].[BotDataEntity] ADD  DEFAULT (getutcdate()) FOR [CreatedTime]
  GO
  ALTER TABLE [dbo].[BotDataEntity] ADD  DEFAULT (getutcdate()) FOR [TimeStamp]
  GO

  CREATE TABLE [dbo].[TranscriptEntity](
    [Id] [int] IDENTITY(1,1) NOT NULL,
    [Channel] [varchar](256) NOT NULL,
    [Conversation] [varchar](1024) NOT NULL,
      [Activity] [nvarchar](max) NOT NULL,
    [TimeStamp] [datetimeoffset](7) NOT NULL,
  CONSTRAINT [PK_TranscriptEntity] PRIMARY KEY CLUSTERED 
  (
    [Id] ASC
  )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY])
  GO
  CREATE NONCLUSTERED INDEX [IX_TranscriptChannel] ON [dbo].[TranscriptEntity]
  (
    [Channel] ASC
  )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
  GO
  CREATE NONCLUSTERED INDEX [IX_TranscriptTimeStamp] ON [dbo].[TranscriptEntity]
  (
    [TimeStamp] ASC
  )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
  GO
  CREATE NONCLUSTERED INDEX [IX_TranscriptConversation] ON [dbo].[TranscriptEntity]
  (
    [Conversation] ASC
  )WITH (PAD_INDEX = OFF, STATISTICS_NORECOMPUTE = OFF, SORT_IN_TEMPDB = OFF, DROP_EXISTING = OFF, ONLINE = OFF, ALLOW_ROW_LOCKS = ON, ALLOW_PAGE_LOCKS = ON) ON [PRIMARY]
  GO
  ALTER TABLE [dbo].[TranscriptEntity] ADD  DEFAULT (getutcdate()) FOR [TimeStamp]
  GO

  </details>

- Once you execute the `CreateTableScript.sql` script in your Azure SQL Database, you should be able to see the below two tables (`BotDataEntity` & `TranscriptEntity`) created.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/EmptyBotTables.png" width="600" border="0"><br/>
>We have just created the table structure, it doesn't contain any data yet. 

### Getting Bot project ready:
For this Demo, we will be using [Bot Framework](https://dev.botframework.com/) v4 Echo bot sample which accepts input from the user and echoes it back.

You may download the working code sample from this [GitHub link](https://github.com/abhimantiwari/botbuilder-community-dotnet/tree/develop/samples/EntityFramework%20Storage%20Sample), or do the following changes into your existing Bot project.

- Import these packages into your Bot project.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/BotBuilderPackages.png" width="600" border="0"><br/>

- Add the following code to your bot project's Startup.cs.
```ruby
 var loggerConnectionString = Configuration["StoreConnectionString"];
 var logger = new EntityFrameworkTranscriptStore(loggerConnectionString);
 services.AddSingleton<ITranscriptStore>(logger);
```
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/BotStartupClass.png" width="700" border="0"><br/>

- Add ITranscriptStore as a parameters to AdapterWithErrorHandler class, and .Use TranscriptLoggerMiddleware:
```ruby
     public AdapterWithErrorHandler(ITranscriptStore transcriptLogger, IConfiguration configuration, ILogger<BotFrameworkHttpAdapter> logger, ConversationState conversationState = null)
            : base(configuration, logger)
        {
            Use(new TranscriptLoggerMiddleware(transcriptLogger));
```
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="500" alt="image" src="/Content/AdapterWithErrorHandler.png" width="900" border="0"><br/>

- The connection string you have copied from Azure sql database, put that into your Bot project against `StoreConnectionString` key in appsettings.json file, something like shown below - 
    ```ruby
    {
      "MicrosoftAppId": "",
      "MicrosoftAppPassword": "",
      "StoreConnectionString": "Server=tcp:YourSQLServerName.database.windows.net,1433;Initial Catalog=YourDatabaseName;Persist Security Info=False;User ID=YourSqlUserId;Password=YourSqlPassword;MultipleActiveResultSets=False;Encrypt=True;TrustServerCertificate=False;Connection Timeout=30;"
    }
    ```
>Node: I have left `MicrosoftAppId` and `MicrosoftAppPassword` blank as I'll be running my Bot locally in Emulator but if your Bot is deployed on some remote server e.g. Azure app service or Bot needs to communicate to other external services e.g., Cognitive services, then you will need to put `MicrosoftAppId` and `MicrosoftAppPassword` as well to authenticate your Bot.

We are almost done. Now you should be able to run your Bot project.

## Let's see it in Action:
Run your bot code/ project (if you have already hosted it to some server e.g., Azure app service, you may skip this part) - 

- If you are running from terminal, navigate to your Bot project folder and run dotnet run command.
```ruby
#run the bot
dotnet run
```
- If you are running from Visual Studio
  - Launch Visual Studio
  - File -> Open -> Project/Solution
  - Navigate to your Bot project folder e.g. EntityFrameworkTranscriptStoreExample
  - Select your Bot project file e.g. EntityFrameworkTranscriptStoreExample.csproj file
  - Press F5 to run the project
>Note: As a prerequisite to run this project, make sure you have [.NET Core](https://dotnet.microsoft.com/en-us/download) (>=3.1) version installed on your machine.

### Test the Bot using Bot Framework Emulator:
[Bot Framework Emulator](https://github.com/microsoft/botframework-emulator) is a desktop application that allows bot developers to test and debug their bots on localhost or running remotely through a tunnel. You may install the Bot Framework Emulator from [here](https://github.com/Microsoft/BotFramework-Emulator/releases).

### Connect to the bot using Bot Framework Emulator:
- Launch Bot Framework Emulator
- File -> Open Bot
- Enter a Bot URL e.g., http://localhost:3978/api/messages . You may also enter the URL of remote web server hosting the Bot code but in that case, you will also need to provide `MicrosoftAppId` and `MicrosoftAppPassword` for Bot to successfully authenticate.

*Once connected you can send some messages to Bot, and it should echo it back.*
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="400" alt="image" src="/Content/BotActivitiesinEmulator.png" width="700" border="0"><br/>

### Logs:
Let's [Connect](https://abhimantiwari.github.io/blog/Saving-Bot-Activities-in-Azure-SQL/#connect-to-the-database) database to see if Bot Activities are being recorded.
You should see the messages saved into the Azure SQL tables we have created.
- `BotDataEntity` should store one entry for each unique connection.
- `TranscriptEntity` should contain all the conversations.
<img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px" height="600" alt="image" src="/Content/BotSqlTableTransactions.png" width="900" border="0"><br/>

If you query the tables, you should be able to fetch all the messages exchanged between user and Bot.                                                                    
  <img title="image" style="BORDER-RIGHT: 0px; BORDER-TOP: 0px; DISPLAY: inline; BORDER-LEFT: 0px; BORDER-BOTTOM: 0px; margin:0px 40px" height="400" alt="image" src="/Content/SavedBotActivities.png" width="600" border="0"><br/>

>Please note that depending on your Bot, you may need to specify in your Bot’s terms of use directly to notify your users whenever storing user data is involved.

Hope it helps! :-)