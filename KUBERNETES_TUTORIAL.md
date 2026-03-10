<h1>Nasadenie do klastra kubernetes</h1>
<p>V predchádzajúcom príklade sme si predstavili jeden zo spôsobov nasadenia našej aplikácie, pri ktorom sme využili služby typu <book-link href="https://azure.microsoft.com/en-in/resources/cloud-computing-dictionary/what-is-paas/" class="hydrated"><em>Platform as a Service</em></book-link> v dátovom centre <book-link href="https://azure.microsoft.com/" class="hydrated">Microsoft Azure</book-link>.
Hoci je tento spôsob nasadenia aplikácie funkčný, je vhodný len pre veľmi malé projekty. Na projektoch, kde spolupracuje niekoľko tímov a kde finálne dielo pozostáva z rôznych subsystémov, je tento spôsob nedostatočný.</p>
<p>V tomto cvičení si ukážeme, ako nasadiť našu aplikáciu do klastra <book-link href="https://kubernetes.io/" class="hydrated">Kubernetes</book-link> manuálne a v ďalšej časti, ako zabezpečiť kontinuálne nasadenie aplikácie technikou <book-link href="https://www.gitops.tech/" class="hydrated">GitOps</book-link> s využitím aplikácie <book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link>.</p>
<p>Pred samotnou prácou sa uistite, že máte zapnutú podporu Kubernetes v rámci <code>Docker Desktop</code> aplikácie, viď obrázok:</p>
<p><book-img src="book/dojo/01.Web-Components/img/060-01-docker-kubernetes.png?version=20260310045022" alt="Kubernetes v Docker for desktop" counter="1" class="hydrated"></book-img></p>
<h3>1. Vytvorenie adresárovej štruktúry pre GitOps nasadenie aplikácie</h3>
<p>V priečinku <code>${WAC_ROOT}</code> vytvorte nový priečinok s názvom <code>ambulance-gitops</code>. Vo VS Code pridajte tento priečinok do aktuálneho pracovného priestoru pomocou voľby menu <em>File-&gt;Add Folder to Workspace...</em></p>
<p>V priečinku <code>${WAC_ROOT}/ambulance-gitops</code> vytvorte nasledujúcu adresárovú štruktúru:</p>
<code-highlight language="plain" class="hydrated">ambulance-gitops
|
|- apps
|  L &lt;pfx&gt;-ambulance-ufe
|
|- infrastructure
|  L polyfea
|
|- components
|- clusters
    L localhost
        </code-highlight><p>Aplikáciu budeme nasadzovať s využitím aplikácie <book-link href="https://kustomize.io/" class="hydrated">kustomize</book-link> pre  riadenie konfigurácií v systéme kubernetes.  <book-link href="https://kustomize.io/" class="hydrated">Kustomize</book-link> je natívnou  súčasťou systému kubernetes a je tiež integrovaná do aplikácie <code>kubectl</code>. Často používanou  alternatívou k tejto aplikácii je aplikácia <book-link href="https://helm.sh/" class="hydrated">Helm</book-link>. Účel použitia je pri  týchto aplikáciách rovnaký, Helm sa zameriava na techniku šablón, ktoré sú parametrizované konfiguráciou pre špecifické nasadenie. <book-link href="https://kustomize.io/" class="hydrated">Kustomize</book-link> šablóny odmieta, keďže nie sú priamo  nasaditeľné alebo použiteľné a pracuje na princípe prispôsobovania funkčných manifestov pomocou  čiastkových úprav - <em>patch</em>-ov - pre špecifické nasadenie. V tomto cvičení preferujeme  <book-link href="https://kustomize.io/" class="hydrated">Kustomize</book-link> najmä z dôvodu, že manifesty si v základnej forme zachovávajú syntaktickú a  sémantickú presnosť definovanú <book-link href="https://kubernetes.io/docs/concepts/overview/kubernetes-api/" class="hydrated">Kubernetes API</book-link>.</p>
