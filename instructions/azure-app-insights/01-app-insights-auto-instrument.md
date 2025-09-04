---
lab:
  topic: Application Insights
  title: Überwachen einer Anwendung mit automatischer Instrumentierung
  description: 'Hier erfahren Sie, wie Sie eine Anwendung in Application Insights ohne Codeänderungen überwachen, indem Sie die automatische Instrumentierung konfigurieren. '
---

# Überwachen einer Anwendung mit automatischer Instrumentierung

In dieser Übung erstellen Sie eine Azure App Service-Web-App mit aktivierter Application Insights-Funktion, Sie konfigurieren die automatische Instrumentierung ohne Codeänderungen, erstellen eine Blazor-Anwendung und stellen diese bereit, und Sie zeigen anschließend Anwendungsmetriken und Fehlerdaten in Application Insights an. Durch die Implementierung einer umfassenden Anwendungsüberwachung sowie des Einblicks in die Anwendung, ohne dass Änderungen an Ihrem Code vorgenommen werden müssen, werden Bereitstellungen und Migrationen vereinfacht.

In dieser Übung ausgeführte Aufgaben:

* Erstellen einer Web-App-Ressource mit aktivierter Application Insights-Funktion
* Konfigurieren Sie die Instrumentierung für die Web-App.
* Erstellen Sie eine neue Blazor-App, und stellen Sie diese in der Web-App-Ressource bereit.
* Anzeigen der Anwendungsaktivität in Application Insights
* Bereinigen von Ressourcen

Diese Übung dauert etwa **20** Minuten.

## Erstellen von Ressourcen in Azure

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.
1. Wählen Sie die Option **+ Ressource erstellen** in der **Azure Services**-Überschrift oben auf der Homepage. 
1. Geben Sie in der **Im Marketplace suchen**-Suchleiste *Web-App* ein und drücken Sie die **Eingabetaste**, um die Suche zu starten.
1. Wählen Sie in der Kachel Web-App das Dropdownmenü **Erstellen** und dann die Option **Web-App** aus.

    ![Screenshot der Web-App-Kachel.](./media/create-web-app-tile.png)

Wenn Sie **Erstellen** auswählen, wird eine Vorlage mit einigen Registerkarten geöffnet, die Sie mit Informationen zu Ihrer Bereitstellung ausfüllen müssen. Die folgenden Schritte führen Sie durch die Änderungen, die auf den relevanten Registerkarten vorgenommen werden sollen.

1. Füllen Sie die Registerkarte **Grundlagen** mit den Informationen aus der folgenden Tabelle aus:

    | Einstellung | Aktion |
    |--|--|
    | **Abonnement** | Übernehmen Sie den Standardwert. |
    | **Ressourcengruppe** | Wählen Sie „Neu erstellen“, geben Sie `rg-WebApp` ein und wählen Sie dann „OK“. Bei Bedarf können Sie ebenfalls eine vorhandene Ressourcengruppe auswählen. |
    | **Name** | Geben Sie einen eindeutigen Namen wie **IHRE-INITIALEN-monitorapp** ein. Ersetzen Sie **IHRE-INITIALEN** durch Ihre Initialen oder einen anderen Wert. Der Name muss eindeutig sein, daher sind möglicherweise einige Änderungen erforderlich. |
    | Schieberegler unter der Einstellung **Name** | Wählen Sie den Schieberegler aus, um ihn zu deaktivieren. Dieser Schieberegler wird nur in manchen Azure-Konfigurationen angezeigt. |
    | **Veröffentlichen** | Wählen Sie die Option **Code** aus. |
    | **Runtimestapel** | Wählen Sie im Dropdownmenü **.NET 8 (LTS)** aus. |
    | **Betriebssystem** | Wählen Sie **Windows** aus. |
    | **Region** | Behalten Sie die Standardauswahl bei, oder wählen Sie einen Bereich in Ihrer Nähe aus. |
    | **Windows-Plan** | Behalten Sie die Standardauswahl bei. |
    | **Tarif** | Wählen Sie die Dropdownliste und dann den kostenlosen **F1-Plan** aus. |

