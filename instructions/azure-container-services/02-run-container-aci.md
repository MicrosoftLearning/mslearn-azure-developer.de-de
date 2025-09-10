---
lab:
  topic: Azure container services
  title: Bereitstellen eines Containers für Azure Container Instances mithilfe von Azure CLI-Befehlen
  description: 'Hier erfahren Sie, wie Sie Azure CLI Befehle verwenden, um einen Container für Azure Container Instances bereitzustellen.'
---

# Bereitstellen eines Containers für Azure Container Instances mithilfe von Azure CLI-Befehlen

In dieser Übung stellen Sie mithilfe der Azure CLI einen Container in Azure Container Instances (ACI) bereit und führen diesen aus. Sie lernen, wie Sie eine Containergruppe erstellen, Containereinstellungen angeben und überprüfen, ob Ihre Containeranwendung in der Cloud ausgeführt wird.

In dieser Übung ausgeführte Aufgaben:

* Erstellen von Azure Container Instance-Ressourcen in Azure
* Erstellen und Bereitstellen eines Containers
* Überprüfen, ob der Container ausgeführt wird

Diese Übung dauert etwa **15** Minuten.

## Erstellen einer Ressourcengruppe

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Erstellen Sie eine Ressourcengruppe für die Ressourcen, die für diese Übung benötigt werden. Ersetzen Sie **myResourceGroup** durch einen Namen, den Sie für die Ressourcengruppe verwenden möchten. Sie können **eastus** bei Bedarf durch eine Region in Ihrer Nähe ersetzen. Wenn Sie bereits über eine Ressourcengruppe verfügen, die Sie verwenden möchten, fahren Sie mit dem nächsten Schritt fort.

    ```
    az group create --location eastus --name myResourceGroup
    ```

## Erstellen und Bereitstellen eines Containers

Um einen Container zu erstellen, müssen Sie einen Namen, ein Docker-Image und eine Azure-Ressourcengruppe für den Befehl **az container create** angeben. Durch Angeben einer DNS-Namensbezeichnung machen Sie den Container über das Internet verfügbar.

1. Führen Sie den folgenden Befehl aus, um einen DNS-Namen zu erstellen, der verwendet wird, um Ihren Container für das Internet verfügbar zu machen. Ihr DNS-Name muss eindeutig sein. Führen Sie diesen Befehl über Cloud Shell aus, um so eine Variable zu erstellen, die einen eindeutigen Namen enthält.

    ```bash
    DNS_NAME_LABEL=aci-example-$RANDOM
    ```

1. Führen Sie den folgenden Befehl aus, um eine Containerinstanz zu erstellen. Ersetzen Sie **myResourceGroup** und **myLocation** durch die zuvor von Ihnen verwendeten Werte. Es dauert einige Minuten, bis der Vorgang abgeschlossen ist.

    ```bash
    az container create --resource-group myResourceGroup \
        --name mycontainer \
        --image mcr.microsoft.com/azuredocs/aci-helloworld \
        --ports 80 \
        --dns-name-label $DNS_NAME_LABEL --location myLocation \
        --os-type Linux \
        --cpu 1 \
        --memory 1.5 
    ```

    Im vorherigen Befehl gibt **$DNS_NAME_LABEL** Ihren DNS-Namen an. Der Imagename **mcr.microsoft.com/azuredocs/aci-helloworld**bezieht sich auf ein Docker-Image, das eine einfache Node.js-Webanwendung ausführt.

Wechseln Sie zum nächsten Abschnitt, nachdem der Befehl **az container create** abgeschlossen ist.

## Überprüfen, ob der Container ausgeführt wird

Sie können den Containerbuildstatus mit dem Befehl **az container show** überprüfen. 

1. Führen Sie den folgenden Befehl aus, um den Bereitstellungsstatus des von Ihnen erstellten Containers zu überprüfen. Ersetzen Sie **myResourceGroup** durch den zuvor von Ihnen verwendeten Wert.

    ```bash
    az container show --resource-group myResourceGroup \
        --name mycontainer \
        --query "{FQDN:ipAddress.fqdn,ProvisioningState:provisioningState}" \
        --out table 
    ```

    Der vollqualifizierte Domänenname (FQDN) und der Bereitstellungsstatus des Containers werden angezeigt. Im Folgenden sehen Sie ein Beispiel.

    ```
    FQDN                                    ProvisioningState
    --------------------------------------  -------------------
    aci-wt.eastus.azurecontainer.io         Succeeded
    ```

    > **Hinweis:** Wenn Ihr Container den Status **Wird erstellt** aufweist, sollten Sie kurz warten und den Befehl erneut ausführen, bis der Status **Erfolgreich** angezeigt wird.

1. Navigieren Sie in einem Browser zum FQDN des Containers, um ihn im ausgeführten Zustand zu sehen. Möglicherweise erhalten Sie eine Warnung, dass die Website nicht sicher ist.

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
