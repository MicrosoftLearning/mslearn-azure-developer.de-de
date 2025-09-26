---
lab:
  topic: Azure events and messaging
  title: Senden und Empfangen von Nachrichten aus Azure Queue Storage
  description: 'Hier erfahren Sie, wie Sie mit dem .NET Azure.StorageQueues-SDK Nachrichten aus Azure Queue Storage senden und empfangen.'
---

# Senden und Empfangen von Nachrichten aus Azure Queue Storage

In dieser Übung erstellen und konfigurieren Sie Azure Queue Storage-Ressourcen, und Sie erstellen anschließend eine .NET-App zum Senden und Empfangen von Nachrichten mithilfe des **Azure.Storage.Queues**-SDK. Sie erfahren, wie Sie Speicherressourcen bereitstellen, Warteschlangennachrichten verwalten und Ihre Umgebung bereinigen, wenn Sie fertig sind. 

In dieser Übung ausgeführte Aufgaben:

* Erstellen von Azure Queue Storage-Ressourcen
* Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen
* Erstellen einer .NET-Konsolen-App zum Senden und Empfangen von Nachrichten
* Bereinigen von Ressourcen

Diese Übung dauert ca. **30** Minuten.

## Erstellen von Azure Queue Storage-Ressourcen

In diesem Abschnitt der Übung erstellen Sie mit der Azure CLI die erforderlichen Ressourcen in Azure.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Wählen Sie in der Cloud Shell-Symbolleiste im Menü **Einstellungen** das Menüelement **Zur klassischen Version wechseln** aus (dies ist für die Verwendung des Code-Editors erforderlich).

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen.

    ```
    az group create --name myResourceGroup --location eastus
    ```

1. Viele der Befehle erfordern eindeutige Namen und verwenden dieselben Parameter. Durch das Erstellen einiger Variablen werden die Änderungen reduziert, die an den Befehlen zum Erstellen von Ressourcen erforderlich sind. Führen Sie die folgenden Befehle aus, um die erforderlichen Variablen zu erstellen. Ersetzen Sie **myResourceGroup** durch den Namen, den Sie für diese Übung verwenden. Wenn Sie den Speicherort im vorherigen Schritt geändert haben, nehmen Sie dieselbe Änderung in der Variablen **location** vor.

    ```
    resourceGroup=myResourceGroup
    location=eastus
    storAcctName=storactname$RANDOM
    ```

1. Sie benötigen den Namen, der dem Speicherkonto später in dieser Übung zugewiesen wird. Führen Sie den folgenden Befehl aus, und hinterlegen Sie die Ausgabe.

    ```
    echo $storAcctName
    ```

1. Führen Sie den folgenden Befehl aus, um mithilfe der zuvor erstellten Variablen ein Speicherkonto zu erstellen. Der Vorgang dauert einige Minuten.

    ```bash
    az storage account create --resource-group $resourceGroup \
        --name $storAcctName --location $location --sku Standard_LRS
    ```

### Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen

Damit Ihre App Nachrichten senden und empfangen kann, weisen Sie Ihrem Microsoft Entra-Benutzer die Rolle **Mitwirkender an Storage-Warteschlangendaten** zu. Dadurch erhält Ihr Benutzerkonto die Berechtigung, Warteschlangen zu erstellen und Nachrichten mithilfe von Azure RBAC zu senden bzw. zu empfangen. Führen Sie die folgenden Schritte in Cloud Shell aus.

1. Führen Sie den folgenden Befehl aus, um **userPrincipalName** aus Ihrem Konto abzurufen. Dadurch wird dargestellt, wem die Rolle zugewiesen wird.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Ressourcen-ID des Speicherkontos abzurufen. Die Ressourcen-ID legt den Bereich für die Rollenzuweisung auf einen bestimmten Namespace fest.

    ```
    resourceID=$(az storage account show --resource-group $resourceGroup \
        --name $storAcctName --query id --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Rolle **Mitwirkender an Storage-Warteschlangendaten** zu erstellen und zuzuweisen.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Storage Queue Data Contributor" \
        --scope $resourceID
    ```

