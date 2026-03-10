<h1>Kontinuálne nasadenie pomocou nástroja Flux (lokálne do k8s v rámci Docker Desktop)</h1>
<p><book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link> patrí v súčasnosti medzi najpoužívanejšie nástroje na kontinuálne nasadenie. Funguje na princípe <em>Pull Deployment</em> a plne podporuje metódy <book-link href="https://www.gitops.tech/" class="hydrated">GitOps</book-link>. V skratke, GitOps sa skladá z troch hlavných komponentov:</p>
<ul>
<li>IaC (Infrastructure as Code) - popis infraštruktúry (<em>deploymentu</em>) je uložený v súboroch v git repozitári a slúži ako "jediný zdroj pravdy".</li>
<li>MRs (Merge requests) - zmeny infraštruktúry sa dejú v kóde pomocou merge requestov. Typicky sa tu nastavuje povinné code review.</li>
<li>CI/CD (Continuous Integration and Continuous Deployment) - automatická aktualizácia infraštruktúry je zabezpečovaná pomocou CI/CD.</li>
</ul>
<p><book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link> je založený na sade rozšírení <book-link href="https://kubernetes.io/docs/concepts/overview/kubernetes-api/" class="hydrated">kubernetes API</book-link>, tzv. <book-link href="https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/" class="hydrated"><em>Custom resources</em></book-link>, ktoré kontrolujú, ako sa zmeny v git repozitároch aplikujú do kubernetes klastra. Dva základné objekty, s ktorými budeme pracovať, sú:</p>
<ul>
<li><book-link href="https://fluxcd.io/flux/components/source/gitrepositories/" class="hydrated"><code>GitRepository</code></book-link> objekt - zrkadlí konfigurácie z daného git repozitára</li>
<li><book-link href="https://fluxcd.io/flux/components/kustomize/kustomization/" class="hydrated"><code>Kustomization</code></book-link> objekt - presne špecifikuje adresár v rámci repozitára a frekvenciu synchronizácie.</li>
</ul>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">warning</md-icon></book-hint-type> <code>Flux Kustomization</code> objekt a <code>Kustomize Kustomization</code> objekt sú dva rôzne objekty. V ďalšom texte sa budú miešať, preto sa budeme na ne odvolávať vždy presne.</p>
</blockquote><p>Na obrázku sú znázornené komponenty Flux.</p>
<p><book-img src="book/dojo/01.Web-Components/img/081-01-flux-gitops-toolkit.png?version=20260310045022" alt="Flux gitops" counter="1" class="hydrated"></book-img></p>
<p>Ukážeme si inštaláciu a konfiguráciu Flux na lokálnom kubernetes klastri, ktorý poskytuje aplikácia Docker Desktop.</p>
<p>Na kubernetes klastri v dátovom centre (Azure, AWS, Google, ...) Flux nainštaluje administrátor daného klastra.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> Na konci ďalšej kapitoly je obrázok, ktorý znázorňuje všetky komponenty tu spomínané a ich vzájomné prepojenie.</p>
</blockquote><h3>1. Príprava konfigurácie</h3>
<p>V tejto časti si najprv nakonfigurujeme všetky potrebné súbory a
uložíme do git repozitára, aby boli pripravené pre priebežné nasadenie do kubernetes klastra.</p>
<p>Vytvorte nový adresár <code>${WAC_ROOT}/ambulance-gitops/infrastructure/fluxcd</code> a v ňom vytvorte súbor <code>kustomization.yaml</code> s obsahom:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
- https://github.com/fluxcd/flux2//manifests/install?ref=v2.8.1</code-highlight><p>Týmto spôsobom sme vytvorili závislosť na konkrétnom vydaní systému <book-link href="https://fluxcd.io/" class="hydrated">Flux CD</book-link>. Do súboru
<code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/prepare/kustomization.yaml</code> pridajte referenciu na flux cd konfiguráciu:</p>
<code-highlight language="yaml" class="hydrated">...
resources:
- ../../../infrastructure/polyfea-controller
- ../../../infrastructure/fluxcd @_add_@</code-highlight><p>Vytvorte adresár <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops</code> a v ňom súbor <code>git-repository.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: gitops-repo
  namespace: wac-hospital