<p>Štruktúra priečinkov je hierarchická. Priečinok <code>clusters</code> bude obsahovať konfiguráciu systému  pre jednotlivé špecifické klastre, väčšinu času budeme počas cvičenia pracovať s lokálnym  klastrom označeným ako <code>localhost</code>.</p>
<p>Priečinok <code>infrastructure</code> bude obsahovať komponenty systému, ktoré sa buď inštalujú typicky len raz počas životného cyklu klastra alebo sú zdieľané medzi tímami a sú potrebné k behu  ostatných subsystémov. Čo presne je súčasťou priečinku <code>infrastructure</code> sa môže líšiť prípad od prípadu, v zásade tam patria zdieľané služby s inými aplikáciami, ktoré na niektorých klastroch už sú predpripravené a nie je možné ich "preinštalovať" v rámci daného projektu/tímu, a na  iných klastroch ich treba predinštalovať, aby bola zabezpečená funkčnosť dodávanej aplikácie.</p>
<p>Priečinok <code>apps</code> obsahuje komponenty systému, ktoré projektový tím zabezpečuje pre všetky  cieľové prostredia - klastre - a má ich plne pod kontrolou.</p>
<p>Priečinok <code>components</code> umožňuje zdieľať konfiguráciu medzi jednotlivými variantami nasadenia, napríklad pokiaľ by náš systém mal možnosť použiť rôzne systémy DBMS, mohli by sme príslušné konfigurácie a úpravy existujúcich komponentov umiestniť do tohto priečinka. V našom prípade budeme tento priečinok primárne používať na definíciu jednotlivých verzií nášho systému.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> V týchto cvičeniach používame takzvanú <book-link href="https://fluxcd.io/flux/guides/repository-structure/#monorepo" class="hydrated"><em>MonoRepo</em></book-link> štruktúru repozitára pre priebežné nasadenie. Nie je to ale jediná možnosť, viac možností je opísaných vo <book-link href="https://fluxcd.io/flux/guides/repository-structure/" class="hydrated">flux dokumentácii</book-link>.</p>
</blockquote><h3>2. Vytvorenie a nasadenie Deployment manifestu pre web komponent</h3>
<p>V priečinku <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe</code> vytvorte súbor <code>deployment.yaml</code> s nasledujúcim obsahom (komentáre môžete vynechať):</p>
<code-highlight language="yaml" class="hydrated">apiVersion: apps/v1
kind: Deployment
metadata:
  name: &lt;pfx&gt;-ambulance-ufe-deployment      # meno deployment-u od ktorého sa odvádza meno podu
spec:
  replicas: 2 @_important_@
  selector:
    matchLabels:
        pod: &lt;pfx&gt;-ambulance-ufe-label    # tento riadok je len označenie štítku 
                                    # key-value pair pod=ambualnce ufe
                                    # určený pre výber podov s rovnakým štítkom
  template:                    # šablóna pre vytvorenie replík podov
    metadata:
      labels: 
        pod: &lt;pfx&gt;-ambulance-ufe-label    # štítky podu
    spec:
      containers:
      - name: ambulance-ufe-container # meno kontaineru - v pod-e ich môže byť viac
        image: &lt;pfx-hub&gt;/ambulance-ufe
        imagePullPolicy: Always
        
        ports:
        - name: http
          containerPort: 8080       # port na ktorom počúva kontajner, pomenovaný ako `http`

        resources:                  # určenie kvót pre využívané fyzické prostriedky
                                    # je dôležité pre riadenie záťaže v rámci kubernetes
                                    # klastra
            requests:               
                memory: "32M"
                cpu: "0.1"
            limits:
                memory: "320M"
                cpu: "0.3"</code-highlight><p>Súbor <code>deployment.yaml</code> je deklaráciou nasadenia našej služby - tzv. <book-link href="https://kubernetes.io/docs/concepts/workloads/" class="hydrated"><em>workload</em></book-link> - do klastra. Všimnite si, že požaduje nasadenie dvoch replík - čo znamená, že v klastri budú vytvorené dva <em>pod</em>-y a v rámci každého z nich bude vytvorený aj bežiaci kontajner s našou aplikáciou. Všimnite si tiež, že v šablóne pre vytvorenie podu je definovaný štítok <code>pod: &lt;pfx&gt;-ambulance-ufe-label</code>, ktorý je použitý v selektore pre výber podov, ktoré majú byť súčasťou tohto nasadenia. Tento štítok je použitý aj v ďalších manifestoch, ktoré budeme vytvárať. Všetky objekty (<em>resources</em>) v systéme kubernetes môžu byť označené štítkami a tiež môžu byť vyhľadávané pomocou selektorov, ktoré štítky používajú. Prípony <code>-label</code>, <code>-deployment</code> a <code>-container</code> nie sú štandardnou konvenciou, sú tu uvedené najmä pre lepšie sa zorientovanie ako jednotlivé názvy v manifeste súvisia.</p>
