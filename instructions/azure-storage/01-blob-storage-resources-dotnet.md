---
lab:
  topic: Azure Storage
  title: Erstellen von Blob Storage-Ressourcen mit der .NET-Clientbibliothek
  description: 'Hier erfahren Sie, wie Sie die Azure Storage .NET-Clientbibliothek verwenden, um Container zu erstellen, Blobs hochzuladen und aufzulisten sowie Container zu löschen.'
---

# Erstellen von Blob Storage-Ressourcen mit der .NET-Clientbibliothek

In dieser Übung erstellen Sie ein Azure Storage-Konto, und Sie erstellen eine .NET-Konsolenanwendung mithilfe der Azure Storage Blob-Clientbibliothek, um Container zu erstellen, Dateien in Blob Storage hochzuladen, Blobs aufzulisten und Dateien herunterzuladen. Sie erfahren, wie Sie sich bei Azure authentifizieren, Blob Storage-Vorgänge programmgesteuert ausführen und die Ergebnisse im Azure-Portal überprüfen.

In dieser Übung ausgeführte Aufgaben:

* Vorbereiten der Azure-Ressourcen
* Erstellen einer Konsolen-App zum Erstellen und Herunterladen von Daten
* Ausführen der App und Überprüfen der Ergebnisse
* Bereinigen von Ressourcen

Diese Übung dauert ca. **30** Minuten.

## Erstellen Sie ein Azure Storage-Konto.

In diesem Abschnitt der Übung erstellen Sie mit der Azure CLI die erforderlichen Ressourcen in Azure.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Wählen Sie in der Cloud Shell-Symbolleiste im Menü **Einstellungen** das Menüelement **Zur klassischen Version wechseln** aus (dies ist für die Verwendung des Code-Editors erforderlich).

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus2** bei Bedarf durch eine Region in Ihrer Nähe ersetzen. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort.

    ```
    az group create --location eastus2 --name myResourceGroup
    ```

