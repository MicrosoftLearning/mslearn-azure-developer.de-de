---
lab:
  topic: Azure events and messaging
  title: Senden und Abrufen von Ereignissen aus Azure Event Hubs
  description: 'Hier erfahren Sie, wie Ereignisse aus Azure Event Hubs mit dem .NET-SDK Azure.Messaging.EventHubs gesendet und abgerufen werden.'
---

# Senden und Abrufen von Ereignissen aus Azure Event Hubs

In dieser Übung erstellen Sie Azure Event Hubs-Ressourcen und eine .NET-Konsolen-App zum Senden und Empfangen von Ereignissen mithilfe des SDK **Azure.Messaging.EventHubs**. Sie erfahren, wie Sie Cloudressourcen bereitstellen, mit Event Hubs interagieren und Ihre Umgebung nach Abschluss bereinigen.

In dieser Übung ausgeführte Aufgaben:

* Erstellen einer Ressourcengruppe
* Erstellen von Azure Event Hubs-Ressourcen
* Erstellen einer .NET-Konsolen-App zum Senden und Abrufen von Ereignissen
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
    namespaceName=eventhubsns$RANDOM
    ```

### Erstellen eines Azure Event Hubs-Namespace und eines Event Hubs

Ein Azure Event Hubs-Namespace ist ein logischer Container für Event Hub-Ressourcen in Azure. Er bietet einen eindeutigen Bereichscontainer, in dem Sie einen oder mehrere Event Hubs erstellen können, die zum Erfassen, Verarbeiten und Speichern großer Mengen von Ereignisdaten verwendet werden. Die folgenden Anweisungen werden in Cloud Shell ausgeführt. 

1. Führen Sie den folgenden Befehl aus, um einen Event Hubs-Namespace zu erstellen.

    ```
    az eventhubs namespace create --name $namespaceName --resource-group $resourceGroup -l $location
    ```

1. Führen Sie den folgenden Befehl aus, um im Event Hubs-Namespace einen Event Hub namens **myEventHub** zu erstellen. 

    ```
    az eventhubs eventhub create --name myEventHub --resource-group $resourceGroup \
      --namespace-name $namespaceName
    ```

### Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen

Damit Ihre App Nachrichten senden und empfangen kann, weisen Sie auf Ebene des Event Hubs-Namespace Ihrem Microsoft Entra-Benutzer die Rolle **Azure Event Hubs-Datenbesitzer** zu. Dadurch erhält Ihr Benutzerkonto die Berechtigung, mithilfe von Azure RBAC Warteschlangen und Themen zu verwalten und auf diese zuzugreifen. Führen Sie die folgenden Schritte in Cloud Shell aus.

1. Führen Sie den folgenden Befehl aus, um **userPrincipalName** aus Ihrem Konto abzurufen. Dadurch wird dargestellt, wem die Rolle zugewiesen wird.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Ressourcen-ID des Event Hubs-Namespace abzurufen. Die Ressourcen-ID legt den Bereich für die Rollenzuweisung auf einen bestimmten Namespace fest.

    ```
    resourceID=$(az eventhubs namespace show --resource-group $resourceGroup \
        --name $namespaceName --query id --output tsv)
    ```
1. Führen Sie den folgenden Befehl aus, um die Rolle **Azure Event Hubs-Datenbesitzer** zu erstellen und zuzuweisen, durch die Sie die Berechtigung zum Senden und Abrufen von Ereignissen erhalten.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "Azure Event Hubs Data Owner" \
        --scope $resourceID
    ```

## Senden und Abrufen von Ereignissen mit einer .NET-Konsolenanwendung

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```
    mkdir eventhubs
    cd eventhubs
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Messaging.EventHubs** und **Azure.Identity** hinzuzufügen.

    ```
    dotnet add package Azure.Messaging.EventHubs
    dotnet add package Azure.Identity
    ```

Nun ist es an der Zeit, den Vorlagencode in der Datei **Program.cs** mithilfe des Editors in der Cloud Shell zu ersetzen.

### Hinzufügen des Startercodes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```
    code Program.cs
    ```

1. Ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Überprüfen Sie unbedingt die Kommentare im Code, und ersetzen Sie **YOUR_EVENT_HUB_NAMESPACE** durch Ihren Event Hubs-Namespace.

    ```csharp
    using Azure.Messaging.EventHubs;
    using Azure.Messaging.EventHubs.Producer;
    using Azure.Messaging.EventHubs.Consumer;
    using Azure.Identity;
    using System.Text;
    
    // TO-DO: Replace YOUR_EVENT_HUB_NAMESPACE with your actual Event Hub namespace
    string namespaceURL = "YOUR_EVENT_HUB_NAMESPACE.servicebus.windows.net";
    string eventHubName = "myEventHub"; 
    
    // Create a DefaultAzureCredentialOptions object to exclude certain credentials
    DefaultAzureCredentialOptions options = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Number of events to be sent to the event hub
    int numOfEvents = 3;
    
    // CREATE A PRODUCER CLIENT AND SEND EVENTS
    
    
    
    // CREATE A CONSUMER CLIENT AND RECEIVE EVENTS
    
    
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern.

