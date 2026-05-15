# samhallsanalys-rapporter

Repository för Samhällsanalys statiska webbrapporter (HTML från RMarkdown/Quarto).
Renderade rapporter i `public/` deployas automatiskt till Shiny-servern och
serveras under `https://samhallsanalys.regiondalarna.se/<rapportnamn>/`.

## Struktur

```
samhallsanalys-rapporter/
├── .github/workflows/deploy.yml   # GitHub Actions — triggar deploy vid push till publicera
├── .githooks/pre-commit           # Varnar för inaktuell HTML
├── .gitattributes                 # Markerar HTML/CSS/JS som binära
├── public/                        # Allt här deployas till /srv/rapporter/
│   ├── laget_i_dalarna/
│   │   ├── index.html
│   │   └── ...
│   └── annan_rapport/
│       └── ...
└── README.md
```

Allt som ligger i `public/` rsync:as till `/srv/rapporter/` på Shiny-servern och
blir åtkomligt på webben. Filer utanför `public/` (workflow-filer, README m.m.)
deployas inte.

## Branch-modell

Samma princip som våra Shiny-app-repon:

- **`master`** — arbetsbranch. Här gör man revideringar, lägger till nya rapporter
  och pushar löpande utan att deploy triggas.
- **`publicera`** — produktionsbranch. När `master` är redo att publiceras
  merge:as den in i `publicera`. Push till `publicera` triggar GitHub Actions
  som kör deploy till Shiny-servern.

## Komma igång på en ny dator

Efter `git clone`, kör en gång:

```bash
git config core.hooksPath .githooks
```

Det aktiverar pre-commit-hooken som varnar om HTML i `public/` ser inaktuell ut
(filens mtime > 7 dagar). Hooken kan kringgås med `git commit --no-verify` när
det behövs.

## Lägga till en ny rapport

1. Rendera rapporten lokalt (Quarto/RMarkdown) med
   `self-contained: true` så att HTML-filen är fristående (ingen `_files/`-mapp
   med beroenden).
2. Lägg den renderade filen som `public/<rapportnamn>/index.html`.
3. Commita och pusha till `master`.
4. När det är dags att publicera: merge:a `master` → `publicera` och pusha.

Rapportnamnet (mappnamnet under `public/`) blir url:en:
`https://samhallsanalys.regiondalarna.se/<rapportnamn>/`.

**Viktigt:** Rapportnamn får inte krocka med namn på Shiny-appar (samma
domän). Deploy-skriptet kontrollerar detta automatiskt och avbryter deployen
med ett tydligt felmeddelande om en kollision upptäcks.

## Publicera

```bash
git checkout publicera
git merge master
git push
git checkout master
```

GitHub Actions kör då `/usr/local/bin/rapport_deploy.sh public/` på servern,
som rsync:ar innehållet till `/srv/rapporter/` och sätter rätt ägarskap och
rättigheter (`rapport-deploy:www-data`, `2755`/`644`).