<p>Takto vytvorený manifest je funkčný a možno ho nasadiť do klastra. Predpokladáme, že máte nainštalovaný systém <book-link href="https://kubernetes.io/" class="hydrated">kubernetes</book-link> na svojom počítači a zvolený <book-link href="https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/#define-clusters-users-and-contexts" class="hydrated">aktívny kontext</book-link> pre tento klaster. To, ktorý kontext je aktívny môžete zistiť vykonaním príkazu</p>
<code-highlight language="ps" class="hydrated">kubectl config get-contexts</code-highlight><p>ktorý zobrazí výsledky v podobe:</p>
<code-highlight language="plain" class="hydrated">CURRENT   NAME           CLUSTER           AUTHINFO 
*         aks            aks-cluster       user-admin
          docker-desktop docker-desktop    docker-desktop</code-highlight><p>kde znak <code>*</code> označuje práve aktívny kontext. Pokiaľ nie je aktívny vami požadovaný kontext, môžete ho zmeniť príkazom:</p>
<code-highlight language="ps" class="hydrated">kubectl config use-context docker-desktop</code-highlight><blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> Odporúčame nainštalovať si na svojom počítači aplikácie <book-link href="https://github.com/ahmetb/kubectx" class="hydrated">kubectx</book-link>, <book-link href="https://github.com/ahmetb/kubectx/blob/master/kubens" class="hydrated">kubens</book-link>, <book-link href="https://github.com/MuhammedKalkan/OpenLens" class="hydrated">OpenLens</book-link>, respektíve <book-link href="https://k9scli.io/" class="hydrated">k9s</book-link> a naučiť sa ich používať pri práci s kubernetes klastrami. V cvičeniach sú uvedené len príkazy založené na <book-link href="https://kubectl.docs.kubernetes.io/" class="hydrated">kubectl</book-link> nástroji, ktorý je súčasťou nainštalovaného <book-link href="https://kubernetes.io/" class="hydrated">kubernetes</book-link> systému.</p>
</blockquote><p>Teraz prejdite do priečinka <code>${WAC_ROOT}/ambulance-gitops</code> a vykonajte príkaz</p>
<code-highlight language="ps" class="hydrated">kubectl apply -f apps/&lt;pfx&gt;-ambulance-ufe/deployment.yaml</code-highlight><p>Následne môžete vykonať (opakovane) príkaz</p>
<code-highlight language="ps" class="hydrated">kubectl get pods</code-highlight><p>a zopakovať ho až pokiaľ neuvidíte výpis v podobe</p>
<code-highlight language="plain" class="hydrated">NAME                                        READY   STATUS    RESTARTS   AGE
ambulance-ufe-deployment-7f45494d6b-qj58z   1/1     Running   0          37s
ambulance-ufe-deployment-7f45494d6b-slppc   1/1     Running   0          61s</code-highlight><blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> Status môže mať dočasne aj výpis vo forme <code>Container Creating</code>. Pokiaľ stĺpec indikuje nejakú chybu, opravte manifest <code>deployment.yaml</code> a znova ho aplikujte príkazom <code>kubectl apply -f apps/&lt;pfx&gt;-ambulance-ufe/deployment.yaml</code>.</p>
</blockquote><p>Výpis indikuje, že v našom klastri máme teraz bežiace dva <book-link href="https://kubernetes.io/docs/concepts/workloads/pods" class="hydrated">pody</book-link> - zjednodušene dva virtuálne počítače pripojené k virtuálnej sieti kubernetes klastra.</p>
<h3>3. Presmerovanie portov pre prístup k podom v Kubernetes a odstránenie nasadených objektov</h3>
<p>Momentálne sú tieto pody dostupné len na virtuálnej sieti systému kubernetes. Pokiaľ k nim chceme pristúpiť, môžeme využiť funkciu <book-link href="https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/" class="hydrated">presmerovania portov</book-link> z lokálneho systému na cieľový pod (prípadne <em>service</em>) aktívneho klastra.</p>
<p>Vykonajte príkaz:</p>
<code-highlight language="ps" class="hydrated">kubectl port-forward &lt;pfx&gt;-ambulance-ufe-deployment-&lt;your-hash&gt; 8081:8080</code-highlight><p>a následne ak otvoríme v prehliadači stránku <book-link href="http://localhost:8081" class="hydrated">http://localhost:8081</book-link>, uvidíme náš zoznam čakajúcich.</p>
<p>Funkcia <code>port-forward</code> funguje aj pri vzdialených klastroch a je vhodná na ladenie podporných mikroslužieb, ktoré nie sú dostupné z externých sietí mimo klastra. V našom prípade bude zverejnená len služba integrujúca jednotlivé mikro-Front-End komponenty, čo je ukázané nižšie.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> Nástroj <book-link href="https://kubectl.docs.kubernetes.io/" class="hydrated">kubectl</book-link> funguje ako web klient, ktorý pristupuje k <book-link href="https://en.wikipedia.org/wiki/Representational_state_transfer" class="hydrated">REST API</book-link>/<book-link href="https://kubernetes.io/docs/concepts/overview/kubernetes-api/" class="hydrated">Kubernetes API</book-link> systému kubernetes. Je možné pristupovať k tomuto API aj priamo, museli by sme ale zabezpečiť správnu autenticitu a autorizáciu našich požiadaviek, ktorú nástroj <book-link href="https://kubectl.docs.kubernetes.io/" class="hydrated">kubectl</book-link> vykonáva automaticky na základe konfigurácie štandardne uloženej v adresári <code>~/.kube</code>.</p>
</blockquote><p>Nakoniec odstránime nami vytvorené objekty pomocou príkazu:</p>
<code-highlight language="ps" class="hydrated">kubectl delete -f apps/&lt;pfx&gt;-ambulance-ufe/deployment.yaml</code-highlight><h3>4. Príprava manifestov pre web komponent</h3>
<p>Teraz vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/service.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">kind: Service
apiVersion: v1
metadata:
  name: &lt;pfx&gt;-ambulance-ufe   #  určuje aj záznam DNS v rámci virtuálne siete klastra
