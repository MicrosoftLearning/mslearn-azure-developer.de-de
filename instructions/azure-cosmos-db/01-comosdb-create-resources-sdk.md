---
lab:
  topic: Azure Cosmos DB
  title: Erstellen von Ressourcen in Azure Cosmos DB for NoSQL mit .NET
  description: 'Erfahren Sie, wie Sie Datenbank- und Container-Ressourcen in Azure Cosmos DB mit dem Microsoft .NET SDK v3 erstellen.'
---

# Erstellen von Ressourcen in Azure Cosmos DB for NoSQL mit .NET

In dieser Übung erstellen Sie ein Azure Cosmos DB-Konto und eine .NET-Konsolenanwendung, die das Microsoft Azure Cosmos DB-SDK verwendet, um eine Datenbank, einen Container und ein Beispielelement zu erstellen. Sie erfahren, wie Sie die Authentifizierung konfigurieren, Datenbankvorgänge programmgesteuert ausführen und Ihre Ergebnisse im Azure-Portal überprüfen.

In dieser Übung ausgeführte Aufgaben:

* Erstellen eines Azure Cosmos DB-Kontos
* Erstellen einer Konsolen-App, die eine Datenbank, einen Container und ein Element erstellt
* Ausführen der Konsolen-App und Überprüfen der Ergebnisse

Diese Übung dauert ca. **30** Minuten.

## Erstellen eines Azure Cosmos DB-Kontos

In diesem Abschnitt der Übung erstellen Sie eine Ressourcengruppe und Azure Cosmos DB-Konto. Sie notieren auch den Endpunkt und den Zugriffsschlüssel für das Konto.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Wählen Sie in der Cloud Shell-Symbolleiste im Menü **Einstellungen** das Menüelement **Zur klassischen Version wechseln** aus (dies ist für die Verwendung des Code-Editors erforderlich).

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. Viele der Befehle erfordern eindeutige Namen und verwenden dieselben Parameter. Durch das Erstellen einiger Variablen werden die Änderungen reduziert, die an den Befehlen zum Erstellen von Ressourcen erforderlich sind. Führen Sie die folgenden Befehle aus, um die erforderlichen Variablen zu erstellen. Ersetzen Sie **myResourceGroup** durch den Namen, den Sie für diese Übung verwenden.

    ```
    resourceGroup=myResourceGroup
    accountName=cosmosexercise$RANDOM
    ```

1. Führen Sie die folgenden Befehle aus, um das Azure Cosmos DB-Konto zu erstellen. Dabei muss jeder Kontoname eindeutig sein. 

    ```
    az cosmosdb create --name $accountName \
        --resource-group $resourceGroup
    ```

1.  Führen Sie den folgenden Befehl aus, um den **documentEndpoint** für das Azure Cosmos DB-Konto abzurufen. Notieren Sie sich den Endpunkt aus den Befehlsergebnissen, er wird später in der Übung benötigt.

    ```
    az cosmosdb show --name $accountName \
        --resource-group $resourceGroup \
        --query "documentEndpoint" --output tsv
    ```

1. Rufen Sie mit dem folgenden Befehl den Primärschlüssel für das Konto ab. Notieren Sie sich den Primärschlüssel aus den Befehlsergebnissen. Dieser wird später in der Übung benötigt.

    ```
    az cosmosdb keys list --name $accountName \
        --resource-group $resourceGroup \
        --query "primaryMasterKey" --output tsv
    ```

## Erstellen von Datenressourcen und eines Elements mit einer .NET-Konsolenanwendung

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Erstellen Sie einen Ordner für das Projekt, und öffnen Sie diesen.

    ```bash
    mkdir cosmosdb
    cd cosmosdb
    ```

1. Erstellen Sie die .NET-Konsolen-App.

    ```bash
    dotnet new console
    ```

### Konfigurieren der Konsolenanwendung

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Microsoft.Azure.Cosmos**, **Newtonsoft.Json**und **dotenv.net** hinzuzufügen.

    ```bash
    dotnet add package Microsoft.Azure.Cosmos --version 3.*
    dotnet add package Newtonsoft.Json --version 13.*
    dotnet add package dotenv.net
    ```

1. Führen Sie den folgenden Befehl aus, um die **ENV**-Datei zum Aufbewahren der Geheimnisse zu erstellen, und öffnen Sie diese dann im Code-Editor.

    ```bash
    touch .env
    code .env
    ```