## Erstellen einer .NET-Konsolen-App zum Senden und Empfangen von Nachrichten

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```
    mkdir queuestor
    cd queuestor
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Storage.Queues** und **Azure.Identity** hinzuzufügen.

    ```
    dotnet add package Azure.Storage.Queues
    dotnet add package Azure.Identity
    ```

### Hinzufügen des Startercodes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```
    code Program.cs
    ```

1. Ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Überprüfen Sie die Kommentare im Code, und ersetzen Sie **<YOUR-STORAGE-ACCT-NAME>** durch den Zuvor notierten Speicherkontonamen.

    ```csharp
    using Azure;
    using Azure.Identity;
    using Azure.Storage.Queues;
    using Azure.Storage.Queues.Models;
    using System;
    using System.Threading.Tasks;
    
    // Create a unique name for the queue
    // TODO: Replace the <YOUR-STORAGE-ACCT-NAME> placeholder 
    string queueName = "myqueue-" + Guid.NewGuid().ToString();
    string storageAccountName = "<YOUR-STORAGE-ACCT-NAME>";
    
    // ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE
    
    
    
    // ADD CODE TO SEND AND LIST MESSAGES
    
    
    
    // ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES
    
    
    
    // ADD CODE TO DELETE MESSAGES AND THE QUEUE
    
    
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern.

### Hinzufügen von Code zum Erstellen eines Warteschlangenclients und Erstellen einer Warteschlange

Jetzt können Sie Code hinzuzufügen, um den Warteschlangenspeicherclient und eine Warteschlange zu erstellen.

1. Suchen Sie den Kommentar **// ADD CODE TO CREATE A QUEUE CLIENT AND CREATE A QUEUE**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Instantiate a QueueClient to create and interact with the queue
    QueueClient queueClient = new QueueClient(
        new Uri($"https://{storageAccountName}.queue.core.windows.net/{queueName}"),
        new DefaultAzureCredential(options));
    
    Console.WriteLine($"Creating queue: {queueName}");
    
    // Create the queue
    await queueClient.CreateAsync();
    
    Console.WriteLine("Queue created, press Enter to add messages to the queue...");
    Console.ReadLine();
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und fahren Sie dann mit der Übung fort.

### Hinzufügen von Code zum Senden und Auflisten von Nachrichten in einer Warteschlange

1. Suchen Sie den Kommentar **// ADD CODE TO SEND AND LIST MESSAGES**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Send several messages to the queue with the SendMessageAsync method.
    await queueClient.SendMessageAsync("Message 1");
    await queueClient.SendMessageAsync("Message 2");
    
    // Send a message and save the receipt for later use
    SendReceipt receipt = await queueClient.SendMessageAsync("Message 3");
    
    Console.WriteLine("Messages added to the queue. Press Enter to peek at the messages...");
    Console.ReadLine();
    
    // Peeking messages lets you view the messages without removing them from the queue.
    
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to update a message in the queue...");
    Console.ReadLine();
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und fahren Sie dann mit der Übung fort.

### Hinzufügen von Code zum Aktualisieren einer Nachricht und Auflisten der Ergebnisse

1. Suchen Sie den Kommentar **// ADD CODE TO UPDATE A MESSAGE AND LIST MESSAGES**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Update a message with the UpdateMessageAsync method and the saved receipt
    await queueClient.UpdateMessageAsync(receipt.MessageId, receipt.PopReceipt, "Message 3 has been updated");
    
    Console.WriteLine("Message three updated. Press Enter to peek at the messages again...");
    Console.ReadLine();
    
    
    // Peek messages from the queue to compare updated content
    foreach (var message in (await queueClient.PeekMessagesAsync(maxMessages: 10)).Value)
    {
        Console.WriteLine($"Message: {message.MessageText}");
    }
    
    Console.WriteLine("\nPress Enter to delete messages from the queue...");
    Console.ReadLine();
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und fahren Sie dann mit der Übung fort.

### Hinzufügen von Code zum Löschen von Nachrichten und der Warteschlange

1. Suchen Sie den Kommentar **// ADD CODE TO DELETE MESSAGES AND THE QUEUE**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Delete messages from the queue with the DeleteMessagesAsync method.
    foreach (var message in (await queueClient.ReceiveMessagesAsync(maxMessages: 10)).Value)
    {
        // "Process" the message
        Console.WriteLine($"Deleting message: {message.MessageText}");
    
        // Let the service know we're finished with the message and it can be safely deleted.
        await queueClient.DeleteMessageAsync(message.MessageId, message.PopReceipt);
    }
    Console.WriteLine("Messages deleted from the queue.");
    Console.WriteLine("\nPress Enter key to delete the queue...");
    Console.ReadLine();
    
    // Delete the queue with the DeleteAsync method.
    Console.WriteLine($"Deleting queue: {queueClient.Name}");
    await queueClient.DeleteAsync();
    
    Console.WriteLine("Done");
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

1. Erweitern Sie im linken Navigationsbereich **> Datenspeicherung**, und wählen Sie **Warteschlangen** aus.

1. Wählen Sie die Warteschlange aus, die die Anwendung erstellt. Daraufhin können Sie die gesendeten Nachrichten anzeigen und überwachen, was die Anwendung ausführt.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.