1. Wählen Sie die Registerkarte **Überwachen und sichern** aus, oder navigieren Sie zu dieser Registerkarte, und geben Sie die Informationen in der folgenden Tabelle ein:

    | Einstellung | Aktion |
    |--|--|
    | **Application Insights aktivieren** | Wählen Sie **Ja** aus. |
    | **Application Insights** | Wählen Sie **Neu erstellen** aus. Daraufhin wird ein Dialogfeld angezeigt. Geben Sie in das Feld **Name** des Dialogfelds `autoinstrument-insights` ein. Wählen Sie dann **OK** aus, um den Namen anzunehmen. |
    | **Arbeitsbereich** | Geben Sie `Workspace` ein, wenn das Feld noch nicht ausgefüllt und gesperrt ist. |

1. Wählen Sie **Überprüfen und erstellen** aus, und überprüfen Sie die Details Ihrer Bereitstellung. Wählen Sie dann **Erstellen** aus, um die Ressourcen zu erstellen.

Es dauert einige Minuten, bis die Bereitstellung abgeschlossen ist. Wählen Sie nach Abschluss des Vorgangs die Schaltfläche **Zu Ressource wechseln** aus.

### Konfigurieren von Instrumentierungseinstellungen

Um die Überwachung ohne Änderungen an Ihrem Code zu aktivieren, müssen Sie die Instrumentierung für Ihre App auf Dienstebene konfigurieren.

1. Erweitern Sie im linken Navigationsmenü die Option **Überwachung**, und wählen Sie **Application Insights** aus.

1. Suchen Sie den Abschnitt **Anwendung instrumentieren**, und wählen Sie **.NET Core** aus.

1. Wählen Sie im Abschnitt **Sammlungsebene** die Option **Empfohlen** aus.

1. Wählen Sie **Anwenden** aus, und bestätigen Sie anschließend die Änderungen.

1. Wählen Sie im linken Navigationsmenü die Option **Übersicht** aus.

## Erstellen und Bereitstellen einer Blazor-App

In diesem Abschnitt der Übung erstellen Sie eine Blazor-App in Cloud Shell und stellen diese in der Web-App bereit, die Sie erstellt haben. Alle Schritte in diesem Abschnitt werden in Cloud Shell ausgeführt.

1. Verwenden Sie die Schaltfläche **[\>_]** rechts neben der Suchleiste oben auf der Seite, um eine neue Cloud Shell im Azure-Portal zu erstellen, und wählen Sie eine ***Bash***-Umgebung aus. Die Cloud Shell bietet eine Befehlszeilenschnittstelle in einem Bereich am unteren Rand des Azure-Portals. Wenn Sie aufgefordert werden, ein Speicherkonto auszuwählen, damit Ihre Dateien beibehalten werden, wählen Sie **Kein Speicherkonto erforderlich**, Ihr Abonnement und dann **Anwenden** aus.

    > **Hinweis**: Wenn Sie zuvor eine Cloud Shell erstellt haben, die eine *PowerShell*-Umgebung verwendet, wechseln Sie diese zu ***Bash***.

1. Führen Sie die folgenden Befehle aus, um ein Verzeichnis für die Blazor-App zu erstellen und in das Verzeichnis zu wechseln.

    ```
    mkdir blazor
    cd blazor
    ```

1. Führen Sie den folgenden Befehl aus, um im Ordner eine neue Blazor-App zu erstellen.

    ```
    dotnet new blazor
    ```

1. Führen Sie den folgenden Befehl aus, um die Anwendung zu erstellen und sicherzustellen, dass während der Erstellung keine Probleme aufgetreten sind.

    ```
    dotnet build
    ```

### Bereitstellen der App in App Service

Um die App bereitzustellen, müssen Sie diese zunächst mit dem Befehl **dotnet publish** veröffentlichen und dann eine *ZIP*-Datei für die Bereitstellung erstellen.

1. Führen Sie den folgenden Befehl aus, um die App im Verzeichnis *publish* zu veröffentlichen.

    ```
    dotnet publish -c Release -o ./publish
    ```

