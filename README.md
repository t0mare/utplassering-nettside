# Fra nettside til egen deploy-pipeline

Du skal lage en nettside og sette opp en pipeline som publiserer den automatisk hver gang du pusher til GitHub. Først til GitHub Pages, så til din egen nginx-server på Proxmox.

## Kom i gang

1. Trykk **Use this template → Create a new repository** øverst på denne siden.
2. Velg din egen konto som eier, gi repoet et navn (f.eks. `min-side`) og la det være **Public**.
3. Klon repoet til maskinen din og åpne det i VS Code.

```bash
git config --global user.name "Fornavn Etternavn"
git config --global user.email "epost@eksempel.no"
git clone https://github.com/<brukernavn>/<repo>.git
cd <repo>
code .
```

## Slik ser repoet ut

```text
.
├── README.md            <- denne filen, og loggen din nederst
├── site/                <- alt som publiseres
│   ├── index.html
│   └── style.css
└── .github/
    └── workflows/       <- her legger du workflow-filene
```

Alt som skal ut på nettet, ligger i `site/`. Begge workflowene publiserer bare den mappen.

## Oppgave 1: Lag nettsiden

- Lag innhold i `site/index.html` og `site/style.css`. Tema: deg selv, en hobby eller en tenkt bedrift.
- Bilder legger du i `site/bilder/`.
- Bruk **relative stier**: `style.css` og `bilder/logo.png`, ikke `/style.css`.
- Commit og push ofte, med meldinger som sier hva du endret.

```bash
git add .
git commit -m "Legger til forside med meny"
git push
```

**Ferdig når:** siden ligger på GitHub og ser riktig ut når du åpner `site/index.html` i nettleseren.

## Oppgave 2: Publiser til GitHub Pages

1. Gå til **Settings → Pages → Build and deployment** og sett **Source** til **GitHub Actions**.
2. Lag filen `.github/workflows/pages.yml` med innholdet under. Commit og push.
3. Følg kjøringen under fanen **Actions**. Adressen til siden vises på `deploy`-jobben.

```yaml
name: Publiser til GitHub Pages

on:
  push:
    branches: [main]
  workflow_dispatch:          # gir en "Run workflow"-knapp under Actions

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: pages
  cancel-in-progress: false

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: actions/upload-pages-artifact@v5
        with:
          path: site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - id: deployment
        uses: actions/deploy-pages@v4
```

**Ferdig når:** en endring du pusher, vises på `https://<brukernavn>.github.io/<repo>/` etter et par minutter.

Du skal kunne svare på:

- Når kjører workflowen?
- Hvor kjører jobbene, og hvem eier de maskinene?
- Hvorfor har `deploy` linjen `needs: build`?
- Hvor havner filene? (Tips: se etter dem i repoet, og se nederst på sammendragssiden for kjøringen.)

## Oppgave 3: Installer en runner på serveren din

En *self-hosted runner* er et program på din server som spør GitHub om det finnes jobber til den. Serveren tar kontakt med GitHub, ikke omvendt. Derfor trenger den ingen åpne porter inn fra internett.

Sjekk først at serveren når GitHub:

```bash
curl -sI https://github.com     # skal svare HTTP/2 200
```

Lag en egen bruker for runneren. Den skal **ikke** ha sudo og **ikke** være med i `docker`-gruppen.

```bash
sudo dnf update && sudo dnf install curl tar git rsync
sudo useradd -m -s /bin/bash runner
sudo -iu runner
mkdir actions-runner && cd actions-runner
```

Gå til **Settings → Actions → Runners → New self-hosted runner → Linux, x64** i repoet ditt. Kjør nedlastingskommandoene derfra som `runner`-brukeren. Kjør så `config.sh` med disse valgene (tokenet varer én time):

```bash
./config.sh --url https://github.com/<brukernavn>/<repo> \
            --token <TOKEN> \
            --name <navnet-ditt>-proxmox \
            --labels proxmox \
            --unattended
exit
```

Installer runneren som en tjeneste, slik at den starter av seg selv:

```bash
cd /home/runner/actions-runner
sudo ./svc.sh install runner
sudo ./svc.sh start
sudo ./svc.sh status
journalctl -u 'actions.runner.*' -f     # følg loggen (Ctrl+C for å avslutte)
```

**Ferdig når:** runneren står som **Idle** under Settings → Actions → Runners.

## Oppgave 4: Deploy til din egen nginx

nginx kjører i Docker med en mappe fra serveren montert inn. Finn ut:

- Hvilken mappe på serveren er montert inn i containeren? (`docker ps` og `docker inspect <container> --format '{{json .Mounts}}'`)
- Hvilken port svarer nginx på?
- Hvem eier mappen? Runner-brukeren må kunne skrive til den.

Lag `.github/workflows/deploy.yml` selv. Her er et skjelett:

```yaml
name: Deploy til egen server

on:
  # TODO: kjør ved push til main, og manuelt fra Actions-fanen
  # NB: aldri pull_request – se Sikkerhet under

env:
  WEBROOT:   # TODO: mappen på serveren som er montert inn i nginx-containeren
  URL:       # TODO: f.eks. http://localhost:8080/

jobs:
  deploy:
    runs-on:   # TODO: hvordan sender du jobben til DIN runner?
    steps:
      - uses: actions/checkout@v6
      # TODO: kopier innholdet i site/ til $WEBROOT
      #       tips: rsync -av --delete --chmod=D755,F644 site/ "$WEBROOT/"
      # TODO: sjekk at siden svarer, og la jobben feile hvis den ikke gjør det
      #       tips: curl -f
```

**Ferdig når:**

- en push oppdaterer siden på serveren uten at du kopierer noe manuelt
- jobben blir rød hvis siden ikke svarer (test det!)
- du kan forklare hvorfor samme push nå gir to kjøringer, og hva som avgjør hvor hver av dem kjører

## Oppgave 5: Ekstra

Velg selv, omtrent i stigende vanskelighetsgrad:

1. **Versjonsinfo:** skriv commit-ID og tidspunkt til `site/version.txt` i workflowen før kopieringen.
2. **Kvalitetssjekk:** valider HTML-en før deploy (f.eks. `npx htmlhint site`). Feil skal stoppe deploy.
3. **Staging og produksjon:** branchen `dev` deployer til en egen mappe og port, `main` til produksjon.
4. **Rollback:** finn ut hvordan du ruller tilbake til forrige versjon med git, og test det.
5. **Overvåking:** sett opp noe som varsler når siden er nede. Stopp containeren og se at varselet kommer.

## Sikkerhet

- Repoet er offentlig, og runneren kjører kode på serveren din. Hvem som helst kan forke repoet og lage en pull request. Workflows som bruker runneren skal derfor bare trigges av `push` og `workflow_dispatch`, **aldri** `pull_request`.
- Under **Settings → Actions → General → Fork pull request workflows**: velg at alle eksterne bidragsytere må godkjennes.
- Runner-brukeren skal ikke ha sudo og ikke være med i `docker`-gruppen. Tilgang til Docker er i praksis root.
- Aldri commit tokens, passord eller nøkler.

## Min logg

Fyll inn underveis: hva du satte opp, kommandoene du brukte, hva som gikk galt og hvordan du løste det. Du presenterer dette på slutten.

| Hva | Kommando eller fil | Hva skjedde / hva lærte jeg |
| --- | --- | --- |
|     |                    |                             |
