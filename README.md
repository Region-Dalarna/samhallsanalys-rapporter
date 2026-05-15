# samhallsanalys-rapporter

Repository för Samhällsanalys statiska webbrapporter (HTML från RMarkdown/Quarto).
Renderade rapporter i `public/` deployas automatiskt till Shiny-servern och
serveras under `https://samhallsanalys.regiondalarna.se/<rapportnamn>/`.

## Struktur

```
samhallsanalys-rapporter/
├── .github/workflows/deploy.yml   # GitHub Actions — triggar deploy vid push till publicera
├── .githooks/pre-commit           # Stoppar commit om HTML är äldre än .qmd/.Rmd
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

## Konventioner

- Varje rapport bor i ett eget repo under `c:/gh/<rapportnamn>/` med en
  `<rapportnamn>.qmd` (eller `.Rmd`) som renderas till `<rapportnamn>.html`.
- Repo-namnet styr även mappnamnet under `public/` och därmed url:en på webben.
- HTML-filer ska renderas med `self-contained: true` så att de är fristående
  (ingen `_files/`-mapp med beroenden).
- Rapportnamn får inte krocka med namn på Shiny-appar (samma domän).
  Deploy-skriptet kontrollerar detta automatiskt och avbryter deployen med ett
  tydligt felmeddelande om en kollision upptäcks.

## Komma igång på en ny dator

Efter `git clone`, kör en gång:

```bash
git config core.hooksPath .githooks
```

Det aktiverar pre-commit-hooken (se nedan). Hooken kan kringgås med
`git commit --no-verify` när det behövs.

## Publicera med `webbrapport_publicera()`

Det rekommenderade sättet att publicera en rapport är via R-funktionen
`webbrapport_publicera()` (finns i `dalarnasUtilsR`). Den sköter hela kedjan:
kopierar HTML-filen till rätt plats, commit:ar och pushar till `master`, gör
en selektiv merge till `publicera` och triggar deployen.

```r
# Standardfall: c:/gh/laget_i_dalarna/laget_i_dalarna.html  →  public/laget_i_dalarna/index.html
webbrapport_publicera("laget_i_dalarna")

# Annan källfil i samma repo
webbrapport_publicera("laget_i_dalarna", "output/rapport_v2.html")
# → c:/gh/laget_i_dalarna/output/rapport_v2.html  →  public/rapport_v2/index.html
```

Funktionen:

- Kräver att HTML-filen ligger inom `c:/gh/<rapport_repo>/`. Filer utanför
  rapport-repot tillåts inte.
- Skapar målmappen under `public/` automatiskt om den inte finns.
- **Stoppar publiceringen om HTML-filen är äldre än motsvarande
  `.qmd`/`.Rmd`-källfil** (sätt
  `publicera_aven_om_html_fil_aldre_an_rmd_qmd_fil = TRUE` för att tvinga fram
  publicering ändå).
- Gör en *selektiv* merge — endast den enskilda rapporten flyttas över till
  `publicera`, så andra rapporter på live-sajten påverkas inte.

## Publicera manuellt (alternativ)

Om man av någon anledning vill göra det för hand:

1. Rendera rapporten lokalt (Quarto/RMarkdown).
2. Kopiera HTML-filen till `public/<rapportnamn>/index.html`.
3. Commita och pusha till `master`.
4. När det är dags att publicera: merge:a `master` → `publicera` och pusha.

```bash
git checkout publicera
git merge master
git push
git checkout master
```

GitHub Actions kör då `/usr/local/bin/rapport_deploy.sh public/` på servern,
som rsync:ar innehållet till `/srv/rapporter/` och sätter rätt ägarskap och
rättigheter (`rapport-deploy:www-data`, `2755`/`644`).

## Pre-commit-hooken

`.githooks/pre-commit` körs vid varje `git commit` (om den är aktiverad via
`git config core.hooksPath .githooks`) och **stoppar commiten** om en staged
HTML-fil i `public/<rapportnamn>/` är äldre än motsvarande
`<rapportnamn>.qmd` eller `<rapportnamn>.Rmd` i syster-repot
`c:/gh/<rapportnamn>/`.

Den är icke-interaktiv — inga y/N-prompts — så schemalagda jobb funkar utan
att hänga. Kringgås vid behov med `git commit --no-verify`.

Hittar hooken ingen källfil i syster-repot hoppar den över kontrollen för den
filen (men `webbrapport_publicera()` har en strängare kontroll på samma sak,
så stale HTML fångas ändå när man publicerar via R-funktionen).