spec:
  interval: 1m0s
  ref:
    branch: main
  secretRef:
    name: repository-pat
  timeout: 1m0s
  url: https://github.com/&lt;your-account&gt;/ambulance-gitops @_important_@</code-highlight><p>Tento súbor definuje git repozitár a vetvu, ktoré bude Flux CD sledovať a zabezpečovať priebežné nasadenie podľa konfigurácie v tomto repozitári. Pre prístup bude používať <em>personal access token</em> (PAT), ktorý si vygenerujete neskôr.</p>
<p>V tom istom priečinku vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/prepare.kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: prepare
  namespace: wac-hospital
spec:
  wait: true
  interval: 120s
  path: clusters/localhost/prepare @_important_@
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo</code-highlight><p>Tento súbor definuje, že Flux CD bude sledovať zmeny v priečinku <code>clusters/localhost/prepare</code> a ak nastanú, tak ich aplikuje do kubernetes klastra.</p>
<p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/install.kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: install
  namespace: wac-hospital
spec:
  wait: true
  force: true 
  dependsOn:  @_important_@
  - name: prepare @_important_@
  interval: 120s
  path: clusters/localhost/install @_important_@
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo</code-highlight><p>Tento súbor definuje, že Flux CD bude sledovať zmeny v priečinku <code>clusters/localhost/install</code> a ak nastanú, tak ich aplikuje do kubernetes klastra. Všimnite si, že sme zároveň určili, že táto konfigurácia je závislá od nasadenia a pripravenosti konfigurácie <code>prepare</code>.</p>
<p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/cd.kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cd
  namespace: wac-hospital
spec:
  wait: true
  interval: 42s
  path: clusters/localhost @_important_@
  prune: true
  sourceRef:
    kind: GitRepository
    name: gitops-repo</code-highlight><p>Tento súbor definuje, že Flux CD bude sledovať konfiguráciu v priečinku <code>clusters/localhost/</code> a ak sa zmení, tak zmeny aplikuje do kubernetes klastra - tzn. ak budeme chcieť zmeniť detaily priebežnej integrácie, tak bude postačovať, ak ich zapíšeme do nášho repozitára.</p>
<p>Vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops/kustomization.yaml</code>, ktorý vyššie uvedené súbory zintegruje:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

labels:
- pairs:
    app.kubernetes.io/part-of: wac-hospital

namespace: wac-hospital

resources:
- prepare.kustomization.yaml
- cd.kustomization.yaml
- install.kustomization.yaml
- git-repository.yaml</code-highlight><p>V priečinku <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost</code> vytvorte súbor <code>kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

labels:
- pairs:
    app.kubernetes.io/name: wac-hospital
    app.kubernetes.io/instance: development
    app.kubernetes.io/version: "0.1"

resources:
  - gitops</code-highlight><p>Táto konfigurácia sa odkazuje na priečinok <code>gitops</code>, ktorý sme vytvorili v predchádzajúcom kroku. Znamená to, že konfigurácia klastra je riadená objektmi Flux CD, ktorý zabezpečuje priebežné nasadenie podľa konfigurácie v git repozitári na príslušných cestách.</p>
