---
lab:
  topic: Secure solutions in Azure
  title: Abrufen von Konfigurationseinstellungen aus Azure App Configuration
  description: 'Hier erfahren Sie, wie Sie eine Azure App Configuration-Ressource erstellen und mit der Azure CLI Konfigurationsinformationen festlegen. Verwenden Sie dann **ConfigurationBuilder**, um Einstellungen für Ihre Anwendung abzurufen.'
---

# Abrufen von Konfigurationseinstellungen aus Azure App Configuration

In dieser Übung erstellen Sie eine Azure App Configuration-Ressource, Sie speichern mit der Azure CLI Konfigurationseinstellungen und eine .NET-Konsolenanwendung, die **ConfigurationBuilder** zum Abrufen von Konfigurationswerten verwendet. Sie erfahren, wie Sie Einstellungen mit hierarchischen Schlüsseln organisieren und Ihre Anwendung für den Zugriff auf cloudbasierte Konfigurationsdaten authentifizieren.

In dieser Übung ausgeführte Aufgaben:

* Erstellen einer Azure App Configuration-Ressource
* Konfigurationsinformationen zur Speicherverbindungszeichenfolge
* Erstellen einer .NET-Konsolen-App zum Abrufen der Konfigurationsinformationen
* Bereinigen von Ressourcen

Diese Übung dauert etwa **15** Minuten.

