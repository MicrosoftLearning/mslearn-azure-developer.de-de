---
lab:
  topic: Azure events and messaging
  title: Weiterleiten von Ereignissen an einen benutzerdefinierten Endpunkt mit Azure Event Grid
  description: 'Hier erfahren Sie, wie Sie Azure Event Grid verwenden, um Ereignisse an einen benutzerdefinierten Endpunkt weiterzuleiten.'
---

# Weiterleiten von Ereignissen an einen benutzerdefinierten Endpunkt mit Azure Event Grid

In dieser Übung erstellen Sie ein Azure Event Grid-Thema und einen Web-App-Endpunkt, und Sie erstellen anschließend eine .NET-Konsolenanwendung, die benutzerdefinierte Ereignisse an das Event Grid-Thema sendet. Sie erfahren, wie Sie Ereignisabonnements konfigurieren, sich mit Event Grid authentifizieren und überprüfen, ob Ihre Ereignisse erfolgreich an den Endpunkt weitergeleitet werden, indem Sie diese in der Web-App anzeigen.

In dieser Übung ausgeführte Aufgaben:

* Erstellen von Azure Event Grid-Ressourcen
* Aktivieren eines Event Grid-Ressourcenanbieters
* Erstellen eines Themas in Event Grid
* Erstellen eines Nachrichtenendpunkts
* Abonnieren des Themas
* Senden eines Ereignisses mit einer .NET-Konsolen-App
* Bereinigen von Ressourcen

Diese Übung dauert ca. **30** Minuten.

## Erstellen von Azure Event Grid-Ressourcen

In diesem Abschnitt der Übung erstellen Sie mit der Azure CLI die erforderlichen Ressourcen in Azure.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Wählen Sie in der Cloud Shell-Symbolleiste im Menü **Einstellungen** das Menüelement **Zur klassischen Version wechseln** aus (dies ist für die Verwendung des Code-Editors erforderlich).

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen.

    ```bash
    az group create --name myResourceGroup --location eastus
    ```

1. Viele der Befehle erfordern eindeutige Namen und verwenden dieselben Parameter. Durch das Erstellen einiger Variablen werden die Änderungen reduziert, die an den Befehlen zum Erstellen von Ressourcen erforderlich sind. Führen Sie die folgenden Befehle aus, um die erforderlichen Variablen zu erstellen. Ersetzen Sie **myResourceGroup** durch den Namen, den Sie für diese Übung verwenden. Wenn Sie den Speicherort im vorherigen Schritt geändert haben, nehmen Sie dieselbe Änderung in der Variablen **location** vor.

    ```bash
    let rNum=$RANDOM
    resourceGroup=myResourceGroup
    location=eastus
    topicName="mytopic-evgtopic-${rNum}"
    siteName="evgsite-${rNum}"
    siteURL="https://${siteName}.azurewebsites.net"
    ```

### Aktivieren eines Event Grid-Ressourcenanbieters

Ein Azure-Ressourcenanbieter ist ein Dienst, der bestimmte Ressourcentypen in Azure definiert und verwaltet. Dieser wird von Azure im Hintergrund verwendet, wenn Sie Ressourcen bereitstellen oder verwalten. Registrieren Sie den Event Grid-Ressourcenanbieter mit dem Befehl **az provider register**. 

```bash
az provider register --namespace Microsoft.EventGrid
```

Es kann einige Minuten dauern, bis die Registrierung abgeschlossen ist. Sie können den Status mit dem folgenden Befehl überprüfen.

```bash
az provider show --namespace Microsoft.EventGrid --query "registrationState"
```

> **Hinweis:** Dieser Schritt ist nur für Abonnements erforderlich, die Event Grid zuvor noch nicht verwendet haben.

### Erstellen eines Themas in Event Grid

Erstellen Sie mithilfe des Befehls **az eventgrid topic create** ein Thema. Der Name muss eindeutig sein, da er Teil des DNS-Eintrags ist.  

```bash
az eventgrid topic create --name $topicName \
    --location $location \
    --resource-group $resourceGroup
```

### Erstellen eines Nachrichtenendpunkts

Vor dem Abonnieren des benutzerdefinierten Themas müssen Sie den Endpunkt für die Ereignisnachricht erstellen. Der Endpunkt führt in der Regel Aktionen auf der Grundlage der Ereignisdaten aus. Das folgende Skript verwendet eine vordefinierte Web-App, die die Ereignismeldungen anzeigt. Die bereitgestellte Lösung umfasst einen App Service-Plan, eine App Service-Web-App und Quellcode von GitHub.

1. Führen Sie die folgenden Befehle aus, um einen Nachrichtenendpunkt zu erstellen. Der **echo**-Befehl zeigt die Website-URL für den Endpunkt an.

    ```bash
    az deployment group create \
        --resource-group $resourceGroup \
        --template-uri "https://raw.githubusercontent.com/Azure-Samples/azure-event-grid-viewer/main/azuredeploy.json" \
        --parameters siteName=$siteName hostingPlanName=viewerhost
    
    echo "Your web app URL: ${siteURL}"
    ```

    > **Hinweis:** Die Ausführung dieses Befehls kann einige Minuten in Anspruch nehmen.

1. Öffnen Sie in Ihrem Browser eine neue Registerkarte, und navigieren Sie zu der URL, die am Ende des vorherigen Skripts generiert wurde, um sicherzustellen, dass die Web-App ausgeführt wird. Die Website sollte angezeigt werden, und es sollten momentan keine Nachrichten vorliegen.

    > **Tipp:** Setzen Sie die Ausführung des Browsers fort. Er wird zum Anzeigen von Aktualisierungen verwendet.

