# FL3App — Procédure de génération et publication d'une mise à jour

> **À adapter à chaque release :** le numéro de version utilisé dans les exemples ci-dessous.
> Le token GitHub ne doit jamais être écrit dans le code source ni commité dans Git.

Ce document décrit la procédure complète pour générer et publier une mise à jour de **FL3App** avec **Velopack 1.2.0** et **GitHub Releases**.

---

## 1. Architecture utilisée

- Application : **WPF .NET 9**
- Package NuGet : **Velopack 1.2.0**
- CLI : **vpk 1.2.0**
- Repository de mises à jour :
  `https://github.com/Groupe-SERAP/FL3App_Update`
- Repository : **public**
- Pack ID Velopack :
  `SERAP.FL3App`
- Runtime cible :
  `win-x64`
- Publication :
  **self-contained**

Le repository GitHub étant public, **aucun token n'est nécessaire dans l'application pour rechercher ou télécharger les mises à jour**.

Le token GitHub est uniquement nécessaire pour **publier une nouvelle release** avec `vpk upload github`.

---

# 2. Configuration initiale — à faire une seule fois

## 2.1 Installer Velopack dans le projet

Depuis le dossier contenant `FL3App.csproj` :

```powershell
dotnet add package Velopack --version 1.2.0
```

Vérification :

```powershell
dotnet list package | findstr Velopack
```

Résultat attendu :

```text
Velopack    1.2.0    1.2.0
```

---

## 2.2 Installer le CLI Velopack

```powershell
dotnet tool install -g vpk --version 1.2.0
```

Pour vérifier que le CLI fonctionne :

```powershell
vpk -h
```

L'en-tête doit indiquer Velopack CLI `1.2.0`.

---

# 3. Configuration du projet WPF

## 3.1 Version dans le `.csproj`

Le projet doit avoir une seule version principale :

```xml
<PropertyGroup>
    <OutputType>WinExe</OutputType>
    <TargetFramework>net9.0-windows</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <UseWPF>true</UseWPF>
    <ApplicationIcon>Resources\Icon\serap_logo_vertical.ico</ApplicationIcon>

    <Deterministic>false</Deterministic>

    <Version>0.11.0</Version>

    <StartupObject>FL3App.App</StartupObject>
</PropertyGroup>
```

Pour chaque nouvelle release, modifier :

```xml
<Version>0.11.0</Version>
```

Exemples :

```text
0.11.0
0.11.1
0.11.2
0.12.0
1.0.0
```

---

## 3.2 Entrée principale Velopack

Velopack doit être exécuté au tout début du `Main()` :

```csharp
[STAThread]
private static void Main(string[] args)
{
    VelopackApp.Build().Run();

    App app = new();
    app.InitializeComponent();
    app.Run();
}
```

---

## 3.3 Point critique dans `OnStartup`

Le contrôle suivant est obligatoire :

```csharp
protected override async void OnStartup(StartupEventArgs e)
{
    base.OnStartup(e);

    if (!await CheckForUpdatesAsync())
        return;

    Logger.Log("INFO", "APP", "Application is starting.");

    // Initialisation normale de l'application...
}
```

Le `return` est important.

Lorsqu'une mise à jour est lancée, l'application ne doit **pas continuer son initialisation**, sinon des threads, fichiers ou ressources peuvent rester actifs et empêcher Velopack de remplacer le dossier :

```text
%LOCALAPPDATA%\SERAP.FL3App\current
```

Cela provoque typiquement :

```text
PermissionDenied
Access denied
Unable to start the update, because one or more running processes prevented it
```

---

# 4. Configuration GitHub côté application

Le repository de mise à jour est public.

La source Velopack doit donc être :

```csharp
var source = new Velopack.Sources.GithubSource(
    "https://github.com/Groupe-SERAP/FL3App_Update",
    accessToken: null,
    prerelease: false);
```

Ne pas utiliser :

```text
https://github.com/Groupe-SERAP/FL3App_Update.git
```

ni :

```text
https://github.com/Groupe-SERAP/FL3App_Update/releases
```

---

# 5. Générer un token GitHub pour publier les releases

Le token sert uniquement à **écrire sur GitHub** lors de la publication.

Il n'est pas utilisé par l'application cliente.

## 5.1 Création du token

Dans GitHub :

1. Ouvrir **Settings**
2. Ouvrir **Developer settings**
3. Ouvrir **Personal access tokens**
4. Ouvrir **Fine-grained tokens**
5. Cliquer sur **Generate new token**

