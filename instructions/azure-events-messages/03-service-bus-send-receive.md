---
lab:
  topic: Azure events and messaging
  title: Senden und Empfangen von Nachrichten von Azure Service Bus
  description: 'Hier erfahren Sie, wie Sie Nachrichten von Azure Service Bus mit dem .NET Azure.Messaging.ServiceBus-SDK senden und empfangen.'
---

# Senden und Empfangen von Nachrichten von Azure Service Bus

In dieser Übung erstellen und konfigurieren Sie Azure Service Bus-Ressourcen und dann eine .NET-App zum Senden und Empfangen von Nachrichten mithilfe des **Azure.Messaging.ServiceBus**-SDK. Sie erfahren, wie Sie einen Service Bus-Namespace und eine Warteschlange bereitstellen, Berechtigungen zuweisen und programmgesteuert mit Nachrichten interagieren. 

In dieser Übung ausgeführte Aufgaben:

* Erstellen von Azure Service Bus-Ressourcen
* Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen
* Erstellen einer .NET-Konsolen-App zum Senden und Empfangen von Nachrichten
* Bereinigen von Ressourcen

Diese Übung dauert ca. **30** Minuten.

## Erstellen von Azure Event Hubs-Ressourcen

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
    namespaceName=svcbusns$RANDOM
    ```

1. Sie benötigen den Namen, der dem Namespace später in dieser Übung zugewiesen wird. Führen Sie den folgenden Befehl aus, und hinterlegen Sie die Ausgabe.

    ```
    echo $namespaceName
    ```

### Erstellen eines Azure Service Bus-Namespace und einer Warteschlange

1. Erstellen eines Namespaces für Service Bus-Messaging. Mit dem folgenden Befehl wird ein Namespace mit der zuvor erstellten Variable erstellt. Der Vorgang dauert einige Minuten.

    ```bash
    az servicebus namespace create \
        --resource-group $resourceGroup \
        --name $namespaceName \
        --location $location
    ```

1. Nachdem ein Namespace erstellt wurde, müssen Sie eine Warteschlange erstellen, in der die Nachrichten gespeichert werden. Führen Sie den folgenden Befehl aus, um eine Warteschlange mit dem Namen **myqueue** zu erstellen.

    ```bash
    az servicebus queue create --resource-group $resourceGroup \
        --namespace-name $namespaceName \
        --name myqueue
    ```

### Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen

Damit Ihre App Nachrichten senden und empfangen kann, weisen Sie auf Ebene des Service Bus-Namespace Ihrem Microsoft Entra-Benutzer die Rolle **Azure Service Bus-Datenbesitzer** zu. Dadurch erhält Ihr Benutzerkonto die Berechtigung, mithilfe von Azure RBAC Warteschlangen und Themen zu verwalten und auf diese zuzugreifen. Führen Sie die folgenden Schritte in Cloud Shell aus.

1. Führen Sie den folgenden Befehl aus, um **userPrincipalName** aus Ihrem Konto abzurufen. Dadurch wird dargestellt, wem die Rolle zugewiesen wird.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Ressourcen-ID des Service Bus-Namespace abzurufen. Die Ressourcen-ID legt den Bereich für die Rollenzuweisung auf einen bestimmten Namespace fest.

    ```
    resourceID=$(az servicebus namespace show --name $namespaceName \
        --resource-group $resourceGroup \
        --query id --output tsv)
    ```
1. Führen Sie den folgenden Befehl aus, um die Rolle **Azure Service Bus-Datenbesitzer** zu erstellen und zuzuweisen.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Service Bus Data Owner" \
        --scope $resourceID
    ```

## Erstellen einer .NET-Konsolen-App zum Senden und Empfangen von Nachrichten

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```
    mkdir svcbus
    cd svcbus
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Messaging.ServiceBus** und **Azure.Identity** hinzuzufügen.

    ```
    dotnet add package Azure.Messaging.ServiceBus
    dotnet add package Azure.Identity
    ```

### Hinzufügen des Startercodes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```
    code Program.cs
    ```

1. Ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Überprüfen Sie unbedingt die Kommentare im Code, und ersetzen Sie **<YOUR-NAMESPACE>** durch den Service Bus-Namespace, den Sie zuvor notiert haben.

    ```csharp
    using Azure.Messaging.ServiceBus;
    using Azure.Identity;
    using System.Timers;
    
    
    // TODO: Replace <YOUR-NAMESPACE> with your Service Bus namespace
    string svcbusNameSpace = "<YOUR-NAMESPACE>.servicebus.windows.net";
    string queueName = "myQueue";
    
    
    // ADD CODE TO CREATE A SERVICE BUS CLIENT
    
    
    
    // ADD CODE TO SEND MESSAGES TO THE QUEUE
    
    
    
    // ADD CODE TO PROCESS MESSAGES FROM THE QUEUE
    
    
    
    // Dispose client after use
    await client.DisposeAsync();
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern.

### Hinzufügen von Code zum Senden von Nachrichten an die Warteschlange

Jetzt können Sie Code hinzufügen, um den Service Bus-Client zu erstellen und einen Batch von Nachrichten an die Warteschlange zu senden.