spec:  
  selector:
    pod: &lt;pfx&gt;-ambulance-ufe-label     # zhoduje sa s metadata.labels šablóny pod-u
  ports:
  - name: http
    protocol: TCP
    port: 80            # port pomenovanej služby, môže sa líšiť 
                        # od portu na ktorom počúva kontajner
    targetPort: http    # viď meno portu v deployment.yaml</code-highlight><p>Tento objekt v systéme kubernetes deklaruje <book-link href="https://kubernetes.io/docs/concepts/services-networking/service/" class="hydrated">sieťovú službu</book-link>, ktorá zároveň implementuje jednoduchý <em>load balancer</em> pre distribúciu HTTP požiadaviek medzi dve repliky našej webovej služby a implementuje aj <book-link href="https://en.wikipedia.org/wiki/Service_discovery" class="hydrated"><em>service discovery</em></book-link> založený na systéme <book-link href="https://en.wikipedia.org/wiki/Domain_Name_System" class="hydrated">DNS</book-link>, to znamená, že meno služby <code>&lt;pfx&gt;-ambulance-ufe</code> zároveň reprezentuje DNS záznam v rámci virtuálnej siete a teda naša webová služba bude interne v rámci systému kubernetes dostupná na adrese <code>http://&lt;pfx&gt;-ambulance-ufe</code> v rámci toho istého <em>namespace</em> alebo na adrese <code>http://&lt;pfx&gt;-ambulance-ufe.&lt;namespace&gt;</code> z ľuboľného <book-link href="https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/" class="hydrated"><em>namespace</em></book-link>.</p>
<p>Ďalej vytvoríme súbor, ktorý do našej obálky vloží link na náš webcomponent, <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/webcomponent-link.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: &lt;pfx&gt;-ambulance-ufe-link
spec:
  microFrontend: polyfea-md-shell # referencia na microfrontend, z ktorého sa načíta element
  element: polyfea-md-app # element, ktorý bude vložený na stránku
  attributes: # atribúty elementu polyfea-md-app
    - name: headline
      value: Zoznam čakajúcich &lt;pfx&gt;
    - name: short-headline
      value: Zoznam čakajúcich &lt;pfx&gt; 
    - name: supporting-text
      value: |
        Spravuje zoznam pacientov čakajúcich na vyšetrenie v ambulancii
    - name: material-icon
      value: home_health
    - name: href # presmerovanie po kliknutí na element polyfea-md-app
      value: ./&lt;pfx&gt;-ambulance-wl 
  priority: 10
  displayRules: # pravidlá, kde sa má element vložiť v obálke
    - anyOf:
      - context-name: applications
      - context-name: rail-content
      - context-name: drawer-content</code-highlight><p>Tento súbor je neštandardným objektom - <em>Custom Resource</em> -  systému kubernetes. V ďalšom bode budeme vytvárať manifesty pre mikro-front-end <em>controller</em>, ktorý tieto objekty obhospodáruje. V zásade tu vykonávame registráciu mikro aplikácie - webového komponentu - do hlavnej aplikačnej obálky.</p>