## Erstellen einer Azure App Configuration-Ressource und Hinzufügen von Konfigurationsinformationen

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
    appConfigName=appconfigname$RANDOM
    ```

1. Führen Sie den folgenden Befehl aus, um den Namen der App Configuration-Ressource abzurufen. Notieren Sie sich den Namen. Sie benötigen ihn später in der Übung.

    ```
    echo $appConfigName
    ```

1. Führen Sie den folgenden Befehl aus, um sicherzustellen, dass der Anbieter **Microsoft.AppConfiguration** für Ihr Abonnement registriert ist.

    ```
    az provider register --namespace Microsoft.AppConfiguration
    ```

1. Es kann einige Minuten dauern, bis die Registrierung abgeschlossen ist. Führen Sie den folgenden Befehl aus, um den Status der Registrierung zu überprüfen. Fahren Sie mit dem nächsten Schritt fort, wenn in den Ergebnissen **Registered** zurückgegeben werden.

    ```
    az provider show --namespace Microsoft.AppConfiguration --query "registrationState"
    ```

1. Führen Sie den folgenden Befehl aus, um eine Azure App Configuration-Ressource zu erstellen. Die Ausführung des Befehls kann einige Minuten dauern.

    ```
    az appconfig create --location $location \
        --name $appConfigName \
        --resource-group $resourceGroup
        --sku Free
    ```

    >**Tipp:** Wenn beim Erstellen der AppConfig-Ressource aufgrund von Kontingenteinschränkungen mit dem SKU-Wert **Free** ein Problem auftritt, verwenden Sie stattdessen **Developer**.
    

### Zuweisen einer Rolle zu Ihrem Microsoft Entra-Benutzernamen

Um Konfigurationsinformationen abzurufen, müssen Sie Ihrem Microsoft Entra-Benutzer die Rolle **App Configuration-Datenleser** zuweisen. 

1. Führen Sie den folgenden Befehl aus, um **userPrincipalName** aus Ihrem Konto abzurufen. Dadurch wird dargestellt, wem die Rolle zugewiesen wird.

    ```
    userPrincipal=$(az rest --method GET --url https://graph.microsoft.com/v1.0/me \
        --headers 'Content-Type=application/json' \
        --query userPrincipalName --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Ressourcen-ID Ihres App Configuration-Diensts abzurufen. Die Ressourcen-ID legt den Bereich für die Rollenzuweisung fest.

    ```
    resourceID=$(az appconfig show --resource-group $resourceGroup \
        --name $appConfigName --query id --output tsv)
    ```

1. Führen Sie den folgenden Befehl aus, um die Rolle **App Configuration-Datenleser** zu erstellen und zuzuweisen.

    ```
    az role assignment create --assignee $userPrincipal \
        --role "App Configuration Data Reader" \
        --scope $resourceID
    ```

Fügen Sie App Configuration als Nächstes eine Platzhalterverbindungszeichenfolge hinzu.

### Hinzufügen von Konfigurationsinformationen mit der Azure CLI

In Azure App Configuration ist ein Schlüssel wie **Dev:conStr** ein hierarchischer Schlüssel oder ein Namespaceschlüssel. Der Doppelpunkt (:) fungiert als Trennzeichen, das eine logische Hierarchie erstellt. Dabei gilt Folgendes:

* **Dev** stellt den Namespace oder das Umgebungspräfix dar (gibt an, dass diese Konfiguration für die Entwicklungsumgebung gilt).
* **conStr** stellt den Konfigurationsnamen dar.

Mit dieser hierarchischen Struktur können Sie Konfigurationseinstellungen nach Umgebung, Feature oder Anwendungskomponente so organisieren, dass verwandte Einstellungen einfacher verwaltet und abgerufen werden können.

Führen Sie den folgenden Befehl aus, um die Platzhalterverbindungszeichenfolge zu speichern. 

```
az appconfig kv set --name $appConfigName \
    --key Dev:conStr \
    --value connectionString \
    --yes
```

Dieser Befehl gibt JSON-Code zurück. Die letzte Zeile enthält den Wert im Nur-Text-Format. 

```json
"value": "connectionString"
```

## Erstellen einer .NET-Konsolen-App zum Abrufen von Konfigurationsinformationen

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Cloud Shell ausgeführt.

>**Tipp:** Ändern Sie die Größe von Cloud Shell, um weitere Informationen und mehr Code anzuzeigen, indem Sie den oberen Rahmen ziehen. Sie können außerdem die Schaltflächen zum Minimieren und Maximieren verwenden, um zwischen Cloud Shell und der Hauptschnittstelle des Portals zu wechseln.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis zu erstellen, das das Projekt enthält, und in das Projektverzeichnis zu wechseln.

    ```
    mkdir appconfig
    cd appconfig
    ```

1. Erstellen Sie die .NET-Konsolenanwendung.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Identity** und **Microsoft.Extensions.Configuration.AzureAppConfiguration** hinzuzufügen.

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Extensions.Configuration.AzureAppConfiguration
    ```

### Hinzufügen des Codes für das Projekt

1. Führen Sie den folgenden Befehl in der Cloud-Shell aus, um mit der Bearbeitung der Anwendung zu beginnen.

    ```
    code Program.cs
    ```

1. Ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Ersetzen Sie **YOUR_APP_CONFIGURATION_NAME** unbedingt durch den Namen, den Sie zuvor notiert haben, und lesen Sie die Kommentare im Code durch.

    ```csharp
    using Microsoft.Extensions.Configuration;
    using Microsoft.Extensions.Configuration.AzureAppConfiguration;
    using Azure.Identity;
    
    // Set the Azure App Configuration endpoint, replace YOUR_APP_CONFIGURATION_NAME
    // with the name of your actual App Configuration service
    
    string endpoint = "https://YOUR_APP_CONFIGURATION_NAME.azconfig.io"; 
    
    // Configure which authentication methods to use
    // DefaultAzureCredential tries multiple auth methods automatically
    DefaultAzureCredentialOptions credentialOptions = new()
    {
        ExcludeEnvironmentCredential = true,
        ExcludeManagedIdentityCredential = true
    };
    
    // Create a configuration builder to combine multiple config sources
    var builder = new ConfigurationBuilder();
    
    // Add Azure App Configuration as a source
    // This connects to Azure and loads configuration values
    builder.AddAzureAppConfiguration(options =>
    {
        
        options.Connect(new Uri(endpoint), new DefaultAzureCredential(credentialOptions));
    });
    
    // Build the final configuration object
    try
    {
        var config = builder.Build();
        
        // Retrieve a configuration value by key name
        Console.WriteLine(config["Dev:conStr"]);
    }
    catch (Exception ex)
    {
        Console.WriteLine($"Error connecting to Azure App Configuration: {ex.Message}");
    }
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern, und dann **STRG+Q**, um den Editor zu beenden.

## Bei Azure anmelden und die App ausführen

1. Geben Sie in Cloud Shell den folgenden Befehl ein, um sich bei Azure anzumelden.

    ```
    az login
    ```

    **<font color="red">Sie müssen sich bei Azure anmelden - auch wenn die Cloud-Shell-Sitzung bereits authentifiziert ist.</font>**

    > **Hinweis**: In den meisten Szenarien ist nur die Verwendung von *az login* ausreichend. Wenn Sie jedoch Abonnements in mehreren Mandqanten haben, müssen Sie möglicherweise den Mandanten mit dem Parameter *--tenant* angeben. Weitere Informationen finden Sie unter [Interaktives Anmelden bei Azure mithilfe der Azure CLI](https://learn.microsoft.com/cli/azure/authenticate-azure-cli-interactively).

1. Führen Sie den folgenden Befehl aus, um die Konsolen-App zu starten. Die App zeigt den Wert **connectionString** an, den Sie weiter oben in der Übung der Einstellung **Dev:conStr** zugewiesen haben.

    ```
    dotnet run
    ```

    Die App zeigt den Wert **connectionString** an, den Sie weiter oben in der Übung der Einstellung **Dev:conStr** zugewiesen haben.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
