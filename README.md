# iac-packer-ansible-azure-VM

Demonstrace přístupu Infrastructure as Code (IaC) za využití technologií Packer a Ansible pro správu obrazů virtuálních počítačů v Azure.

## Infrastructure as Code

Infrastructure as Code (dále jen **IaC**) je přístup, při kterém řídíme infrastrukturu stejně jako např. zdrojový kód aplikace.
Řekněme, že si chci na počítač nainstalovat webserver. To lze samozřejmě udělat tak, že se na počítač přihlásím, odněkud si stáhnu aplikaci webserveru, tu nainstaluji, webserver pustím a popřípadně donastavím nějaké konfigurační soubory, aby webserver nejen správně běžel, ale i správně fungoval.

To vše mi možná nevadí udělat jednou za čas a pro jeden počítač, problém ale nastává ve chvíli, když to potřebuji udělat na padesáti počítačích najednou, nebo na nich několikrát týdně upravit konfiguraci.

Jeden ze způsobů jak tento problém vyřešit je využití IaC nástrojů. Takový nástroj je v podstatě *něco*, co pustím, čemu předhodím konfigurační soubor (běžně `json` nebo `yaml`), a ono se mi to postará o to, aby se konfigurace popsaná v konfiguráku promítla na cílové počítače tím, že to nastavení tam skutečně bude. Tzn. že pokud řeknu, že chci mít nainstalovaný webserver `nginx` v poslední verzi, tak skutečně bude nainstalovaný a skutečně tam bude poslední verze.

Kromě očividného ušetření manuální práce, má IaC další výhody

- konfigurace je rovnou zadokumentovaná, nespoléhame se na to, co má admin v hlavě
- změny jsou snadno replikovatelné (determiničnost)
- změny jsou automatizovatelné
- lepší kontrola nad změnami (princip GitOps)

## Ansible

