Då jag började med projektet sent så kommer jag skriva ner mina noteringar här då jag stött på VÄLDIGT mycket utmaningar och felsökt själv. 

Det betyder att jag inte kommer kunna följa kursens studiematerial utan får själv skapa omvägar för att uppnå samma resultat.


# Start av lab uppgift
## Steg
1. Skapning av redhat konto
2. Start av graits openshift trial.
3. Installering av requirements.
- När jag började analysera Lab report(https://github.com/jonasbjork/lab-argocd/tree/main) ser jag att ett par kommandon används.
4. Clonar repot `git@github.com:example/demo-argocd.git` & Skapar yaml filerna i k8s mappen.
5. Efter installatonen så börjar jag prova pusha kommandon och det fungerar inte riktigt, tar reda på vad namespace mostvarar men möter även på strul.
- Ser att jag har ett project namn som heter "myUsername-dev" och antar det är den, och sätter oc till att använda den som namespace. 
6. Nu så hoppar projektet direkt till "skapa ArgoCD Applicationen" som hoppar över många fler steg än jag har möjlighet att komma åt.
7. Installera ArgoCD
8. Kunde ej installera ArgoCD det blev för böckigt och ansluter mig till handlerare på hur jag ska fortsätta...

### Requirements
#### OC
OC i sig säger mig inte mycket. När jag analyserar koden och projectet så kan jag anta att det har med Openshift att göra så jag googlar "command line oc" och installerar det efter instruktionerna. - https://docs.okd.io/4.18/cli_reference/openshift_cli/getting-started-cli.html#cli-installing-cli-windows_cli-developer-commands

#### Kubectl
Ser att kubectl även uttnytjas och installerar det. Jag googlade men fick inga bra svar så frågade gemini.
som ger mig följande `winget install -e --id Kubernetes.kubectl` att kopier in i Powershell.
- Restart av vscode så fungerar kommandot nu.      

#### Steg 6, installera ArgoCD
Detta är steget som tog mest tid då jag gick igenom en crash course på youtube, googla en massa och testa mig fram utan lycka.

När jag försöker skapa ett kubectl namespace så får jag en massa fel och verkar som openshit policies blockerar mig. Testar mig fram, och den säger jag inte har rättigheter, efter ett par timmar klurande så har jag kommit till slutsatsen att jag inte får skapa project eller namespaces med Trail licensen jag har. Så istället MÅSTE jag köra på den jag fick av redhat, "myUsername-dev".

Kör - oc projects och får upp 3 projekt och "myUsername-dev" som är det jag blivit tilldelat.

Väljer projektet
```bash
oc project myUsername-dev
```

Applyar alla filer till projectet
```bash
oc apply -f .
```
Kollar om podar körs
```bash
oc get pods
```
Får respons,

```
NAME                                READY   STATUS    RESTARTS   AGE
myUsername-nginx-demo-6749c96f7-gglgk   1/1     Running   0          40m
myUsername-nginx-demo-6749c96f7-pxgmp   1/1     Running   0          40m
```
OK!!! det funkar!!
Då jag inte kan skapa nytt namespace så måste allting leva i projektet.
Jag kör installations kommandot i mitt project.
```bash
oc apply -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
Nästan allt installerades men vissa saker som saknar permissions som inte gjordes.
Vi skiter i det och kör på.
Vi provar exposa servern så vi kan nå den: 
```bash
oc expose svc argocd-server
```
- "route.route.openshift.io/argocd-server exposed"

Nu hämtar vi routen för att få anslutning
```bash
oc get route

##### RESULT #####
argocd-server       argocd-server-syvrix-dev.apps.rm2.thpm.p1.openshiftapps.com              argocd-server       http                  None
syvrix-nginx-demo   myUsername-nginx-demo-syvrix-dev.apps.rm2.thpm.p1.openshiftapps.com          syvrix-nginx-demo   <all>                 None
```

Provar surfa till ArgoCD servern...
Får upp felmeddelande som säger;
"Application is not available
The application is currently not serving requests at this endpoint. It may not have been started or is still starting."

Nu får vi ta och fixa installations problemen som vi fick... (dumt att prova men live and you learn)

Vi går igenom podarna och kollar efter fel..
```bash
oc describe pod argocd-server-79fdfd7f5b-bl56p
```
Vi får fel meddelande "Error: secret "argocd-redis" not found"... efter lite sökning så visar det sig vara pga att installationen inte gick igenom som den gjorde pga policies i openshift så vi rensar allt och provar på nytt...
```bash
oc delete all -l app.kubernetes.io/part-of=argocd
```

Vi tar bort secrets om det skulle tillföra problem... (rensar allt så vi kan göra om det clean)
```bash
oc delete secret -l app.kubernetes.io/part-of=argocd
```

Efter flera timmars felsökande behöver vi skapa en YML fil just för att undan komma problemet..

```yaml
# Redis Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-redis
spec:
  replicas: 1
  selector:
    matchLabels:
      app: argocd-redis
  template:
    metadata:
      labels:
        app: argocd-redis
    spec:
      containers:
      - name: redis
        image: redis:6.2
        ports:
        - containerPort: 6379
---
apiVersion: v1
kind: Service
metadata:
  name: argocd-redis
spec:
  selector:
    app: argocd-redis
  ports:
    - port: 6379
---
# Argo CD server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: argocd-server
spec:
  replicas: 1
  selector:
    matchLabels:
      app: argocd-server
  template:
    metadata:
      labels:
        app: argocd-server
    spec:
      containers:
      - name: argocd-server
        image: quay.io/argoproj/argocd:v3.3.6
        ports:
        - containerPort: 8080
        env:
        - name: ARGOCD_REDIS
          value: "argocd-redis:6379"
        securityContext:
          runAsNonRoot: true
---
apiVersion: v1
kind: Service
metadata:
  name: argocd-server
spec:
  selector:
    app: argocd-server
  ports:
    - port: 8080
```

Vi skapar filen och sen applyar vi den.
```bash
oc apply -f argocd-sandbox.yaml
```

Den går igenom och nu kollar vi om podarna körs.
```
oc get pods -w
```

Vi ser att dom körs nu.
```result
argocd-redis-78cc5fc968-tnv62       1/1     Running            0             2m5s
argocd-server-b885cf9f8-vqf4n       0/1     CrashLoopBackOff   4 (34s ago)   2m5s
```

Vi väntar tills servern är 1/1 så vi kan prova exposa argocd-server...
Servern vill ej installeras...
problemet verkar vara länkat till root användaren, så vi provar att köra utan den och patchar servern.
```bash
oc patch deployment argocd-server -p '{"spec":{"template":{"spec":{"containers":[{"name":"argocd-server","securityContext":{"runAsNonRoot":true}}]}}}}'                                                     
deployment.apps/argocd-server patched (no change)
```

Vi tar bort pod servern
``` bash
oc delete pod argocd-server-b885cf9f8-vqf4n
```

vi troubleshootar den nya porden geonm att kolla "oc logs"
- Vi ser att vi saknar privilieges som kan bero på root.
Vi provar att patcha igen.
```shell
oc patch deployment argocd-server `
>> -p '{ "spec": { "template": { "spec": { "containers": [ { "name": "argocd-server", "command": ["argocd-server"], "args": ["--insecure"], "securityContext": { "runAsNonRoot": true, "allowPrivilegeEscalation": false } } ] } } } }'
```
Result: deployment.apps/argocd-server patched
Vi restartar poden... genom att köra "oc delete pod xxxx"
och nu kollar vi resultatet igen  med "oc get pods -w" och nu så kör den som den ska
PS C:\Users\MarcusNilsson\OneDrive - Eurofinans AB\Dokument\Sortering\Code\Repositories\demo-argocd> oc get pods -w                              
```
NAME                                READY   STATUS    RESTARTS   AGE
argocd-redis-78cc5fc968-tnv62       1/1     Running   0          16m
argocd-server-56b75c84f6-r5drp      1/1     Running   0          8s
```
WOOOOO!!!

No exposar vi frontended. 
```bash
oc expose svc argocd-server
```

Hämta routes
```bash
oc get route
```

och vi får samma problem igen att applicationen inte svarar......
Efter lite klurande så provar vi att ta  bort routern då den har varit skapad nu i ett tag.
Skapar om den och patchar poden
```bash
oc create route edge argocd-server --service=argocd-server --port=8080
```
Får fortfarande samma problem meddelande från servern "Applcation is not available" 
- Kan vara local routing problem...
vi patchar det 
```bash
oc patch deployment argocd-server `
-p '{ "spec": { "template": { "spec": { "containers": [ { "name": "argocd-server", "command": ["argocd-server"], "args": ["--insecure", "--address", "0.0.0.0"], "securityContext": { "runAsNonRoot": true, "allowPrivilegeEscalation": false } } ] } } } }'
```

Vi tar bort poden så den restartar.
Fortfarande samma problem... 
Felsöker oc loggar och vi ser att min användare är "forbidden"
- Skapar service konto i return.
```
oc create serviceaccount argocd-server
```

Finns redan... Ger det rättigheter istället
```bash
oc adm policy add-role-to-user edit -z argocd-server
```

Säger till poden att använda sig av service kontot.
```bash
oc patch deployment argocd-server `
-p '{ "spec": { "template": { "spec": { "serviceAccountName": "argocd-server" } } } }'
```


Får det fortfarnade inte att funka och nu har den problem secreten...
Avslutar för dagen för det har varit för mycket klurande.

Kan ej få det funka så jag rensar allting och städar upp efter mig...

```bash
oc delete deployment argocd-server argocd-redis -n syvrix-dev
oc delete secret argocd-secret -n syvrix-dev     
```

Force deletar pods som verkar ligga kvar
```bash
oc delete pod argocd-redis-78cc5fc968-xfn5t  --grace-period=0 --force -n syvrix-dev
```