Configuration recommandée :

```text
Token name:
FL3App Velopack Release

Resource owner:
Groupe-SERAP

Repository access:
Only select repositories

Repository:
FL3App_Update
```

Dans **Repository permissions** :

```text
Contents → Read and write
```

Aucune permission `Administration`, `Issues`, `Pull requests`, etc. n'est nécessaire pour ce workflow.

Après création, copier immédiatement le token.

**Ne jamais :**

- mettre le token dans le code C# ;
- mettre le token dans le `.csproj` ;
- commiter le token dans Git ;
- publier le token dans un README ;
- envoyer le token dans un ticket ou un chat.

---

# 6. Charger le token dans PowerShell

À faire au début d'une session PowerShell avant de publier une release.

Utiliser :

```powershell
$secureToken = Read-Host "Token GitHub" -AsSecureString
$env:VPK_TOKEN = [System.Net.NetworkCredential]::new("", $secureToken).Password
```

PowerShell demande alors le token sans l'afficher à l'écran.

Le token est disponible dans la session courante via :

```text
VPK_TOKEN
```

Fermer la fenêtre PowerShell supprime cette variable de cette session.

---

# 7. Procédure complète pour publier une nouvelle version

Exemple utilisé dans cette section :

```text
Version actuelle : 0.11.0
Nouvelle version : 0.11.1
```

---

## Étape 1 — Modifier la version du projet

Dans `FL3App.csproj` :

```xml
<Version>0.11.1</Version>
```

La version doit être cohérente avec la version Velopack qui sera utilisée plus tard.

---

## Étape 2 — Se placer dans le dossier projet

Exemple :

```powershell
cd C:\Projets_Serap\FL3_Software\FL3App\FL3App
```

---

## Étape 3 — Nettoyer l'ancien dossier `Publish`

Recommandé avant chaque publication :

```powershell
Remove-Item -Recurse -Force .\Publish -ErrorAction SilentlyContinue
```

Il n'est normalement **pas nécessaire de supprimer le dossier `Releases`** entre deux versions successives.

---

## Étape 4 — Publier l'application

```powershell
dotnet publish -c Release -r win-x64 --self-contained true -o .\Publish
```

La sortie doit se terminer par un build réussi.

Vérifier que l'exécutable existe :

```powershell
Test-Path .\Publish\FL3App.exe
```

Résultat attendu :

```text
True
```

---

## Étape 5 — Vérifier éventuellement la version compilée

```powershell
[Reflection.AssemblyName]::GetAssemblyName(
    (Resolve-Path ".\Publish\FL3App.dll")
).Version
```

Pour :

```xml
<Version>0.11.1</Version>
```

la version de l'assembly doit correspondre à `0.11.1.x`.

---

## Étape 6 — Créer le package Velopack

```powershell
vpk pack `
    --packId SERAP.FL3App `
    --packTitle "FL3App" `
    --packVersion 0.11.1 `
    --packDir .\Publish `
    --mainExe FL3App.exe `
    --icon .\Resources\Icon\serap_logo_vertical.ico
```

La valeur :

```text
--packVersion 0.11.1
```

doit correspondre à :

```xml
<Version>0.11.1</Version>
```

Après le packaging :

```powershell
Get-ChildItem .\Releases
```

On doit notamment retrouver :

```text
SERAP.FL3App-0.11.1-full.nupkg
SERAP.FL3App-win-Setup.exe
SERAP.FL3App-win-Portable.zip
releases.win.json
assets.win.json
RELEASES
```

Le message suivant est normal si l'application n'est pas encore signée numériquement :

```text
No signing parameters provided
```

---

## Étape 7 — Charger le token GitHub

Si ce n'est pas déjà fait dans la fenêtre PowerShell actuelle :

```powershell
$secureToken = Read-Host "Token GitHub" -AsSecureString
$env:VPK_TOKEN = [System.Net.NetworkCredential]::new("", $secureToken).Password
```

---

## Étape 8 — Publier la release GitHub

```powershell
vpk upload github `
    --repoUrl https://github.com/Groupe-SERAP/FL3App_Update `
    --outputDir .\Releases `
    --publish `
    --releaseName "FL3App V0.11.1" `
    --tag V0.11.1
```

Sortie attendue, approximativement :