<p>Všimnite si že používame referenciu na <code>microFrontend: polyfea-md-shell</code> a <code>element: polyfea-md-app</code>. Oba sú súčasťou riadiča, ktorý si popíšeme v bode 5.</p>
<p>Ďalej vytvoríme súbor, vďaka ktorému sa načíta obsah nášho webcomponentu, <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/webcomponent-content.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: polyfea.github.io/v1alpha1
kind: WebComponent
metadata:
  name: &lt;pfx&gt;-ambulance-ufe-content
spec:
  microFrontend: &lt;pfx&gt;-ambulance-ufe
  element: &lt;pfx&gt;-ambulance-wl-list
  displayRules:
    - allOf:
      - context-name: main-content
      - path: "^(\\.?/)?&lt;pfx&gt;-ambulance-wl(/.*)?$"</code-highlight><blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">warning</md-icon></book-hint-type> Meno elementu <code>&lt;pfx&gt;-ambulance-wl-list</code> musí zodpovedať komponentu, ktorý sme vytvorili predtým, pozri súbor <code>${WAC_ROOT}/ambulance-ufe/src/components/&lt;pfx&gt;-ambulance-wl-list/&lt;pfx&gt;-ambulance-wl-list.tsx</code></p>
</blockquote><p>Teraz nám však chýba referencia na microfrontend pre náš nový element. Vytvoríme súbor <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/microfrontend.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: polyfea.github.io/v1alpha1
kind: MicroFrontend
metadata:
  name: &lt;pfx&gt;-ambulance-ufe
spec:
  frontendClass: md-shell # default obsiahnutý v základe polyfea md-shell
  service: 
    name: &lt;pfx&gt;-ambulance-ufe # referencia na službu odkiaľ sa má načítať modul
  modulePath: build/ambulance-ufe.esm.js # cesta v rámci služby kde sa modul nachádza</code-highlight><p>Nakoniec vytvoríme súbor <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/kustomization.yaml</code>. Správa konfigurácií založená na systéme <book-link href="https://kustomize.io/" class="hydrated">Kustomize</book-link> využíva súbory <code>kustomization.yaml</code> ako integračný bod jednotlivých objektov deklarovaných v ostatných manifestoch. Pritom ostatné manifesty ostávajú stále použiteľné a možno ich priamo použiť ako argument aplikácie <code>kubectl</code>. Súbor <code>kustomization.yaml</code> je v zásade predpisom ako automatizovane <em>editovať</em> originálne súbory pre nasadenie jednotlivých objektov v iných prostrediach - rozumej v iných klastroch - a tieto úpravy možno hierarchicky kumulovať. V tomto prípade ide len o vymenovanie jednotlivých manifestov a priradenie spoločných značiek.</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- deployment.yaml
- service.yaml
- webcomponent-link.yaml
- webcomponent-content.yaml
- microfrontend.yaml

labels:
- pairs:
    app.kubernetes.io/component: &lt;pfx&gt;-ambulance-ufe</code-highlight><p>V tejto chvíli ešte nebudeme našu aplikáciu nasadzovať, potrebujeme najprv pripraviť manifesty pre infraštruktúru klastra. Pokiaľ ale chceme vidieť výsledok "kustomizacie" našej aplikácie, môžeme ho získať vykonaním príkazu</p>