<p>Upravte súbor <code>${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

resources:
  - https://github.com/polyfea/manifests//controller #?ref=v1.0.0 @_remove_@
  - https://github.com/polyfea/manifests//md-shell # should be in the later step, we will replace it later @_remove_@
  - https://github.com/polyfea/manifests//gitops/fluxcd @_add_@

patches: @_remove_@
  - path: ./patches/service.yaml @_remove_@

configMapGenerator: @_add_@
- name: polyfea-shell-cfg @_add_@
  namespace: polyfea @_add_@
  options: @_add_@
    disableNameSuffixHash: true @_add_@
  literals: @_add_@
  - APP_TITLE=Nemocnica WAC @_add_@
  - SERVICE_TYPE=NodePort @_add_@
  - NODE_PORT=30331 @_add_@</code-highlight><p>Táto úprava prispôsobuje nasadenie Polyfea aplikácie pre  systém FluxCD, napríklad zabezpečí postupné nasadenie definícii objektov a následne nasadenie objektu typu <code>MicrofronendClass</code>.
Zároveň môžete zmazať súbor <code>${WAC_ROOT}/ambulance-gitops/infrastructure/polyfea/patches/service.yaml</code> ktorý sa už nepoužíva.</p>
<h3>2. Vytvorenie a konfigurácia Personal Access Token (PAT) pre FluxCD</h3>
<p>Ak je náš repozitár verejný, FluxCD môže získať informácie bez potreby konfigurovania prístupových práv. Avšak v praxi je bežnejšie, že konfigurácie sú špecifické a súkromné. Preto predpokladáme, že tento repozitár je súkromný a musíme nakonfigurovať prístupové údaje pre FluxCD.</p>
<p>Prejdite na stránku <book-link href="https://github.com/" class="hydrated">GitHub</book-link>, prihláste sa a následne prejdite na stránku <book-link href="https://github.com/settings/apps" class="hydrated">Developer Settings</book-link>. <em>Z hlavnej stránky sa tu môžete dostať stlačením ikony Vášho avatara, menu "Settings" a potom "Developer settings".</em> Na ľavej strane vyberte <em>Personal access tokens-&gt;Tokens (classic)</em> a stlačte tlačidlo <em>Generate new token</em>. Prihláste sa, zadajte meno tokenu, napr. <em>WAC-FluxCD token</em>, a <em>Expiration</em> nastavte  na <em>Custom defined</em> (dátum aspoň do konca semestra). V časti <em>Scopes</em> označte položku <em>repo</em>. Stlačte tlačidlo <em>Generate token</em> a <strong>NEZABUDNITE SI ULOŽIŤ DO SCHRÁNKY</strong> vygenerovaný PAT.</p>
<p><book-img src="book/dojo/01.Web-Components/img/081-01-GitHubPAT.png?version=20260310045022" alt="Vytvorenie Personal Access Token-u" counter="2" class="hydrated"></book-img></p>
<p>Následne musíme toto heslo uložiť do kubernetes klastra. V kontexte GitOps máme teraz niekoľko možností ako postupovať. Tu si ukážeme zatiaľ jednoduchší postup, a to, že heslá a iné dôverné informácie nebudú súčasťou nášho repozitára, ale ich budeme mať k dispozícii len lokálne. Tento postup sa ale skomplikuje v momente, kedy budeme musieť pracovať s viac ako len s jedným heslom alebo klastrom. V takom prípade je oveľa vhodnejšie použiť tzv. <book-link href="https://fluxcd.io/flux/guides/mozilla-sops/" class="hydrated"><em>SOPS - Service Operations</em></book-link>, kedy sú šifrované heslá a verejný kľúč uložené priamo v repozitári a súkromný kľúč, potrebný na odšifrovanie dôverných informácií je prístupný len v samotnom klastri, kde ho ručne nasadzuje jeho administrátor. Druhý spôsob si preto ukážeme až v inej kapitole.</p>
<p>Vytvorte priečinok <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets</code> a v ňom súbor <code>repository-pat.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: v1
kind: Secret
metadata:
  name: repository-pat
  namespace: wac-hospital

type: Opaque
stringData:
  username: &lt;your-acount&gt; # zvyčajne môže byť akékoľvek meno @_important_@
  password: &lt;your-personal-access-token&gt;  # vygenerovaný PAT @_important_@</code-highlight><p>V tom istom priečinku vytvorte súbor <code>kustomization.yaml</code>:</p>
<code-highlight language="yaml" class="hydrated">apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

labels:
- pairs:
    app.kubernetes.io/part-of: wac-hospital

namespace: wac-hospital

resources:
  - repository-pat.yaml</code-highlight><p>Nakoniec vytvorte súbor <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/secrets/.gitignore</code> s obsahom:</p>
<code-highlight language="plain" class="hydrated">*
!.gitignore
!kustomization.yaml</code-highlight><p>Posledným krokom sme zabezpečili to, aby bol súbor <code>kustomization.yaml</code> prítomný v git repozitári, ale súbor <code>repository-pat.yaml</code> nie. Týmto spôsobom sme zabezpečili, že heslo nebude súčasťou nášho git repozitára. Nevýhodou je, že v prípade ak dôjde k strate lokálnych údajov, musíme heslo vygenerovať znova.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">warning</md-icon></book-hint-type> Uloženie citlivých údajov v súbore na disku je tiež vystavené riziku, že sa k nim dostane neoprávnená osoba. Uistite sa, že k Vášmu počítaču a uvedenému priečinku má prístup iba oprávnená osoba, prípadne citlivé súbory po nasadení zmažte a vytvorte nanovo až v prípade potreby.</p>
</blockquote><h3>3. Testovanie konfigurácie a archivácia repozitára</h3>
<p>Otestujte správnosť Vašej konfigurácie príkazmi vykonanými z priečnka <code>${WAC_ROOT}/ambulance-gitops/</code>:</p>
<code-highlight language="ps" class="hydrated">kubectl kustomize clusters/localhost
kubectl kustomize clusters/localhost/prepare
kubectl kustomize clusters/localhost/install
kubectl kustomize clusters/localhost/secrets</code-highlight><p>Pokiaľ všetky príkazy prebehli bez chybového hlásenia, archivujte svoj repozitár:</p>
<code-highlight language="ps" class="hydrated">git add .
git commit -m 'fluxcd configuration'
git push</code-highlight><h3>4. Bootstraping FluxCD</h3>
<p>Aby sme mohli začať využívať služby <book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link>, musíme ich prvotne nasadiť do klastra manuálne. Po prvom nasadení dôjde k synchronizácii stavu medzi klastrom a našim repozitárom a pri ďalšej práci nám preto bude postačovať, keď zmeny konfigurácie uložíme do repozitára. Jednou z výhod tohto prístupu je aj to, že môžeme riadiť kto z vývojového tímu potrebuje mať prístup k jednotlivým nasadeniam/klastrom - v prípade vývoja formou DevOps predpokladáme, že to je väčšina vývojárov a zároveň môžeme riadiť, aké oprávnenia sú jednotlivým členom poskytnuté.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">info</md-icon></book-hint-type> V tomto cvičení nasadzujeme <book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link> formou referencie na zverejnené manifesty v repozitári <book-link href="https://github.com/fluxcd/flux2" class="hydrated">fluxcd/flux2</book-link>. Alernatívne spôsoby inštalácie sú opísané v <book-link href="https://fluxcd.io/flux/installation/" class="hydrated">dokumentácii Fluxu</book-link>.</p>
</blockquote><p>Nasadíme Flux operátor do nášho klastra. Uistite sa, že máte vybraný správny kontext - <code>kubectl config get-contexts</code>, predite do priečinku <code>${WAC_ROOT}/ambulance-gitops/</code> a vykonajte príkaz</p>
<code-highlight language="ps" class="hydrated">kubectl apply -k infrastructure/fluxcd</code-highlight><p>Týmto príkazom sme nasadili Flux do klastra. Skontrolujte, či bol Flux nasadený a či je všetko v poriadku, prípadne počkajte kým sú všetky pody v stave <code>Running</code>:</p>
<code-highlight language="ps" class="hydrated">kubectl get all -n flux-system</code-highlight><p>Do klastra potrebujeme umiestniť prístupové udáje k nášmu repozitáru. Stále v priečinku <code>${WAC_ROOT}/ambulance-gitops/</code> vykonajte príkazy:</p>
<code-highlight language="ps" class="hydrated">kubectl create namespace wac-hospital
kubectl apply -k clusters/localhost/secrets</code-highlight><p>Teraz nasadíme našu konfiguráciu pre klaster <code>localhost</code>. V priečinku <code>${WAC_ROOT}/ambulance-gitops/</code> vykonajte príkaz:</p>
<code-highlight language="ps" class="hydrated">kubectl apply -k clusters/localhost</code-highlight><p>Týmto príkazom sme do klastra priamo nasadili objekty z priečinku <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost/gitops</code>. Pomocou objektu <code>gitops-repo</code> typu <code>GitRepository</code> vytvorí Flux lokálnu kópiu určenej vetvy nášho repozitára. Následne, v neurčenom poradí, pomocou objektu <code>cd</code> typu <code>Kustomization.kustomize.toolkit.fluxcd.io</code> aplikuje v klastri konfiguráciu z priečinka <code>clusters/localhost</code>, čím zabezpečí obnovenie konfigurácie samotného predpisu priebežného nasadenia. Zároveň pomocou objektu <code>prepare</code> tiež typu  <code>Kustomization.kustomize.toolkit.fluxcd.io</code>, nainštaluje do klastra služby, ktoré tam naša aplikácia implicitne predpokladá. V tomto prípade to je služba <code>polyfea-controller</code> a samotný operátor <book-link href="https://fluxcd.io/" class="hydrated">Flux CD</book-link>, ktorý môžeme takto napríklad obnoviť na novšiu verziu.</p>
<p>Po aplikovaní a priravenosti konfigurácie pomocou objektu <code>prepare</code> sa začne aplikovať konfigurácia uvedená v objekte <code>install</code> typu <code>Kustomization.kustomize.toolkit.fluxcd.io</code>, ktorá nasadí vlastné služby a objekty nášho projektu.</p>
<p><book-link href="https://fluxcd.io/" class="hydrated">Flux CD</book-link> pravidelne kontroluje, či nedošlo k zmenám v repozitári alebo či stav klastra nie je odlišný od konfigurácie určenej niektorým z objektov typu <code>Kustomization</code>. Pri akejkoľvek zistenej zmene sa pokúsi dosiahnuť stav totožný so stavom predpísaným v konfigurácii. V prípade, že sa zmení konfigurácia v repozitári, Flux CD automaticky zmení konfiguráciu v klastri.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> Niekedy potrebujeme dočasne zmeniť stav objektov v klastri - napríklad pri analýze hláseného problému môžeme chcieť dočasne zmeniť úroveň logov generovaných našou mikroslužbou. Pokiaľ pridáte v klastri objektu anotáciu <code>kustomize.toolkit.fluxcd.io/reconcile: disabled</code>, tak sa stav objektu nezmení až do momentu, kedy túto anotáciu odstránite. Anotáciu môžete aplikovať napríklad príkazom:</p>
<code-highlight language="ps" class="hydrated">kubectl annotate deployment &lt;name&gt; kustomize.toolkit.fluxcd.io/reconcile=disabled</code-highlight><p>Nezabudnite túto anotáciu odstrániť po skončení analýzy problému.</p>
</blockquote><p>Priebežne overte stav Vášho nasadenia príkazom:</p>
<code-highlight language="ps" class="hydrated">kubectl get gitrepository -n wac-hospital
kubectl get kustomization -n wac-hospital</code-highlight><p>Výstup by mal vyzerať obdobne ako na nasledujúcom výpise:</p>
<code-highlight language="plain" class="hydrated">kubectl get gitrepository -n wac-hospital
NAME          URL                                          AGE    READY   STATUS
gitops-repo   https://github.com/milung/ambulance-gitops   119s   True    stored artifact for revision 'main@sha1:...'

kubectl get kustomization -n wac-hospital         
NAME      AGE   READY   STATUS
cd        16m   True    Applied revision: main@sha1...
install   16m   True    Applied revision: main@sha1:...
prepare   11m   True    Applied revision: main@sha1:...</code-highlight><p>Pokiaľ je stav <code>READY</code> na hodnote <code>True</code>, znamená to, že Flux CD úspešne
nasadil konfiguráciu do klastra.</p>
<blockquote><p><book-hint-type class="hydrated"><md-icon slot="icon" aria-hidden="true">build_circle</md-icon></book-hint-type> Ak je stav <code>READY</code> na hodnote <code>False</code>, skontrolujte 
položku <code>Status</code> vo výpise príkazu <code>kubectl describe kustomization &lt;name&gt; -n wac-hospital</code>,
a opravte prípadnú chybu. Aplikujte opravu komitom vašich zmien do repozitára. Pokiaľ sa chyba týka
objektu <code>gitops-repo</code>, vykonajte aj príkaz <code>kubectl apply -k clusters/localhost</code>, v opačnom prípade je komit do repozitára postačujúci.</p>
</blockquote><p>Nakoniec overte, či sú všetky nasadené objekty pripravené a či sú všetky pody v stave <code>Running</code>:</p>
<code-highlight language="ps" class="hydrated">kubectl get all -n wac-hospital</code-highlight><p>Výstup by mal byť podobný tomuto:</p>
<code-highlight language="plain" class="hydrated">NAME                                                        READY   STATUS    RESTARTS   AGE
pod/polyfea-controller-controller-manager-84b649fd6-jn7bv   2/2     Running   0          3m8s
pod/polyfea-md-shell-657c94985d-98zj8                       1/1     Running   0          2m45s
pod/wacdemo-ambulance-ufe-deployment-868d8ff49b-nxd86       1/1     Running   0          2m45s
pod/wacdemo-ambulance-ufe-deployment-868d8ff49b-zf5jq       1/1     Running   0          2m45s

NAME                                                            TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
service/polyfea-controller-controller-manager-metrics-service   ClusterIP   10.99.167.36    &lt;none&gt;        8443/TCP       3m8s
service/polyfea-controller-manager                              NodePort    10.96.174.220   &lt;none&gt;        80:30331/TCP   3m8s
service/polyfea-md-shell                                        ClusterIP   10.99.38.173    &lt;none&gt;        80/TCP         2m45s
service/wacdemo-ambulance-ufe                                   ClusterIP   10.96.68.53     &lt;none&gt;        80/TCP         2m45s

NAME                                                    READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/polyfea-controller-controller-manager   1/1     1            1           3m8s
deployment.apps/polyfea-md-shell                        1/1     1            1           2m45s
deployment.apps/wacdemo-ambulance-ufe-deployment        2/2     2            2           2m45s

NAME                                                              DESIRED   CURRENT   READY   AGE
replicaset.apps/polyfea-controller-controller-manager-84b649fd6   1         1         1       3m8s
replicaset.apps/polyfea-md-shell-657c94985d                       1         1         1       2m45s
replicaset.apps/wacdemo-ambulance-ufe-deployment-868d8ff49b       2         2         2       2m45s</code-highlight><p>Pre každý z našich dvoch komponentov (ambulance-ufe a polyfea-controller) boli vytvorené nasledovné objekty:</p>
<ul>
<li>1x service</li>
<li>1x deployment</li>
<li>1x replicaset (vytvorený automaticky pre každý deployment)</li>
<li>2x pody (v deploymente sme mali uvedené 2 repliky)</li>
</ul>
<p>Zadajte do prehliadača adresu <book-link href="http://localhost:30331/fea/" class="hydrated">http://localhost:30331/fea/</book-link>.
Mali by ste vidieť stránku s aplikačnou obálkou s integrovanou mikro aplikáciou. Po stlačení na odkaz <em>Zoznam čakajúcich</em> by ste mali vidieť nasledujúci výstup:</p>
<p><book-img src="book/dojo/01.Web-Components/img/060-02-appshell-list.png?version=20260310045022" alt="Integrovaný zoznam čakajúcich" counter="3" class="hydrated"></book-img></p>
<h3>5. Overenie funkcionality priebežného nasadenia</h3>
<p>V princípe máme prvú verziu priebežného nasadenia hotovú. Ak sa teraz zmení niektorý z yaml manifestov v <code>ambulance-gitops</code> repozitári (musí byť ale referencovaný z <code>${WAC_ROOT}/ambulance-gitops/clusters/localhost</code> adresára), tak <book-link href="https://fluxcd.io/" class="hydrated">Flux</book-link> zabezpečí, že sa zmeny automaticky prejavia aj v klastri. Vyskúšame si to.</p>
<p>V súbore <code>${WAC_ROOT}/ambulance-gitops/apps/&lt;pfx&gt;-ambulance-ufe/deployment.yaml</code> zmeňte počet replík na 1, uložte zmeny a archivujte ich (<em>commit</em> a <em>push</em>) do vzdialeného repozitára.</p>
<p>Po chvíli overte, že sa zmeny prejavili v klastri. Výstup nasledujúceho príkazu by mal ukazovať 1 pod s menom <em>ambulance-ufe-deployment</em>.</p>
<code-highlight language="ps" class="hydrated">kubectl get pods -n wac-hospital</code-highlight>