```text
Preparing to upload ...
Creating draft release ...
Uploading asset 'SERAP.FL3App-0.11.1-full.nupkg' ...
Uploading asset 'SERAP.FL3App-win-Setup.exe' ...
Uploading asset 'SERAP.FL3App-win-Portable.zip' ...
Uploading releases.win.json
Uploading legacy RELEASES
Converting draft to full published release.
```

---

# 8. Vérification sur GitHub

Dans :

```text
https://github.com/Groupe-SERAP/FL3App_Update
```

ouvrir **Releases**.

La nouvelle release doit être :

```text
FL3App V0.11.1
Tag : v0.11.1
```

Elle doit être :

```text
Published
```

et non :

```text
Draft
```

Si l'application utilise :

```csharp
prerelease: false
```

la release ne doit pas être marquée **Pre-release**.

---

# 9. Tester la mise à jour

Pour tester correctement une mise à jour :

```text
Version installée sur le PC : 0.11.0
Version publiée sur GitHub   : 0.11.1
```

Ne pas installer directement le nouveau `Setup.exe`.

Lancer normalement la version `0.11.0` déjà installée :

```text
%LOCALAPPDATA%\SERAP.FL3App\current\FL3App.exe
```

Le comportement attendu est :

```text
FL3App 0.11.0
      ↓
CheckForUpdatesAsync()
      ↓
GitHub
      ↓
0.11.1 détectée
      ↓
message de mise à jour
      ↓
téléchargement
      ↓
fermeture FL3App
      ↓
Velopack applique 0.11.1
      ↓
redémarrage
      ↓
FL3App 0.11.1
```

Le log doit ensuite contenir :

```text
Velopack CurrentVersion=0.11.1
No update available.
```

---

# 10. Emplacement de l'application installée

Velopack installe FL3App dans :

```text
%LOCALAPPDATA%\SERAP.FL3App
```

L'exécutable actif se trouve dans :

```text
%LOCALAPPDATA%\SERAP.FL3App\current\FL3App.exe
```

Le dossier contenant les packages téléchargés se trouve dans :

```text
%LOCALAPPDATA%\SERAP.FL3App\packages
```

---

# 11. Logs Velopack

En cas de problème d'installation de mise à jour :

```powershell
notepad "$env:LOCALAPPDATA\velopack\velopack_SERAP.FL3App.log"
```

Ou pour afficher les dernières lignes :

```powershell
Get-Content "$env:LOCALAPPDATA\velopack\velopack_SERAP.FL3App.log" -Tail 150
```

---

# 12. Erreur : mise à jour téléchargée mais non appliquée

Symptôme :

```text
Update detected: 0.11.0 -> 0.11.1
Update downloaded: 0.11.1
```

mais au redémarrage :

```text
Velopack CurrentVersion=0.11.0
```

Dans le log Velopack, on peut voir :

```text
PermissionDenied
Accès refusé
Unable to start the update, because one or more running processes prevented it
```

Vérifier que le `OnStartup()` contient bien :

```csharp
if (await CheckForUpdatesAsync())
    return;
```

Sans ce `return`, l'application continue son initialisation après avoir déclenché la mise à jour et peut garder des ressources ou processus ouverts.

Vérifier également qu'aucun processus FL3App ne reste actif :

```powershell
Get-Process FL3App -ErrorAction SilentlyContinue |
    Select-Object Id, Path
```

Et :

```powershell
Get-CimInstance Win32_Process |
Where-Object {
    $_.ExecutablePath -like "*\SERAP.FL3App\current\*"
} |
Select-Object ProcessId, Name, ExecutablePath, CommandLine
```

---

# 13. Ne pas stocker de données persistantes dans `current`

Le dossier :

```text
%LOCALAPPDATA%\SERAP.FL3App\current
```

est remplacé pendant une mise à jour.

Ne pas y stocker de données utilisateur ou de fichiers devant survivre aux mises à jour.

Utiliser plutôt un dossier dédié, par exemple :

```text
%LOCALAPPDATA%\SERAP\FL3App
```

---

# 14. Avertissement Windows / SmartScreen

L'installateur peut afficher :

```text
SERAP.FL3App-win-Setup.exe n'est pas fréquemment téléchargé.
```

ou un avertissement SmartScreen.

Cela vient notamment du fait que les exécutables ne sont actuellement pas signés numériquement.

Le packaging peut également afficher :

```text
No signing parameters provided
```

Cela ne bloque pas le fonctionnement de Velopack, mais une signature de code est recommandée avant une distribution plus large.

---