1. Führen Sie die folgenden Befehle aus, um eine *ZIP*-Datei der veröffentlichten App zu erstellen. Die *ZIP*-Datei befindet sich im Stammverzeichnis der Anwendung.

    ```
    cd publish
    zip -r ../app.zip .
    cd ..
    ```

1. Führen Sie den folgenden Befehl aus, um die App für App Service bereitzustellen. Ersetzen Sie **YOUR-WEB-APP-NAME** UND **YOUR-RESOURCE-GROUP** durch die Werte, die Sie zuvor in der Übung beim Erstellen der App Service-Ressourcen verwendet haben.

    ```
    az webapp deploy --name YOUR-WEB-APP-NAME \
        --resource-group YOUR-RESOURCE-GROUP \
        --src-path ./app.zip
    ```

1. Wenn die Bereitstellung abgeschlossen ist, wählen Sie den Link im Feld **Standarddomäne** im Abschnitt **Essentials** aus, um die App in Ihrem Browser auf einer neuen Registerkarte zu öffnen.

Jetzt können Sie in Application Insights einige grundlegende Anwendungsmetriken anzeigen. Schließen Sie diese Registerkarte nicht, denn Sie verwenden sie in der restlichen Übung.

## Anzeigen von Metriken in Application Insights

Kehren Sie zur Registerkarte mit dem Azure-Portal zurück, und navigieren Sie zu der Application Insights-Ressource, die Sie zuvor erstellt haben. Auf der Registerkarte **Übersicht** werden einige einfache Diagramme angezeigt:

* Anforderungsfehler
* Serverantwortzeit
* Serveranforderungen
* Verfügbarkeit

In diesem Abschnitt führen Sie einige Aktionen in der Web-App aus und kehren dann zu dieser Seite zurück, um die Aktivität anzuzeigen. Die Aktivitätsberichterstattung ist verzögert, wodurch es einige Minuten dauern kann, bis sie in den Diagrammen angezeigt wird.

Führen Sie die folgenden Schritte in der Web-App aus.

1. Navigieren Sie im Menü der Web-App zwischen den Navigationsoptionen **Home**, **+ Zähler** und **Wetter**.

1. Aktualisieren Sie die Webseite mehrmals, um Daten zur **Serverantwortzeit** und zu **Serveranforderungen** zu generieren.

1. Um einige Fehler zu erstellen, wählen Sie die Schaltfläche **Home** aus, und fügen Sie an die URL **/failures** an. Diese Route ist in der Web-App nicht vorhanden und generiert einen Fehler. Aktualisieren Sie die Seite mehrmals, um Fehlerdaten zu generieren.

1. Kehren Sie zur Registerkarte zurück, auf der Application Insights ausgeführt wird, und warten Sie ein oder zwei Minuten, bis die Informationen in den Diagrammen angezeigt werden. 

1. Erweitern Sie im linken Navigationsbereich den Abschnitt **Untersuchen**, und wählen Sie **Fehler** aus. Es zeigt die Anzahl der Anforderungen mit einem Fehler zusammen mit ausführlicheren Informationen zu den Antwortcodes für die Fehler an.

Untersuchen Sie weitere Berichtsoptionen, um eine Vorstellung davon zu erhalten, welche weiteren Arten von Informationen verfügbar sind. 

## Bereinigen von Ressourcen

Nachdem Sie die Übung beendet haben, sollten Sie die von Ihnen erstellten Cloud-Ressourcen löschen, um eine unnötige Ressourcennutzung zu vermeiden.

1. Navigieren Sie zu der Ressourcengruppe, die Sie erstellt haben, und zeigen Sie den Inhalt der in dieser Übung verwendeten Ressourcen an.
1. Wählen Sie auf der Symbolleiste die Option **Ressourcengruppe löschen** aus.
1. Geben Sie den Namen der Ressourcengruppe ein, und bestätigen Sie, dass Sie sie löschen möchten.

> **VORSICHT:** Beim Löschen einer Ressourcengruppe werden alle darin enthaltenen Ressourcen gelöscht. Wenn Sie eine vorhandene Ressourcengruppe für diese Übung ausgewählt haben, werden alle vorhandenen Ressourcen ebenfalls gelöscht, die nicht in dieser Übung verwendet werden.