1. Fügen Sie der **ENV**-Datei den folgenden Code hinzu. Ersetzen Sie **YOUR_DOCUMENT_ENDPOINT** und **YOUR_ACCOUNT_KEY** durch die zuvor von Ihnen notierten Werte.

    ```
    DOCUMENT_ENDPOINT="YOUR_DOCUMENT_ENDPOINT"
    ACCOUNT_KEY="YOUR_ACCOUNT_KEY"
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und dann **STRG+Q**, um den Editor zu beenden.

Nun ist es an der Zeit, den Vorlagencode in der Datei **Program.cs** mithilfe des Editors in der Cloud Shell zu ersetzen.

### Hinzufügen des Startcodes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```bash
    code Program.cs
    ```

1. Ersetzen Sie den vorhandenen Code durch den folgenden Codeschnipsel. 

    Der Code bildet die Gesamtstruktur der App. Überprüfen Sie die Kommentare im Code, um die Funktionsweise nachzuvollziehen. Später in der Übung fügen Sie in den angegebenen Bereichen Code hinzu, um die Anwendung zu vervollständigen. 

    ```csharp
    using Microsoft.Azure.Cosmos;
    using dotenv.net;
    
    string databaseName = "myDatabase"; // Name of the database to create or use
    string containerName = "myContainer"; // Name of the container to create or use
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    string cosmosDbAccountUrl = envVars["DOCUMENT_ENDPOINT"];
    string accountKey = envVars["ACCOUNT_KEY"];
    
    if (string.IsNullOrEmpty(cosmosDbAccountUrl) || string.IsNullOrEmpty(accountKey))
    {
        Console.WriteLine("Please set the DOCUMENT_ENDPOINT and ACCOUNT_KEY environment variables.");
        return;
    }
    
    // CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY
    
    
    try
    {
        // CREATE A DATABASE IF IT DOESN'T ALREADY EXIST
    
    
        // CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY
    
    
        // DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER
    
    
        // ADD THE ITEM TO THE CONTAINER
    
    
    }
    catch (CosmosException ex)
    {
        // Handle Cosmos DB-specific exceptions
        // Log the status code and error message for debugging
        Console.WriteLine($"Cosmos DB Error: {ex.StatusCode} - {ex.Message}");
    }
    catch (Exception ex)
    {
        // Handle general exceptions
        // Log the error message for debugging
        Console.WriteLine($"Error: {ex.Message}");
    }
    
    // This class represents a product in the Cosmos DB container
    public class Product
    {
        public string? id { get; set; }
        public string? name { get; set; }
        public string? description { get; set; }
    }
    ```

Als Nächstes fügen Sie in den angegebenen Bereichen der Projekte Code hinzu, um Folgendes zu erstellen: Client, Datenbank und Container. Außerdem fügen Sie dem Container ein Beispielelement hinzu.

### Hinzufügen von Code zum Erstellen des Clients und Ausführen von Vorgängen 

1. Fügen Sie den folgenden Code im Bereich nach dem Kommentar **// CREATE THE COSMOS DB CLIENT USING THE ACCOUNT URL AND KEY** hinzu. Dieser Code definiert den Client, der zum Herstellen einer Verbindung mit Ihrem Azure Cosmos DB-Konto verwendet wird.

    ```csharp
    CosmosClient client = new(
        accountEndpoint: cosmosDbAccountUrl,
        authKeyOrResourceToken: accountKey
    );
    ```

    >Hinweis: Es empfiehlt sich, **DefaultAzureCredential** aus der *Azure Identity-Bibliothek* zu verwenden. Je nachdem, wie Ihr Abonnement eingerichtet ist, sind hierfür möglicherweise einige zusätzliche Konfigurationen in Azure erforderlich. 

1. Fügen Sie den folgenden Code im Bereich nach dem Kommentar **// CREATE A DATABASE IF IT DOESN'T ALREADY EXIST** hinzu. 

    ```csharp
    Database database = await client.CreateDatabaseIfNotExistsAsync(databaseName);
    Console.WriteLine($"Created or retrieved database: {database.Id}");
    ```

1. Fügen Sie den folgenden Code im Bereich nach dem Kommentar **// CREATE A CONTAINER WITH A SPECIFIED PARTITION KEY** hinzu. 

    ```csharp
    Container container = await database.CreateContainerIfNotExistsAsync(
        id: containerName,
        partitionKeyPath: "/id"
    );
    Console.WriteLine($"Created or retrieved container: {container.Id}");
    ```

1. Fügen Sie den folgenden Code im Bereich nach dem Kommentar **// DEFINE A TYPED ITEM (PRODUCT) TO ADD TO THE CONTAINER** hinzu. Dadurch wird das Element definiert, das dem Container hinzugefügt wird.

    ```csharp
    Product newItem = new Product
    {
        id = Guid.NewGuid().ToString(), // Generate a unique ID for the product
        name = "Sample Item",
        description = "This is a sample item in my Azure Cosmos DB exercise."
    };
    ```

1. Fügen Sie den folgenden Code im Bereich nach dem Kommentar **// ADD THE ITEM TO THE CONTAINER** hinzu. 

    ```csharp
    ItemResponse<Product> createResponse = await container.CreateItemAsync(
        item: newItem,
        partitionKey: new PartitionKey(newItem.id)
    );

    Console.WriteLine($"Created item with ID: {createResponse.Resource.id}");
    Console.WriteLine($"Request charge: {createResponse.RequestCharge} RUs");
    ```

1. Nachdem der Code vollständig ist, speichern Sie den Fortschritt mit **STRG+S**, um die Datei zu speichern, und **STRG+Q**, um den Editor zu beenden.

1. Führen Sie den folgenden Befehl in der Cloudshell aus, um auf Fehler im Projekt zu testen. Sollten Sie Fehler feststellen, öffnen Sie die Datei *Program.cs* im Editor und überprüfen Sie diese auf fehlenden Code oder Fehler beim Einfügen.

    ```
    dotnet build
    ```

Nachdem das Projekt fertig ist, ist es an der Zeit, die Anwendung auszuführen und die Ergebnisse im Azure-Portal zu überprüfen.

## Ausführen der Anwendung und Überprüfen der Ergebnisse

1. Führen Sie den Befehl `dotnet run` aus, wenn Sie sich in der Cloud Shell befinden. Die Ausgabe sollte in etwa dem folgenden Beispiel entsprechen.

    ```
    Created or retrieved database: myDatabase
    Created or retrieved container: myContainer
    Created item: c549c3fa-054d-40db-a42b-c05deabbc4a6
    Request charge: 6.29 RUs
    ```

1. Navigieren Sie im Azure-Portal zu der zuvor erstellten Azure Cosmos DB-Ressource. Wählen Sie in der linken Navigation **Daten-Explorer** aus. Wählen Sie im **Daten-Explorer** **myDatabase** und erweitern Sie dann **myContainer**. Sie können den von Ihnen erstellten Artikel anzeigen, indem Sie **Artikel** auswählen.

    ![Screenshot, der die Position der Elemente im Daten-Explorer anzeigt.](./media/01/cosmos-data-explorer.png)

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
