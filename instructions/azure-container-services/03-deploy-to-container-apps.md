---
lab:
  topic: Azure container services
  title: Bereitstellen eines Containers in Azure Container Apps mit der Azure CLI
  description: 'Hier erfahren Sie, wie Sie Azure CLI-Befehle verwenden, um eine sichere Azure Container Apps-Umgebung zu erstellen und einen Container bereitzustellen.'
---

# Bereitstellen eines Containers in Azure Container Apps mit der Azure CLI

In dieser Übung stellen Sie mithilfe der Azure CLI eine Containeranwendung in Azure Container Apps bereit. Sie lernen, wie Sie eine Container-App-Umgebung erstellen, Ihren Container bereitstellen und überprüfen, ob Ihre Anwendung in Azure ausgeführt wird.

In dieser Übung ausgeführte Aufgaben:

* Erstellen von Ressourcen in Azure
* Azure Container Apps-Umgebung erstellen
* Container-App in der Umgebung bereitstellen

Diese Übung dauert etwa **15** Minuten.

## Erstellen einer Ressourcengruppe und Vorbereiten der Azure-Umgebung

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort.

    ```azurecli
    az group create --location eastus --name myResourceGroup
    ```

1. Führen Sie den folgenden Befehl aus, um sicherzustellen, dass die neueste Version der Azure Container Apps-Erweiterung für die CLI installiert ist.

    ```azurecli
    az extension add --name containerapp --upgrade
    ```

### Registrieren von Namespaces

Es gibt zwei Namespaces, die für Azure Container Apps registriert werden müssen. Mit den folgenden Schritten stellen Sie sicher, dass diese registriert sind. Es kann einige Minuten dauern, bis jede Registrierung abgeschlossen ist, wenn sie noch nicht in Ihrem Abonnement konfiguriert ist. 

1. Registrieren Sie den Namespace **Microsoft.App**. 

    ```bash
    az provider register --namespace Microsoft.App
    ```

1. Registrieren Sie den Anbieter **Microsoft.OperationalInsights** für den Azure Monitor Log Analytics-Arbeitsbereich, wenn Sie diesen noch nicht verwendet haben.

    ```bash
    az provider register --namespace Microsoft.OperationalInsights
    ```

## Azure Container Apps-Umgebung erstellen

Eine Umgebung in Azure Container Apps erstellt eine sichere Grenze für eine Gruppe von Container-Apps. Container-Apps, die in derselben Umgebung bereitgestellt werden, werden im gleichen virtuellen Netzwerk bereitgestellt und schreiben Protokolle in denselben Log Analytics-Arbeitsbereich.

1. Erstellen Sie mit dem Befehl **az containerapp env create** eine Umgebung. Ersetzen Sie **myResourceGroup** und **myLocation** durch die zuvor von Ihnen verwendeten Werte. Es dauert einige Minuten, bis der Vorgang abgeschlossen ist.

    ```bash
    az containerapp env create \
        --name my-container-env \
        --resource-group myResourceGroup \
        --location myLocation
    ```

## Container-App in der Umgebung bereitstellen

Nachdem die Bereitstellung der Container-App-Umgebung abgeschlossen ist, können Sie in Ihrer Umgebung ein Containerimage bereitstellen.

1. Stellen Sie mit dem Befehl **containerapp create** ein Beispiel-App-Containerimage bereit. Ersetzen Sie **myResourceGroup** durch den zuvor von Ihnen verwendeten Wert.

    ```bash
    az containerapp create \
        --name my-container-app \
        --resource-group myResourceGroup \
        --environment my-container-env \
        --image mcr.microsoft.com/azuredocs/containerapps-helloworld:latest \
        --target-port 80 \
        --ingress 'external' \
        --query properties.configuration.ingress.fqdn
    ```

    Indem Sie **--ingress** auf **external** festlegen, stellen Sie die Container-App für öffentliche Anforderungen zur Verfügung. Der Befehl gibt einen Link zurück, über den Sie Ihre App aufrufen können.

    ```
    Container app created. Access your app at <url>
    ```

Wählen Sie zum Überprüfen der Bereitstellung die URL aus, die durch den Befehl **az containerapp create** zurückgegeben wird, um zu überprüfen, ob die Container-App ausgeführt wird.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
