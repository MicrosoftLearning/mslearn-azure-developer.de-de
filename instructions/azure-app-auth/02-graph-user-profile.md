---
lab:
  topic: Azure authentication and authorization
  title: Abrufen von Benutzerprofilinformationen mit dem Microsoft Graph-SDK
  description: 'Hier erfahren Sie, wie Sie Benutzerprofilinformationen aus Microsoft Graph abrufen.'
---

# Abrufen von Benutzerprofilinformationen mit dem Microsoft Graph-SDK

In dieser Übung erstellen Sie eine .NET-App für die Authentifizierung mit der Microsoft Entra ID, Sie fordern ein Zugriffstoken an, und Sie rufen anschließend die Microsoft Graph-API auf, um Ihre Benutzerprofilinformationen abzurufen und anzuzeigen. Sie erfahren, wie Sie Berechtigungen konfigurieren und mit Microsoft Graph aus Ihrer Anwendung interagieren.

In dieser Übung ausgeführte Aufgaben:

* Registrieren einer Anwendung bei Microsoft Identity Platform
* Erstellen Sie eine .NET-Konsolenanwendung, die die interaktive Authentifizierung implementiert, und verwenden Sie die Klasse **GraphServiceClient**, um Benutzerprofilinformationen abzurufen.

Diese Übung dauert etwa **15** Minuten.

## Vor der Installation

Zum Abschließen der Übung benötigen Sie Folgendes:

