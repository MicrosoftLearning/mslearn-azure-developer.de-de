---
lab:
  topic: Azure container services
  title: Erstellen und Ausführen eines Containerimages mit Azure Container Registry Tasks
  description: 'Hier erfahren Sie, wie Sie Azure CLI-Befehle verwenden, um Containerimages mit Azure Container Registry Tasks zu erstellen und auszuführen.'
---

# Erstellen und Ausführen eines Containerimages mit Azure Container Registry Tasks

In dieser Übung erstellen Sie ein Containerimage aus Ihrem Anwendungscode und pushen es mithilfe der Azure CLI an Azure Container Registry. Sie lernen, wie Sie Ihre App für die Containerisierung vorbereiten, eine ACR-Instanz erstellen und Ihr Containerimage in Azure speichern.

In dieser Übung ausgeführte Aufgaben:

* Erstellen einer Azure Container Registry-Ressource
* Erstellen und Pushen eines Images aus einer Dockerfile-Datei
* Überprüfen der Ergebnisse
* Ausführen des Images in der Azure Container Registry

Diese Übung dauert etwa **20** Minuten.

## Erstellen einer Azure Container Registry-Ressource

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort.

    ```
    az group create --location eastus --name myResourceGroup
    ```

1. Führen Sie den folgenden Befehl aus, um eine grundlegende Containerregistrierung zu erstellen. Der Registrierungsname muss innerhalb von Azure eindeutig sein und zwischen 5 und 50 alphanumerische Zeichen enthalten. Ersetzen Sie **myResourceGroup** durch den zuvor von Ihnen verwendeten Namen, und **myContainerRegistry** durch einen eindeutigen Wert.

    ```bash
    az acr create --resource-group myResourceGroup \
        --name myContainerRegistry --sku Basic
    ```

    > **Hinweis:** Mit dem Befehl wird eine *Basic*-Registrierung erstellt, eine kostenoptimierte Option für Entwickler, die sich in Azure Container Registry einarbeiten.

## Erstellen und Pushen eines Images aus einer Dockerfile-Datei

Als Nächstes erstellen und pushen Sie ein Image, das auf einer Dockerfile-Datei basiert.

1. Führen Sie den folgenden Befehl aus, um die Dockerfile-Datei zu erstellen. Die Dockerfile enthält eine einzelne Zeile, die auf das in Microsoft Container Registry gehostete Image *hello-world* verweist.

    ```bash
    echo FROM mcr.microsoft.com/hello-world > Dockerfile
    ```

1. Führen Sie den folgenden Befehl **az acr build** aus, der das Image erstellt und es nach seiner erfolgreichen Erstellung in Ihre Registrierung pusht. Ersetzen Sie **myContainerRegistry** durch den Namen, den Sie zuvor erstellt haben.

    ```bash
    az acr build --image sample/hello-world:v1  \
        --registry myContainerRegistry \
        --file Dockerfile .
    ```

    Im Folgenden sehen Sie ein verkürztes Beispiel der Ausgabe des vorherigen Befehls, das die letzten Zeilen mit den endgültigen Ergebnissen zeigt. Im Feld *Repository* wird das Image *sample/hello-word* aufgeführt.

    ```
    - image:
        registry: myContainerRegistry.azurecr.io
        repository: sample/hello-world
        tag: v1
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      runtime-dependency:
        registry: mcr.microsoft.com
        repository: hello-world
        tag: latest
        digest: sha256:92c7f9c92844bbbb5d0a101b22f7c2a7949e40f8ea90c8b3bc396879d95e899a
      git: {}
    
    
    Run ID: cf1 was successful after 11s
    ```

## Überprüfen der Ergebnisse

1. Führen Sie den folgenden Befehl aus, um die Repositorys in Ihrer Registrierung auflisten. Ersetzen Sie **myContainerRegistry** durch den Namen, den Sie zuvor erstellt haben.

    ```bash
    az acr repository list --name myContainerRegistry --output table
    ```

    Ausgabe:

    ```
    Result
    ----------------
    sample/hello-world
    ```

1. Führen Sie den folgenden Befehl aus, um die Tags im Repository **sample/hello-world** aufzulisten. Ersetzen Sie **myContainerRegistry** durch den Namen, den Sie zuvor verwendet haben.

    ```bash
    az acr repository show-tags --name myContainerRegistry \
        --repository sample/hello-world --output table
    ```

    Ausgabe:

    ```
    Result
    --------
    v1
    ```

## Ausführen des Images in ACR

1. Führen Sie mit dem Befehl **az acr run** das Containerimage *sample/hello-world:v1* aus Ihrer Containerregistrierung aus. Im folgenden Beispiel wird **$Registry** verwendet, um die Registrierung anzugeben, in der Sie den Befehl ausführen. Ersetzen Sie **myContainerRegistry** durch den Namen, den Sie zuvor verwendet haben.

    ```bash
    az acr run --registry myContainerRegistry \
        --cmd '$Registry/sample/hello-world:v1' /dev/null
    ```

    Der Parameter **cmd** in diesem Beispiel führt den Container in seiner Standardkonfiguration aus. **cmd** unterstützt jedoch weitere Parameter für **docker run** oder selbst weitere **Docker**-Befehlen. 

    Die folgende Beispielausgabe ist gekürzt:

    ```
    Packing source code into tar to upload...
    Uploading archived source code from '/tmp/run_archive_ebf74da7fcb04683867b129e2ccad5e1.tar.gz'...
    Sending context (1.855 KiB) to registry: mycontainerre...
    Queued a run with ID: cab
    Waiting for an agent...
    2019/03/19 19:01:53 Using acb_vol_60e9a538-b466-475f-9565-80c5b93eaa15 as the home volume
    2019/03/19 19:01:53 Creating Docker network: acb_default_network, driver: 'bridge'
    2019/03/19 19:01:53 Successfully set up Docker network: acb_default_network
    2019/03/19 19:01:53 Setting up Docker configuration...
    2019/03/19 19:01:54 Successfully set up Docker configuration
    2019/03/19 19:01:54 Logging in to registry: mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Successfully logged into mycontainerregistry008.azurecr.io
    2019/03/19 19:01:55 Executing step ID: acb_step_0. Working directory: '', Network: 'acb_default_network'
    2019/03/19 19:01:55 Launching container with name: acb_step_0
    
    Hello from Docker!
    This message shows that your installation appears to be working correctly.
    
    2019/03/19 19:01:56 Successfully executed container: acb_step_0
    2019/03/19 19:01:56 Step ID: acb_step_0 marked as successful (elapsed time in seconds: 0.843801)
    
    Run ID: cab was successful after 6s
    ```

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