1. Viele der Befehle erfordern eindeutige Namen und verwenden dieselben Parameter. Durch das Erstellen einiger Variablen werden die Änderungen reduziert, die an den Befehlen zum Erstellen von Ressourcen erforderlich sind. Führen Sie die folgenden Befehle aus, um die erforderlichen Variablen zu erstellen. Ersetzen Sie **myResourceGroup** durch den Namen, den Sie für diese Übung verwenden.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    accountName=storageacct$RANDOM
    ```

1. Führen Sie die folgenden Befehle aus, um das Azure Storage-Konto zu erstellen. Dabei muss jeder Kontoname eindeutig sein. Mit dem ersten Befehl wird eine Variable mit einem eindeutigen Namen für Ihr Speicherkonto erstellt. Notieren Sie den Namen Ihres Kontos aus der Ausgabe des **echo**-Befehls. 

    ```
    az storage account create --name $accountName \
        --resource-group $resourceGroup \
        --location $location \
        --sku Standard_LRS 
    
    echo $accountName
    ```

### Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen

Damit Ihre App Ressourcen und Elemente erstellen kann, weisen Sie Ihrem Microsoft Entra-Benutzer die Rolle **Storage Blob-Datenbesitzer** zu. Führen Sie die folgenden Schritte in Cloud Shell aus.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie den folgenden Befehl aus, um **userPrincipalName** aus Ihrem Konto abzurufen. Dadurch wird dargestellt, wem die Rolle zugewiesen wird.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Ressourcen-ID des Speicherkontos abzurufen. Die Ressourcen-ID legt den Bereich für die Rollenzuweisung auf einen bestimmten Namespace fest.

    ```
    resourceID=$(az storage account show --name $accountName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. Führen Sie den folgenden Befehl aus, um die Rolle **Storage Blob-Datenbesitzer** zu erstellen und zuzuweisen. Mit dieser Rolle erhalten Sie die Berechtigungen zum Verwalten von Containern und Elementen.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Blob Data Owner" \
        --scope $resourceID
    ```

## Erstellen einer .NET-Konsolen-App zum Erstellen von Containern und Elementen

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```
    mkdir azstor
    cd azstor
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um die erforderlichen Pakete in der Anwendung hinzuzufügen.

    ```
    dotnet add package Azure.Storage.Blobs
    dotnet add package Azure.Identity
    ```

1. Führen Sie den folgenden Befehl aus, um in Ihrem Projekt den Ordner **data** zu erstellen. 

    ```
    mkdir data
    ```

Jetzt können Sie den Code für das Projekt hinzufügen.

### Hinzufügen des Startercodes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```
    code Program.cs
    ```

1. Ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    using Azure.Storage.Blobs;
    using Azure.Storage.Blobs.Models;
    using Azure.Identity;
    
    Console.WriteLine("Azure Blob Storage exercise\n");
    
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Run the examples asynchronously, wait for the results before proceeding
    await ProcessAsync();
    
    Console.WriteLine("\nPress enter to exit the sample application.");
    Console.ReadLine();
    
    async Task ProcessAsync()
    {
        // CREATE A BLOB STORAGE CLIENT
        
    
    
        // CREATE A CONTAINER
        
    
    
        // CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE
        
    
    
        // UPLOAD THE FILE TO BLOB STORAGE
        
    
    
        // LIST BLOBS IN THE CONTAINER
        
    
    
        // DOWNLOAD THE BLOB TO A LOCAL FILE
        
    
    }
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.


## Hinzufügen von Code zum des Projekts

Während der restlichen Übung fügen Sie Code in bestimmten Bereichen hinzu, um die vollständige Anwendung zu erstellen. 

1. Suchen Sie den Kommentar **// CREATE A BLOB STORAGE CLIENT**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. **BlobServiceClient** fungiert als primärer Einstiegspunkt für die Verwaltung von Containern und Blobs in einem Speicherkonto. Der Client verwendet *DefaultAzureCredential* für die Authentifizierung. Ersetzen Sie **IHR_KONTONAME** unbedingt durch den Namen, den Sie zuvor notiert haben.

    ```csharp
    // Create a credential using DefaultAzureCredential with configured options
    string accountName = "YOUR_ACCOUNT_NAME"; // Replace with your storage account name
    
    // Use the DefaultAzureCredential with the options configured at the top of the program
    DefaultAzureCredential credential = new DefaultAzureCredential(options);
    
    // Create the BlobServiceClient using the endpoint and DefaultAzureCredential
    string blobServiceEndpoint = $"https://{accountName}.blob.core.windows.net";
    BlobServiceClient blobServiceClient = new BlobServiceClient(new Uri(blobServiceEndpoint), credential);
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.

1. Suchen Sie den Kommentar **// CREATE A CONTAINER**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. Das Erstellen eines Containers umfasst das Erstellen einer Instanz der **BlobServiceClient**-Klasse und das anschließende Aufrufen der **CreateBlobContainerAsync**-Methode, um den Container in Ihrem Speicherkonto zu erstellen. Ein GUID-Wert wird an den Containernamen angehängt, um sicherzustellen, dass er eindeutig ist. Bei der **CreateBlobContainerAsync**-Methode tritt ein Fehler auf, wenn der Container bereits vorhanden ist.

    ```csharp
    // Create a unique name for the container
    string containerName = "wtblob" + Guid.NewGuid().ToString();
    
    // Create the container and return a container client object
    Console.WriteLine("Creating container: " + containerName);
    BlobContainerClient containerClient = 
        await blobServiceClient.CreateBlobContainerAsync(containerName);
    
    // Check if the container was created successfully
    if (containerClient != null)
    {
        Console.WriteLine("Container created successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("Failed to create the container, exiting program.");
        return;
    }
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.

1. Suchen Sie den Kommentar **// CREATE A LOCAL FILE FOR UPLOAD TO BLOB STORAGE**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. Dadurch wird eine Datei in dem Datenverzeichnis erstellt, das in den Container hochgeladen wird.

    ```csharp
    // Create a local file in the ./data/ directory for uploading and downloading
    Console.WriteLine("Creating a local file for upload to Blob storage...");
    string localPath = "./data/";
    string fileName = "wtfile" + Guid.NewGuid().ToString() + ".txt";
    string localFilePath = Path.Combine(localPath, fileName);
    
    // Write text to the file
    await File.WriteAllTextAsync(localFilePath, "Hello, World!");
    Console.WriteLine("Local file created, press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.

1. Suchen Sie den Kommentar **// UPLOAD THE FILE TO BLOB STORAGE**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. Der Code ruft einen Verweis auf ein **BlobClient**-Objekt ab, indem die **GetBlobClient**-Methode für den im vorherigen Abschnitt erstellten Container aufgerufen wird. Anschließend wird mithilfe der **UploadAsync**-Methode eine generierte lokale Datei hochgeladen. Mit dieser Methode wird das Blob erstellt, falls es nicht vorhanden ist, oder überschrieben, sofern es bereits vorhanden ist.

    ```csharp
    // Get a reference to the blob and upload the file
    BlobClient blobClient = containerClient.GetBlobClient(fileName);
    
    Console.WriteLine("Uploading to Blob storage as blob:\n\t {0}", blobClient.Uri);
    
    // Open the file and upload its data
    using (FileStream uploadFileStream = File.OpenRead(localFilePath))
    {
        await blobClient.UploadAsync(uploadFileStream);
        uploadFileStream.Close();
    }
    
    // Verify if the file was uploaded successfully
    bool blobExists = await blobClient.ExistsAsync();
    if (blobExists)
    {
        Console.WriteLine("File uploaded successfully, press 'Enter' to continue.");
        Console.ReadLine();
    }
    else
    {
        Console.WriteLine("File upload failed, exiting program..");
        return;
    }
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.

1. Suchen Sie den Kommentar **// LIST BLOBS IN THE CONTAINER**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. Sie listen die Blobs im Container mit der **GetBlobsAsync**-Methode auf. In diesem Fall wurde dem Container nur ein Blob hinzugefügt, sodass beim Auflisten auch nur ein Blob zurückgegeben wird. 

    ```csharp
    Console.WriteLine("Listing blobs in container...");
    await foreach (BlobItem blobItem in containerClient.GetBlobsAsync())
    {
        Console.WriteLine("\t" + blobItem.Name);
    }
    
    Console.WriteLine("Press 'Enter' to continue.");
    Console.ReadLine();
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern, und fahren Sie mit dem nächsten Schritt fort.

1. Suchen Sie den Kommentar **// DOWNLOAD THE BLOB TO A LOCAL FILE**, und fügen Sie dann den folgenden Code direkt unter dem Kommentar hinzu. Im Code wird die **DownloadAsync**-Methode verwendet, um das zuvor erstellte Blob in Ihr lokales Dateisystem herunterzuladen. Der Beispielcode fügt dem Blobnamen den Suffix „DOWNLOADED“ hinzu, damit Sie beide Dateien im lokalen Dateisystem anzeigen können. 

    ```csharp
    // Adds the string "DOWNLOADED" before the .txt extension so it doesn't 
    // overwrite the original file
    
    string downloadFilePath = localFilePath.Replace(".txt", "DOWNLOADED.txt");
    
    Console.WriteLine("Downloading blob to: {0}", downloadFilePath);
    
    // Download the blob's contents and save it to a file
    BlobDownloadInfo download = await blobClient.DownloadAsync();
    
    using (FileStream downloadFileStream = File.OpenWrite(downloadFilePath))
    {
        await download.Content.CopyToAsync(downloadFileStream);
    }
    
    Console.WriteLine("Blob downloaded successfully to: {0}", downloadFilePath);
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und dann **STRG+Q**, um den Editor zu beenden.

## Bei Azure anmelden und die App ausführen

1. Geben Sie im Befehlszeilenbereich der Cloud-Shell den folgenden Befehl ein, um sich bei Azure anzumelden.

    ```
    az login
    ```

    **<font color="red">Sie müssen sich bei Azure anmelden - auch wenn die Cloud-Shell-Sitzung bereits authentifiziert ist.</font>**

    > **Hinweis**: In den meisten Szenarien ist nur die Verwendung von *az login* ausreichend. Wenn Sie jedoch Abonnements in mehreren Mandqanten haben, müssen Sie möglicherweise den Mandanten mit dem Parameter *--tenant* angeben. Weitere Informationen finden Sie unter [Interaktives Anmelden bei Azure mithilfe der Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Führen Sie den folgenden Befehl aus, um die Konsolen-App zu starten. Die App wird während der Ausführung mehrmals angehalten und wartet darauf, dass Sie eine beliebige Taste drücken, um den Vorgang fortzusetzen. Dadurch haben Sie die Möglichkeit, die Nachrichten im Azure-Portal anzuzeigen.

    ```
    dotnet run
    ```

1. Navigieren Sie im Azure-Portal zu dem Azure Storage-Konto, das Sie erstellt haben. 

1. Erweitern Sie im linken Navigationsbereich **> Datenspeicherung**, und wählen Sie **Container** aus.

1. Wählen Sie den Container aus, den die Anwendung erstellt hat. Daraufhin können Sie das hochgeladene Blob anzeigen.

1. Führen Sie die beiden folgenden Befehle aus, um in das Verzeichnis **data** zu wechseln und die Dateien auflisten, die hoch- und heruntergeladen wurden.

    ```
    cd data
    ls
    ```

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.