* Ein Azure-Abonnement. Wenn Sie noch keines besitzen, können Sie sich dafür [registrieren](https://azure.microsoft.com/).

* [Visual Studio Code](https://code.visualstudio.com/) auf einer der [unterstützten Plattformen](https://code.visualstudio.com/docs/supporting/requirements#_platforms)

* [.NET 8](https://dotnet.microsoft.com/en-us/download/dotnet/8.0) oder höher

* [C#-Entwicklerkit](https://marketplace.visualstudio.com/items?itemName=ms-dotnettools.csdevkit) für Visual Studio Code

## Registrieren einer neuen Anwendung

1. Navigieren Sie in Ihrem Browser zum Azure-Portal [https://portal.azure.com](https://portal.azure.com) und melden Sie sich mit Ihren Azure-Anmeldeinformationen an, wenn Sie dazu aufgefordert werden.

1. Suchen Sie im Portal nach **App-Registrierungen**, und wählen Sie diese aus. 

1. Wählen Sie **+ Neue Registrierung** aus. Wenn die Seite **Anwendung registrieren** angezeigt wird, geben Sie die Registrierungsinformationen Ihrer Anwendung ein:

    | Feld | Wert |
    |--|--|
    | **Name** | Geben Sie `myGraphApplication` ein.  |
    | **Unterstützte Kontotypen** | Wählen Sie **Nur Konten in diesem Organisationsverzeichnis** aus. |
    | **Umleitungs-URI (optional)** | Wählen Sie **Öffentlicher Client/nativ (mobil & Desktop)** aus, und geben Sie rechts in das Feld `http://localhost` ein. |

1. Wählen Sie **Registrieren** aus. Microsoft Entra ID weist Ihrer App eine eindeutige Anwendungs-ID (Client) zu und leitet Sie zur Seite **Übersicht** Ihrer Anwendung weiter. 

1. Notieren Sie im Abschnitt **Essentials** der Seite **Übersicht** die **Anwendungs-ID (Client)** und die **Verzeichnis-ID (Mandant)**. Die Informationen werden für die Anwendung benötigt.

    ![Screenshot: Position der zu kopierenden Felder](./media/01-app-directory-id-location.png)
 
## Erstellen einer .NET-Konsolen-App zum Senden und Empfangen von Nachrichten

Nachdem die erforderlichen Ressourcen nun in Azure bereitgestellt wurden, besteht der nächste Schritt darin, die Konsolenanwendung einzurichten. Die folgenden Schritte werden in Ihrer lokalen Umgebung ausgeführt.

1. Erstellen Sie für das Projekt einen Ordner mit dem Namen **graphapp** oder einem Namen Ihrer Wahl.

1. Starten Sie **Visual Studio Code**, und wählen Sie **Datei > Ordner öffnen...** und dann den Projektordner aus.

1. Wählen Sie **Ansicht > Terminal** aus, um ein Terminal zu öffnen.

1. Führen Sie im VS Code-Terminal den folgenden Befehl aus, um die .NET-Konsolenanwendung zu erstellen.

    ```
    dotnet new console
    ```

1. Führen Sie die folgenden Befehle aus, um dem Projekt die Pakete **Azure.Identity**, **Microsoft.Graph** und die **dotenv.net** hinzuzufügen.

    ```
    dotnet add package Azure.Identity
    dotnet add package Microsoft.Graph
    dotnet add package dotenv.net
    ```

### Konfigurieren der Konsolenanwendung

In diesem Abschnitt erstellen und bearbeiten Sie eine **ENV**-Datei, in der die zuvor notierten Geheimnisse gespeichert werden. 

1. Wählen Sie **Datei > Neue Datei...** aus, und erstellen Sie im Projektordner eine Datei mit dem Namen *.env*.

1. Öffnen Sie die **ENV**-Datei, und fügen Sie den folgenden Code hinzu. Ersetzen Sie **YOUR_CLIENT_ID** und **YOUR_TENANT_ID** durch die zuvor notierten Werte.

    ```
    CLIENT_ID="YOUR_CLIENT_ID"
    TENANT_ID="YOUR_TENANT_ID"
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern.

### Hinzufügen des Startercodes für das Projekt

1. Öffnen Sie die Datei *Program.cs*, und ersetzen Sie alle vorhandenen Inhalte durch den folgenden Code. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    using Microsoft.Graph;
    using Azure.Identity;
    using dotenv.net;
    
    // Load environment variables from .env file (if present)
    DotEnv.Load();
    var envVars = DotEnv.Read();
    
    // Read Azure AD app registration values from environment
    string clientId = envVars["CLIENT_ID"];
    string tenantId = envVars["TENANT_ID"];
    
    // Validate that required environment variables are set
    if (string.IsNullOrEmpty(clientId) || string.IsNullOrEmpty(tenantId))
    {
        Console.WriteLine("Please set CLIENT_ID and TENANT_ID environment variables.");
        return;
    }
    
    // ADD CODE TO DEFINE SCOPE AND CONFIGURE AUTHENTICATION
    
    
    
    // ADD CODE TO CREATE GRAPH CLIENT AND RETRIEVE USER PROFILE
    
    
    ```

1. Drücken Sie **STRG+S**, um Ihre Änderungen zu speichern.

### Hinzufügen von Code zum Vervollständigen der Anwendung

1. Suchen Sie den Kommentar **// ADD CODE TO DEFINE SCOPE AND CONFIGURE AUTHENTICATION**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    // Define the Microsoft Graph permission scopes required by this app
    var scopes = new[] { "User.Read" };
    
    // Configure interactive browser authentication for the user
    var options = new InteractiveBrowserCredentialOptions
    {
        ClientId = clientId, // Azure AD app client ID
        TenantId = tenantId, // Azure AD tenant ID
        RedirectUri = new Uri("http://localhost") // Redirect URI for auth flow
    };
    var credential = new InteractiveBrowserCredential(options);
    ```

1. Suchen Sie den Kommentar **// ADD CODE TO CREATE GRAPH CLIENT AND RETRIEVE USER PROFILE**, und fügen Sie den folgenden Code direkt nach dem Kommentar hinzu. Überprüfen Sie unbedingt die Kommentare im Code.

    ```csharp
    // Create a Microsoft Graph client using the credential
    var graphClient = new GraphServiceClient(credential);
    
    // Retrieve and display the user's profile information
    Console.WriteLine("Retrieving user profile...");
    await GetUserProfile(graphClient);
    
    // Function to get and print the signed-in user's profile
    async Task GetUserProfile(GraphServiceClient graphClient)
    {
        try
        {
            // Call Microsoft Graph /me endpoint to get user info
            var me = await graphClient.Me.GetAsync();
            Console.WriteLine($"Display Name: {me?.DisplayName}");
            Console.WriteLine($"Principal Name: {me?.UserPrincipalName}");
            Console.WriteLine($"User Id: {me?.Id}");
        }
        catch (Exception ex)
        {
            // Print any errors encountered during the call
            Console.WriteLine($"Error retrieving profile: {ex.Message}");
        }
    }
    ```

1. Drücken Sie **STRG+S**, um die Datei zu speichern.

## Ausführen der Anwendung

Nachdem die App nun vollständig ist, können Sie sie ausführen. 

1. Führen Sie den folgenden Befehl aus, um die Anwendung zu starten:

    ```
    dotnet run
    ```

1. Durch die App wird der Standardbrowser geöffnet, und Sie werden aufgefordert, das Konto auszuwählen, mit dem Sie sich authentifizieren möchten. Wenn mehrere Konten aufgeführt sind, wählen Sie das Konto aus, das dem in der App verwendeten Mandanten zugeordnet ist.

1. Wenn Sie sich zum ersten Mal bei der registrierten App authentifizieren, erhalten Sie die Benachrichtigung **Angeforderte Berechtigungen**, in der Sie aufgefordert werden, zu genehmigen, dass die App Sie anmeldet und Ihr Profil liest, und den Zugriff auf Daten beizubehalten, für die Sie der App Zugriff gewährt haben. Wählen Sie **Annehmen** aus.

    ![Screenshot: Benachrichtigung über die angeforderten Berechtigungen](./media/01-granting-permission.png)

1. Die in der Konsole angezeigten Ergebnisse sollten ähnlich wie im folgenden Beispiel aussehen.

    ```
    Retrieving user profile...
    Display Name: <Your account display name>
    Principal Name: <Your principal name>
    User Id: 9f5...
    ```

1. Starten Sie die Anwendung ein zweites Mal, und beachten Sie, dass die Benachrichtigung **Angeforderte Benachrichtigung** nicht mehr angezeigt wird. Die zuvor von Ihnen erteilte Berechtigung wurde zwischengespeichert.

## Bereinigen von Ressourcen

Nachdem Sie die Übung abgeschlossen haben, sollten Sie die zuvor von Ihnen erstellte App-Registrierung löschen.

1. Navigieren Sie im Azure-Portal zu der App-Registrierung, die Sie erstellt haben.
1. Wählen Sie in der Symbolleiste die Option **Löschen** aus.
1. Bestätigen Sie den Löschvorgang.