<code-highlight language="yaml" class="hydrated">kubectl kustomize ./apps/&lt;pfx&gt;-ambulance-ufe/</code-highlight><h3>5. Príprava manifestov infraštruktúry pre nasadenie mikro-aplikácií</h3>
<p>V tomto kroku pripravíme manifesty pre objekty našej <em>infraštruktúry</em>. V našom prípade sa teraz jedná najmä o radič mikro-aplikácií, teda akúsi aplikačnú obálku pre zobrazovanie jednotlivých web-komponentov. Pri skúmaní súčasného stavu sme nenašli implementáciu <book-link href="https://martinfowler.com/articles/micro-frontends.html" class="hydrated">micro-front-end systému</book-link>, ktorý by vhodne kombinoval technológie <book-link href="https://developer.mozilla.org/en-US/docs/Web/Web_Components" class="hydrated">Web Components</book-link>, <book-link href="https://kubernetes.io/" class="hydrated">Kubernetes</book-link> a <book-link href="https://martinfowler.com/articles/micro-frontends.html" class="hydrated">micro-Front-Ends</book-link>. Táto kombinácia sa pritom javí najvhodnejšia vzhľadom na súčasné trendy vývoja, keďže vytvára priestor na samostatný vývoj jednotlivých web komponentov založený na všeobecne uznávaných štandardoch, a ktoré je možno potom nasadiť deklaratívnym spôsobom v rôznych situáciách, čo priamo zodpovedá filozofii mikro služieb. Existujúce implementácie sú napríklad <book-link href="https://bit.dev/" class="hydrated">bit</book-link>, alebo <book-link href="https://single-spa.js.org/" class="hydrated">single-spa</book-link>, ich integrácia ale väčšinou vyžaduje úzke previazanie mikro-front-end služieb.</p>
<p>Aby sme využili deklaratívne princípy Kubernetes API a nezávislosť tímov vyvíjajúcich jednotlivé mikro-aplikácie, bol pre potreby cvičenia vytvorený jednoduchý <book-link href="https://kubernetes.io/docs/concepts/architecture/controller/" class="hydrated"><em>kubernetes controller</em></book-link>, ktorý obsluhuje neštandardné objekty systému kubernetes definované pomocou <book-link href="https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/" class="hydrated"><em>Custom Resource Definition</em></book-link>. Tento <em>controller</em> nepretržite sleduje zmeny deklarovaných objektov - <em>webcomponents</em> - a poskytuje ich zabudovanému webovému serveru, ktorý implementuje aplikačnú obálku. Jednotlivé webové komponenty sú potom dynamicky načítavané podľa potreby a zobrazené na základe špecifikácie v týchto neštandardných objektoch. <em>Controller</em> je implementovaný jazyku Go v programovacom (back-end) a v jazyku Typescript (front-end) a jeho zdrojové súbory sú dostupné na <book-link href="https://github.com/polyfea" class="hydrated">https://github.com/polyfea</book-link>. Slúži zároveň ako motivácia a ukážka rozšíriteľnosti <book-link href="https://kubernetes.io/docs/concepts/overview/kubernetes-api/" class="hydrated">kubernetes api</book-link>, ktoré sa zameriava na deklaratívny popis želaného stavu, namiesto procedurálneho popisu, ako želaný stav dosiahnuť.</p>
<p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/kustomization.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/polyfea/manifests//controller #?ref=v1.0.0
  - https://github.com/polyfea/manifests//md-shell # should be in the later step, we will replace it later

patches:
  - path: ./patches/service.yaml
</code-highlight><p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/patches/service.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">kind: Service
apiVersion: v1
metadata:
  name: polyfea-controller
  namespace: polyfea
spec:
  ports:
  - name: webserver
    protocol: TCP
    port: 80
    targetPort: http
    nodePort: 30331
  type: NodePort</code-highlight><p>Táto úprava mení typ služby <code>polyfea</code>, ktorá je pôvodne špecifikovaná <book-link href="https://github.com/polyfea/manifests/blob/main/controller/service.yaml" class="hydrated">vo vzdialenej špecifickácii</book-link>. Pôvodná špecifikácia implicitne používa typ <code>ClusterIP</code>, ktorý sprístupní službu len na internej sieti kubernetes klastra. Upravená verzia používa typ <code>NodePort</code> a nastavuje parameter <code>nodePort: 30331</code>. To znamená, že k službe <code>polyfea</code> možno pristúpiť na porte <code>30331</code> hostiteľského počítača klastra. Tento <book-link href="https://kubectl.docs.kubernetes.io/references/kustomize/kustomization/patches/#patch-using-path-strategic-merge" class="hydrated">strategic-merge patch</book-link> súbor nie je úplnou deklaráciou <em>Service</em>, obsahuje len identifikátory pre vyhľadanie konkrétneho záznamu (<code>kind</code>, <code>apiVersion</code>, <code>name</code>) a položky, ktoré majú byť upravené (<code>ports</code>, <code>type</code>).</p>