1. Suchen Sie den Kommentar **// ADD CODE TO CREATE A SERVICE BUS CLIENT**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Create a DefaultAzureCredentialOptions object to configure the DefaultAzureCredential
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a Service Bus client using the namespace and DefaultAzureCredential
    // The DefaultAzureCredential will use the Azure CLI credentials, so ensure you are logged in
    ServiceBusClient client = new(svcbusNameSpace, new DefaultAzureCredential(options));
    ```

1. Suchen Sie den Kommentar **// ADD CODE TO SEND MESSAGES TO THE QUEUE**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Create a sender for the specified queue
    ServiceBusSender sender = client.CreateSender(queueName);
    
    // create a batch 
    using ServiceBusMessageBatch messageBatch = await sender.CreateMessageBatchAsync();
    
    // number of messages to be sent to the queue
    const int numOfMessages = 3;
    
    for (int i = 1; i <= numOfMessages; i++)
    {
        // try adding a message to the batch
        if (!messageBatch.TryAddMessage(new ServiceBusMessage($"Message {i}")))
        {
            // if it is too large for the batch
            throw new Exception($"The message {i} is too large to fit in the batch.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of messages to the Service Bus queue
        await sender.SendMessagesAsync(messageBatch);
        Console.WriteLine($"A batch of {numOfMessages} messages has been published to the queue.");
    }
    finally
    {
        // Calling DisposeAsync on client types is required to ensure that network
        // resources and other unmanaged objects are properly cleaned up.
        await sender.DisposeAsync();
    }
    
    Console.WriteLine("Press any key to continue");
    Console.ReadKey();
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und fahren Sie dann mit der Übung fort.

### Hinzufügen von Code zum Verarbeiten von Nachrichten in der Warteschlange

1. Suchen Sie den Kommentar **// ADD CODE TO PROCESS MESSAGES FROM THE QUEUE**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt den Code und die Kommentare.

    ```csharp
    // Create a processor that we can use to process the messages in the queue
    ServiceBusProcessor processor = client.CreateProcessor(queueName, new ServiceBusProcessorOptions());
    
    // Idle timeout in milliseconds, the idle timer will stop the processor if there are no more 
    // messages in the queue to process
    const int idleTimeoutMs = 3000;
    System.Timers.Timer idleTimer = new(idleTimeoutMs);
    idleTimer.Elapsed += async (s, e) =>
    {
        Console.WriteLine($"No messages received for {idleTimeoutMs / 1000} seconds. Stopping processor...");
        await processor.StopProcessingAsync();
    };
    
    try
    {
        // add handler to process messages
        processor.ProcessMessageAsync += MessageHandler;
    
        // add handler to process any errors
        processor.ProcessErrorAsync += ErrorHandler;
    
        // start processing 
        idleTimer.Start();
        await processor.StartProcessingAsync();
    
        Console.WriteLine($"Processor started. Will stop after {idleTimeoutMs / 1000} seconds of inactivity.");
        // Wait for the processor to stop
        while (processor.IsProcessing)
        {
            await Task.Delay(500);
        }
        idleTimer.Stop();
        Console.WriteLine("Stopped receiving messages");
    }
    finally
    {
        // Dispose processor after use
        await processor.DisposeAsync();
    }
    
    // handle received messages
    async Task MessageHandler(ProcessMessageEventArgs args)
    {
        string body = args.Message.Body.ToString();
        Console.WriteLine($"Received: {body}");
    
        // Reset the idle timer on each message
        idleTimer.Stop();
        idleTimer.Start();
    
        // complete the message. message is deleted from the queue. 
        await args.CompleteMessageAsync(args.Message);
    }
    
    // handle any errors when receiving messages
    Task ErrorHandler(ProcessErrorEventArgs args)
    {
        Console.WriteLine(args.Exception.ToString());
        return Task.CompletedTask;
    }
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und dann **STRG+Q**, um den Editor zu beenden.

## Bei Azure anmelden und die App ausführen

1. Geben Sie im Befehlszeilenbereich der Cloud-Shell den folgenden Befehl ein, um sich bei Azure anzumelden.

    ```
    az login
    ```

    **<font color="red">Sie müssen sich bei Azure anmelden - auch wenn die Cloud-Shell-Sitzung bereits authentifiziert ist.</font>**

    > **Hinweis**: In den meisten Szenarien ist nur die Verwendung von *az login* ausreichend. Wenn Sie jedoch Abonnements in mehreren Mandqanten haben, müssen Sie möglicherweise den Mandanten mit dem Parameter *--tenant* angeben. Weitere Informationen finden Sie unter [Interaktives Anmelden bei Azure mithilfe der Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Führen Sie den folgenden Befehl aus, um die Konsolen-App zu starten. Die App hält in verschiedenen Phasen an und fordert Sie auf, eine Taste zu drücken, um fortzufahren. Dadurch haben Sie die Möglichkeit, die Nachrichten im Azure-Portal anzuzeigen.

    ```
    dotnet run
    ```

    

1. Navigieren Sie im Azure-Portal zu dem Service Bus-Namespace, den Sie erstellt haben. 

1. Wählen Sie am unteren Rand des Fensters **Übersicht** die Option **myqueue** aus.

1. Wählen Sie im linken Navigationsbereich die Option **Service Bus Explorer** aus.

1. Wählen Sie **Vorschau ab Start** aus. Nach einigen Sekunden sollten daraufhin die drei Nachrichten angezeigt werden.

1. Drücken Sie in Cloud Shell eine beliebige Taste, um fortzufahren. Daraufhin werden die drei Nachrichten durch die Anwendung verarbeitet. 
 
1. Kehren Sie zum Portal zurück, nachdem die Anwendung die Verarbeitung der Nachrichten abgeschlossen hat. Wählen Sie **Vorschau ab Start** erneut aus, und beachten Sie, dass keine Nachrichten in der Warteschlange vorhanden sind.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht. 