### Hinzufügen von Code zum Vervollständigen der Anwendung

In diesem Abschnitt fügen Sie Code zum Erstellen der Produzenten- und Consumerclients zum Senden und Empfangen von Ereignissen hinzu.

1. Suchen Sie den Kommentar **// CREATE A PRODUCER CLIENT AND SEND EVENTS**, und fügen Sie den folgenden Code direkt hinter dem Kommentar hinzu. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    // Create a producer client to send events to the event hub
    EventHubProducerClient producerClient = new EventHubProducerClient(
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    // Create a batch of events 
    using EventDataBatch eventBatch = await producerClient.CreateBatchAsync();
    
    
    // Adding a random number to the event body and sending the events. 
    var random = new Random();
    for (int i = 1; i <= numOfEvents; i++)
    {
        int randomNumber = random.Next(1, 101); // 1 to 100 inclusive
        string eventBody = $"Event {randomNumber}";
        if (!eventBatch.TryAdd(new EventData(Encoding.UTF8.GetBytes(eventBody))))
        {
            // if it is too large for the batch
            throw new Exception($"Event {i} is too large for the batch and cannot be sent.");
        }
    }
    
    try
    {
        // Use the producer client to send the batch of events to the event hub
        await producerClient.SendAsync(eventBatch);
    
        Console.WriteLine($"A batch of {numOfEvents} events has been published.");
        Console.WriteLine("Press Enter to retrieve and print the events...");
        Console.ReadLine();
    }
    finally
    {
        await producerClient.DisposeAsync();
    }
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern.

1. Suchen Sie den Kommentar **// CREATE A CONSUMER CLIENT AND RETRIEVE EVENTS**, und fügen Sie den folgenden Code direkt hinter dem Kommentar hinzu. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    // Create an EventHubConsumerClient
    await using var consumerClient = new EventHubConsumerClient(
        EventHubConsumerClient.DefaultConsumerGroupName,
        namespaceURL,
        eventHubName,
        new DefaultAzureCredential(options));
    
    Console.Clear();
    Console.WriteLine("Retrieving all events from the hub...");
    
    // Get total number of events in the hub by summing (last - first + 1) for all partitions
    // This count is used to determine when to stop reading events
    long totalEventCount = 0;
    string[] partitionIds = await consumerClient.GetPartitionIdsAsync();
    foreach (var partitionId in partitionIds)
    {
        PartitionProperties properties = await consumerClient.GetPartitionPropertiesAsync(partitionId);
        if (!properties.IsEmpty && properties.LastEnqueuedSequenceNumber >= properties.BeginningSequenceNumber)
        {
            totalEventCount += (properties.LastEnqueuedSequenceNumber - properties.BeginningSequenceNumber + 1);
        }
    }
    
    // Start retrieving events from the event hub and print to the console
    int retrievedCount = 0;
    await foreach (PartitionEvent partitionEvent in consumerClient.ReadEventsAsync(startReadingAtEarliestEvent: true))
    {
        if (partitionEvent.Data != null)
        {
            string body = Encoding.UTF8.GetString(partitionEvent.Data.Body.ToArray());
            Console.WriteLine($"Retrieved event: {body}");
            retrievedCount++;
            if (retrievedCount >= totalEventCount)
            {
                Console.WriteLine("Done retrieving events. Press Enter to exit...");
                Console.ReadLine();
                return;
            }
        }
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

1. Führen Sie den folgenden Befehl aus, um die Anwendung zu starten:

    ```
    dotnet run
    ```

    Nach ein paar Sekunden sollte die Ausgabe etwa folgendem Beispiel entsprechen:
    
    ```
    A batch of 3 events has been published.
    Press Enter to retrieve and print the events...
    
    Retrieving all events from the hub...
    Retrieved event: Event 4
    Retrieved event: Event 96
    Retrieved event: Event 74
    Done retrieving events. Press Enter to exit...
    ```

Die Anwendung sendet immer drei Ereignisse an den Hub, ruft jedoch alle Ereignisse im Hub ab. Wenn Sie die Anwendung mehrmals ausführen, wird eine zunehmende Anzahl an Ereignissen abgerufen. Die für die Ereigniserstellung verwendeten Zufallszahlen helfen Ihnen, unterschiedliche Ereignisse zu identifizieren.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden. 
