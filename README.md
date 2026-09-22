# FL3App — Génération et publication d'une mise à jour

Pour une nouvelle version `0.11.1` :

### 1. Modifier le `.csproj`

```xml
<Version>0.11.1</Version>
```

### 2. Publier

```powershell
Remove-Item -Recurse -Force .\Publish -ErrorAction SilentlyContinue

dotnet publish -c Release -r win-x64 --self-contained true -o .\Publish
```

### 3. Packager

```powershell
vpk pack `
    --packId SERAP.FL3App `
    --packVersion 0.11.1 `
    --packDir .\Publish `
    --mainExe FL3App.exe
```

### 4. Charger le token

```powershell
$secureToken = Read-Host "Token GitHub" -AsSecureString
$env:VPK_TOKEN = [System.Net.NetworkCredential]::new("", $secureToken).Password
```

### 5. Publier sur GitHub

```powershell
vpk upload github `
    --repoUrl https://github.com/Groupe-SERAP/FL3App_Update `
    --outputDir .\Releases `
    --publish `
    --releaseName "FL3App V0.11.1" `
    --tag v0.11.1
```

### 6. Tester

Lancer l'ancienne version installée.

Ne pas installer manuellement le nouveau Setup pour tester l'auto-update.

---

# 16. Checklist avant publication

- [ ] La version du `.csproj` a été mise à jour
- [ ] Le numéro `--packVersion` est identique
- [ ] Le dossier `Publish` a été régénéré
- [ ] `FL3App.exe` existe dans `Publish`
- [ ] `vpk pack` s'est terminé sans erreur
- [ ] Le token GitHub est chargé dans `VPK_TOKEN`
- [ ] La release GitHub est publiée et non en Draft
- [ ] Le tag correspond à la version
- [ ] `releases.win.json` est présent dans la release
- [ ] Le package `*-full.nupkg` est présent
- [ ] L'ancienne version installée détecte la nouvelle version
- [ ] Le log final indique la nouvelle `CurrentVersion`