> Pokud si chcete následující skripty vyzkoušet, potřebujete běžící linuxový stroj, na který máte přístup a který můžete nastavovat. Pokud takový nemáte, můžete si ho vytvořit v Azure dle [dokumentace](https://docs.microsoft.com/cs-cz/azure/virtual-machines/linux/quick-create-cli) kde skončete před instalací webserveru (tedy se jen připojte pro ověření, že vám ssh připojení funguje a počítač běží).

Ansible je právě jeden z nástrojů kterým můžeme řídít konfiguraci. Používá `yaml` formát a jeho použití si ukážeme na instalaci `nginx`u ze souboru `ansible_demo/playbook.yaml`.

Dva důležité konfigurační soubory jsou v našem případě `playbook.yaml` a `inventory.yaml`.

### Ansible Playbook

Ansible playbook je něco, co mi popisuje samotnou konfiguraci, tedy popisuje mi **co** se má udělat. To je za pomocí *modulů*, které jsou definované pomocí názvu modulu a pak jeho parametrů.

```yaml
- apt:  # název modulu
    name: nginx  # parametr `name` s hodnotou `nginx`
    state: latest  # parametr `state` s hodnotou `latest`
    update_cache: yes  # tady už je snad jasné o co jde
```

Ansiblu tedy řekneme, že chceme použít modul `apt` a chceme ho spustit s danými parametry. Modul je pak v podstatě python (pro windows powershell) kód, který je nakopírován na cílový stroj a spuštěn.

### Ansible Inventory

Inventář je v podstatě jen seznam strojů, které máme k dispozici a které chceme ovládat. Počítače můžeme sdružovat do skupin a kromě nad celým inventářem, tak můžeme vykonávat úlohy pouze nad určitými skupinami nebo jednotlivými stroji.

### Spuštění ansible

> Před spouštěním následujícíh příkazů v `ansible_demo/inventory.yaml` nahraďte `REMOTE_IP_ADDRESS` IP adresou počítače, ke kterému se budete připojovat a `REMOTE_USER` uživatelem, kterým jste schopni se připojit přes ssh.

Pro ověření, že jsme schopni se ansiblem připojit, můžeme ansiblu podsunout pouze jeden modul, který chceme vykonat (ad-hoc command). Běžně se používá modul `ping`.

```
ansible all -m ping -i ansible_demo/inventory.yaml
```

Kde `all` je jméno skupiny (nebo stroje) oproti kterému chci modul pustit. V našem případě bysme mohli `all` nahradit za `nginx_server`.

Pro jednoduchou instalaci nginxu použijeme playbook `ansible_demo/playbook.yaml`. Ten spustíme příkazem

```
ansible-playbook -i ansible_demo/inventory.yaml ansible_demo/playbook.yaml
```

### Ansible Windows vs. Linux

Jako spoustu jiných nástrojů, ansible funguje s linuxem bezproblémově. Pro kontrolu vzdáleného stroje se používá ssh připojení, které je v linuxovém světě standardem. Windows ovšem defaultně ssh nepodporuje. Takže musíme buďto na cílový stroj nainstalovat ssh server, nebo použít `winrm` protokol (windows remote management). Z podstaty věci to vyžaduje konfiguraci předtím, než můžu stroj konfigurovat. V cloudovém prostředí se to řešit různými inicializačními skripty, nicméně obzvláště v CI prostředí chci snížit věci, které se běží při spouštění instance na minimum. Další možnost je předpřipravit si obraz operačního systému, který mám nějakým způsobem už přizpůsoben. 

Některé další problémy, které vznikají s použitím Windows jsou:

- vykonání tasku pod jiným uživatelem (`runas`) nefunguje úplně spolehlivě
- uživatel použitý pro připojení musí být administrátor
- testování pomocí Molecule je výrazně složitější (nelze použít docker)
- windows defaultně nemá python, proto se musí používat jiné moduly napsané v powershellu -> dvojí role pro linux a windows nebo musím rozlišovat OS

```yaml
# ukázka playbooku pro oboje platformy
tasks:
  - name: Configure debian
    include_tasks: "configure_debian.yaml"
    when: ansible_os_family == 'Debian'
    runas: <some_user>

  - name: Configure windows
    include_tasks: "configure_windows.yaml"
    when: ansible_os_family == 'Windows'
```

## Packer

Jak již bylo zmíněno, jedním z řešení jak vyřešit problém s přednastavením windows komunikace a zároveň výrazně snížit čas, který CI pipeline stráví konfigurací, je vytvoření vlastního obrazu, ze kterého později vytvořím stroj pro build.

Packer je open-source nástroj, který nám umožńuje vyrábět identické image pro několik cloudových platform z jedné zdrojové šablony. Nejčastějším use-casem je vytvoření tzv. *zlatých obrazů* které jsou pak využitelné různými vývojářskými týmy napříč organizací v různých cloudových prostředích.

Workflow je poměrně jednoduché. V podstatě si Packer vytvoří instanci z předem definovaného obrazu (např. čistého Windows serveru), tu nakonfiguruje, udělá z ní obraz a následně ji smaže.

Packer má několik možností, jak si obraz OS přizpůsobit, v tomto případě jsou pro nás důležité skripty spuštěné na operačním systému a ansible playbooky, které je packer schopen na dočasné instanci spustit.

```json
    "provisioners": [
        {
          "type": "powershell",
          "script": "./scripts/ConfigureRemotingForAnsible.ps1"
        }, 
          {
            "type": "ansible",
            "playbook_file": "./playbooks/iis.yaml",  
            "user": "packer",
            "use_proxy": false,
            "extra_arguments": ["-e", "ansible_winrm_server_cert_validation=ignore"]
        },
        {
          "type": "powershell",
          "script" : "./scripts/sysprep.ps1"
        }]
```

Protože ja packer multiplatformní, používá se stejně pro obě platformy s tím rozdílem, že vytvoření windows image trvá několikrát déle. Vzhledem k tomu, že vytváříme různé image, musíme udržovat i různé image. Díky integraci ansible ovšem máme zaručený stejný zdroj konfigurace.

### Spuštění

Vytvoření image je velmi jednoduché a to pomocí

```
packer build packer_demo/nginx_image.json
```

## Využití zlatých obrazů v CI

Výhoda přednastaveného obrazu, ze kterého můžu rovnou spustit instanci pro build agenta je především ta, že nemusím konfiguraci dělat v CI pipelině a tím pádem prodlužovat čas buildu. Obzvláště u Windows, kde vše trvá mnohonásobně déle, je toto značná výhoda. Díky přepoužitelnosti také ušetřím úsilí strávené udržováním různých *příchutí* operačních systémů. Zároveň snadno můžu tento proces automatizovat a tím pádem se nemusím starat o aktualizace apod. protože to za mě může udělat pipeline.

### Využití pro Azure DevOps (CI)

Vytvořený image nemusím použít pouze pro vytvoření serveru. Image se dá použít i jako zdroj pro Azure Scale Set. Scale Sety slouží především ke škálování aplikací na více instancí. Nicméně se dají použít i jako Agent Pooly v Azure DevOps. Díky tomu místo toho abych měl vytvořeno např. 20 počítačů, které budou k dispozici jako build agenti, můžu použít Scale Set, který mi vytvoří/zničí agenty podle potřeby pipeline. Workflow je tedy takové, že si za pomocí pipeline jsem schopen vytvářet image, ty použiji jako zdroj pro Azure Scale Set, který pak mám nastaven jako Agent Pool pro projektové pipelines. Díky tomu jsem schopen propagovat změny ze svého IaC repozitáře až do reálného build prostředí s minimálním úsilím na údržbu.

## Závěr

Závěr toho všeho je nejspíš asi ten - nevymýšlet kolo, problém se snažit roztrhat na modulární celky a na ty použít existující a pokud možno standardní nástroje.