### Abonnieren des Themas

Sie abonnieren ein Event Grid-Thema, um Event Grid mitzuteilen, welche Ereignisse Sie nachverfolgen möchten und wohin diese gesendet werden sollen. 

1. Abonnieren Sie mithilfe des Befehls **az eventgrid event-subscription create** ein Thema. Das folgende Skript ruft die Abonnement-ID aus Ihrem Konto ab und verwendet diese bei der Erstellung des Ereignisabonnements.

    ```bash
    endpoint="${siteURL}/api/updates"
    topicId=$(az eventgrid topic show --resource-group $resourceGroup \
        --name $topicName --query "id" --output tsv)
    
    az eventgrid event-subscription create \
        --source-resource-id $topicId \
        --name TopicSubscription \
        --endpoint $endpoint
    ```

1. Zeigen Sie wieder Ihre Web-App an. Wie Sie sehen, wurde ein Abonnementüberprüfungsereignis an sie gesendet. Klicken Sie auf das Augensymbol, um die Ereignisdaten zu erweitern. Event Grid sendet das Überprüfungsereignis, damit der Endpunkt bestätigen kann, dass er Ereignisdaten empfangen möchte. Die Web-App enthält Code zur Überprüfung des Abonnements.

## Senden eines Ereignisses mit einer .NET-Konsolenanwendung

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```bash
    mkdir eventgrid
    cd eventgrid
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```bash
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Messaging.EventGrid** und **dotenv.net** hinzuzufügen.

    ```bash
    dotnet add package Azure.Messaging.EventGrid
    dotnet add package dotenv.net
    ```

### Konfigurieren der Konsolenanwendung

In diesem Abschnitt rufen Sie den Endpunkt für das Thema und den Zugriffsschlüssel ab, damit diese einer **ENV**-Datei hinzugefügt werden können, um diese Geheimnisse zu speichern.

1. Führen Sie die folgenden Befehle aus, um die URL und den Zugriffsschlüssel für das zuvor von Ihnen erstellte Thema abzurufen. Notieren Sie diese Werte unbedingt.

    ```bash
    az eventgrid topic show --name $topicName -g $resourceGroup --query "endpoint" --output tsv
    az eventgrid topic key list --name $topicName -g $resourceGroup --query "key1" --output tsv
    ```

1. Führen Sie den folgenden Befehl aus, um die **ENV**-Datei zum Aufbewahren der Geheimnisse zu erstellen, und öffnen Sie diese dann im Code-Editor.

    ```bash
    touch .env
    code .env
    ```

1. Fügen Sie der **ENV**-Datei den folgenden Code hinzu. Ersetzen Sie **YOUR_TOPIC_ENDPOINT** und **YOUR_TOPIC_ACCESS_KEY** durch die zuvor von Ihnen aufgezeichneten Werte.

    ```
    TOPIC_ENDPOINT="YOUR_TOPIC_ENDPOINT"
    TOPIC_ACCESS_KEY="YOUR_TOPIC_ACCESS_KEY"
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und dann **STRG+Q**, um den Editor zu beenden.

Nun ist es an der Zeit, den Vorlagencode in der Datei **Program.cs** mithilfe des Editors in der Cloud Shell zu ersetzen.

### Hinzufügen des Codes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```bash
    code Program.cs
    ```

1. Ersetzen Sie vorhandenen Code durch den folgenden Code. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    using dotenv.net; 
    using Azure.Messaging.EventGrid; 
    
    // Load environment variables from .env file
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Start the asynchronous process to send an Event Grid event
    ProcessAsync().GetAwaiter().GetResult();
    
    async Task ProcessAsync()
    {
        // Retrieve Event Grid topic endpoint and access key from environment variables
        var topicEndpoint = envVars["TOPIC_ENDPOINT"];
        var topicKey = envVars["TOPIC_ACCESS_KEY"];
        
        // Check if the required environment variables are set
        if (string.IsNullOrEmpty(topicEndpoint) || string.IsNullOrEmpty(topicKey))
        {
            Console.WriteLine("Please set TOPIC_ENDPOINT and TOPIC_ACCESS_KEY in your .env file.");
            return;
        }
    
        // Create an EventGridPublisherClient to send events to the specified topic
        EventGridPublisherClient client = new EventGridPublisherClient
            (new Uri(topicEndpoint),
            new Azure.AzureKeyCredential(topicKey));
    
        // Create a new EventGridEvent with sample data
        var eventGridEvent = new EventGridEvent(
            subject: "ExampleSubject",
            eventType: "ExampleEventType",
            dataVersion: "1.0",
            data: new { Message = "Hello, Event Grid!" }
        );
    
        // Send the event to Azure Event Grid
        await client.SendEventAsync(eventGridEvent);
        Console.WriteLine("Event sent successfully.");
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

1. Führen Sie in Cloud Shell den folgenden Befehl aus, um die Konsolenanwendung zu starten. Die Meldung **Ereignis erfolgreich gesendet** wird angezeigt , wenn die Nachricht gesendet wird.

    ```bash
    dotnet run
    ```

1. Zeigen Sie Ihre Web-App an, um das soeben gesendete Ereignis anzuzeigen. Klicken Sie auf das Augensymbol, um die Ereignisdaten zu erweitern.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.