<p>Ďalším typom <em>service</em>, ktorý by sme mohli použiť je typ <code>LoadBalancer</code>. Konfigurácia tohto typu je závislá od poskytovateľa klastra, v prípade <book-link href="https://www.docker.com/products/docker-desktop" class="hydrated">Docker Desktop</book-link> by bola služba dostupná na porte 80 nášho počítača. V tomto prípade ale možno použiť iba jednu službu typu <code>LoadBalancer</code> v rámci celého klastra. (V prípade klastrov v prostredí Azure alebo AWS, sa každej službe typu <code>LoadBalancer</code> priraďuje samostatná <em>verejná</em> IP adresa).</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> Pokiaľ používate minikube alebo iný lokálny klaster než <em>Docker Desktop</em>, tak možno budete nútený použiť prikaz <code>kubectl --namespace polyfea port-forward svc/polyfea-controller 30331:80</code> na presmerovanie portov predtým než otvoríte stránku <code>http:localhost:30331/fea</code>.</p>
</blockquote><p>Týmto spôsobom sme vytvorili konfiguráciu založenú na vopred pripravenej konfigurácii pre systém Polyfea, ktorým budeme spravovať naše microfrontend komponenty. Viac si o tomto riadiči viete prečítať tu: <book-link href="https://github.com/polyfea/polyfea/blob/main/README.md" class="hydrated">https://github.com/polyfea/polyfea/blob/main/README.md</book-link>.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> Takáto priama závislosť na externých manifestoch, bez špecifikácie príslušnej verzie, je v praxi nežiadúca, keďže riskujeme, že dôjde k zmenám, ktoré narušia funkčnosť našej aplikácie. V praxi by sa tieto manifesty buď skopírovali do lokálneho repozitára, alebo sa určila konkrétna verzia (vetva/tag v GitHub), ktorá bude potom použitá v súbore <code>kustomization.yaml</code>.</p>
</blockquote><h3>6. Nasadenie aplikácie do lokálneho Kubernetes klastra s využitím Kustomize</h3>
<p>V predchádzajúcich krokoch sme vytvorili deklarácie pre našu aplikáciu <code>ambulance-ufe</code> a pre infraštrukturálnu aplikáciu <code>polyfea</code>. Teraz prejdeme k deklarácii konfigurácie do špecifických prostredí - klastrov. Keďže budeme nasadzovať do nášho lokálneho kubernetes klastra, potrebujeme nasadiť obe aplikácie a vytvoriť subdoménu klastra - <book-link href="https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/" class="hydrated"><em>namespace</em></book-link> - do ktorých ich umiestnime. Nasadenie do klastra rozdelíme do dvoch krokov:</p>
<p>Krok <code>prepare</code> - vytvorenie <em>namespace</em> a nasadenie infraštruktúry klastra. Tu budeme nasadzovať služby, ktoré sú podmienkou na to, aby náš klaster bol pripravený na nasadenie samotnej aplikácie;</p>
<p>Krok <code>install</code> - nasadenie služieb samotnej aplikácie/aplikácií do klastra. Ak by sme mali klaster kompletne pripravený, tak stačí nasadiť tento krok.</p>
<p>Implicitne tiež predpokladáme, že medzi týmito krokmi existujú nejaké závislosti - napríklad objekty typu <code>webc</code> v kroku <code>install</code> predpokladajú existenciu rozšírení Kubernetes API typu <code>polyfea.github.io</code> z kroku <code>prepare</code>.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> Počet krokov sa snažíme limitovať, niekedy však môže byť potrebné rozdeliť nasadenie do viacerých krokov, aby boli splnené určité predpoklady pre nasadenie našej aplikácie - typicky dostupnosť služieb, ktoré definujú spôsob ako nasadiť iné objekty alebo objekty, ku ktorým chceme limitovať prístup pre lepšiu koordináciu jednotlivých tímov.</p>
</blockquote><p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml</code></p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- ../../../infrastructure/polyfea</code-highlight><p>Teraz nam zostáva pripraviť nasadenie aplikácie <code>ambulance-ufe</code> a <code>polyfea-md-shell</code> do klastra. Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/install/kustomization.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: wac-hospital  # chceme tieto objekty umiestniť do `wac-hospital` namespace
                         # tento zapis preťaží hodnoty "namespace" vo všetkých objektoch
labels:
- pairs:
    app.kubernetes.io/part-of: wac-hospital

resources:
- ../../../apps/&lt;pfx&gt;-ambulance-ufe
</code-highlight><p>Overte, že je vaša konfigurácia správna vykonaním príkazu v priečinku <code>${WAC_ROOT}/ambulance-gitops</code>:</p>
<code-highlight language="ps" class="hydrated">kubectl kustomize clusters/localhost/prepare</code-highlight><p>a následne príkaz</p>
<code-highlight language="ps" class="hydrated">kubectl kustomize clusters/localhost/install</code-highlight><p>Pokiaľ sú vaše konfigurácie správne, zobrazí sa výpis aplikovaných YAML dokumentov, v opačnom prípade musíte opraviť chybu podľa zobrazeného výpisu.</p>
<p>Pred tým ako zmeny nasadíme do klastra, vytvoríme si požadovaný namespace <code>wac-hospital</code>. Tento namespace by mohol byť aj súčasťou definícii objektov pre kustomize, ale pre jednoduchosť odstraňovania a opätovného nasadenia do klastra sme sa ho rozhodli vytvoriť manuálne. Vykonajte príkaz.</p>
<code-highlight language="ps" class="hydrated">  kubectl create namespace wac-hospital</code-highlight><p>Následne aplikujte prípravný krok príkazom v priečinku <code>${WAC_ROOT}/ambulance-gitops</code>:</p>
<code-highlight language="ps" class="hydrated">  kubectl apply -k clusters/localhost/prepare</code-highlight><blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> S najväčšou pravdepodobnosťou sa pri tomto kroku objavia chyby typu
<code>resource mapping not found for name: ...ensure CRDs are installed first</code> . Táto chyba je spôsobená tým, že vytvárame nové objekty typu <code>WebComponent</code> a <code>MicroFrontend</code>, ktoré sú neštandardné a ich definícia sa ešte nestihla aplikovať v klastri. Preto je potrebné príkaz
<code>kubectl apply -k clusters/localhost/prepare</code> zopakovať. V ďalšej časti upravíme nasadenia systému polyfea tak aby sa tento problém nevyskytoval.</p>
</blockquote><p>Je tiež možné, že pri tomto príkaze dostanete upozornenia typu: <code>Warning: resource ... is missing the kubectl.kubernetes.io/last-applied-configuration annotation ...</code>. Tieto upozornenia môžete ignorovať, <code>last-applied-configuration</code> bude implicitne vytvorená predchadzajúcim príkazom.</p>
<p>Následne aplikujte príkaz:</p>
<code-highlight language="ps" class="hydrated">kubectl apply -k clusters/localhost/install</code-highlight><p>a overte, že sa novo vytvorené pody dostanú do stavu <code>Running</code> príkazom:</p>
<code-highlight language="ps" class="hydrated">kubectl get pods --namespace wac-hospital</code-highlight><p>V prehliadači otvorte stránku <book-link href="http://localhost:30331/fea" class="hydrated">http://localhost:30331/fea</book-link>, na ktorej uvidíte aplikačnú obálku s integrovanou mikro aplikáciou. Po stlačení na odkaz <em>Zoznam čakajúcich</em> by ste mali vidieť nasledujúci výstup:</p>
<p><book-img src="book/dojo/01.Web-Components/img/060-02-appshell-list.png?version=20260310045022" alt="Integrovaný zoznam čakajúcich" counter="2" class="hydrated"></book-img></p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> Na niektorých systémoch nemusia byť <code>NodePort</code>-s prístupné na adrese <code>localhost</code>. Pozrite si, akým spôsobom sú prístupné na vašom systéme. Na systéme <book-link href="https://www.docker.com/products/docker-desktop" class="hydrated">Docker Desktop</book-link> je možné použiť adresu <code>host.docker.internal</code> alebo <code>localhost</code> s portom <code>30331</code>.</p>
</blockquote><p>Nasledujúci obrázok znázorňuje deployment a komunikačnú schému nasadenej aplikácie.</p>
<p><book-img src="book/dojo/01.Web-Components/img/060-03-k8s-ufe-komunikacia.png?version=20260310045022" alt="Komunikácia medzi micro-frontend radičom a nasadeným WebComponent" counter="3" class="hydrated"></book-img></p>
<h3>7. Odstránenie manuálne nasadených Kubernetes objektov</h3>
<p>Overili sme, že manuálne nasadenie našej aplikácie do Kubernetes klastra
funguje. Hoci sa môže naša štruktúra adresárov zdať komplikovaná pre potreby nasadenia jednoduchej webovej aplikácie, je dôležitým východiskom pre pridávanie ďaľších mikroslužieb v priebehu týchto cvičení.</p>
<p>V ďalšej časti si ukážeme, ako zabezpečiť kontinuálne nasadenie
s využitím aplikácie <book-link href="https://fluxcd.io/" class="hydrated">FluxCD</book-link></p>
<p>Predtým odstránime všetky manuálne nasadené objekty z klastra:</p>
<code-highlight language="ps" class="hydrated">kubectl delete -k ./clusters/localhost/install
kubectl delete -k ./clusters/localhost/prepare
kubectl delete namespace wac-hospital</code-highlight>
